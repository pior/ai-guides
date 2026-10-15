# Freenove FNK0104N — guide for agents

ESP32-S3 board with a 3.5" 320×480 QSPI touch display, audio codec, speaker amp, mic.
Arduino-ESP32 3.3 on PlatformIO (pioarduino). Written 2026-10 from one working project.

Tags: **[board]** tested on hardware · **[sch]** read from the Freenove schematic ·
**[vendor]** from Freenove code, not tested here. Untagged = how working firmware does it.

## Sources

- Vendor repo (code, schematic, datasheets): https://github.com/Freenove/Freenove_ESP32_S3_Display
  - Schematic: `Schematic/3.5inch_ESP32-S3_Display_Schematic.pdf`
  - Display/touch driver: `Libraries/FNK0104N/TFT_eSPI_v2.5.43.zip` → `ST77922.cpp`, `ST77922_Touch.cpp`
  - ES8311 driver: sketch `Sketch_07.1_Music` → `es8311.cpp`
  - Datasheets: `Datasheet/` (ST77922 spec + TDDI protocol, ES8311, TP4054, LMA2718, ESP32-S3)
  - Pins per board: `#ifdef FNK0104N_3P5_320x480_ST77922` blocks in the sketches
  - **License CC BY-NC-SA 3.0.** Do not paste Freenove code into a permissively licensed repo.
    Espressif's `es8311.c` inside it is Apache-2.0.
- No official mechanical drawing or STEP file exists.
- ESP32-S3 datasheet: https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf
- ESP32-S3 TRM: https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf
- Arduino-ESP32: https://docs.espressif.com/projects/arduino-esp32/en/latest/
- pioarduino platform: https://github.com/pioarduino/platform-espressif32
- ESP-IDF (S3) PCNT: https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/pcnt.html
- esptool / espefuse: https://docs.espressif.com/projects/esptool/en/latest/esp32s3/
- LVGL 8.4: https://docs.lvgl.io/8.4/

## Hardware

| Part | Detail | Tag |
|---|---|---|
| SoC | ESP32-S3R8: 8 MB **octal** PSRAM in package, 40 MHz crystal | board |
| Flash | 16 MB QIO (`esptool flash_id`: mfr 0x5e, dev 0x4018). Schematic symbol `W25X40` is generic: ignore | board |
| Display | 3.5" IPS 320×480, **ST77922**, QSPI. Native portrait | board |
| Touch | capacitive, **inside the ST77922 (TDDI)**, I2C 0x55, 16-bit registers, 5 points | board |
| Codec | ES8311, I2C 0x18, I2S with MCLK | board |
| Speaker amp | SC8002B, enable = `AUDIO_EN` | board |
| Mic | LMA2718 analog MEMS → ES8311 ADC. Sound hole through the PCB, screen side | board |
| Charger | TP4054 on BAT+ | sch |
| RGB LED | WS2812, GPIO40 | vendor |
| SD | SDMMC 4-bit slot | vendor |

## Pin map

| Function | GPIO | Tag |
|---|---|---|
| LCD QSPI CS / SCLK / D0 / D1 / D2 / D3 | 10 / 12 / 11 / 13 / 14 / 9 | board |
| LCD backlight (BSS138, HIGH = on) | 41 | board |
| LCD reset | none driven: on the board RESET net | vendor |
| LCD TE | routed on schematic, GPIO not identified | sch |
| I2C SDA / SCL (touch, codec, header P4) | 38 / 39 | board |
| Touch RST / INT | 48 / 47 (INT unused: polled) | board |
| I2S MCLK / BCLK / WS / DOUT (→codec) / DIN (←codec) | 17 / 18 / 21 / 15 / 16 | board |
| `AUDIO_EN` — **active LOW** | 1 | board |
| UART0 TX / RX (header P2) | 43 / 44 | board |
| Header P3 | 45 / 46 (**strapping pins**) | board |
| BOOT button | 0 | vendor |
| Battery ADC (`BAT_ADC`, 100k/100k divider R14/R15) | 8 | vendor, sch |
| SD CLK / CMD / D0 / D1 / D2 / D3 | 5 / 4 / 6 / 7 / 2 / 3 | vendor |

Only GPIO43, 44, 45, 46 and the I2C bus are exposed on headers [sch].

## Connectors (back side)

