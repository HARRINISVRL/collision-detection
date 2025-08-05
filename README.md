# 🚗 Smart Collision + Alcohol Detection System (ESP8266 + MPU6050 + MQ-3)

An IoT-based real-time accident detection and alerting system using **ESP8266**, **MPU6050**, **Vibration sensor**, and **MQ-3 alcohol sensor**. This project features **sensor fusion, adaptive thresholding, severity classification**, and **ignition disable logic** to improve safety.

---

## 🔧 Features

- 🚨 **Accident Detection** using:
  - Accelerometer & Gyroscope (MPU6050)
  - Vibration Sensor
- 📊 **Severity Classification** (Light / Moderate / Severe) using G-force delta
- 🔁 **EMA (Exponential Moving Average)** baseline for adaptive spike detection
- 🍷 **Alcohol Detection** (MQ-3 sensor on A0)
- 🔒 **Ignition Disable Logic** if alcohol is detected while vehicle is idle
- 📡 **ThingSpeak Integration** for real-time sensor data logging
- 🌐 **Webhook Support** (IFTTT / Discord / Custom alert endpoints)
- 🖥️ **Rich Serial Debugging** output for testing and tuning

---

## 🧰 Hardware Components

| Component | Description |
|----------|-------------|
| ESP8266 (NodeMCU) | Wi-Fi MCU |
| MPU6050 | 3-axis Accelerometer + Gyroscope |
| MQ-3 | Alcohol Gas Sensor |
| Vibration Sensor | SW-420 or equivalent |
| Relay Module (Optional) | To disable ignition based on alcohol |
| Jumper Wires, Breadboard | Basic connections |

---

## ⚙️ Pin Configuration

| Sensor | ESP8266 Pin |
|--------|-------------|
| MPU6050 SDA | D2 |
| MPU6050 SCL | D1 |
| MPU6050 INT | D3 |
| Vibration Sensor | D4 |
| MQ-3 Analog Out | A0 |
| Ignition Disable Relay | D5 |

---

## 🔗 ThingSpeak Setup

1. Create a [ThingSpeak Channel](https://thingspeak.com)
2. Enable **Fields 1-9**:
    - Ax, Ay, Az (Accelerometer)
    - Gx, Gy, Gz (Gyroscope)
    - Vibration value
    - Alcohol level
    - Severity (encoded as numeric: 1=light, 2=moderate, 3=severe)
3. Note down:
    - Channel ID
    - Write API Key

---

## 🌐 IFTTT Webhook Setup

1. Create a webhook applet on [IFTTT](https://ifttt.com/maker_webhooks)
2. Use the webhook URL in the format:  https://maker.ifttt.com/trigger/<event>/with/key/<your_key>
3. Update this URL in the code:
   const char* webhookURL = "https://maker.ifttt.com/trigger/accident_alert/with/key/xyz123";

## 📥 Code Installation
1. Install Arduino IDE
2. Add ESP8266 Board Support via Board Manager
3. Install required libraries:
   - ESP8266WiFi
   - ThingSpeak
   - Wire
   - MPU6050 
4. Clone or download this repository: git clone https://github.com/HARRINISVRL/collision-detection.git
5. Open the .ino file in Arduino IDE.
6. Replace the following placeholders:
   const char* ssid = "YOUR_WIFI_NAME";
   const char* password = "YOUR_WIFI_PASSWORD";
   unsigned long channelID = YOUR_THINGSPEAK_CHANNEL_ID;
   const char* writeAPIKey = "YOUR_THINGSPEAK_WRITE_API_KEY";

## 🛡️ Safety Logic   
  | Condition                             | Action                     |
| ------------------------------------- | -------------------------- |
| 🚗 High G-force + Vibration detected  | Accident Confirmed         |
| 🍺 Alcohol > Threshold + Vehicle Idle | Disable Ignition via Relay |
| 🧠 Cooldown period                    | Prevents repeated triggers |

## 📊 ThingSpeak Data Mapping
| Field  | Data                                           |
| ------ | ---------------------------------------------- |
| Field1 | Accel X                                        |
| Field2 | Accel Y                                        |
| Field3 | Accel Z                                        |
| Field4 | Gyro X                                         |
| Field5 | Gyro Y                                         |
| Field6 | Vibration (0/1)                                |
| Field7 | Alcohol (MQ-3 analog value)                    |
| Field8 | Severity (1 = light, 2 = moderate, 3 = severe) |

## 📜 License
This project is open-source under the MIT License. Feel free to use, modify, and share with attribution.

## 📫 Contact

For questions, collaboration, or feedback:

Open an issue on GitHub

Or contact the developer via email (rlharrini@gmail.com) or via LinkedIn (www.linkedin.com/in/harrinirl26)
---
