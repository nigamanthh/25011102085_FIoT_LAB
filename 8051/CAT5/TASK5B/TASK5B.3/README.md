# EXPERIMENT DETAILS

**EXP:** CONTINUOUS ASSESSMENT 5

**DATE:** 10/08/2026

**NAME OF EXP:** Hardware Timer and Interrupt-Based Timing for LED Fading, Pulse Generation and Stepper Motor Control

**HARDWARE SETUP / PLATFORM:** AT89S52 development kit (8051 core, 11.0592 MHz crystal) + LED + DSO + ULN2003/28BYJ-48 — Keil µVision 5 (C51), progisp + USBasp ISP programmer, DSO

---

## TASK 2C : Different Signal Types

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

Instead of computing individual pulse transitions mathematically, complex digital waveforms can be treated as binary bit arrays and streamed out via a fixed timer interrupt. Timer 0 is configured in Mode 2 auto-reload to generate an interrupt every 100 µs. By grouping 5 timer ticks together, each bit duration corresponds to exactly 0.5 ms. The ISR serializes out predefined bit patterns onto P1.1: a standard square wave (`10`), a customized duty cycle waveform (`11110000`), a single narrow pulse train (`1000000000`), and a periodic burst signal consisting of five pulses followed by ten empty slots (`1010101010` + ten `0`).

**Pseudocode:**

1. Configure Timer 0 in 8-bit auto-reload Mode 2 for a 100 µs period.
2. In the ISR:
   a. Track an interrupt tick counter up to 5.
   b. Every 5 ticks (0.5 ms), output the next bit of the active waveform pattern onto pin P1.1.
   c. Advance the bit pointer and wrap around to the start of the pattern.
3. In `main()`, poll the keypad and cycle through the available waveform patterns on each key event.

**Code Base:**

- Project files: `TASK5B_3.uvproj` (Keil µVision), `STARTUP.A51`, firmware `Objects/TASK5B_3.hex`.
- `main.c` — `PATTERNS[]`, `select_pattern()`

### 3. Inference

We confirmed on the oscilloscope that complex digital pulse trains—such as periodic bursts and irregular duty cycles—can be synthesized cleanly from a single timer interrupt simply by shifting out stored bit arrays.
