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

1. Initialize button pins (`button_col = 0; button_row = 1;`), configure Timer 0 in Mode 2 (`TMOD = (TMOD & 0xF0) | 0x02; TH0 = TL0 = 0xA4;`), initialize pattern with `select_pattern(0)`, and enable interrupts (`EA = 1; TR0 = 1;`).
2. In `select_pattern(n)` (patterns 0 to 3):
   a. Disable Timer 0 interrupt (`ET0 = 0`).
   b. Use a `switch(n)` block to set `cur_pattern` and `cur_len` (case 0: `square` "10", len 2; case 1: `duty` "11110000", len 8; case 2: `pulse` "1000000000", len 10; case 3: `burst` "10101010100000000000", len 20) and show the pattern number 1–4 on 7-segment Port 0 (0x06, 0x5B, 0x4F, 0x66).
   c. Reset `bit_idx = 0; tick_cnt = 0;` and re-enable `ET0 = 1;`.
3. In `timer0_isr() interrupt 1 using 1`:
   a. Track `tick_cnt`; every 5 ticks (0.5 ms), reset `tick_cnt = 0`.
   b. Output the active bit: `signal = (cur_pattern[bit_idx] == '1');` (`sbit signal = P1^1;`).
   c. Advance `bit_idx` and wrap around to 0 when `bit_idx >= cur_len`.
4. In `main()`, poll `button_pressed()` in an infinite loop (debounced with 20 ms delay and wait-for-release). When pressed, advance `n` (wrapping at $\ge 4$ back to 0) and call `select_pattern(n)`.

**Code Base:**

- Project files: `TASK5B_3.uvproj` (Keil µVision), `STARTUP.A51`, firmware `Objects/TASK5B_3.hex`.
- `main.c` — `select_pattern()`, `timer0_isr() interrupt 1 using 1`, `button_pressed()`
- `delay.c/.h` — software delay for button debounce

### 3. Inference

We confirmed on the oscilloscope that complex digital pulse trains—such as periodic bursts and irregular duty cycles—can be synthesized cleanly from a single timer interrupt simply by shifting out stored bit arrays.
