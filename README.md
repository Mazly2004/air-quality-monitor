# Air Quality Monitoring System (AQI_Project_Sprint)

**Last Updated:** March 2026  
**Type:** Embedded IoT + Full-Stack Data Pipeline  
**Hardware:** ESP32 + ZPHS01B Multi-Gas Sensor  
**Infrastructure:** Dual Raspberry Pi HA cluster + Optional Research Server

---

## 1. Project Overview

This is an end-to-end **Air Quality Monitoring (AQM)** system that:
1. Reads environmental data from a **ZPHS01B multi-gas sensor** via an **ESP32 microcontroller**
2. Transmits data over **WiFi → MQTT** to a **Raspberry Pi server cluster**
3. Stores the data in **InfluxDB** (primary pipeline) and optionally **TimescaleDB** (research comparison)
4. Visualises it in a **Grafana** dashboard

---

## 2. System Architecture

### High-Level Data Flow

```
[ESP32 + ZPHS01B Sensor]
        │
        │  WiFi / MQTT (JSON payload)
        ▼
[Mosquitto MQTT Broker]  ←── Primary: Pi 1 (172.20.10.10)
        │                ←── Backup:  Pi 2 (172.20.10.11)  (automatic failover)
        ▼
[Telegraf]  (data bridge, subscribes to MQTT, parses JSON)
        │
        ▼
[InfluxDB 2.7]  (time-series storage)
        │
        ▼
[Grafana]  (live dashboards on port 3000)
```

### High-Availability (HA) Failover

The system implements **automatic MQTT broker failover** between two Raspberry Pis:

| Role | Host | IP | MQTT Port |
|---|---|---|---|
| Primary | Raspberry Pi 1 | `172.20.10.10` | `1883` |
| Backup | Raspberry Pi 2 | `172.20.10.11` | `1883` |

**Failover logic (in the ESP32 firmware):**
1. Tries to connect to the **Primary** broker
2. After **3 failed attempts**, automatically switches to the **Backup** broker
3. Every **60 seconds**, retries the Primary in the background
4. Reconnects to Primary as soon as it comes back online

Backoff is exponential: 2 s → 4 s → 8 s between retry attempts.

---

## 3. Hardware

| Component | Detail |
|---|---|
| Microcontroller | ESP32 (dev board) |
| Sensor | ZPHS01B multi-pollutant sensor |
| Display | 20×4 I²C LCD (address `0x27`) |
| Sensor interface | UART via `Serial2` (RX=GPIO16, TX=GPIO17, 9600 baud) |
| WiFi | SSID: `Mazly` |

### ZPHS01B Sensor — Measured Parameters

| Parameter | Unit | Byte Offset in Packet |
|---|---|---|
| PM1.0 | µg/m³ | bytes 2–3 |
| PM2.5 | µg/m³ | bytes 4–5 |
| PM10 | µg/m³ | bytes 6–7 |
| CO₂ | ppm | bytes 8–9 |
| TVOC | ppb | bytes 10–11 |
| Temperature | °C | bytes 12–13 (`(raw − 400) / 10`) |
| Humidity | % | bytes 14–15 (`raw / 10`) |
| HCHO (Formaldehyde) | µg/m³ | bytes 16–17 |
| CO | ppm | bytes 18–19 |
| O₃ | ppb | bytes 20–21 |
| NO₂ | ppb | bytes 22–23 |

Packet format: `FF 86 [22 data bytes] [checksum]` — 25 bytes total.  
Checksum: `(~sum(bytes 1–23)) + 1`

> ⚠️ **Current Status:** Real sensor hardware is not working. The firmware is currently running in **mock data mode**, sending randomised-but-realistic values every 10 seconds to validate the data pipeline end-to-end. The real sensor code is commented out in `loop()` and will be re-enabled once hardware is fixed.

---

## 4. Firmware (`src/main.cpp`)

