# 🔌 VirtualLab-HIL — Hardware-in-the-Loop (ESP32) Bridge Guide
**Real-Time Bi-Directional Streaming Between Physical Microcontrollers & Virtual Schematics**

---

## 1. Overview & Architecture

VirtualLab-HIL bridges physical microcontrollers (ESP32, ESP32-S3, Arduino) with simulated circuits running in the browser. 

```
┌─────────────────────────┐          WSS (TLS)          ┌──────────────────────────┐
│   ESP32 Microcontroller ├────────────────────────────►│  FastAPI Cloud Gateway   │
│   (FreeRTOS Dual-Core)  │◄────────────────────────────┤  (Render / WebSocket)    │
└─────────────────────────┘      <35ms Latency          └─────────────┬────────────┘
                                                                      │ WSS (/ws/ui)
                                                                      ▼
                                                        ┌──────────────────────────┐
                                                        │ Browser VirtualLab-HIL   │
                                                        │ (Zero-Order Hold MNA)    │
                                                        └──────────────────────────┘
```

- **Core 0 (TaskNetwork)**: Wi-Fi STA/AP mode, mDNS, captive portal web server, and secure WebSocket TLS stream.
- **Core 1 (TaskHardware)**: High-speed 100 Hz concurrent ADC sampling and DAC/PWM actuation.
- **Zero-Order Hold (ZOH)**: Latches physical hardware samples across all 16 simulation sub-steps per frame, ensuring jitter-free continuous solver evaluation.

---

## 2. ESP32 Hardware Pinout Reference

| Port Name | ESP32 GPIO | Direction | Description & Characteristics |
|---|---|---|---|
| **A0** | GPIO 36 (VP) | Ingress (Input) | 12-bit ADC Input (0–3.3V) |
| **A1** | GPIO 39 (VN) | Ingress (Input) | 12-bit ADC Input (0–3.3V) |
| **A2** | GPIO 34 | Ingress (Input) | 12-bit ADC Input (0–3.3V) |
| **A3** | GPIO 35 | Ingress (Input) | 12-bit ADC Input (0–3.3V) |
| **A4** | GPIO 32 | Ingress (Input) | 12-bit ADC Input (0–3.3V) |
| **A5** | GPIO 33 | Ingress (Input) | 12-bit ADC Input (0–3.3V) |
| **D0** | GPIO 13 | Ingress (Input) | Digital Input with internal pulldown |
| **D1** | GPIO 14 | Ingress (Input) | Digital Input with internal pulldown |
| **D4** | GPIO 16 | Ingress (Input) | Digital Input with internal pulldown |
| **D5** | GPIO 17 | Ingress (Input) | Digital Input with internal pulldown |
| **D6** | GPIO 4 | Ingress (Input) | Digital Input with internal pulldown |
| **DAC0** | GPIO 25 | Egress (Output) | True 8-bit Hardware DAC1 (0–3.3V) |
| **DAC1** | GPIO 26 | Egress (Output) | True 8-bit Hardware DAC2 (0–3.3V) |
| **PWM0** | GPIO 18 | Egress (Output) | High-speed Hardware PWM (LEDC) |
| **PWM1** | GPIO 19 | Egress (Output) | High-speed Hardware PWM (LEDC) |
| **PWM2** | GPIO 21 | Egress (Output) | High-speed Hardware PWM (LEDC) |
| **PWM3** | GPIO 22 | Egress (Output) | High-speed Hardware PWM (LEDC) |
| **D2** | GPIO 2 | Egress (Output) | Built-in Blue LED / Digital GPIO |
| **D3** | GPIO 23 | Egress (Output) | Digital Output GPIO |
| **D7** | GPIO 27 | Egress (Output) | Digital Output GPIO |

### 2.1 Standalone Deliverable Device Option (2.8" TFT Color LCD + DisplayOS)
For standalone, benchtop packaging as a commercial-grade deliverable hardware instrument, connect a **2.8-inch ILI9341 SPI color TFT LCD (240x320)** to run **DisplayOS** (featuring auto-cycling telemetry dashboards, real-time mini oscilloscope, and smart pin auto-detection). 

See the dedicated [🖥️ Deliverable Device & TFT Mini OS Guide](DELIVERABLE_DEVICE_TFT_GUIDE.md) for full wiring tables, FreeRTOS graphics engine details, and packaging instructions.

---

## 3. Firmware Flashing & Zero-Config Setup

### Step 1: Open Firmware in Arduino IDE
1. Open the project firmware at `firmware/esp32_hil_bridge/esp32_hil_bridge.ino`.
2. Select your board: **ESP32 Dev Module**.
3. Install required libraries: `WebSocketsClient`, `Preferences`, `WebServer`.
4. Click **Upload**.

### Step 2: Onboard Wi-Fi Captive Portal (No Re-flashing Needed!)
1. If Wi-Fi credentials are not found, the ESP32 automatically launches an Access Point named:
   ```
   VirtualLab-ESP32-Setup
   ```
2. Connect to this Wi-Fi network from your phone or computer.
3. Your browser automatically opens the configuration portal at `http://192.168.4.1`.
4. Enter your local Wi-Fi SSID, Password, and Device ID (e.g. `esp32_lab_01`), and click **Save & Connect**.
5. The ESP32 saves credentials to non-volatile flash memory and connects to the VirtualLab cloud gateway.

---

## 4. Hardware-in-the-Loop Lab Workflow

### Step 1: Place HIL Components on Canvas
- In the Component Palette, expand the **Hardware-in-the-Loop (ESP32)** category.
- Drag a **HIL Ingress** component onto the canvas and select pin `A0`.
- Drag a **HIL Egress** component and select pin `DAC0`.

### Step 2: Establish Bridge Connection
1. In the top navigation bar, click the **`HIL`** button.
2. Confirm the gateway URL is set to `wss://virtuallabs-hil.onrender.com/ws/ui`.
3. When connected, the indicator in the top navbar flashes green: `ESP32 Hardware Online`.

### Step 3: Closed-Loop Physical Experiment
- Connect a physical potentiometer on your breadboard between 3.3V and GND, with the wiper connected to ESP32 pin **GPIO 36 (A0)**.
- In VirtualLab, wire `hil_ingress:A0` to an analog Op-Amp amplifier.
- Wire the Op-Amp output to `hil_egress:DAC0`.
- Wire ESP32 pin **GPIO 25 (DAC0)** to a physical oscilloscope or multimeter on your desk.
- As you turn the physical potentiometer knob, watch the virtual schematic react in real time, and observe the amplified voltage driven back to your physical benchtop instruments!