| Conn | Pins | Notes |
|---|---|---|
| P2 UART | 5 V, GND, TXD0 (43), RXD0 (44) | 100 Ω series resistors [sch]. 5 V via P-MOSFET Q5, gate pulled to GND = always on [sch]; powered an LD2410C (~80 mA) [board] |
| P3 IO | 3.3 V, GND, GPIO45, GPIO46 | strapping pins, see below |
| P4 I2C | 3.3 V, GND, SDA 38, SCL 39 | same bus as touch + codec; 10 kΩ pull-ups on board (R29/R30) [sch] |
| JP1 BAT | MX1.25, pin 1 BAT+, pin 2 GND | **no reverse-polarity protection** [sch] |
| JP3 | speaker, PH1.25 2-pin | |
| USB-C | native USB (CDC + flashing) | |

- Schematic shows signals per header, **not which physical pin is pin 1**: check silkscreen or multimeter.
- Header cables leave parallel to the board, from the edge.

## Power [sch]

- USB VBUS → D8 (Schottky) → +5 rail.
- JP1 BAT+ → Q3 (SL2305, gate = VBUS) → +5 rail. Q3 off while USB has power: USB and JP1 can be connected together.
- +5 rail feeds P2 5 V, speaker amp, 3.3 V regulator.
- TP4054 BAT pin abs max 7 V (datasheet): a regulated 5 V source on JP1 stays within limits.
- **Not verified:** battery runtime, charging behaviour, current draw, sleep current, `BAT_ADC` accuracy. Measure before designing a battery product.

## PlatformIO

```ini
[env:board]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/55.03.312-1/platform-espressif32.zip
framework = arduino
board = esp32-s3-devkitc-1
board_build.arduino.memory_type = qio_opi   ; QIO flash + octal PSRAM
board_build.flash_mode = qio
board_upload.flash_size = 16MB
board_upload.maximum_size = 16777216
board_build.partitions = default_16MB.csv   ; LittleFS = 3.4 MB [board]
board_build.filesystem = littlefs
monitor_speed = 115200
monitor_filters = esp32_exception_decoder
build_flags =
    -DBOARD_HAS_PSRAM
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1   ; logs on native USB, UART0 stays free for P2
    -DCORE_DEBUG_LEVEL=3
    -DLV_CONF_INCLUDE_SIMPLE
    -Iinclude
lib_deps = lvgl/lvgl@8.4.0
```

- `pio` installed by the VS Code extension is at `~/.platformio/penv/bin/pio` (not on PATH).
- Host unit tests: separate `[env:native]` with `platform = native`, `build_src_filter = +<logic/>`, `test_build_src = yes`; keep hardware-free code in `src/logic/`. Add `test_ignore` for those tests in the board env.

## Serial and flashing (macOS)

- Port: `/dev/cu.usbmodem*`.
- **Opening the port resets the board** (DTR/RTS pulse, even with pyserial `dtr`/`rts` preset False) [board]. Opened during boot, it shows as `esp_reset_reason()` 5 (interrupt watchdog): not a crash.
- USB CDC drops early boot output. To capture boot: open, `rts=True` 200 ms then False (reset), close, reopen in a retry loop.
- **Kill any serial logger before flashing**: it holds the port and the upload fails.
- Read-only checks: `esptool.py flash_id`, `espefuse.py summary`. **Never burn eFuses.**

## Display — ST77922 QSPI

- TFT_eSPI does not drive it. Freenove uses ESP-IDF `spi_master` directly (TFT_eSPI only for sprites) [vendor].
- Bus: `SPI2_HOST`, `SPICOMMON_BUSFLAG_IOMUX_PINS | QUAD`, **80 MHz**, mode 0, half duplex, DMA auto [board].
- Register write: cmd `0x02` (1 line), 8-bit cmd + 24-bit address field = `reg << 8`, data on 1 line.
- Pixels: cmd `0x32` (4 lines, `SPI_TRANS_MODE_QIO`), address `0x3C << 8` (RAM write continue), chunks of 16 K pixels. Window: `0x2A` columns, `0x2B` rows (4 bytes each, big-endian start/end).
- Init: vendor table (~65 writes in `ST77922.cpp`), then `0x11` + 120 ms, `0x29`, `0x2C`, `0x3A=0x01`, `0x36=0x00`, `0x35=0x01` + 20 ms.
- Colors: RGB565 **byte-swapped** (`LV_COLOR_16_SWAP 1`).
- MADCTL supports portrait and portrait-flipped only. Landscape = **rotate in software** per flush into an internal DMA buffer, then send [board]. Touch coordinates must get the same transform.
- Windows must be **4-pixel aligned**: x1/y1 down to a multiple of 4, x2/y2 up to `4n+3` (LVGL `rounder_cb`). 320 and 480 are multiples of 4, so it holds after rotation.
- Backlight: LEDC PWM on GPIO41, 5 kHz, 8-bit works (`ledcAttach`, `ledcWrite`) [board]. Freenove only switches it on/off.

