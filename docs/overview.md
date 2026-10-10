# NetTwin — Technical Overview

**A Digital Twin Architecture for Intelligent, Fault-Tolerant IoT Networks**

## Context

NetTwin is a Digital Twin architecture for IoT networks developed as a 2nd-year Higher Cycle (2CS) engineering project at the Higher School of Computer Science (ESI-SBA), Algeria.

The system addresses three fundamental weaknesses in traditional IoT infrastructures:
- **No fault tolerance** — sensor failures require manual intervention
- **No unified visibility** — devices, switches, and controllers are managed separately
- **No intelligent defence** — attacks are detected too late, after damage is done

NetTwin moves IoT networks from **passive monitoring** to **active network resilience**.

## Architecture — 4 Layers

### Layer 1 — Physical
- ESP8266 nodes (RFID reader, DHT11 temperature/humidity sensor)
- Raspberry Pi 3 (edge computing node)
- Raspberry Pi 3 (camera module)
- Raspberry Pi 4 (Open vSwitch, primary platform)

### Layer 2 — Data
- **MQTT Bridge** — real-time telemetry backbone
- **InfluxDB** — time-series storage
- **Grafana** — real-time visualization and alerting

### Layer 3 — Application
- **SDN Proxy** — application-layer enforcement (3-step pipeline: source validation → VLAN policy matrix → data abstraction)
- **Simulation Engine** — what-if analysis for proactive decision making
- **Shared State** — central coordination layer

### Layer 4 — Intelligence
- **AI Engine** — Random Forest intrusion detection
- **Health Prediction** — failure prediction before it happens
- **SDN Controller (Ryu)** — network-layer flow control

## Network Topology

- **SDN Controller** (Ryu) → **OVS Switch** → **MQTT Broker** → **Modem/Router**
- **VLAN 30 (IoT Zone)** — ESP8266 RFID, ESP8266 DHT11, RPi 3 Camera Node (192.168.1.30)
- **VLAN 20 (Guest Zone)** — Phone, PC

## Three-Zone VLAN Security Model

| Zone | VLAN | Subnet | Purpose |
|---|---|---|---|
| Management | VLAN 10 | 10.10.10.0/24 | Control plane only |
| Guest / Unknown | VLAN 20 | 10.20.20.0/24 | Auto-assigned to new devices |
| IoT Devices | VLAN 30 | 10.30.30.0/24 | MQTT publishing only |

**Two independent enforcement layers:**
- **OpenFlow drop rules** (network layer)
- **SDN Proxy policy matrix** (application layer)

## MQTT Bridge — Real-Time Data Backbone

### Input Topics
- `smartguard/env` — temperature, humidity, LED state (every 5s)
- `smartguard/rfid` — access control events (tag ID, decision)
- `nettwin/device_ip` — dynamic device registration on boot

### Fan-Out (4 simultaneous consumers)
- AI Engine
- InfluxDB
- Grafana
- Shared State

### Reverse Actuation Path
The Bridge can publish commands back to ESP32 devices (toggle LED, adjust threshold, hardware reset).

## SDN Proxy — Application-Layer Enforcement

Every request passes through 3 sequential steps:

1. **Source Validation** — IP checked against blacklist → 403 + audit log if flagged
2. **VLAN Policy Matrix** — source zone vs target zone checked → ZONE_VIOLATION if not permitted
3. **Data Abstraction** — proxied to backend; clients never reach InfluxDB/OVS directly

**Average overhead: +2.3ms per request. Blocked requests: 4.2ms (early exit).**

## Edge Camera Subsystem

### Hardware
- Raspberry Pi 3 Model B + Camera Module v2
- Static IP: 192.168.1.30

### Software Dependencies
- `picamera2` — hardware camera interface
- `opencv-python` — image processing
- `Flask` — API and streaming endpoint
- `ffmpeg` — H.264 transcoding

### Real-Time Video Streaming
- **MJPEG endpoint** (`/video`) — on-demand, 320×240 @ 50% JPEG quality
- **Continuous push** (`push_frames`) — 1 frame/second to Admin PC at 192.168.1.100

### Motion Detection Pipeline
1. Grayscale conversion
2. Gaussian smoothing (21×21 kernel)
3. Absolute frame differencing: `D(x,y) = |F_t(x,y) - F_{t-1}(x,y)|`
4. Binary thresholding (Δ ≥ 25 → 255, otherwise 0)
5. Density check: ratio ≥ 2% → motion trigger

### Cooldown State Machine
- **DETECTED** — triggered immediately, fires alert
- **30s Cooldown** — prevents packet floods
- **CLEAR** — restored after 30s of stillness