**Build system:** PlatformIO  
**Platform:** `espressif32` / `esp32dev` / Arduino framework  

### Libraries

| Library | Purpose |
|---|---|
| `knolleary/PubSubClient@^2.8` | MQTT client |
| `marcoschwartz/LiquidCrystal_I2C@^1.1.4` | I²C LCD driver |

### Key Functions

| Function | Description |
|---|---|
| `connectWiFi()` | Connects to WiFi; shows status on LCD; sets `wifiConnected` flag |
| `reconnect()` | Connects/reconnects to MQTT with HA failover logic |
| `requestSensorData()` | Sends the 9-byte Q&A command to the ZPHS01B over Serial2 |
| `parseZPHS01B()` | Parses the 25-byte sensor response, validates checksum, publishes via MQTT |
| `sendDataMQTT()` | Serialises readings to JSON and publishes to `sensors/esp32_01` |
| `setup()` | I²C scan, LCD init, Serial2 init, WiFi, MQTT init |
| `loop()` | Maintains MQTT connection; sends mock data every 10 s; (real sensor code disabled) |

### MQTT Payload (JSON)

```json
{
  "pm25": 12.50,
  "pm10": 18.00,
  "co2": 450.00,
  "tvoc": 100.00,
  "temp": 22.00,
  "hum": 45.00,
  "hcho": 0.0150
}
```

Published to topic: **`sensors/esp32_01`**

---

## 5. Server Stack

### Primary Server (`server/docker-compose.yml`)

Deployed on **Raspberry Pi 1** (`172.20.10.10`):

| Service | Image | Port | Role |
|---|---|---|---|
| `mosquitto` | `eclipse-mosquitto:latest` | `1883` | MQTT broker |
| `influxdb` | `influxdb:2.7` | `8086` | Time-series database |
| `telegraf` | `telegraf:latest` | — | MQTT → InfluxDB bridge |
| `grafana` | `grafana/grafana-oss:latest` | `3000` | Dashboard |

### Backup Server (`server/docker-compose-backup.yml`)

Deployed on **Raspberry Pi 2** (`172.20.10.11`):
- Mirror of the primary stack with separate named volumes (`influxdb-backup-data`)
- Grafana is commented out by default to save Pi resources
- InfluxDB org: `AQI_Backup`, bucket: `air_quality_backup`

### Telegraf Configuration (`server/telegraf/telegraf.conf`)

- **Input:** MQTT consumer subscribed to `sensors/esp32_01` on `tcp://mosquitto:1883`, JSON data format
- **Output:** InfluxDB v2 at `http://influxdb:8086`, org: `Secondspark`, bucket: `mazly`
- Poll interval: **10 seconds**

### Mosquitto Configuration

```
listener 1883
allow_anonymous true
```

Anonymous connections are permitted (intended for local network use only).

---

## 6. Research / Comparison Stack (`server/docker-compose-timescaledb.yml`)

A **third, independent** stack for academic research — intended for deployment on a **university lab server or cloud VM**, NOT on the Raspberry Pis.

**Purpose:** Compare **TimescaleDB** (relational time-series, PostgreSQL-based) against **InfluxDB** (specialist time-series).

| Service | Image | Port | Role |
|---|---|---|---|
| `timescaledb_research` | `timescale/timescaledb:latest-pg15` | `5432` | PostgreSQL + TimescaleDB |
| `telegraf_research` | `telegraf:latest` | — | Subscribes to BOTH MQTT brokers |
| `pgadmin_research` | `dpage/pgadmin4:latest` | `5050` | DB admin UI |
| `grafana_timescale` | `grafana/grafana-oss:latest` | `3001` | Dashboard (separate port) |

### TimescaleDB Schema (`server/timescaledb/init.sql`)