## Touch — ST77922 TDDI (I2C 0x55)

- Answers **only after a reset pulse**: RST LOW 100 ms, HIGH, wait 100 ms. An I2C scan before that finds only 0x18 [board].
- Init: wait until `STATUS (0x0001) & 0x0F == 0`. Freenove waits forever: **add a timeout**.
- Max points: register `0x0009` (5 here).
- Read: `TOUCH_INFO (0x0010)` bit 3 = new data. Points from `0x0014`, 7 bytes each; byte 0 bit 7 = valid; X = 14 bits (bytes 0–1), Y = 14 bits (bytes 2–3).
- **Read the whole report (7 × max points bytes).** Reading only point 0 never clears the new-data bit: first touch sticks forever [board].
- Release = one report with the valid bit cleared, then `TOUCH_INFO` = 0 [board].

## LVGL (8.4.0, same major as Freenove — v8 and v9 APIs differ)

```c
#define LV_COLOR_DEPTH 16
#define LV_COLOR_16_SWAP 1
#define LV_MEM_CUSTOM 1
#define LV_MEM_CUSTOM_INCLUDE "esp_heap_caps.h"
#define LV_MEM_CUSTOM_ALLOC(size) heap_caps_malloc(size, MALLOC_CAP_SPIRAM)
#define LV_MEM_CUSTOM_FREE heap_caps_free
#define LV_MEM_CUSTOM_REALLOC(p, size) heap_caps_realloc(p, size, MALLOC_CAP_SPIRAM)
#define LV_TICK_CUSTOM 1
#define LV_TICK_CUSTOM_INCLUDE "esp_timer.h"
#define LV_TICK_CUSTOM_SYS_TIME_EXPR (esp_timer_get_time() / 1000LL)
#define LV_DISP_DEF_REFR_PERIOD 30
```

- Draw buffer: one buffer, 480 × 40 px, **in PSRAM**; flush rotates into the internal DMA buffer and sends with `spi_device_polling_transmit` (blocking) [board].
- Only one task calls LVGL.
- Custom fonts with accents and icons: `lv_font_conv` (npx `lv_font_conv@1.5.2`), Montserrat + FontAwesome 5 from `.pio/libdeps/<env>/lvgl/scripts/built_in_font/`, range `0x20-0x7E,0xA0-0xFF`, plus the `LV_SYMBOL_*` code points; `src=>dst` maps a glyph to an emoji code point.

## Audio — ES8311 + SC8002B

- Codec must be configured over I2C before any sound [board].
- **Freenove `es8311.cpp` aborts at boot on Arduino-ESP32 3.3**: it uses the legacy I2C driver while `Wire` uses the new one → `CONFLICT! driver_ng is not allowed to be used with this old driver` [board]. Fix: rewrite its register read/write functions with `Wire` (under the bus mutex).
- I2S: `ESP_I2S.h`, `I2S_MODE_STD`, 16 kHz, 16-bit, mono, left slot; MCLK = 256 × fs; codec `mclk_from_mclk_pin = true` [board].
- `AUDIO_EN` (GPIO1) **active LOW**: LOW = +28 dB vs HIGH = amp fully off [board].
- Working playback sequence [board]: `i2s.begin` → `AUDIO_EN` LOW → 20 ms silence → samples → 50 ms silence (DMA drain) → `AUDIO_EN` HIGH → `i2s.end`. Configure the codec once at boot (I2S running for MCLK), then stop I2S until needed.
- Levels [board]: full-scale tone at codec volume 85 = loud at a desk. Amplitude 1500/32767 at volume 52 = barely audible in an enclosure. Amplitude 6000 at volume 63 = clearly audible in an enclosure.
- Mic works at codec default gain [board]. An I2S RX channel costs internal RAM: open it only if needed.

## I2C bus (GPIO38/39)

