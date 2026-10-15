[K_Portasiak_zadanie1.md](https://github.com/user-attachments/files/33508474/K_Portasiak_zadanie1.md)
# Zadanie 1 – PWR Racing Team Electronics

Program na mikrokontroler STM32G473CET3 z kwarcem 16 MHz. Projekt został zrobiony w STM32CubeIDE z użyciem biblioteki HAL.

Co robi program:

- miga diodą na pinie PA5 (5 razy na sekundę),
- liczy, ile sekund minęło od włączenia,
- co sekundę wysyła tę liczbę przez CAN (ID 0x738),
- ma watchdoga, który resetuje mikrokontroler, gdy program się zawiesi.

## Zegar

Zegar jest ustawiony na zewnętrzny kwarc 16 MHz (HSE) i PLL, dzięki czemu procesor działa na 170 MHz, czyli na maksymalnej prędkości:

16 MHz / 4 × 85 / 2 = 170 MHz

CAN dostaje zegar prosto z kwarcu (16 MHz), bo łatwo go podzielić na 1 Mbit/s.

## Miganie diodą

Miganie 5 Hz oznacza, że dioda zapala się i gaśnie 5 razy na sekundę. Jedno mrugnięcie trwa więc 200 ms, a dioda musi zmieniać stan co 100 ms.

Do odmierzania czasu służy timer TIM6. Timer liczy z częstotliwością 170 MHz, więc ustawienia są takie:

- Prescaler = 16999 → 170 MHz / 17000 = 10 kHz
- Period = 999 → 10 kHz / 1000 = 10 Hz, czyli przerwanie co 100 ms

W każdym przerwaniu program zmienia stan diody funkcją `HAL_GPIO_TogglePin`.

## Liczenie czasu

W tym samym przerwaniu program liczy, ile razy się wykonało. 10 przerwań po 100 ms to 1 sekunda, więc wtedy zmienna `g_uptime_s` zwiększa się o 1.

## CAN

Do komunikacji służy FDCAN1 w trybie zwykłego CAN (nie CAN FD), piny PA11 (RX) i PA12 (TX).

Prędkość 1 Mbit/s:

- zegar CAN = 16 MHz, Prescaler = 1
- jeden bit = 1 + 13 + 2 = 16 kwantów czasu
- 16 MHz / 16 = 1 Mbit/s

Ramka:

- ID: 0x738
- długość: 2 bajty
- bajt 0 = starsza część liczby sekund, bajt 1 = młodsza część

Przykład: po 300 sekundach (300 = 0x012C) ramka ma dane `01 2C`.

Na 2 bajtach zmieści się maksymalnie 65535 sekund (ok. 18 godzin). Potem licznik zaczyna od zera.

Przerwanie timera nie wysyła ramki samo, tylko ustawia flagę. Ramkę wysyła pętla główna, żeby przerwanie było krótkie.

Do podłączenia do prawdziwej magistrali CAN potrzebny jest jeszcze transceiver (np. SN65HVD230).

## Watchdog

Program używa watchdoga IWDG. Ma on własny zegar (LSI, ok. 32 kHz), więc działa nawet wtedy, gdy coś się stanie z głównym zegarem.

- Prescaler = 32 → 32 kHz / 32 = 1 kHz
- Reload = 499 → 500 / 1 kHz = ok. 0,5 s

Jeśli program przez 0,5 s nie odświeży watchdoga, mikrokontroler się zresetuje.

Watchdog jest odświeżany w pętli głównej, ale tylko wtedy, gdy przerwanie timera ustawiło flagę `g_timer_alive`. Dzięki temu reset nastąpi, gdy zawiesi się pętla główna albo gdy przestanie działać timer.

## Jak sprawdzić, czy działa

1. Dioda na PA5 powinna migać 5 razy na sekundę.
2. Analizator CAN ustawiony na 1 Mbit/s powinien co sekundę pokazywać ramkę 0x738 z rosnącą liczbą: `00 01`, `00 02`, `00 03`...
3. Watchdog: można na chwilę wpisać `while(1);` w pętli głównej. Po ok. 0,5 s mikrokontroler powinien się zresetować i liczenie zacznie się od nowa.
