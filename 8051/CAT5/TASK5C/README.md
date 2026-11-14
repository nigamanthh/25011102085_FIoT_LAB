# EXPERIMENT DETAILS

**EXP:** CONTINUOUS ASSESSMENT 5

**DATE:** 10/08/2026

**NAME OF EXP:** Hardware Timer and Interrupt-Based Timing for LED Fading, Pulse Generation and Stepper Motor Control

**HARDWARE SETUP / PLATFORM:** AT89S52 development kit (8051 core, 11.0592 MHz crystal) + LED + DSO + ULN2003/28BYJ-48 — Keil µVision 5 (C51), progisp + USBasp ISP programmer, DSO

---

## TASK 3 : Stepper Motor Speed and Direction (optional)

### 1. Circuit Diagram

> **Note:** this diagram shows all CAT 5 tasks together. P1.1 drives the LED in Tasks 1 and 2 (LED, PWM, square waves) and is ULN2003 IN2 in Task 3 (stepper), so connect only one of them at a time.

Crystal 11.0592 MHz → machine cycle 1.085 µs → Timer 0 → overflow → **interrupt** → software time base → action

| Time base | Counts | Setting |
|---|---|---|
| 1 ms | 922 | Mode 1, TH0:TL0 = 0xFC66 (+12 for ISR entry → 0xFC72), reloaded in the ISR |
| 100 µs | 92 | Mode 2 auto-reload, TH0 = 0xA4 |

![Circuit Diagram](circuit.png)

*Drawn on [Wokwi](https://wokwi.com). To open and edit it: start a new project on wokwi.com, paste [`wokwi/diagram.json`](wokwi/diagram.json) into its diagram.json tab, and add the `wokwi/*.chip.json` files (if any) with **New file**. The AT89S52, USB-TTL, ULN2003, USBasp and ST-LINK boards are custom chips (pin names added as labels). The ESP-01 part stands in for the Wemos D1 mini and the BMP180 module for the BME280 (same AT firmware / same 4-pin I²C header); the Nucleo-64 graphic is labelled as the F401RE.*

### 2. Core Logic / Pseudocode / Code Base

**Working Principle:**

This task drives a 28BYJ-48 unipolar stepper motor through a ULN2003 Darlington transistor array interfaced to pins P1.0 through P1.3. A Timer 0 1 ms interrupt acts as the motor timing engine. Inside the ISR, a millisecond counter increments until it matches the target interval `step_ms`. Once reached, the commutation step index is incremented or decremented (guided by the `clockwise` flag) and the corresponding wave-drive pattern (1000 → 0100 → 0010 → 0001) is driven onto the stator coils via a `switch(phase)` block writing to P1 (preserving the upper nibble `P1 & 0xF0`). Keypad commands dynamically update these parameters: Key 1 configures slow rotation (100 ms per step), Key 2 sets medium speed (50 ms), Key 3 enables fast stepping (10 ms), Key 4 flips the direction of rotation, and Key 5 toggles start/stop. The pressed key is displayed on the common-cathode 7-segment display on Port 0 using `display_digit()`.

**Pseudocode:**

1. Initialize pins and display: energize phase 0 (`P1 = (P1 & 0xF0) | 0x08`), display digit '1' (`P0 = 0x06`), configure Timer 0 in Mode 1 for 1 ms periodic interrupts (`TMOD = (TMOD & 0xF0) | 0x01; TH0 = 0xFC; TL0 = 0x72;`), and enable interrupts (`ET0 = 1; EA = 1; TR0 = 1;`).
2. In `timer0_isr() interrupt 1 using 1`:
   a. Reload `TH0 = 0xFC; TL0 = 0x72;`.
   b. If not running (`!running`), return immediately.
   c. Increment `ms_count`; when it reaches `step_ms`, reset `ms_count = 0`.
   d. Advance `phase = (phase + 1) & 3` if `clockwise`, else `phase = (phase - 1) & 3`.
   e. Output coil pattern via `switch(phase)`: case 0: 0x08, case 1: 0x04, case 2: 0x02, case 3: 0x01 into lower nibble of P1.
3. In `main()`, poll `keypad_get()` in an infinite loop. When a valid key is pressed:
   a. Display the key digit on 7-segment Port 0 with `display_digit(key)`.
   b. Update state via `switch(key)`: case 1 -> `step_ms = 100`, case 2 -> `step_ms = 50`, case 3 -> `step_ms = 10`, case 4 -> `clockwise = !clockwise`, case 5 -> `running = !running`.

**Code Base:**

- Project files: `TASK5C.uvproj` (Keil µVision), `STARTUP.A51`, firmware `Objects/TASK5C.hex`.
- `main.c` — `timer0_isr() interrupt 1 using 1`, `keypad_get()`, `display_digit()`
- `delay.c/.h` — software delay for keypad debounce

### 3. Inference

Motor velocity and rotation direction are dictated cleanly by the interrupt period and phase sequencing. Because step timing is isolated within the timer ISR, keyboard responsiveness does not cause jerky motion or skipped motor steps.
