# Common Libraries

This directory holds the master reference copies of shared driver libraries. Because STM32CubeIDE automatically compiles everything placed within each project's `Core/Src` and `Core/Inc` directories, individual tasks contain their own local copies; the master files here ensure consistency across the repository.

| Folder | Files | Used in | Origin / version | Licence | Limitations |
|---|---|---|---|---|---|
| `BME280_Bosch_API` | `bme280.c/.h`, `bme280_defs.h` | CAT 12, 13 | Bosch Sensortec BME280_SensorAPI v3.5.1, unchanged | BSD-3-Clause | needs the application to supply `read`, `write`, `delay_us`; built with `BME280_DOUBLE_ENABLE` (software doubles on the M4) |
| `ESPLib_MQTT` | `ESPLib.c/.h`, `mqtt_packet.c/.h` | CAT 13 | new implementation of the course ESPLib API (function names kept for the lab sheet) | MIT | single TCP connection (CIPMUX=0); QoS 0 only (QoS 1/2 requests return an error, incoming QoS 1/2 messages are dropped); incoming packets > 256 B, PUBLISH payloads > 127 B and topics > 63 characters are dropped and counted; one pending received message at a time; plaintext MQTT (port 1883) |
| `ESP8266_AT_Driver` | `esp_at.c/.h` | CAT 10D, 12B | original | MIT | blocking `AT+CIPSEND` exchange (bounded by timeouts); lines longer than 127 characters are discarded and counted; `esp_at_read_line()` returns the full line length so a too-small caller buffer is visible; a refused UART re-arm is retried from the main loop; after an RX overflow the buffered data is dropped and parsing restarts at the next line break |
| `Helpers` | `fmt_fixed.h`, `wifi_config.h` | CAT 10C – 13 | original | MIT | `fmt_2dp()` handles values within ±21 474 836.47 (int32 × 100); credentials go in the git-ignored `wifi_secrets.h` |

For the 8051 experiments, modular helpers including `delay.c/.h` (a busy-wait delay routine tuned with volatile counters), `uart.c/.h` (the CAT 6C interrupt-driven UART driver with a 16-byte ring buffer), and `lcd.c/.h` (the CAT 7 HD44780 4-bit interface) reside directly within their respective task directories.

Verification: run `make -C tests` to execute the host-side suite verifying MQTT packet assembly, parsing routines, and the ESPLib driver against a software-simulated ESP8266.
