# ✈️ Wireless Avionics Bus for Flight Simulators (ESP-NOW Bridge)

**High-Performance Wireless Cockpit Interface for MSFS2020 (Fenix A320)**

This project implements a low-latency, distributed wireless architecture for home cockpit builders. It bridges **ESP32-C6 nodes** (handling hardware switches/lights) to a central **ESP32-S3 Gateway**, which communicates with the PC (Node-RED/FSUIPC) via high-speed SLIP Serial.

> **Goal:** Achieve industrial-grade reliability with < 4ms latency wirelessly.

---

## 📊 Performance Benchmark: The "Journey to 3ms"

We conducted a rigorous series of tests to optimize the Round Trip Time (RTT) between the ESP32-S3 (Bridge) and ESP32-C6 (Node). Below is the log of how we reduced latency by **63%**.

| Test Phase | Strategy / Configuration | Average RTT (µs) | Status |
| :--- | :--- | :--- | :--- |
| **Phase 1** | **Stock Settings.** Default ESP-NOW config. WiFi Power Saving enabled by default. | **8,250 µs** (8.2ms) | 🔴 Slow |
| **Phase 2** | **Power Saving Disabled.** `esp_wifi_set_ps(WIFI_PS_NONE)`. Radio forced ON. | **8,230 µs** (8.2ms) | 🔴 No Change |
| **Phase 3** | **Channel Locking.** Forced Channel 1 on both ends to avoid scanning delays. | **8,000 µs** (8.0ms) | 🔴 No Change |
| **Phase 4** | **Force Mode & Protocol.** `WIFI_AP_STA` mode active. Disabled 802.11b (1Mbps). Forced 802.11g/n. | **3,055 µs** (3.0ms) | 🟢 **Target Hit** |

### 🏆 Final Result
* **Stability:** ±30 µs jitter.
* **Throughput:** 330 Hz update rate (Bi-directional).
* **Hardware:** ESP32-S3 (240MHz) <-> ESP32-C6 (160MHz).

---

## 🧮 Theoretical Analysis: The Physics of Latency

Why was the stock configuration stuck at 8ms? It wasn't a processing issue; it was a protocol negotiation issue.

### 1. The "Long Range" Trap (802.11b)
By default, ESP-NOW may negotiate the most robust modulation (1 Mbps / 802.11b) to ensure connectivity.

* **Packet Size:** ~200 bytes (Header + Payload).
* **Transmission Speed:** 1,000,000 bits/sec.
* **Math:** $(200 \text{ bytes} \times 8) / 1\text{Mbps} \approx 1,600 \mu s$ (1.6 ms).
* **Round Trip:** $1.6 \text{ms (Tx)} + 1.6 \text{ms (Rx)} = 3.2 \text{ms}$.
* **Overhead:** Adding WiFi preambles, SIFS, ACK wait times, and backoff slots, the physical airtime aligns perfectly with the **8ms** observed in Phase 1.

### 2. The Optimized Protocol (802.11g/n)
By explicitly banning the 802.11b protocol via `esp_wifi_set_protocol`, we force OFDM modulation (min 6 Mbps).

* **Transmission Speed:** > 6,000,000 bits/sec.
* **Math:** $(1,600 \text{ bits}) / 6\text{Mbps} \approx 266 \mu s$ (0.26 ms).
* **Round Trip:** $< 0.6 \text{ms}$.

### 3. The "Floor" (3ms)
If the physical transmission takes < 1ms, why is the result 3ms?
The remaining **2,000 µs** represents the **OS Overhead**. The ESP32's FreeRTOS kernel must:
1.  Interrupt the User Task.
2.  Switch context to the WiFi Stack.
3.  Copy memory buffers.
4.  Execute the Callback.
5.  Switch context back to User Task.

**Conclusion:** 3ms is the practical limit of the ESP-IDF framework without writing bare-metal assembly. This is sufficient for real-time simulation avionics.

---

## 🛠️ Optimization Strategies Used

To replicate these results in your project, apply these critical configurations in your `setup()`:

### 1. Force CPU Frequency
Ensure the S3 is not idling at 80MHz.
```cpp
setCpuFrequencyMhz(240); // Force 240MHz on ESP32-S3
