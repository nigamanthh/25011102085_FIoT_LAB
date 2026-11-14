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

This task drives a 28BYJ-48 unipolar stepper motor through a ULN2003 Darlington transistor array interfaced to pins P1.0 through P1.3. A Timer 0 1 ms interrupt acts as the motor timing engine. Inside the ISR, a millisecond counter increments until it matches the target interval `step_ms`. Once reached, the commutation step index is incremented or decremented (guided by the `clockwise` flag) and the corresponding wave-drive bit pattern (`STEP_SEQ[]`: 1000 → 0100 → 0010 → 0001) is driven onto the stator coils. Keypad commands dynamically update these parameters: Key 1 configures slow rotation (100 ms per step), Key 2 sets medium speed (50 ms), Key 3 enables fast stepping (10 ms), Key 4 flips the direction of rotation, and Key 5 toggles start/stop.

**Pseudocode:**

1. Configure Timer 0 in Mode 1 for 1 ms periodic interrupts.
2. In the ISR:
   a. If the motor is running (`running == 1`), increment `ms_count`.
   b. When `ms_count` equals `step_ms`, reset `ms_count = 0`.
   c. Advance `phase` clockwise (+1) or counter-clockwise (-1), wrapping modulo 4.
   d. Write `STEP_SEQ[phase]` to pins P1.0–P1.3.
3. In `main()`, poll `keypad_get()` and update `step_ms`, `clockwise`, or `running` state accordingly.

**Code Base:**

- Project files: `TASK5C.uvproj` (Keil µVision), `STARTUP.A51`, firmware `Objects/TASK5C.hex`.
- `main.c` — `STEP_SEQ[]`, `keypad_get()`

### 3. Inference

Motor velocity and rotation direction are dictated cleanly by the interrupt period and phase sequencing. Because step timing is isolated within the timer ISR, keyboard responsiveness does not cause jerky motion or skipped motor steps.
