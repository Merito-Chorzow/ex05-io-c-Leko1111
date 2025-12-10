# Sprawozdanie z zajęć — **C w IoT: analiza, obliczenia, I/O**

**Przedmiot:** Programowanie niskopoziomowe  
**Temat bloku:** Debugger → Obliczenia (PID q15) → I/O nieblokujące

---

## Informacje wstępne

- **Środowisko:** VS Code + GCC/Clang, GDB/LLDB, Makefile (`-O0 -g`, ostrzeżenia `-Wall -Wextra -Wpedantic`).
- **Repozytoria ćwiczeń:**
  - Ćw. 1 – Debug: `ex03_debugger_c`
  - Ćw. 2 – Obliczenia (PID q15): `ex04_compute_c`
  - Ćw. 3 – I/O (ring buffer + mini-shell): `ex05_io_c`
- **Wersje narzędzi:**
  - `gcc --version`: Arm GNU Toolchain 14.2.Rel1
  - `gdb --version` / `lldb --version`: GNU gdb 15.2.90
  - System:indows (uruchomienie przez QEMU)
---

## Ćwiczenie 1 — **Analiza kodu z użyciem debuggera**

### 1.1. Cel

> [!abstract] Cel ćwiczenia
> Zdiagnozować i poprawić błędy pamięci (m.in. *off-by-one* i zapis poza bufor) w kodzie C, wykorzystując **breakpointy**, **watchpointy**, podgląd pamięci oraz disassembly.

### 1.2. Materiały

> [!info] Pliki i uruchomienie
> * **Pliki źródłowe:** `src/bug.c`, `src/bug.h`, `src/main.c`.
> * **Kompilacja i start:**
> ```bash
> make clean && make
> ./build/app   # (W środowisku Windows: uruchomienie przez QEMU)
> ```

### 1.3. Procedura (skrót działań)

> [!todo] Kroki diagnostyczne
> 1. **Uruchomienie debuggera:**
>    Załadowanie programu do GDB, wyłączenie stronicowania i ustawienie pułapki na wejściu do podejrzanej funkcji.
>    ```gdb
>    gdb ./build/app
>    (gdb) set pagination off
>    (gdb) break sum_u8
>    (gdb) run
>    ```
> 2. **Konfiguracja Watchpointa:**
>    Namierzenie adresu bufora i ustawienie sprzętowego punktu obserwacji (watchpoint) na adresie tuż za jego końcem (`BUF+16`), aby wykryć nielegalny zapis.
>    ```gdb
>    (gdb) info address BUF
>    (gdb) watch *(BUF+16)
>    (gdb) continue
>    ```
> 3. **Analiza i naprawa:**
>    Zbadanie pętli w `sum_u8` (błąd *off-by-one*) oraz momentu zatrzymania przez watchpoint w `write_tail`. Poprawa kodu (walidacja indeksów, korekta warunków pętli) i ponowna weryfikacja.
### 1.4. Obserwacje

**Log z debuggera:**

> [!example] Fragment logu GDB
> ```text
> (gdb) break sum_u8
> Breakpoint 1 at 0xa6: file src/bug.c, line 6.
> (gdb) continue
> Continuing.
> 
> Breakpoint 1, sum_u8 (p=0x200006e0 <BUF> "", n=15) at src/bug.c:6
> 6           int s = 0;
> (gdb) info address BUF
> Symbol "BUF" is static storage at address 0x200006e0.
> (gdb) watch *(BUF+16)
> Hardware watchpoint 2: *(BUF+16)
> (gdb) continue
> Continuing.
> 
> Hardware watchpoint 2: *(BUF+16)
> 
> Old value = 0 '\000'
> New value = 170 'Ş'
> write_tail (idx=16, v=170 'Ş') at src/bug.c:16
> 16      }
> ```

**Opis błędu:**

> [!bug] Opis błędu i przyczyny Zdiagnozowano dwa krytyczne błędy pamięci:
> 
> 1. **Out-of-bounds write (Zapis poza zakres):** Watchpoint wykrył, że funkcja `write_tail` nadpisuje pamięć pod adresem `BUF+16`. Tablica ma rozmiar 16 (indeksy 0..15), więc operacja ta wykracza poza zaalokowany obszar. Przyczyną było błędne wywołanie funkcji z argumentem `16` w `main.c`.
>     
> 2. **Off-by-one:** Analiza kodu `sum_u8` wykazała błąd w warunku pętli `i <= n`. Powoduje to wykonanie o jednej iteracji za dużo i odczyt pamięci spoza tablicy.
>     

