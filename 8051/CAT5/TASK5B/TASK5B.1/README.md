# EXPERIMENT DETAILS

**EXP:** CONTINUOUS ASSESSMENT 5

**DATE:** 10/08/2026

**NAME OF EXP:** Hardware Timer and Interrupt-Based Timing for LED Fading, Pulse Generation and Stepper Motor Control

**HARDWARE SETUP / PLATFORM:** AT89S52 development kit (8051 core, 11.0592 MHz crystal) + LED + DSO + ULN2003/28BYJ-48 — Keil µVision 5 (C51), progisp + USBasp ISP programmer, DSO

---

## TASK 2A : Software PWM LED Fading

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

Because the standard AT89S52 lacks dedicated hardware PWM channels, we generate PWM in software using Timer 0 in 8-bit auto-reload mode (Mode 2). Setting `TH0 = TL0 = 0xA4` (256 - 92 machine cycles) yields an interrupt every 100 µs. A repeating cycle of 10 time slots defines our PWM frame (10 × 100 µs = 1 ms, giving a flicker-free 1 kHz carrier frequency). The variable `duty` (ranging from 0 to 10) sets how many slots the LED stays ON (`sbit led = P1^1;`). Within the same ISR, a secondary counter (`step_ticks`) tracks 1000 interrupts (100 ms). Every 100 ms, the ISR sets `step_flag`, prompting the main loop to alter the duty cycle by 10%. Over a 2-second period, the LED smoothly fades from 0% to 100% and back down to 0%.

**Pseudocode:**

1. In `main()`, initialize `led = 0` and `direction = 1`, then call `timer0_init()`.
2. In `timer0_init()`, configure Timer 0 in Mode 2 (8-bit auto-reload) via `TMOD = (TMOD & 0xF0) | 0x02`, set `TH0 = TL0 = 0xA4`, and enable Timer 0 interrupts (`ET0 = 1`, `EA = 1`, `TR0 = 1`).
3. In `timer0_isr() interrupt 1 using 1`:
   a. Increment `pwm_slot` (wraps to 0 when $\ge 10$).
   b. Drive `led = (pwm_slot < duty)`.
   c. Increment `step_ticks`; upon reaching 1000 (100 ms), reset `step_ticks = 0` and assert `step_flag = 1`.
4. In `main()`, poll `step_flag` in an infinite loop.
5. When `step_flag` is set, clear it (`step_flag = 0`) and adjust `duty += direction`; reverse `direction = -1` when `duty >= 10`, and `direction = 1` when `duty == 0`.

**Code Base:**

- Project files: `TASK5B_1.uvproj` (Keil µVision), `STARTUP.A51`, firmware `Objects/TASK5B_1.hex`.
- `main.c` — `timer0_init()`, `timer0_isr() interrupt 1 using 1` (PWM and fade time base in one short ISR)

### 3. Inference

The LED brightens and dims smoothly without optical flicker; the apparent brightness follows the software duty cycle while the carrier frequency remains fixed at 1 kHz.
