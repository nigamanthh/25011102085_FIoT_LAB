# EXPERIMENT DETAILS

**EXP:** CONTINUOUS ASSESSMENT 8

**DATE:** 18/09/2026

**NAME OF EXP:** STM32 GPIO Bring-Up and Switch-Controlled LED

**HARDWARE SETUP / PLATFORM:** Blue Pill STM32F103C6T6 (Cortex-M3, 72 MHz) + push-button — STM32CubeMX + Keil MDK-ARM (STM32CubeIDE project also included), STM32 HAL

---

## TASK 8C : Short Press vs Long Press

### 1. Circuit Diagram

| Pin | CubeMX setting | Connected to |
|---|---|---|
| PC13 | GPIO_Output, push-pull | on-board LED (active-LOW) |
| PA15 | GPIO_Input, pull-up | push-button → GND |
| PA13 / PA14 | SYS: Debug = Serial Wire | ST-LINK (also frees PA15 from JTAG) |
| RCC | HSE crystal → PLL ×9 | 72 MHz |

![Circuit Diagram](circuit.png)

*Drawn on [Wokwi](https://wokwi.com). To open and edit it: start a new project on wokwi.com, paste [`wokwi/diagram.json`](wokwi/diagram.json) into its diagram.json tab, and add the `wokwi/*.chip.json` files (if any) with **New file**. The AT89S52, USB-TTL, ULN2003, USBasp and ST-LINK boards are custom chips (pin names added as labels). The ESP-01 part stands in for the Wemos D1 mini and the BMP180 module for the BME280 (same AT firmware / same 4-pin I²C header); the Nucleo-64 graphic is labelled as the F401RE.*

### 2. Core Logic / Pseudocode / Code Base

**Working Principle:**

This task extends event handling by distinguishing between short and long button presses to trigger different operational modes. When `debounce_update()` detects a valid press edge (`BTN_PRESS_EDGE`), the firmware records the system timestamp in `press_start_ms`. The user continues holding the button while the main loop executes normally. Upon release (`BTN_RELEASE_EDGE`), the duration is calculated: $\Delta t = \text{release\_time} - \text{press\_start\_ms}$. The routine `classify_press()` evaluates the duration: $\Delta t < 1000$ ms is categorized as a short press, while $\Delta t \ge 1000$ ms is categorized as a long press. Note that classification occurs strictly **upon button release**, meaning long-press blinking activates when the button is let go rather than mid-hold. An active blinking sequence (toggling every 200 ms using non-blocking timestamps) can be halted by a subsequent short press, or restarted with another long press.

| Action | Response |
|---|---|
| Short | toggle LED |
| Long | continuous blinking (200 ms) |
| Short while blinking | stop blinking |
| Long again | blink again |

**Pseudocode:**

1. When a debounced press edge occurs, capture `press_start_ms = HAL_GetTick()`.
2. When the corresponding release edge arrives, calculate `duration = HAL_GetTick() - press_start_ms`.
3. Pass `duration` to `classify_press()`:
   a. If duration $\ge 1000$ ms (Long Press): enable blinking mode (`blinking = 1`).
   b. If duration $< 1000$ ms (Short Press): if already blinking, exit blinking mode; otherwise, toggle the LED directly.
4. When `blinking == 1`, check if 200 ms have passed since the last toggle using `HAL_GetTick()`; if so, invert PC13 and refresh the blink timer.

**Code Base:**

- Project files: `CAT08C_Short_vs_Long_Press.ioc` (CubeMX configuration), `MDK-ARM/CAT08C_Short_vs_Long_Press.uvprojx` (Keil MDK), `.project` / `.cproject` (STM32CubeIDE); application code in `Core/Src/main.c`.
- `classify_press()`
- `HAL_GetTick()` for both press timing and blink interval — no `HAL_Delay()`

### 3. Inference

Using elapsed timestamps rather than blocking delay loops allows the STM32 to accurately measure press durations while concurrently running non-blocking LED blinking routines.