> [!check] Zastosowane poprawki
> 
> 3. W pliku `src/main.c` poprawiłem wywołanie na `write_tail(15, 0xAA)`, co odpowiada zapisowi do ostatniego elementu tablicy.
>     
> 4. W pliku `src/bug.c` zmieniłem warunek pętli na `i < n`, co jest poprawną konwencją dla tablic indeksowanych od zera.
>


### 1.5. Wnioski
> [!summary] Wnioski z debugowania
> 
> - **Co pomogło w diagnozie?** Kluczowe było użycie **Hardware Watchpoint** (`watch *(BUF+16)`). Pozwoliło to zatrzymać procesor dokładnie w instrukcji, która niszczyła pamięć. Bez tego musielibyśmy zgadywać, która funkcja wychodzi poza zakres.
>     
> - **Prewencja:** Aby uniknąć powrotu do tego błędu, należy unikać "magicznych liczb" (np. `16`) i stosować `sizeof(BUF)` lub zdefiniowane stałe `#define`. Dodatkowo, w funkcjach przyjmujących wskaźnik i rozmiar, warto stosować asercje (`assert`) sprawdzające granice.
>



Marker autogradingu: OK [A]


---

## Ćwiczenie 2 — Zaawansowana aplikacja obliczeniowa: regulator PID w q15

### 2.1. Cel

> [!abstract] Cel ćwiczenia
> Zaimplementować regulator PID w stałoprzecinkowym formacie q15 (1.15) z filtracją członu D oraz mechanizmem anti-windup dla całki, a następnie przetestować go na symulowanym obiekcie (roślinie) 1-rzędu.

### 2.2. Kontekst merytoryczny

> [!info] Kluczowe pojęcia
> * **Człony P/I/D:**
>     * **P (Proporcjonalny):** Odpowiada za szybkość reakcji na błąd. Im dalej od celu, tym mocniej steruje.
>     * **I (Całkujący):** Sumuje błąd w czasie, co pozwala zlikwidować uchyb stały i "dociągnąć" wynik do wartości zadanej.
>     * **D (Różniczkujący):** Reaguje na zmianę błędu (trend). Działa tłumiąco na gwałtowne zmiany, zapobiegając przeregulowaniom.
> 
> * **Format q15:**
>     Sposób zapisu ułamków na procesorach bez jednostki zmiennoprzecinkowej (FPU). Używamy `int16`, gdzie `32768` to `1.0`.
>     * **Przesunięcie `>>15`:** Konieczne po mnożeniu, aby przeskalować wynik z Q30 z powrotem do Q15.
>     * **Saturacja:** Zabezpieczenie przed "przekręceniem licznika" (overflow) przy dodawaniu dużych liczb.
> 
> * **Model rośliny:**
>     Symulowany obiekt inercyjny. Parametr **alfa** określa jego bezwładność – mała alfa to obiekt powolny, duża alfa to obiekt szybki.


### 2.3. Materiały

> [!example] Pliki i uruchomienie
> * **Moduły źródłowe:** `src/q15.h`, `src/pid.h/.c`, `src/plant.h/.c`, `src/main.c`.
> * **Kompilacja i uruchomienie:**
> ```bash
> make
> ./build/app # (W środowisku Windows: uruchomienie przez QEMU)
> ```




### 2.4. Procedura (kroki, bez rozwiązań)

> [!todo] Etapy realizacji
> 1. **Sanity check:** Uruchomienie pętli symulacyjnej (1000 kroków) z logowaniem co 50 kroków w celu weryfikacji środowiska.
> 2. **Interfejs i skalowanie:** Konwersja wszystkich wielkości do formatu q15, ustalenie limitów sterowania ($u_{min}, u_{max}$) oraz limitu całki (`i_limit`).
> 3. **Implementacja P:** Uruchomienie członu proporcjonalnego i test wpływu wzmocnienia $K_p$.
> 4. **Implementacja I + Anti-windup:** Dodanie akumulatora całki z mechanizmem *clamping* (ograniczenie wartości) i test zachowania przy nasyceniu wyjścia.
> 5. **Implementacja D + filtr:** Obliczenie różniczki błędu z zastosowaniem filtra dolnoprzepustowego (parametr `d_alpha`).
> 6. **Metryki:** Zdefiniowanie i obliczenie czasu narastania, przeregulowania oraz błędu ustalonego.
> 7. **Eksperymenty:** Przeprowadzenie testów dla minimum 3 zestawów nastaw PID oraz 2 wartości inercji obiektu ($\alpha$).
> 8. **Deterministyczność:** Weryfikacja powtarzalności wyników poprzez wielokrotne uruchomienie tej samej konfiguracji.

### 2.5. Wyniki i metryki (wypełnij)

Warunki początkowe: set = 0.5, y0 = 0.0, u_min = -1.0, u_max = 1.0, i_limit = 0.5, d_alpha = 0.85