- Shared by touch (0x55), codec (0x18) and anything on P4. Guard every access with **one recursive mutex**.
- 100 kHz for all devices works [board] (Freenove: touch 100 kHz, codec 400 kHz).
- Arduino-ESP32 3.3 = new I2C driver (`driver_ng`); never mix with legacy `driver/i2c.h` code.
- **Avoid address-only probes** (`Wire` transmission with no data, `i2c_master_probe`): probing absent devices was followed by `StoreProhibited` crashes inside the I2C master ISR (3 in ~4 min). Replaced by a one-byte harmless write (e.g. a register select): no crash since [board]. Root cause not proven.
- A NACK logs `ESP_ERR_INVALID_STATE`: expected for an absent device. Poll for optional devices slowly (e.g. every 60 s).

## GPIO45 / GPIO46 (header P3) — strapping pins

- GPIO45 = flash voltage strap; GPIO46 = matters in download mode (GPIO0 LOW needs GPIO46 LOW) (ESP32-S3 datasheet).
- On our unit (`espefuse.py summary`): `VDD_SPI_FORCE=1`, `VDD_SPI_TIEH=1` (flash 3.3 V from eFuse, GPIO45 ignored), `UART_PRINT_CONTROL=0`. **Check yours** before relying on it.
- Keep both LOW at reset: external pull-downs. Do not use modules with pull-ups to VCC (KY-040-style encoder boards).
- PCNT driver **enables internal pull-ups** on its inputs (`pulse_cnt.h`): call `gpio_pullup_dis()` after creating the channels.
- PCNT glitch filter max ≈ 12.8 µs (1023 APB cycles): does not cover ms contact bounce (see possible fixes).

## Memory

- **Internal RAM is the constraint**, PSRAM is plentiful.
- HTTPS (mbedTLS) failed with `SSL - Memory allocation failed` at 43–60 KB free internal heap; worked at 95 KB free / 54 KB largest block [board].
- LVGL heap in PSRAM (`LV_MEM_CUSTOM_ALLOC` above): internal free 59 KB / 32 KB largest → 125 KB / 82 KB [board].
- Put draw buffers, histories, caches in PSRAM (`heap_caps_malloc(n, MALLOC_CAP_SPIRAM)`). DMA buffers stay internal (`MALLOC_CAP_DMA | MALLOC_CAP_INTERNAL`).
- Track `heap_caps_get_largest_free_block(MALLOC_CAP_INTERNAL)` and `heap_caps_get_minimum_free_size(MALLOC_CAP_INTERNAL)`.
- One HTTPS connection at a time (rule we used, not measured).

## Wi-Fi and network (Arduino-ESP32 3.3)

- **`WiFi.status()` can say not connected while traffic flows**: after a short drop it stays `WL_IDLE_STATUS` until DHCP answers; lwIP keeps the old address. Seen as "offline" UI while TCP worked [board]. Use `WiFi.STA.connected() && WiFi.localIP() != IPAddress()`. Do not call `WiFi.begin()` while associated: it drops a working link.
- Join the strongest AP of a multi-AP SSID: `WiFi.setScanMethod(WIFI_ALL_CHANNEL_SCAN)` + `WiFi.setSortMethod(WIFI_CONNECT_AP_BY_SIGNAL)` before `begin` [board].
- `WiFi.enableIPv6(true)`: without it, macOS waits 5 s for an AAAA mDNS answer (`curl http://name.local` took 5 s) [board].
- Board reaches the internet but not LAN peers (and vice versa): check **AP client isolation** (typical on "IoT" SSIDs) [board].
- Our mesh router dropped/delayed ICMP to itself while traffic was fine: false "packet loss". TCP connect time (non-blocking `connect` + `select`, close with `SO_LINGER {1,0}`) is a reliable probe [board].
- Log disconnect reasons: `ARDUINO_EVENT_WIFI_STA_DISCONNECTED` → `WiFi.disconnectReasonName()`.
- Time: `configTime(0, 0, ...)` keeps the system clock UTC; `sntp_set_time_sync_notification_cb` tells when it is set. Show "--:--" before sync, never 1970. Time zones without global `TZ`: AceTime (`DateStrings` returns a shared buffer: copy it).

## Persistence

- LittleFS (`LittleFS.begin(true)` formats on first use) and NVS (`Preferences`) both work [board].
- Write to a temp file, then rename. Header with magic + record size to detect format changes.
- Persist a **boot record in NVS**: last N boots with `esp_reset_reason()`, boot time, uptime saved periodically. Only way to see crashes when the board ran off-USB. Reasons: 1 power-on, 3 software, 4 panic, 5/6/7 watchdogs, 8 deep-sleep wake, 9 brownout, 11 USB (flash / serial open).

