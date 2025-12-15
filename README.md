# Wireless Avionics Bus: High-Performance ESP-NOW Bridge

**An optimized, low-latency wireless bridge for Home Cockpit Simulators (MSFS2020/Fenix A320).**

This project implements a wireless star topology using **ESP32-S3** as a central Gateway (Bridge) and **ESP32-C6** as peripheral Nodes. It achieves a stable **3ms Round-Trip Time (RTT)**, enabling real-time avionics control without cables.

![Star Network Topology](images/ilustration.png)

## 1. The Challenge

In professional flight simulation, switch latency must be imperceptible. While building a distributed cockpit system, we faced a critical bottleneck: the default ESP-NOW configuration resulted in a **latency of ~8ms (8200µs)** per transaction.

While 8ms is acceptable for temperature sensors, it is noticeable for momentary switches and rotary encoders in a cockpit environment.

**Hardware Setup:**
* **Bridge:** ESP32-S3 (Dual Core, 240MHz) handling USB SLIP + ESP-NOW.
* **Nodes:** ESP32-C6 (RISC-V, 160MHz) handling hardware I/O.
* **Protocol:** Custom Half-Duplex Request/Response.

---

## 2. Theoretical Analysis: The "8ms Barrier"

Why was the system stuck at 8ms? The issue wasn't processing power, but **Physics and Protocols**.

By default, ESP-NOW and the ESP32 Wi-Fi stack prioritize range and compatibility. When an ESP32-C6 (Wi-Fi 6) talks to an ESP32-S3 (Wi-Fi 4), they often negotiate the lowest common denominator: **802.11b (1 Mbps)**.

**The Math of "Slow" Wi-Fi:**
* **Packet Size:** ~200 bytes (Structure overhead).
* **Speed:** 1 Mbps (1,000,000 bits/sec).
* **Transmission Time (One way):** $\approx 1.7ms$ + Preamble/Headers $\approx 2.5ms$.

**Round Trip Calculation:**
$$T_{total} = T_{tx\_req} (2.5ms) + T_{process} + T_{tx\_ack} (2.5ms) + Backoff \approx 8ms$$

To break this barrier, we had to force the physics of the transmission to change.

![Timing Diagram](images/ilutiming_diagram.png)
*(Add a diagram showing the transmit/receive cycle here)*

---

## 3. Optimization Strategy & Results

We applied a 4-stage engineering approach to reduce latency. Below is the progression of the results.

![Latency Reduction Graph](images/latency_chart.png)
*(Add a graph of your results: 8.2ms -> 8.0ms -> 3.2ms -> 3.0ms)*

### Stage 1: Baseline (Default Settings)
* **Configuration:** Standard `WiFi.mode(WIFI_STA)`.
* **Result:** `8236 µs` (Avg).
* **Observation:** High jitter. The radio frequently enters "Modem Sleep" to save power, adding wake-up delays.

### Stage 2: Power Management Fix
* **Action:** Forced `esp_wifi_set_ps(WIFI_PS_NONE)` and switched to `WIFI_AP_STA` mode to keep the hardware radio active.
* **Result:** `8011 µs`.
* **Observation:** The jitter disappeared (results became stable), but the floor remained at 8ms. The "Sleep" was fixed, but the transmission speed was still the bottleneck.

### Stage 3: Protocol Optimization (The Breakthrough)
* **Action 1 (Physics):** Disabled Long Range/Legacy protocols.
    ```cpp
    // Ban 802.11b (1 Mbps). Force OFDM (6 Mbps+).
    esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_11G | WIFI_PROTOCOL_11N);
    ```
* **Action 2 (Payload):** Dynamic payload sizing. Instead of sending full 200-byte structs for heartbeats, we reduced the packet to **1 byte** for latency checks.
* **Result:** `3250 µs`.
* **Improvement:** **~60% reduction**. We moved from 1 Mbps to >6 Mbps, reducing airtime from ~2.5ms to ~0.3ms.

### Stage 4: "The Floor" (CPU & Channel Tuning)
* **Action:** Locked both devices to **Channel 1** (physically) to prevent scanning delays and forced CPU to **240MHz** (S3).
* **Final Result:** `3055 µs` (Stable).

---

## 4. Final Performance Comparison

| Metric | Standard ESP-NOW | Optimized Wireless Bus |
| :--- | :--- | :--- |
| **Protocol** | 802.11b / Auto | **802.11g/n (Forced)** |
| **Power Mode** | Modem Sleep | **No Sleep (Always On)** |
| **Air Time (Est)** | ~5.0 ms | **~0.6 ms** |
| **Total RTT** | **~8.2 ms** | **~3.0 ms** |
| **Throughput** | Low | High |

The remaining 3ms consists almost entirely of **FreeRTOS context switching** and hardware interrupt handling, which is the theoretical limit for the ESP32 Arduino framework.

---

## 5. Implementation Snippet

To replicate these results, ensure your initialization sequence disables power saving *after* Wi-Fi init and strictly defines the protocol.

```cpp
void begin(const uint8_t* targetMac) {
    // 1. Force Dual Mode to keep radio Alive
    WiFi.mode(WIFI_AP_STA);
    
    // 2. CRITICAL: Ban 802.11b to prevent 1Mbps negotiation
    esp_wifi_set_protocol(WIFI_IF_STA, WIFI_PROTOCOL_11G | WIFI_PROTOCOL_11N);
    esp_wifi_set_protocol(WIFI_IF_AP,  WIFI_PROTOCOL_11G | WIFI_PROTOCOL_11N);

    // 3. Force Channel 1 (Prevent scanning)
    esp_wifi_set_promiscuous(true);
    esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);
    esp_wifi_set_promiscuous(false);

    // 4. Brute Force Power - NO SLEEP
    WiFi.setSleep(false);
    esp_wifi_set_ps(WIFI_PS_NONE);
    
    // ... Initialize ESP-NOW ...
}
