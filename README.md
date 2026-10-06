# 🌐 Multi-Node IoT Environmental Monitoring System

[![Platform](https://img.shields.io/badge/Platform-ESP32-00599C.svg?logo=espressif&logoColor=white)](https://www.espressif.com/)
[![Protocol](https://img.shields.io/badge/Protocol-ESP--NOW-E7352C.svg)](https://www.espressif.com/en/solutions/low-power-solutions/esp-now)
[![Language](https://img.shields.io/badge/Language-C%2B%2B%20%2F%20Arduino-00979D.svg?logo=cplusplus&logoColor=white)](https://isocpp.org/)
[![Sensors](https://img.shields.io/badge/Sensors-AHT20%20%7C%20LDR-4CAF50.svg)]()
[![Frontend](https://img.shields.io/badge/UI-HTML5%20%2F%20CSS3%20%2F%20JS-E34F26.svg?logo=html5&logoColor=white)](UI_Interface.html)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A distributed, multi-node environmental monitoring ecosystem built on **ESP32 microcontrollers**. The system acquires environmental telemetry across four sender nodes using Espressif's connectionless **ESP-NOW** protocol at 5-second sampling intervals, transmitting the data to a central gateway receiver node that hosts an asynchronous real-time web dashboard.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [System Architecture](#-system-architecture)
- [Implementation Diagram](#-implementation-diagram)
- [Node Configuration](#-node-configuration)
- [Hardware & Pin Configuration](#-hardware--pin-configuration)
- [Network Protocol & Packet Structure](#-network-protocol--packet-structure)
- [Web Dashboard UI](#-web-dashboard-ui)
- [Repository Structure](#-repository-structure)
- [Getting Started](#-getting-started)
  - [Prerequisites & Libraries](#1-prerequisites--libraries)
  - [Hardware Setup](#2-hardware-setup)
  - [Configuration & Flashing](#3-configuration--flashing)
- [Future Enhancements](#-future-enhancements)
- [License](#-license)

---

## 📖 Overview

This project implements an end-to-end telemetry pipeline designed to address real-time monitoring across distinct physical zones without relying on traditional router-dependent Wi-Fi handshakes for peripheral communication:

* **Edge Sensing:** Dedicated ESP32 nodes capture digital temperature, relative humidity, and ambient light intensity.
* **Low-Power Mesh Transport:** Uses **ESP-NOW** (a fast, connectionless protocol operating at 2.4 GHz) to minimize transmission latency and power consumption.
* **Central Gateway & Dashboard:** A dedicated receiver ESP32 acts as a local web server (`ESPAsyncWebServer`), dynamically synchronizing Wi-Fi channels, parsing incoming JSON telemetry packets, and streaming live updates to a responsive browser interface.

---

## 🏛️ System Architecture

```text
 ┌────────────────────────┐
 │  Sender Node 1 (ID: 1) │ (AHT20 Temp/Humidity + LDR Light)
 └───────────┬────────────┘
             │
 ┌───────────┴────────────┐
 │  Sender Node 2 (ID: 2) │ (AHT20 Temp/Humidity + LDR Light)
 └───────────┬────────────┘
             │             ESP-NOW Broadcast (2.4 GHz)
             ├─────────────────────────────────────────────────┐
             │                                                 ▼
 ┌───────────┴────────────┐                        ┌───────────────────────┐
 │  Sender Node 3 (ID: 3) │ (AHT20 Temp only)      │  ESP32 Gateway Node   │
 └───────────┬────────────┘                        │  (Receiver / Server)  │
             │                                     └───────────┬───────────┘
 ┌───────────┴────────────┐                                    │ ESPAsyncWebServer
 │  Sender Node 4 (ID: 4) │ (AHT20 Humidity + LDR Light)       ▼
 └────────────────────────┘                         ┌─────────────────────┐
                                                    │ Real-Time Dashboard │
                                                    │ (HTTP / JSON / SSE) │
                                                    └─────────────────────┘
```

---

## 📐 Implementation Diagram

![Implementation Diagram](https://github.com/user-attachments/assets/7efce2a9-8a80-4eec-a2a6-a30af7ef20b6)

---

## 🧩 Node Configuration

| Node | Board ID | Sensor Hardware | Metrics Tracked | Interval |
| :--- | :---: | :--- | :--- | :---: |
| **Sender Node 1** | `1` | DFRobot AHT20 + LDR | Temperature (°C), Humidity (%), Light Intensity | 5s |
| **Sender Node 2** | `2` | DFRobot AHT20 + LDR | Temperature (°C), Humidity (%), Light Intensity | 5s |
| **Sender Node 3** | `3` | DFRobot AHT20 | Temperature (°C) | 5s |
| **Sender Node 4** | `4` | DFRobot AHT20 + LDR | Humidity (%), Light Intensity | 5s |
| **Receiver Node** | Gateway | ESP32 Receiver | Ingestion, JSON serialization, Web Server | Real-Time |

---

## 🛠️ Hardware & Pin Configuration

| Component | Interface / Pin | Description |
| :--- | :--- | :--- |
| **ESP32 DevKit V1 (x5)** | Controller Boards | Dual-core Tensilica LX6, 240 MHz, 2.4 GHz radio |
| **DFRobot AHT20** | I2C (`SDA: GPIO 21`, `SCL: GPIO 22`) | High-accuracy digital temperature & humidity sensor |
| **Photoresistor (LDR)** | ADC1 (`GPIO 36` / VP) | 12-bit analog-to-digital ambient light intensity converter |
| **Pull-down Resistor** | 10kΩ Resistor | Voltage divider circuit for analog LDR light sensing |

---

## 📡 Network Protocol & Packet Structure

Telemetry data packets are encapsulated in a packed C-structure across all nodes to avoid memory alignment and serialization mismatches:

```cpp
typedef struct __attribute__((packed)) struct_message {
    int id;             // Sender Node ID (1, 2, 3, or 4)
    float temperature;  // Ambient temperature in Celsius
    float humidity;     // Relative humidity in percentage (%)
    int light;          // Ambient light level (ADC value 0–4095)
    int readingId;      // Incremental packet transmission sequence ID
} struct_message;
```

### Channel Synchronization
ESP-NOW operates over the underlying 802.11 physical layer. Senders dynamically scan local Wi-Fi channels to match the active operating channel of the receiver node before registering the peer MAC address:

```cpp
int32_t getWiFiChannel(const char *ssid) {
    if (int32_t n = WiFi.scanNetworks()) {
        for (uint8_t i = 0; i < n; i++) {
            if (!strcmp(ssid, WiFi.SSID(i).c_str())) {
                return WiFi.channel(i);
            }
        }
    }
    return 0;
}
```

---

## 🖥️ Web Dashboard UI

The receiver node serves a responsive, glassmorphism-styled dashboard. The interface color-codes metric cards, displays real-time telemetry updates per node, and validates network connection status.

![UI Dashboard](https://github.com/user-attachments/assets/1deae4e3-aada-4da9-955e-438ac5c6a8de)

---

## 📁 Repository Structure

```text
├── Final_Dashboard.ino       # Main receiver firmware with ESPAsyncWebServer & UI
├── sketch_RecieverCode.ino   # Baseline headless receiver firmware
├── Sender_Node1.ino          # Firmware for Sender Node 1 (ID: 1)
├── Sender_Node2.ino          # Firmware for Sender Node 2 (ID: 2)
├── Sender_Node3.ino          # Firmware for Sender Node 3 (ID: 3)
├── Sender_Node4.ino          # Firmware for Sender Node 4 (ID: 4)
├── UI_Interface.html         # Standalone HTML/CSS/JS frontend dashboard template
├── README.md                 # Project documentation and setup guide
└── LICENSE                   # MIT License
```

---

## ⚡ Getting Started

### 1. Prerequisites & Libraries

Install the [Arduino IDE](https://www.arduino.cc/en/software) (or PlatformIO in VS Code) and add the ESP32 board package via the Boards Manager:
```text
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Install the required libraries:
* **ESPAsyncWebServer** (`me-no-dev/ESPAsyncWebServer`)
* **AsyncTCP** (`me-no-dev/AsyncTCP`)
* **Arduino_JSON** (bblanchon / Arduino official)
* **DFRobot_AHT20** (DFRobot)

### 2. Hardware Setup
1. Wire each **AHT20** sensor to the ESP32 using the default I2C pins (`SDA` $\rightarrow$ `GPIO 21`, `SCL` $\rightarrow$ `GPIO 22`, `VCC` $\rightarrow$ `3.3V`, `GND` $\rightarrow$ `GND`).
2. Build a voltage divider with the **LDR photoresistor** and a 10kΩ resistor, connecting the analog output to `GPIO 36` (VP).
3. Connect all five ESP32 boards via micro-USB.

### 3. Configuration & Flashing

1. **Obtain the Receiver MAC Address:**
   * Flash a simple sketch or run `WiFi.macAddress()` on your gateway ESP32.
   * Note the MAC address (e.g., `A0:B7:65:25:D9:68`).

2. **Configure Sender Nodes:**
   * Open `Sender_Node1.ino` through `Sender_Node4.ino`.
   * Update the receiver MAC address array:
     ```cpp
     uint8_t broadcastAddress[] = {0xA0, 0xB7, 0x65, 0x25, 0xD9, 0x68};
     ```
   * Set your local Wi-Fi SSID for channel discovery:
     ```cpp
     constexpr char WIFI_SSID[] = "YOUR_WIFI_SSID";
     ```
   * Flash each sender script to its designated board.

3. **Configure & Flash the Receiver Node:**
   * Open `Final_Dashboard.ino`.
   * Enter your Wi-Fi credentials for the local network station:
     ```cpp
     const char* ssid = "YOUR_WIFI_SSID";
     const char* password = "YOUR_WIFI_PASSWORD";
     ```
   * Flash `Final_Dashboard.ino` to the gateway ESP32.
   * Open the Serial Monitor at `115200 baud` to view the assigned local IP address.
   * Open the IP address in your browser on any local device to view the live dashboard.

---

## 🔮 Future Enhancements

- [ ] **Data Persistence:** Integrate SPIFFS/LittleFS or an SD card module for offline logging.
- [ ] **MQTT / Home Assistant Integration:** Forward incoming telemetry over MQTT to integrate into a centralized smart home setup.
- [ ] **Telemetry Visualizations:** Add Chart.js to graph historical temperature and humidity fluctuations.
- [ ] **Deep Sleep Optimization:** Implement timed ESP32 deep-sleep states to run sender nodes on LiPo batteries for several months.

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