## Firmware structure that worked

- FreeRTOS tasks pinned explicitly; UI (LVGL) alone on core 1, services on core 0; delete the Arduino `loop` task.
- Services never call each other: a mutex-protected state struct (UI snapshots it ~every 150 ms) + queues for one-off events.
- Pure C++ logic (no Arduino headers) tested on the host.
- Secrets in a gitignored header; commit an example copy.
- `GET /status` JSON with heap, reset reason, Wi-Fi BSSID/RSSI/disconnect reason: debug without USB.
- mDNS hostname + `_http._tcp` service; plain HTTP + raw TCP line server for notifications from scripts.

## Agent test rig (macOS)

- See the screen: `imagesnap -w 2 photo.jpg`, then read the image. If the screen is not in frame, ask the human.
- Hear the speaker: `rec -q -c 1 -r 16000 -b 16 out.wav` (sox) while triggering a sound over HTTP. Analyse with Goertzel in 50 ms windows: compare to the max of quiet windows before the trigger, plus a control frequency. **One window above noise is not evidence**; check the speaker is plugged in.
- Touch tests need a human: send a notification to the screen + a quiet Mac sound to call them.
- Keep test sounds moderate; none at night.
- Expose test endpoints (show page X, play sound Y) so an agent can drive the UI over HTTP.

## Add-on modules (tested)

| Module | Connection | Findings |
|---|---|---|
| HLK-LD2410C radar | P2: 5 V, TX→GPIO44; 256000 baud | RX only needed. Ours was wired TX/RX swapped at first: look for bytes on both pins. Raw distance ±15 cm; trimmed mean of 20 samples (drop 20 % each end) ±5 cm [board]. Datasheet: https://d.hlktech.net/download/HLK-LD2410C/1/HLK%20LD2410C%20Human%20%20Presence%20Sensor%20Module%20Data%20Sheet%20V1.00.pdf |
| HTU21D (SparkFun) | P4, 0x40 fixed, 3.3 V only | No-hold commands, CRC-8 poly 0x131. Rose 24.5 → 25.1 °C in 20 min after boot, close to the board: compare with a reference thermometer [board] |
| TSL2561 (Adafruit) | P4, 0x39 (ADDR float) / 0x29 / 0x49 | Datasheet lux formula, 402 ms integration, 1×/16× auto gain [board] |

Module pull-ups on P4 in parallel with the board's 10 kΩ (e.g. 4.7 kΩ → ≈ 3.2 kΩ) work at 100 kHz [board].

## Possible fixes (unconfirmed)

Not proven. Each one was applied in working firmware, but the problem it was meant to fix was never isolated. Try them when you see the same symptom.

- **Interrupt watchdog resets** (`Interrupt wdt timeout on CPU0`, both cores in the idle task, about every 20 min). The firmware ran I2S TX all the time, so the DMA interrupt fired on core 0 every 15 ms to send silence. Fix applied: start I2S only to play a sound, stop it afterwards. The address-only I2C probe fix (see I2C) landed in the same period, and nothing showed which change stopped the crashes.
- **Board unreachable from the LAN while its own traffic works.** Fix applied: a gratuitous ARP every 15 s (`etharp_gratuitous(netif_default)` through `tcpip_callback`). In our case the real cause turned out to be AP client isolation; whether the ARP helps on its own is unknown.
- **Diagnosing lost multicast**: resolve a known LAN host with `mdns_query_generic(..., MDNS_QUERY_MULTICAST, ...)`. `MDNS.queryHost()` asks for a unicast reply, so it does not test multicast reception.
- **Wi-Fi power save off** (`WiFi.setSleep(false)`) on a mains-powered device, for latency and reachability. Its effect was not measured.
- **Encoder contact bounce on PCNT**: 10 kΩ pull-down ∥ 10 nF on each input (τ = 100 µs). Stay below ≈ 100 nF (1 ms) so fast turns are kept. Calculated, not measured.
- **Mesh roaming**: the ESP32 joins one AP and stays on it. Every 60 s, when RSSI < −70 dBm, scan and `WiFi.begin(ssid, pass, channel, bssid)` to an AP at least 10 dB stronger. Never exercised away from the desk.

## Not verified

Battery operation and charging, current/sleep consumption, `BAT_ADC`, SD card, RGB LED, LCD TE, touch INT pin, mic gain tuning, ESP-IDF without Arduino, LVGL 9, Espressif's `esp_lcd` ST77922 component.