- **`sensor_data`** hypertable — raw readings, primary key `(time, device_id)`, 90-day retention policy
- **`sensor_data_hourly`** continuous aggregate — 1-hour bucket averages, auto-refreshed every hour
- **`db_performance_metrics`** hypertable — for comparing DB performance
- `insert_sensor_data()` function handles duplicate data with `ON CONFLICT` — primary broker data always wins

---

## 7. Project File Structure

```
AQI_Project_Sprint/
├── src/
│   └── main.cpp                        # ESP32 firmware (Arduino/PlatformIO)
├── platformio.ini                      # PlatformIO build config
├── server/
│   ├── docker-compose.yml              # Primary Pi stack
│   ├── docker-compose-backup.yml       # Backup Pi stack
│   ├── docker-compose-timescaledb.yml  # Research stack (TimescaleDB)
│   ├── mosquitto/config/
│   │   └── mosquitto.conf              # MQTT broker config
│   ├── telegraf/
│   │   └── telegraf.conf               # Telegraf: MQTT → InfluxDB
│   ├── timescaledb/
│   │   ├── init.sql                    # TimescaleDB schema & policies
│   │   └── telegraf.conf               # Telegraf config for research stack
│   └── grafana/                        # Dashboards managed via Grafana UI
├── firmware/lib/                       # PlatformIO extra libs (currently empty)
├── include/                            # C++ headers (currently empty)
├── data/                               # SPIFFS data (currently empty)
├── docs/                               # Extended docs (in progress)
├── MERMAID_CODE                        # Architecture diagram (graph TB)
└── failover_mermaidcode                # Failover sequence diagram
```

---

## 8. Current State & Known Issues

| Area | Status | Notes |
|---|---|---|
| Firmware compiles | ✅ | PlatformIO build passes |
| WiFi connection | ✅ | Connects to `Mazly` network |
| MQTT failover logic | ✅ | Implemented & tested in code |
| Mock data pipeline | ✅ | Data flows end-to-end every 10 s |
| Real ZPHS01B sensor | ❌ | Hardware issue — sensor not responding; real code disabled in `loop()` |
| LCD display | ✅ | I²C at `0x27`, 20×4, shows status & IP |
| Primary server stack | ✅ | Docker Compose defined |
| Backup server stack | ✅ | Docker Compose defined |
| TimescaleDB research stack | ✅ | Defined with full schema |
| Grafana dashboards | 🔧 | Infrastructure ready, dashboards not yet built |
| `docs/` folder | 🔧 | Empty — tracked in README |
| Upload port (PlatformIO) | 🔧 | Commented out — must be set before flashing |

---

## 9. Network Map

| Device | IP | Services |
|---|---|---|
| ESP32 | DHCP | WiFi client → MQTT publisher |
| Raspberry Pi 1 (Primary) | `172.20.10.10` | Mosquitto `:1883`, InfluxDB `:8086`, Grafana `:3000` |
| Raspberry Pi 2 (Backup) | `172.20.10.11` | Mosquitto `:1883`, InfluxDB `:8086` |
| Research Server (TBD) | Cloud / Lab | TimescaleDB `:5432`, pgAdmin `:5050`, Grafana `:3001` |

---

## 10. How to Deploy

### Flash the ESP32

1. Set `upload_port` and `monitor_port` in `platformio.ini` to your serial device (e.g. `/dev/cu.usbserial-XXXX`)
2. Run:
   ```bash
   pio run --target upload
   ```
3. Monitor serial output:
   ```bash
   pio device monitor
   ```

### Start the Primary Server (Pi 1)

```bash
cd server
docker compose up -d
```

### Start the Backup Server (Pi 2)

```bash
cd server
docker compose -f docker-compose-backup.yml up -d
```

### Start the Research Stack

```bash
cd server
docker compose -f docker-compose-timescaledb.yml up -d
```

### Re-enable the Real Sensor

In `src/main.cpp` inside `loop()`:
1. Remove or disable the mock data block (the `lastSend` timer section)
2. Uncomment the `Serial2.available() >= 25` block at the bottom of `loop()`
