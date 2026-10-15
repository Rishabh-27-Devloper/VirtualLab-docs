# 🖥️ VirtualLab-HIL — Standalone Deliverable Device & 2.8" TFT Mini OS Guide

> **Transform your ESP32 into a standalone, commercial-grade Hardware-in-the-Loop laboratory instrument featuring an onboard 2.8-inch ILI9341 color TFT LCD and DisplayOS.**

---

## 1. Overview & Concept

The **VirtualLab-HIL Standalone Deliverable Node** packages an ESP32 microcontroller with a vibrant **2.8-inch 240x320 TFT LCD (ILI9341 SPI)** to create a self-contained, benchtop hardware device. 

Instead of operating as a "headless" blind black box, the device runs **DisplayOS** — a lightweight, cyberpunk-styled real-time operating display system that automatically rotates through interactive diagnostic dashboards, visualizes live waveforms on a mini oscilloscope, detects physical wire connections automatically, and provides instant status updates.

```
┌────────────────────────────────────────────────────────────┐
│                    VIRTUALLAB-HIL NODE                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ ⚡ VIRTUALLAB-HIL           WIFI:-54dBm [LIVE <35ms] │  │
│  │ ──────────────────────────────────────────────────── │  │
│  │  ANALOG INGRESS (ADC1 0-3.3V)                        │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐         │  │
│  │  │ A0: 1.65V │  │ A1: 3.30V │  │ A2: 0.00V │         │  │
│  │  │ [====   ] │  │ [=======] │  │ [       ] │         │  │
│  │  └───────────┘  └───────────┘  └───────────┘         │  │
│  │  DIGITAL INGRESS               ACTUATOR EGRESS       │  │
│  │  [D0: 1] [D1: 0] [D4: 1]       DAC0: 2.50V  PWM0: 75%│  │
│  │  ─────────────────────────────────────────────────── │  │
│  │  ● ○ ○ ○ ○  Dashboard Overview          [====      ] │  │
│  └──────────────────────────────────────────────────────┘  │
│    [A0] [A1] [A2] [A3] [A4] [A5]  [DAC0] [DAC1] [PWM0] [D2] │
└────────────────────────────────────────────────────────────┘
```

---

## 2. Hardware Bill of Materials (BOM)

| Component | Specification | Purpose |
|---|---|---|
| **Microcontroller** | ESP32 WROOM-32 / DevKit v1 (30 or 38 pins) | Dual-core 240 MHz compute engine with Wi-Fi & hardware ADC/DAC |
| **Color Display** | 2.8-inch TFT LCD Module (ILI9341 controller, SPI, non-touch) | 240x320 resolution, vibrant 16-bit 65K color graphics |
| **Push Button** | Tactile Push Button (or use onboard BOOT button) | Screen navigation & auto-carousel freeze/hold control |
| **Connecting Wires** | Female-to-Female jumper wires or PCB ribbon cable | Connects display header to ESP32 pins |
| **Enclosure (Optional)** | 3D-printed enclosure or transparent acrylic case | Packages the unit into a finished, portable lab instrument |
| **Power Supply** | Standard Micro-USB / USB-C 5V 1A adapter | Powers ESP32 and TFT backlight |

---

## 3. Physical Wiring Pinout Reference

The SPI display connects to the ESP32's native **VSPI hardware bus** to deliver 40 MHz high-speed refresh rates with zero screen flicker.

### 3.1 TFT LCD Pin Connections (ILI9341 SPI & Navigation Button)

| 2.8" TFT Pin / Control | ESP32 GPIO | Description / Notes |
|---|---|---|
| **VCC** | **3.3V** or **5V** | If module has onboard 3.3V LDO regulator (red PCB), connect to 5V or 3.3V. |
| **GND** | **GND** | Common Ground |
| **CS** | **GPIO 15** | TFT Chip Select (Active Low) |
| **RESET / RST** | **GPIO 4** | Hardware Reset (Can alternatively be wired to ESP32 `EN` pin) |
| **DC / RS** | **GPIO 2** | Data / Command selector line |
| **SDI / MOSI** | **GPIO 23** | VSPI Hardware Master-Out-Slave-In |
| **SCK / CLK** | **GPIO 18** | VSPI Hardware Serial Clock |
| **LED / BL** | **GPIO 21** or **3.3V** | Backlight control (Wired to GPIO 21 for PWM dimming, or direct 3.3V for max brightness) |
| **SDO / MISO** | *Not Connected* | Display is write-only; leaving disconnected frees an extra ESP32 GPIO! |
| **Push Button** | **GPIO 0 (BOOT)** | Press to advance to the next screen. Auto-carousel is permanently stopped. (One leg to GPIO 0, other to GND; internal pullup enabled). |

---

### 3.2 Complete Hardware I/O Assignment for Deliverable Device

With the display connected as above, the remaining pins provide a complete, conflict-free mixed-signal testbench:

```
                  ┌──────────────────────────────┐
                  │          ESP32 NODE          │
                  │                              │
[ADC1 Safe Wi-Fi] │ A0 (GPIO 36 / VP)            │ [SPI Display]
                  │ A1 (GPIO 39 / VN)            │   CS   -> GPIO 15
                  │ A2 (GPIO 34)                 │   DC   -> GPIO 2
                  │ A3 (GPIO 35)                 │   RST  -> GPIO 4
                  │ A4 (GPIO 32)                 │   MOSI -> GPIO 23
                  │ A5 (GPIO 33)                 │   SCK  -> GPIO 18
                  │                              │   BL   -> GPIO 21
[Digital Inputs]  │ D0 (GPIO 13)                 │
                  │ D1 (GPIO 14)                 │ [Hardware Outputs]
                  │ D4 (GPIO 27)                 │   DAC0 -> GPIO 25 (8-bit)
                  │ D5 (GPIO 16)                 │   DAC1 -> GPIO 26 (8-bit)
                  │ D6 (GPIO 17)                 │   PWM0 -> GPIO 19
                  │                              │   PWM1 -> GPIO 22
                  │                              │   D2   -> GPIO 12 (Status)
                  │                              │   D3   -> GPIO 5  (Output)
[Navigation Key]  │ BTN (GPIO 0 / BOOT -> GND)   │
                  └──────────────────────────────┘
```

---

## 4. DisplayOS Architecture: 5 Specialized Lab Dashboards

**DisplayOS** runs as a dedicated FreeRTOS graphics task on **Core 1**, updating the screen at a smooth 20 FPS (every 50 ms) without loading Core 0's WebSocket and Wi-Fi streaming.

### 4.1 Dual Orientation Engine (Portrait 240x320 & Landscape 320x240)
The graphics engine natively supports both orientations:
- **Portrait Mode (`TFT_ORIENTATION 0` - 240x320) [DEFAULT]**: Tailored specifically for handheld deliverable instruments (multimeter form-factor). All 6 ADC cards, digital tiles, oscilloscope frame, pin auto-detection table, and dials fit inside the 240-pixel screen width with zero clipping.
- **Landscape Mode (`TFT_ORIENTATION 1` or `3` - 320x240)**: Reconfigures all elements into a wide benchtop layout.

### 4.2 Zero-Flicker Differential Redraw Engine
- **Setup Screen Fix**: The Wi-Fi Captive Portal screen is rendered **once** when entering AP mode, eliminating the violent 20 Hz `fillScreen` strobe.
- **Oscilloscope Differential Trace Erasing**: Erases only the previous line trace coordinates rather than performing full-area canvas wipes, achieving crystal-clear, flicker-free waveform rendering.
- **Opaque Typography**: Live numerical readouts overwrite directly with background glyph fills, avoiding full rectangular clearing.
- **27 MHz SPI Clock**: Configured for clean signal integrity over standard breadboard jumper wires.

### 4.3 Permanent Manual Navigation
**Auto-carousel is permanently stopped.** Just like a real benchtop oscilloscope or laboratory multimeter, the screen **never** switches unexpectedly on a timer. It remains locked on the active dashboard, continuously updating live voltages, waveforms, and gauges with zero flicker, and **only advances when you physically press the navigation button**.

### Page 0: Multi-Channel Telemetry Dashboard
- Displays all 6 Analog ADC channels ($A_0$–$A_5$) with numerical readouts and horizontal level fill bars ($0.00\text{V} - 3.30\text{V}$).
- Shows 5 Digital Inputs ($D_0, D_1, D_4, D_5, D_6$) as glowing green (`1`) or dark (`0`) status tiles.
- Shows real-time Actuator Egress states (DAC0/DAC1 voltages, PWM duty percentages, and digital relays).

### Page 1: Real-Time Mini Oscilloscope (Waveform Viewer)
- 140-sample rolling waveform graph with dotted graticule divisions.
- Prioritizes active DAC0 output or A0 input.
- Real-time telemetry readouts: Peak-to-Peak voltage ($V_{pp}$), RMS voltage ($V_{rms}$), signal frequency ($f = 10\text{ Hz}$), and timebase scale ($10\text{ ms/div}$).

### Page 2: Smart Pin Connection & Auto-Detection Map
- An automated diagnostic algorithm continuously calculates sliding voltage variance, steady-state rail levels, and pulldown response across all pins.
- **Auto-Detection States**:
  - `ACTIVE SENSING (WAVE)`: Glowing green LED indicator (connected to AC signal, potentiometer, or sensor).
  - `TIED TO 3.3V VCC`: Glowing cyan LED indicator.
  - `TIED TO 0.0V GND`: Purple indicator.
  - `DISCONNECTED / FLOATING`: Muted gray indicator.
- Quickly verify which breadboard wires are plugged in without needing a multimeter!

### Page 3: Actuator Gauges & PWM Meters
- Large semicircular dials for continuous DAC0 and DAC1 outputs ($0.00\text{V} - 3.30\text{V}$).
- PWM duty cycle bar meters ($0\% - 100\%$) for high-speed motor and LED control.
- Digital output toggle states (D2, D3).

