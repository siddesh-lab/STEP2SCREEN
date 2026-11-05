# ⚡ ESP32-C3 Smart Pedometer & Relay Circuit Diagram & Wiring Guide

## 📌 System Architecture Overview

The system consists of the **ESP32-C3** microcontroller connected to:
1. **ADXL345 Accelerometer**: Detects walking footfall impacts on **I2C Bus #1** (`SDA: GPIO 4`, `SCL: GPIO 5`). Footfalls feed into the 20-second walking accumulator — they do **not** directly add steps anymore.
2. **0.91" SSD1306 OLED Display (128x32)**: Shows step count (TV seconds), GPS distance/speed, walking stickman animation, motivational quotes, 20s progress bar, and relay status on **I2C Bus #1** (`SDA: GPIO 4`, `SCL: GPIO 5`).
3. **5V Relay Module**: Signal line on **GPIO 2** (`Active-LOW: LOW = ON, HIGH = OFF`) and connection sense line on **GPIO 6** (`INPUT_PULLUP: LOW = Connected to GND, HIGH = Unplugged`).
4. **NEO-6M GPS Module**: Connected via **UART1** (`TX→GPIO 0`, `RX←GPIO 1`). Tracks real-world walking distance (m/km) and speed (km/h). GPS validates steps — footfalls only count toward the accumulator when GPS speed ≥ 0.5 km/h. Falls back to ADXL345-only detection when indoors (no GPS fix).

---

## 🔌 Pin Connection Table

| Component | Component Pin | Target ESP32-C3 Pin | Wire Color | Signal / Function Description |
| :--- | :--- | :--- | :--- | :--- |
| **ADXL345 Sensor** | `VCC` | `3.3V` / `5V` | Red | Sensor Power Supply |
| **ADXL345 Sensor** | `GND` | `GND` | Black | Sensor Ground |
| **ADXL345 Sensor** | `SDA` | `GPIO 4` | Blue | I2C Bus Data |
| **ADXL345 Sensor** | `SCL` | `GPIO 5` | White | I2C Bus Clock |
| **0.91" Pedometer OLED**| `VCC` | `3.3V` / `5V` | Red | Display Power Supply |
| **0.91" Pedometer OLED**| `GND` | `GND` | Black | Display Ground |
| **0.91" Pedometer OLED**| `SDA` | `GPIO 4` | Blue | I2C Bus Data |
| **0.91" Pedometer OLED**| `SCL` | `GPIO 5` | White | I2C Bus Clock |
| **5V Relay Module** | `VCC` | `5V (VBUS)` | Red | Relay Coil Power (5V) |
| **5V Relay Module** | `GND` | `GND` | Black | Common Ground |
| **5V Relay Module** | `IN1` | `GPIO 2` | Yellow | Relay Signal Control (Active-LOW: LOW = ON, HIGH = OFF) |
| **Relay Connection Detect**| **Sense Line** | `GPIO 6` | Green | Hardware Connection Sense Pin (`LOW` = Relay Connected to GND) |
| **NEO-6M GPS Module** | `VCC` | `3.3V` | Red | GPS Power Supply |
| **NEO-6M GPS Module** | `GND` | `GND` | Black | GPS Ground |
| **NEO-6M GPS Module** | `TX` | `GPIO 0` | White | GPS NMEA Data → ESP32-C3 UART1 RX |
| **NEO-6M GPS Module** | `RX` | `GPIO 1` | Blue | ESP32-C3 UART1 TX → GPS (optional config) |

---

## 🛠️ Hardware Relay Detection Setup (GPIO 6)

To allow the ESP32-C3 to automatically detect when the Relay module is physically plugged in:
1. Connect **ESP32-C3 GPIO 6** to a **GND pin on the Relay Module / Breadboard connector**.
2. **GPIO 6** is configured with an internal pull-up (`INPUT_PULLUP`).
   - **Relay Plugged In**: GPIO 6 is pulled to `GND` (`LOW`). The ESP32-C3 detects **"Relay Connected!"** and displays `"Relay Connected!"` on the 0.91" OLED screen.
   - **Relay Unplugged**: GPIO 6 remains `HIGH`. The ESP32-C3 displays `"Plug in Relay!"` on the 0.91" OLED screen.

---

## 🔄 Operating Logic Summary

1. **Step Accumulation (1-Second Walking Engine & High Intensity)**:
   - ADXL345 configured in **High-Intensity Mode** (`ADXL345_RANGE_16_G`, dynamic scale $\pm 16\text{g}$) to capture intense footsteps and running decelerations without clipping.
   - Dynamic footfall impact threshold: `dynamicDelta > 3.0 m/s²` requiring brisk, vigorous walking footfalls (casual dragging/fidgeting rejected).
   - **1 second of walking = 1 step earned** (= 1 second of TV time).
   - Accumulator **pauses** when the user stops walking (progress is preserved, not lost).
   - GPS speed gate: footfalls only count when GPS speed ≥ 0.5 km/h (or when no GPS fix is available indoors).
   - Steps saved into Preferences NVS memory (retained across power cycles).
   - ESP32-C3 hardware RESET button resets `stepCount` to 0.

2. **GPS Location & Movement Tracking**:
   - NEO-6M GPS module streams NMEA sentences via UART1 (GPIO 0 RX) at 9600 baud.
   - `TinyGPSPlus` library parses location, speed, and satellite count in real time.
   - Accumulated walking **distance** (metres / km) is calculated between consecutive GPS fixes.
   - GPS fixes with jumps < 0.5m (noise) or > 50m (glitch) are ignored.
   - OLED alternates every **3 seconds** between step count screen and GPS distance/speed screen.
   - A filled dot appears in the top-right corner of the OLED when GPS has a valid satellite fix.
   - First fix typically takes **30–90 seconds** outdoors.

3. **OLED Display Screens**:
   - **Screen A (3s)**: Walking stickman + step count (TV seconds remaining) + speech bubble text.
   - **Screen B (3s)**: Walking stickman + GPS distance (m/km) + GPS speed (km/h) + fix dot.
   - **Progress Bar**: Bottom pixel row of OLED fills left→right showing 0→20s accumulator progress.

4. **Relay Detection & Control Logic (Active-LOW)**:
   - **When Relay is Connected (`GPIO 6 == LOW`) & `stepCount > 0`**:
     - GPIO 2 outputs `LOW` (Relay ON & IN1 LED lights up).
     - 0.91" OLED displays `"Relay Connected!"` (or motivational quote if walking).
     - 1-Second countdown decrements steps.
   - **When Relay is Connected (`GPIO 6 == LOW`) & `stepCount == 0`**:
     - GPIO 2 outputs `HIGH` (Relay OFF & IN1 LED turns off).
     - 0.91" OLED displays `"Walk 2 get steps!"`.
   - **When Relay is Unplugged (`GPIO 6 == HIGH`)**:
     - GPIO 2 outputs `HIGH` (Relay OFF).
     - 0.91" OLED displays `"Plug in Relay!"`.