| Konfiguracja |        Kp |       Ki |   Kd |    α (plant) |  Czas narastania |  Przeregulowanie [%] | Błąd ustalony | Uwagi                                                                       |
| ------------ | --------: | -------: | ---: | -----------: | ---------------: | -------------------: | ------------: | --------------------------------------------------------------------------- |
| #1           |       0.6 |     0.05 | 0.01 |         0.05 |               50 |                   0% |  mały (~0,01) | Stabilny powlny dolot                                                       |
| #2           | 4.0 (Max) |     0.05 | 0.01 |         0.05 |    ok. 150 - 200 | Brak (lub minimalne) |          mały | Układ nasycony (u=max), roślina zbyt wolna na oscylacje                     |
| #3           | 4.0 (Max) |     0.05 | 0.01 | 0.5 (szybka) |               70 |                   0% |     minimalny | Bardzo szybka reakcja, brak przeregulowania mimo dużego Kp                  |
| #4           |       0.6 | 0.8(Max) | 0.01 | 0.5 (Szybko) | < 50 (B. szybko) |                   0% |          brak | Zadziałał Anti-Windup! Mimo ogromnego Ki, limit całki zapobiegł oscylacjom. |




> [!example] Fragment logu GDB (Konfiguracja domyślna)
> ```text
> (gdb) break main.c:28
> Breakpoint 1 at 0x23e: file src/main.c, line 28.
> (gdb) continue
> 
> Breakpoint 1, main () at src/main.c:28
> 28                  printf("k=%4d set=%+.3f y=%+.3f u=%+.3f\n",
> (gdb) print k
> $1 = 0
> (gdb) print set
> $2 = 16384
> (gdb) print y
> $3 = 0
> (gdb) print u
> $4 = 13107
> 
> (gdb) continue
> Breakpoint 1, main () at src/main.c:28
> (gdb) print k
> $5 = 50
> (gdb) print y
> $6 = 14840
> (gdb) print u
> $7 = 15661
> ```



### 2.6. Wnioski (5–10 zdań)

> [!summary] Wnioski końcowe
> **1. Analiza parametrów:**
> Przeprowadzone eksperymenty wykazały, że zwiększenie wzmocnienia $K_p$ przyspiesza reakcję układu, jednak efekt ten jest limitowany przez nasycenie wyjścia sterującego ($u_{max}$). Kluczowym parametrem okazała się stała czasowa obiektu ($\alpha$) – przy małym $\alpha$ (duża inercja) układ jest bardzo odporny na oscylacje nawet przy dużych wzmocnieniach, natomiast przy dużym $\alpha$ reaguje niemal natychmiastowo.
>
> **2. Skuteczność Anti-Windup:**
> Mechanizm *clamping* zadziałał wzorowo. W eksperymencie z gigantycznym wzmocnieniem całkującym ($K_i = 0.8$), ograniczenie akumulatora całki (`i_limit`) zapobiegło zjawisku *integrator windup*. Dzięki temu układ doszedł do wartości zadanej bez przeregulowania, mimo że teoretycznie powinien wpaść w silne oscylacje.
>
> **3. Deterministyczność:**
> Wyniki symulacji były w 100% deterministyczne (powtarzalne). Uruchomienie programu wielokrotnie z tymi samymi nastawami zawsze generowało identyczne logi. Wynika to z faktu, że symulacja na QEMU w trybie *bare-metal* (bez systemu operacyjnego i losowych przerwań zewnętrznych) wykonuje instrukcje w sposób sekwencyjny i przewidywalny.


Marker autogradingu: OK [B].

---

## Ćwiczenie 3 — I/O nieblokujące: ring buffer + mini-shell

### 3.1. Cel

> [!abstract] Cel ćwiczenia
> Zaprojektować nieblokującą obsługę wejścia/wyjścia opartą o pierścieniowy bufor (RB) i prosty parser linii (mini-shell) z komendami `set`, `get`, `stat`, `echo` oraz licznikami `dropped` i `broken_lines`.

### 3.2. Materiały

> [!info] Wykorzystane zasoby
> * **Moduły:** `src/ringbuf.h/.c`, `src/shell.h/.c`, `src/main.c`.
> * **Uruchomienie:**
> ```bash
> make
> ./build/app # (W środowisku Windows: uruchomienie przez QEMU)
> ```


### 3.3. Procedura (kroki, bez rozwiązań)