### Page 4: System Diagnostics & Network Telemetry
- Local IP address, Subnet, and Gateway.
- Target cloud server (`virtuallabs-hil.onrender.com:443`) and WebSocket link latency ($<35\text{ ms}$).
- Free SRAM heap memory with graphic progress meter and dual-core FreeRTOS task status.
- System uptime counter (`HH:MM:SS`).

### Page 5: Mobile Web Settings & Scannable QR Code
- **Scannable ISO/IEC 18004 29x29 QR Code**: Point any smartphone camera at the screen to immediately launch the web control panel (`http://192.168.4.1/settings`).
- **Interactive On-Screen Status**: Displays current Theme mode (`Dark` / `Light`), Screen orientation (`Portrait` / `Landscape`), and active hardware driver.
- **Direct Web Access**: URL prominently printed for quick manual browser entry.

### 4.6 Smartphone Gesture Engine (Single BOOT Button Navigation)
To give you total hands-on control over the physical instrument with zero touchscreens:
- **Single Click**: Advance to the **Next Screen** (`1/6` $\to$ `2/6` $\to$ `3/6` $\to$ `4/6` $\to$ `5/6` $\to$ `6/6` $\to$ `1/6`).
- **Double Click (Quick)**: **Toggle Theme** instantly (**Dark Mode** $\leftrightarrow$ **Light Mode**)!
- **Long Press (Hold 1s)**: **Toggle Orientation** (**Portrait** $\leftrightarrow$ **Landscape**)!
- **Bottom Navigation Dock**: Displays capsule indicator dots (`● ○ ○ ○ ○ ○`), active page label (`1/6 Telemetry`), and an iOS-style centered home bar.

---

## 5. Instant Event Override & Captive Portal Mode

### Instant Event Notifications (Toast Banners)
Whenever a virtual actuator state changes from the browser canvas (e.g., the user adjusts a DAC voltage or toggles an output) or an input pin transitions, DisplayOS immediately displays a floating event banner across the bottom bar (`EVENT: Actuators Updated`), guaranteeing you never miss an active event while the display is auto-cycling.

### Captive Portal On-Screen Setup
If Wi-Fi credentials are not yet configured or the router is unreachable, DisplayOS automatically switches to the **Wi-Fi Setup Portal Screen**:
1. It displays the Access Point SSID: **`VirtualLab-ESP32-Setup`**.
2. It guides the user to open browser URL: **`http://192.168.4.1/`**.
3. Once credentials are saved, it displays a success confirmation and reboots directly into normal instrument mode!

---

## 6. Software Flashing & Arduino IDE Setup

### Step 1: Install Required Libraries in Arduino IDE
Open Arduino IDE $\to$ **Tools** $\to$ **Manage Libraries...** and install:
1. `Adafruit ST7735 and ST7789 Library` (by Adafruit — required if using ST7789 driver)
2. `Adafruit ILI9341` (by Adafruit)
3. `Adafruit GFX Library` (by Adafruit)
4. `WebSocketsClient` (by Markus Sattler)
5. `ArduinoJson` (version 6.x by Benoît Blanchon)

> **Note on 80px Noise Band:** Many modern 2.8" SPI panels sold as "ILI9341" contain an **ST7789V** driver IC. Driving an ST7789 with the ILI9341 library leaves an ~80-pixel un-cleared corrupted static band on one edge. Simply keep `#define TFT_DRIVER_TYPE 1` in the firmware to use the native ST7789 driver with `tft->init(240, 320)`, which eliminates the static noise and drives 100% of the 2.8" glass.

### Step 2: Open Firmware & Upload
1. Open [`firmware/VirtualLab_ESP32/VirtualLab_ESP32.ino`](file:///c:/Users/Anand%20Shukla/Documents/VirtualLab/firmware/VirtualLab_ESP32/VirtualLab_ESP32.ino).
2. Ensure `DisplayOS.h` is present in the same sketch folder.
3. Select Board: **ESP32 Dev Module**.
4. Settings:
   - CPU Frequency: **240MHz**
   - Flash Frequency: **80MHz**
   - Upload Speed: **921600**
5. Click **Upload**.

---

## 7. Packaging into a Physical Product

To produce a deliverable, commercial-quality product:
- **Enclosure**: Use a standard 3D-printable ESP32 benchtop enclosure with a front bezel cutout for the 2.8" LCD ($50\text{mm} \times 69\text{mm}$ viewable area).
- **Terminal Connections**: Mount two 8-pin screw terminal blocks or colored female header sockets on the sides labeled `A0`–`A5`, `GND`, `3.3V`, and `DAC0`/`DAC1`.
- **Power**: Use the ESP32 onboard USB port for clean 5V benchtop power.

Your ESP32 is now a fully functional, self-contained **VirtualLab-HIL Hardware Instrument**!