### Decentralized Storage & DVR
- 5-minute segments at 10 FPS (XVID raw → H.264 MP4 via ffmpeg)
- Self-healing disk watchdog: keeps 5 GB free, FIFO deletion of oldest clips

### Camera Node REST API
| Route | Method | Description |
|---|---|---|
| `/video` | GET | Live MJPEG stream |
| `/motion` | GET | Current state (DETECTED/CLEAR) |
| `/motionlog` | GET | Last 100 sessions |
| `/motionlog` | DELETE | Reset log |
| `/recordings` | GET | List all MP4 clips |
| `/recordings/<file>` | GET | Stream specific clip |
| `/recordings/<file>` | DELETE | Remove clip |
| `/recordings` | DELETE | Purge all clips |

## AI-Based Threat Detection

- **Dataset:** NSL-KDD (41 features)
- **Model:** Random Forest Classifier
- **Classes:** Normal, DDoS, Probe, R2L, U2R

### Detection Rates (10-fold cross-validation)
| Class | Detection | FP Rate |
|---|---|---|
| Normal | 97.2% | 1.8% |
| DDoS | 98.1% | 0.9% |
| Probe | 96.0% | 2.1% |
| R2L | 94.3% | 2.8% |
| U2R | 92.7% | 3.2% |
| **Overall** | **96.4%** | **1.2%** |

**End-to-end response time: 25–35 ms**

## Health Prediction Engine

- **Model:** Random Forest Predictor
- **Input:** IoT telemetry + system metrics
- **Output:** Risk score (0.0–1.0) → Normal / Warning / Critical

### Accuracy
| Condition | Accuracy |
|---|---|
| Normal identified | 99.1% |
| Warning detected | 93.8% |
| Critical detected | 91.4% |
| False Positive Rate | 1.2% |

## Simulation Engine

5 scenarios validated end-to-end:
- AP / Sensor Failure
- Switch Downtime
- Sensor Flooding
- Unauthorised Access
- Bandwidth Saturation

Injected into shared state — indistinguishable from real events. Outputs: fault-tolerant path, recovery strategy, auto-remediation.

## Benchmarks

### SDN Proxy Latency Overhead
| Request Type | Without | With | Overhead |
|---|---|---|---|
| Normal GET | 6.8ms | 9.1ms | +2.3ms |
| VLAN Policy Check | 10.1ms | 12.4ms | +2.3ms |
| Blocked Request | N/A | 4.2ms | Fast |
| Audit Log Write | 0ms | 1.1ms | +1.1ms |
| **Average** | | | **+2.3ms** |

### Security Coverage
| Metric | Without Proxy | With Proxy |
|---|---|---|
| Cross-VLAN Blocking | 0% | 100% |
| API Call Rejection | ~15% | 100% |
| Audit Coverage | 0% | 100% |

## Objectives Achieved (5/5)

- ✅ **SDN Segmentation** — Three-zone VLAN model, zero breaches
- ✅ **Real-Time Digital Twin** — Sub-second latency via MQTT
- ✅ **AI Threat Detection** — 96.4% accuracy, <35ms response
- ✅ **Health Prediction** — 91.4–99.1% accuracy
- ✅ **Simulation Engine** — 5 scenarios validated on real system

## Technology Stack

- **SDN:** Ryu Framework, OpenFlow
- **Virtual Switching:** Open vSwitch
- **IoT Transport:** MQTT / Mosquitto
- **Machine Learning:** scikit-learn (Random Forest)
- **Time-Series DB:** InfluxDB
- **Visualization:** Grafana
- **API Layer:** Python / Flask
- **Compute:** Raspberry Pi 4
- **IoT Nodes:** ESP32 / ESP8266
- **Edge Vision:** picamera2, OpenCV, ffmpeg

## Future Development Roadmap

### Phase 1 — Scalability
- Multi-node RPi Cluster
- Hardware 802.1Q VLAN
- MQTT Load Balancing
- InfluxDB Clustering

### Phase 2 — Adaptive AI
- LSTM / Autoencoder models
- Online Learning
- Federated Learning
- SHAP Explainable AI

### Phase 3 — Enterprise
- Cloud Digital Twin
- 5G / NB-IoT Support
- Twin Federation
- ISO 27001 Reporting

## Conclusion

NetTwin demonstrates that enterprise-grade network intelligence is achievable on commodity hardware — not in theory, but in a live, tested system.

**From passive monitoring → to active network resilience.**

---

*Full report available upon request.*