> [!info] Opis procedury i polityki bufora
> **Wybrana polityka:** `drop-new` (odrzucanie nowych danych).
> 
> **Uzasadnienie (Konsekwencje):**
> W implementacji `ringbuf.c`, funkcja `rb_put` sprawdza ilość wolnego miejsca. Jeśli bufor jest pełny, inkrementuje licznik `dropped` i nie zapisuje nowego bajtu.
> * **Zaleta:** Zachowujemy spójność starych danych, które już są w buforze i czekają na przetworzenie. W protokołach tekstowych (jak nasz Shell) jest to bezpieczniejsze, bo zapobiega "szatkowaniu" komend, które są już w połowie wpisane.
> * **Wada:** Tracimy najnowsze komunikaty, co w systemach czasu rzeczywistego (np. sterowanie dronem) mogłoby być problemem (tam wolimy `drop-old`).
> 
> **Weryfikacja:**
> Został przeprowadzony test przepełnienia (burst) polegający na wysłaniu 200 komend (1200 bajtów) do bufora 128-bajtowego. Spodziewana liczba odrzuconych bajtów: $1200 - 128 \approx 1072$.


### 3.4. Wyniki (logi – wklej własne)

> [!example] Logi interakcji i weryfikacja przepełnienia
> **1. Symulacja interakcji:**
> ```text
> READY
> set=0.000 ticks=5 drop=0 broken=0
> OK set=0.420
> rx_free=128 tx_free=128 rx_count=0
> ECHO hello world
> ```
> 
> **2. Weryfikacja przepełnienia (Burst test w GDB):**
> Wysłano serię danych przekraczającą rozmiar bufora. Licznik `dropped` potwierdza odrzucenie nadmiarowych bajtów:
> ```text
> (gdb) break main.c:28
> Breakpoint 1 at 0x170: file src/main.c, line 28.
> (gdb) continue
> 
> Breakpoint 1, main () at src/main.c:28
> 28          return 0;
> (gdb) print sh.rx.dropped
> $1 = 1073
> ```

### 3.5. Wnioski (3–6 zdań)

> [!summary] Wnioski z implementacji I/O
> 1. **Ocena polityki:** Polityka `drop-new` jest właściwa dla interfejsu tekstowego (Shell). Nadpisywanie starych danych (`drop-old`) mogłoby uszkodzić początek bufora, w którym znajduje się aktualnie przetwarzana, niekompletna komenda, prowadząc do błędów parsowania.
> 
> 2. **Realne zastosowanie:** W rzeczywistym systemie (z UART), funkcja `rb_put` byłaby wywoływana w przerwaniu (ISR) przy odebraniu znaku, a `shell_tick` w pętli głównej. Bufor kołowy pozwala na rozdzielenie tych dwóch domen czasowych (producent-konsument) bez blokowania procesora.
> 
> 3. **Refaktoryzacja:** W obecnym API brakuje funkcji `rb_peek` (podgląd bez zdejmowania), która ułatwiłaby parsowanie bardziej złożonych protokołów bez konieczności kopiowania danych do lokalnych buforów.
>


Marker autogradingu: OK [C].

---

### Podsumowanie całych zajęć (2–4 akapity)

> [!quote] Podsumowanie kursu
> **1. Analiza błędów i Debugging (Ćw. 1):**
> Nauczyłem się, że w systemach wbudowanych błędy pamięci (jak *Stack Overflow* czy *Hard Fault*) często objawiają się w nieoczywisty sposób (np. nadpisanie zmiennych, blokada procesora). Kluczową umiejętnością było wykorzystanie GDB: analiza rejestrów (PC, LR, SP), stosowanie *Hardware Watchpoints* do łapania momentu nielegalnego zapisu oraz manualna nawigacja po pamięci, gdy standardowe wyjście (printf) nie działa.
> 
> **2. Algorytmy q15 i sterowanie (Ćw. 2):**
> Zrozumiałem specyfikę obliczeń stałoprzecinkowych – konieczność ręcznego skalowania, pilnowania przepełnień i stosowania saturacji. Projekt regulatora PID pokazał, jak ważna jest deterministyczność kodu oraz zabezpieczenia takie jak *Anti-Windup*. Symulacja wykazała, że dobór parametrów (wzmocnienia vs. inercja obiektu) jest kompromisem między szybkością a stabilnością.
> 
> **3. Nieblokujące I/O (Ćw. 3):**
> Implementacja *Ring Buffera* uświadomiła mi, jak zarządzać asynchronicznym przepływem danych bez używania funkcji blokujących (jak `scanf`). Kluczowym wnioskiem jest konieczność obsługi sytuacji brzegowych, takich jak przepełnienie bufora, i świadomy wybór strategii (odrzucanie nowych vs starych danych) w zależności od zastosowania systemu.


## Załączniki

* Fragmenty logów GDB/LLDB (Ćw. 1): *Patrz sekcja 1.4*
* Logi z przebiegów symulacji PID (Ćw. 2): *Patrz sekcja 2.5*
* Logi interakcji shell/RB (Ćw. 3): *Patrz sekcja 3.4*
---

