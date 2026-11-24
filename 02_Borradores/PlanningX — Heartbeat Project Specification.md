# **PlanningX — Heartbeat Project Specification**

**Platform Ecosystem:** Xi Platform / xManager / Android Device Manager (ADM)  
**Version:** 1.3 — Architecture, Tenant Detailed Views & Headend Hardware (IRD / ISDB-T) Integration1.3 — IRD Transponder Monitoring & Heartbeat  
Date: October 19, 2026  
Status: Approved for Implementation

# **1\. Executive Summary & Architecture Context**

The PlanningX Heartbeat Project establishes the operational liveness, real-time availability tracking, and telemetry infrastructure for the Xi platform fleet (Android TV boxes, IPTV set-top boxes, and edge streaming appliances managed via xManager and ADM backend services).

## **Architectural Overview**

* **Device Agent (Android TV):** Emits lightweight periodic heartbeat signals over MQTT topic `adm/v1/devices/{deviceId}/heartbeat` every 30 seconds.  
* **MQTT Broker (Eclipse Mosquitto / EMQX):** Handles secure MQTTS ingress (TLS 1.3, port 8883\) with strict per-device dynamic ACLs.  
* **xManager Ingestion Engine:** Microservice subscribed to uplink telemetry queues that validates tenant scope, processes incoming heartbeats, and maintains fast ephemeral state.  
* **Redis Cache Layer:** In-memory metadata and liveness registry running on xManager to absorb high-frequency heartbeat I/O and serve sub-millisecond status lookups.  
* **Relational Persistence (PostgreSQL):** Stores persistent device inventory, tenant hierarchy (Owner, Distributor, ISP, Department), and historical audit logs.  
* **Monitoring Dashboard (Consola Web):** Displays real-time device availability, visual health states, and fleet telemetry via WebSockets.  
* **xManager Local Database (PostgreSQL / Relational Config Store):** Serves as the authoritative master repository for tenant hierarchies, node network configurations (100.64.X.2 overlay VPN), and headend broadcast hardware configurations (IRD satellite receivers and ISDB-T digital modulators).  
* Satellite IRD Ingestion Gateway: Ingests real-time telemetry and heartbeat signals from headend Integrated Receiver Decoders (IRDs, such as multi-channel satellite receivers) into xManager, expanding the monitoring umbrella beyond edge Android TV devices to upstream satellite transponder reception.  
* IRD Telemetry Ingestion: Ingestion mechanism for IRD telemetry via SNMP polling and dedicated uplink channel adm/v1/irds/{irdId}/telemetry.  
* API Console Endpoints: The API console endpoints receiving the health check requests will be reviewed and checked directly to ensure robust ingress verification, service availability, and operational integrity.  
* **Tenant Detailed Views (Consola Web):** Dedicated drill-down interfaces per tenant displaying real-time aggregated fleet heartbeat telemetry alongside local headend broadcast equipment configurations (IRD and ISDB-T).

# **2\. Monitoring Dashboard Logic: Real-Time Green/Red States**

The monitoring dashboard provides real-time visibility into the operational state of all fleet devices within the authenticated user's organization scope.

## **2.1 State Definitions & Thresholds**

Device status is determined strictly by the delta between current server time (now) and the last recorded heartbeat timestamp (last\_heartbeat):

| State | Visual Indicator | Condition | Operational Meaning |
| :---- | :---- | :---- | :---- |
| **ONLINE** | **GREEN (●)** | \$\\Delta t \\le 60\\text{ seconds}\$ | Device is connected, active, and reporting within the 1-minute window. |
| **OFFLINE** | **RED (●)** | \$\\Delta t \> 60\\text{ seconds}\$ | Heartbeat timed out. Device is disconnected, powered down, or network degraded. |

## **2.2 Transition Rules & State Machine**

1. **Transition from RED to GREEN (Online Event):**  
   * Triggered immediately upon receiving a valid heartbeat packet on `adm/v1/devices/{deviceId}/heartbeat`.  
   * xManager updates the timestamp in Redis (`lastSeen`), sets state to `ONLINE`, and emits a WebSocket broadcast `DEVICE_STATUS_UPDATE` to active dashboard sessions.  
2. **Transition from GREEN to RED (Timeout Expiration):**  
   * An asynchronous Watchdog Sweeper in xManager queries the Redis Sorted Set every 10 seconds for devices whose `lastSeen < (now - 60)`.  
   * Identified devices are marked `OFFLINE`, a status transition event is published to WebSockets, and a disconnect log is scheduled for batch insertion into PostgreSQL.

## **2.3 Frontend Dashboard Components**

* **Live Device Inventory Table:** Real-time table displaying Device ID, Organization/Department, IP Address, App/Firmware Version, Relative Last Seen (e.g., "15s ago" vs "2m ago"), and a prominent Green/Red status badge.  
* **Summary KPI Cards:**  
  * Total Provisioned Devices (\$N\_{\\text{total}}\$)  
  * Active Online Count (\$N\_{\\text{green}}\$)  
  * Offline / Disconnected Count (\$N\_{\\text{red}}\$)  
  * Fleet Availability Percentage: \$\\frac{N\_{\\text{green}}}{N\_{\\text{total}}} \\times 100\$  
* **Filtering & Alerting:** Instant filter by status (All, Online Only, Offline Only), search by Device ID or IP, and configurable webhook notifications when device drops offline for more than 5 consecutive minutes.

## **2.4 IRD Transponder Monitoring & Zero-Metric Alarms**

**Scope & Configuration:** Each IRD managed in xManager supports up to 5 configurable transponder monitoring slots (Transponders 1 to 5), configurable with Transponder ID, Carrier Frequency (MHz), Polarization, Symbol Rate, and Description.  
**Monitored Metrics:**

* **Signal Level:** RF input signal power / level in dBm / dBuV (and RF Lock status).  
* **Bandwidth:** Transport stream (TS) aggregate throughput / bitrate in Mbps.

**Zero-Metric Alarm Triggering Logic:**

* **Alarm Condition:** A Critical Alarm is immediately triggered if EITHER metric hits zero:  
  * **Signal Level \== 0:** Indicates loss of satellite carrier, LNB failure, dish misalignment, or RF cabling issue.  
  * **Bandwidth \== 0:** Indicates carrier lock without TS data, stream loss, demux failure, or transponder blackout.  
* **Visual & Sound States:**  
  * **NORMAL (Green):** Signal Level \> 0 AND Bandwidth \> 0 across all configured transponders.  
  * **CRITICAL ALARM (Red / Flashing / Audio Alert):** Signal Level \== 0 OR Bandwidth \== 0 on any active configured transponder.  
* **Debouncing & Clear Conditions:** Evaluated on every reporting cycle (default 10s). Alarm triggers immediately upon detection of zero and clears only after both Signal Level and Bandwidth register values greater than zero for two consecutive reporting cycles.  
* **Notification Dispatch:** Immediate WebSocket push to Consola Web (channel:ird\_alarms), visual alert banner in the monitoring dashboard, and dispatch of configured webhooks/notifications.

**Consola Web Frontend Widget:**

* Dedicated IRD Transponder Matrix widget displaying up to 5 transponder cards per IRD.  
* Real-time gauges and numerical indicators for Signal Level (dBuV/dBm) and Bandwidth (Mbps).  
* Configuration modal allowing operators to enable/disable and set parameters for up to 5 transponders per IRD.

## **2.4 Direct Review of API Console Endpoints**

The API console endpoints receiving the health check requests will be reviewed and checked directly. This direct review and verification ensures that incoming health check requests, telemetry signals, and availability probes dispatched to the console API services are thoroughly validated for latency, authentication integrity, correct payload handling, and seamless status synchronization with the real-time monitoring interface.

## **2.4 Detailed Views for Each Tenant (Headend & Hardware Integration)**

The Consola Web provides dedicated tenant cockpits for granular operational oversight and headend management:

* **Navigation & Context:** Operators can drill down from the fleet overview directly into a specific tenant operational view.  
* **Tenant Overview Card:** Displays Tenant Name, Tenant Nro/ID, Management Server IP (100.64.X.2), RustDesk Remote ID, InfluxDB instance, Grafana ID, active device counts, and fleet availability percentage.  
* **IRD (Integrated Receiver/Decoder) Management Card:** Outlines satellite signal ingest roles (e.g., Arsat 1/2), Management IP / Web GUI URLs (e.g., 192.168.100.135, 192.168.0.150, 192.168.1.135), Web GUI Access Credentials, Provisioning Status (Deployed, Proveído, Pendiente), and satellite dish/LNB configurations.  
* **ISDB-T (Digital Terrestrial Television Modulator) Management Card:** Outlines terrestrial RF transmission parameters (e.g., Dexin 16-port modulators), Management IP / Web GUI URLs (e.g., 192.168.100.136, 192.168.0.140, 192.168.1.136), Access Credentials, Equipment specifications (e.g., 2 x 16-port Dexin), and Deployment States (Deployed, On Road, Fail, Guardado).  
* **Headend Switching & Distribution Network:** Details management switch provisioning and network distribution infrastructure types (Fiber Optic / FO, Hybrid Fiber-Coaxial / HFC, FO+HFC).

# **3\. Redis Architecture for Metadata Storage in xManager**

To maintain responsiveness and scalability across tens of thousands of concurrent devices, xManager utilizes an in-memory Redis instance (`redis-server` installed on xManager) as the primary state buffer, protecting PostgreSQL from write amplification and connection exhaustion.

## **3.1 Redis Data Structures & Key Schemas**

### **1\. Device Metadata Hash (`HSET`)**

* **Key Pattern:** `device:{deviceId}:meta`  
* **Fields:**  
  * `deviceId`: Hardware unique identifier (string)  
  * `orgId`: UUID of the parent Department/ISP organization  
  * `status`: `"ONLINE"` or `"OFFLINE"`  
  * `lastSeen`: Unix epoch timestamp in milliseconds  
  * `ip`: Local network IP address (e.g., `192.168.100.45`)  
  * `firmwareVersion`: Installed system firmware build (e.g., `v2.1.4-build8`)  
  * `appVersion`: Agent software version (e.g., `adm-agent-v1.0.2`)  
  * `uptimeSeconds`: Cumulative device uptime  
  * `updatedAt`: Server-side ingest timestamp  
* **TTL Strategy:** Retained persistently or set with a sliding 7-day TTL, refreshed on each heartbeat.

### **2\. Heartbeat Timeout Sorted Set (`ZSET`)**

* **Key:** `devices:heartbeat:zset`  
* **Member:** `{deviceId}`  
* **Score:** Unix timestamp of last received heartbeat (seconds).  
* **Timeout Lookup Operation:**  
  * Query: `ZRANGEBYSCORE devices:heartbeat:zset 0 (now - 60)`  
  * Efficiency: \$O(\\log N \+ M)\$, where \$N\$ is total fleet size and \$M\$ is the number of timed-out devices. Eliminates full keyspace scans.

### **3\. Organization Device Index Set (`SET`)**

* **Key Pattern:** `org:{orgId}:devices`  
* **Members:** Set of `{deviceId}` strings belonging to the organization. Enables instant multi-tenant scope resolution without traversing SQL foreign keys.

### **4\. IRD Configuration Hash (`HSET`)**

* **Key Pattern:**`ird:{irdId}:config`  
* **Fields:**`irdId`, `name`, `ip`, `enabledTpsCount` (1 to 5), `tp:{1..5}:enabled`, `tp:{1..5}:name`, `tp:{1..5}:freq`, `tp:{1..5}:pol`

### **5\. IRD Transponder Real-Time Metrics Hash (`HSET`)**

* **Key Pattern:**`ird:{irdId}:tp:{slot}:metrics` (slot \= 1..5)  
* **Fields:**`slotIndex`, `tpName`, `signalLevel`, `bandwidthMbps`, `signalZeroAlarm` (0 or 1), `bandwidthZeroAlarm` (0 or 1), `alarmState` ("OK" or "CRITICAL"), `updatedAt` (Unix epoch ms)

### **6\. Active IRD Transponder Alarm Set (`SET`)**

* **Key Pattern:**`irds:alarms:active`  
* **Members:**`{irdId}:tp:{slot}` for each transponder currently experiencing Signal Level \== 0 or Bandwidth \== 0\. Allows O(1) checking of whether any IRD has an active transponder failure.

## **3.2 Ingestion & Synchronization Flow**

1. **MQTT Arrival:** Broker receives packet on `adm/v1/devices/{deviceId}/heartbeat`.  
2. **xManager Ingestion:**  
   * Executes atomic Redis pipeline:HSET device:{deviceId}:meta lastSeen {now\_ms} status "ONLINE" uptimeSeconds {uptime} appVersion {ver} ip {ip}  
       
     ZADD devices:heartbeat:zset {now\_sec} {deviceId}  
       
     PUBLISH channel:device\_events '{"deviceId": "{deviceId}", "status": "ONLINE", "lastSeen": {now\_ms}}'

     
3. **Database Flush:** A background worker flushes batched updates (`last_heartbeat`, `status`) to PostgreSQL every 5 minutes using `UPDATE devices SET last_heartbeat = data.ts, status = data.status FROM (...)`.

## **3.3 xManager Local Database Schema for Tenant Configurations**

The relational database schema hosted locally on xManager persists tenant configurations and headend metadata:

* **tenants Table:**`tenant_id, name, server_nro, server_ip, rustdesk_id, influx_instance, grafana_id, network_type, status`  
* **tenant\_headend\_configs Table:**`config_id, tenant_id, ird_ip, ird_port, ird_user, ird_password, ird_status, isdbt_ip, isdbt_port, isdbt_user, isdbt_password, isdbt_model, isdbt_status, switch_status, antenna_config, notes`  
* **API Endpoints:** REST endpoints providing headend configuration access:  
  `GET /api/v1/tenants/{tenantId}/details`  
  `GET /api/v1/tenants/{tenantId}/headend-config`  
* **Caching Strategy:** Cached in Redis under key `tenant:{tenantId}:headend_meta` with automatic cache invalidation triggers on database update.  
4. IRD Telemetry Processing: Upon receiving an IRD telemetry packet, xManager validates the up to 5 configured transponders, updates ird:{irdId}:tp:{slot}:metrics, evaluates if signalLevel \== 0 or bandwidthMbps \== 0, updates irds:alarms:active, and publishes state changes to channel:ird\_alarms.

# **4\. Scope for Stage 2 Metric Collections**

While Stage 1 (MVP) focuses strictly on connectivity liveness, device inventory, and basic online/offline state, Stage 2 expands metric collection into comprehensive telemetry, quality of service (QoS), and predictive diagnostic monitoring.

## **4.1 Telemetry Delivery Mechanism**

* **MQTT Topic:** `adm/v1/devices/{deviceId}/telemetry`  
* **QoS Level:** QoS 0 (best-effort, non-blocking telemetry stream)  
* **Collection Frequency:** Configurable reporting interval (default: every 120 seconds; dynamically throttled to 300 seconds during active 4K streaming or degraded network conditions).

## **4.2 Granular Telemetry Domains**

### **1\. Hardware & Operating System Health**

* **CPU Metrics:** Aggregate CPU utilization %, per-core load, 1m/5m/15m system load averages, thermal throttling flags.  
* **Memory Metrics:** Total RAM, free RAM, used RAM, cached RAM, Low-Memory Killer (LMK) kill counters and memory pressure warning triggers.  
* **Storage Metrics:** Internal flash total capacity, free bytes, percentage used, flash write cycles/wear estimation where supported.  
* **SoC Temperature:** Thermal sensor readings in Celsius for CPU and GPU cores.

### **2\. Network Quality & Connectivity Diagnostics**

* **Interface Identification:** Active connection type (`eth0` vs `wlan0`).  
* **Wi-Fi Specific Diagnostics:**  
  * Received Signal Strength Indicator (RSSI) in dBm (e.g., `-55 dBm`).  
  * Signal-to-Noise Ratio (SNR) and link negotiation speed (Mbps).  
  * Connected BSSID, SSID, frequency band (2.4 GHz vs 5.0 GHz), and channel number.  
* **Ethernet Diagnostics:** Link speed (100M/1000M), full/half duplex mode, carrier drop events.  
* **Network Path & Latency:** Gateway round-trip time (RTT ping in ms), DNS lookup resolution time, and packet loss percentage over a 50-packet sample.

### **3\. IPTV Video Playback & Quality of Service (QoS)**

* **Player State:** Current playback condition (`PLAYING`, `BUFFERING`, `PAUSED`, `ERROR`, `IDLE`).  
* **Channel Metadata:** Active Channel ID, Stream URL, multicast/unicast protocol source.  
* **Buffer Performance:** Current buffer fullness (ms), buffer underrun/stall count, stall duration.  
* **Stream Performance:** Ingest bitrate (kbps), video resolution (\$1920\\times 1080\$, \$1280\\times 720\$), frame rate (fps), cumulative dropped video frames count.  
* **Audio Synchronization:** Audio buffer underruns and lip-sync drift indicators.

### **4\. System Watchdog & Crash Observability**

* **Process Memory Footprint:** Resident Set Size (RSS) of the agent background service.  
* **Application Crash Logs:** Uncaught exception summaries, ANR (Application Not Responding) incident counters.  
* **HDMI & Display Status:** HDMI connection state (Connected/Disconnected), HDCP negotiation status, HDMI-CEC state.

## **4.3 Storage & Analytics Pipeline for Stage 2**

* High-volume time-series telemetry data will be ingested via the Telemetry Service and routed to **InfluxDB** (leveraging existing InfluxDB instances in the Xi infrastructure: `http://100.64.x.8:8086`) with pre-configured **Grafana** dashboard templates for long-term trending, anomaly alerts, and performance visualization.

# **5\. Tenant Operational Inventory & Headend Configuration Catalog (xManager Local Database)**

The table below details the verified operational inventory and headend hardware configurations stored in the xManager local database across all deployed tenant nodes:

| Nro | Tenant Name (Cliente) | Server Main / IP | RustDesk ID | IRD Configuration | ISDB-T Configuration | Hardware & Deployment Status |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 13 | Las Toscas | Server 13 (100.64.13.2) |  | PASS: admin | PASS: admin | Influx: influxdb-13 | Deployed |
| 15 | PJC | Server 15 (100.64.15.2) |  | PASS: admin | PASS: admin | Influx: influxdb-15 | Deployed |
| 19 | Ibicui | Server 19 (100.64.19.2) |  | http\://192.168.100.135/ (admin) | http\://192.168.100.136 (admin) | Influx: influxdb-19 | Deployed |
| 23 | Fraile Pintado | Server 23 (100.64.23.2) | 298 681 182 | http\://192.168.0.150 (admin) | http\://192.168.0.140 (admin) | Influx: influxdb-23 | Deployed |
| 29 | Puerto Esperanza | Server 29 (100.64.29.2) | 1 935 568 006 | http\://192.168.100.135/ (admin) | http\://192.168.100.136 (admin) | Influx: influxdb-29 | Deployed |
| 31 | Caa Cati | Server 31 (100.64.31.2) | 296 777 462 | http\://192.168.100.135/ (admin) | http\://192.168.100.136 (admin) | Influx: influxdb-31 | Deployed |
| 33 | San Martin 2 | Server 33 (100.64.33.2) | 295 534 992 | http\://192.168.100.135/ (admin) | http\://192.168.100.136 (admin) | Influx: influxdb-33 | Deployed |
| 35 | Avia Terai | Server 35 (100.64.35.2) | 285 826 960 | http\://192.168.100.135/ (Q1w2e3r4p0) | http\://192.168.100.136 (admin) | Influx: influxdb-35 | Deployed |
| 37 | Guemes | Server 37 (100.64.37.2) | 292 483 426 | http\://192.168.100.135/ (admin) | http\://192.168.100.136 (admin) | Influx: influxdb-37 | Deployed |
| 39 | Sauce de Luna | Server 39 (100.64.39.2) | 300 925 315 | http\://192.168.1.135/ (c6w2EUcgzKFW9drc2fyX1aHoK8) | http\://192.168.1.136 (c6w2EUcgzKFW9drc2fyX1aHoK8) | Influx: influxdb-39 | Deployed |
| 41 | Paso de la Patria | Server 41 (100.64.41.2) | 294 128 757 |  |  | Influx: influxdb-41 | Deployed |
| 43 | Frentones | Server 43 (100.64.43.2) | 300 596 523 |  |  | Influx: influxdb-43 | Deployed |
| 45 | Aguaray | Server 45 (100.64.45.2) | 290 877 920 | http\://192.168.100.135/ |  | Influx: influxdb-45 | Deployed |
| 47 | Junin | Server 47 (100.64.47.2) | 298 935 477 | http\://192.168.100.137 (Alem) | http\://181.209.124.98:9091/ (admin/Mega-BTO414) | Influx: influxdb-47 | Deployed |
| 49 | Machagai | Server 49 (100.64.49.2) | 290 877 920 / 1.558.575.440 | http\://192.168.100.135/ (admin) | http\://192.168.100.136 (admin) | Influx: influxdb-49 | Deployed |
| 51 | Embarcacion | Server 51 (100.64.51.2) |  |  |  | Influx: influxdb-51 | Deployed |
| 53 | El Dorado (Wanda) | Server 53 (100.64.53.2) |  |  |  | Influx: influxdb-53 | Deployed |
| 55 | Ituzaingo | Server 55 (100.64.55.2) | 291 875 457 |  | Usuario: kelo / Clave: keloisdbt | Influx: influxdb-55 | On Road |
| 57 | Aguaray 2 | Server 57 (100.64.57.2) | 300 466 803 |  |  | Influx: influxdb-57 | Deployed |
| 59 | Corzuela | Server 59 (100.64.59.2) | 289 485 867 |  |  | Influx: influxdb-59 | Deployed |
| 61 | Apostoles | Server 61 (100.64.61.2) | 291 629 005 |  |  | Influx: influxdb-61 | Fail |
| 63 | 3 Isletas | Server 63 (100.64.63.2) | 289 704 853 |  |  | Influx: influxdb-63 | Deployed |
| 65 | Villafane | Server 65 (100.64.65.2) | 300 452 762 |  |  | Influx: influxdb-65 | Deployed |
| 67 | San Bernardo | Server 67 (100.64.67.2) | 289 370 562 |  |  | Influx: influxdb-67 | Deployed |
| 69 | Rincon de los Sauces | Server 69 (100.64.69.2) | 291 248 115 |  |  | Influx: influxdb-69 | Deployed |
| 71 | Barrio Salta | Server 71 (100.64.71.2) | 298 001 595 |  |  | Influx: influxdb-71 | Deployed |
| 73 | Villa Angela | Server 73 (100.64.73.2) |  | IRD Proveído / Antena Arsat 2 (1 LNB quad) | ISDBT 1 & 2 Proveído | Influx: influxdb-73 | Switch Proveído | Deployed |
| 75 | Las Lomitas | Server 75 (100.64.75.2) | 1 362 682 805 |  | ISDBT Proveer (Recupero de SM) | Deployed |
| 77 | Espinillo | Server 77 (100.64.77.2) | 285 554 118 | Tiene IRD | Tiene ISDBT | Switch tonto | Deployed |
| 79 | 25 de Mayo | Server 79 (100.64.79.2) | 299 318 035 |  | 2 x 16 puertos Dexin | Plataforma Nueva | Switch Proveer (\$140) | Deployed |
| 81 | Cordoba Barrio | Server 81 (100.64.81.2) | 293 615 479 | Verificar IRD | Tiene ISDBT | Switch Proveer | Deployed |
| 83 | Fontana | Server 83 (100.64.83.2) |  |  |  | Influx: influxdb-83 | Deployed |
| 85 | San Martin Chaco | Server 85 (100.64.85.2) | 287 562 760 | Verificar IRD | Verificar ISDBT | Switch Proveer | Deployed |
| 89 | Pampa del Infierno | Server 89 (100.64.89.2) | 287 331 143 | Proveer IRD | Proveer ISDBT | Proveer Switch | Deployed |
| 91 | Ituzaingo Obercom (SD) | Server 91 (100.64.91.2) | 286 474 168 |  |  | Deployed |
| 93 | Fontana 2 | Server 93 (100.64.93.2) |  |  |  | Deployed |
| 97 | Villa Ocampo (Santa Fe) | Server 97 (100.64.97.2) |  |  |  |  |
| 99 | Arenales (Apostoles) | Server 99 (100.64.99.2) |  |  |  |  |
| 101 | San Estanislao (Zarza) | Server 101 (100.64.101.2) |  | IRD A Enviar | Tiene ISDBT | Grupo Soporte\! | Switch A Enviar |
| 103 | Villa Elisa (SM Chaco) | Server 103 (100.64.103.2) |  |  |  | A enviar |
| 105 | Pila Bs As | Server 105 (100.64.105.2) |  |  |  |  |
| 109 | Bolivar | Server 109 (100.64.109.2) |  |  |  |  |
| 111 | Lujan de Cuyo | Server 111 (100.64.111.2) |  |  |  |  |
| 113 | Obera (Obercom) | Server 113 (100.64.113.2) |  |  |  |  |
| 115 | Santiago Barrio | Server 115 (100.64.115.2) |  |  |  | Guardado |

## **64.4 Headend IRD Transponder Telemetry Specification**

```json
{
  "irdId": "ird-headend-01",
  "timestamp": 1787937000,
  "transponders": [
    {
      "slot": 1,
      "name": "TP 11559 Arsat 2",
      "frequencyMhz": 11559,
      "polarization": "V",
      "signalLevelDbuv": 68.5,
      "bandwidthMbps": 38.4,
      "lock": true,
      "alarm": false
    },
    {
      "slot": 2,
      "name": "TP 11639 Arsat 1",
      "frequencyMhz": 11639,
      "polarization": "H",
      "signalLevelDbuv": 0.0,
      "bandwidthMbps": 0.0,
      "lock": false,
      "alarm": true
    }
  ]
}
```

Decision Matrix Table:

| Signal Level | Bandwidth | State | Alarm Action | Root Cause Indication |
| :---- | :---- | :---- | :---- | :---- |
| \> 0 | \> 0 | NORMAL (Green) | None | Nominal carrier and payload stream |
| \== 0 | \== 0 | CRITICAL (Red) | Trigger Alarm | Complete signal outage or transponder power loss |
| \== 0 | \> 0 | CRITICAL (Red) | Trigger Alarm | RF tuner / sensor fault (data present without RF lock) |
| \> 0 | \== 0 | CRITICAL (Red) | Trigger Alarm | Carrier present with zero bitrate (empty TS / feed blackout) |

# **. Implementation Roadmap & Milestones**

| Milestone | Deliverables | Target Status |
| :---- | :---- | :---- |
| **Milestone 1** | Implement 1-minute timeout Watchdog Sweeper in xManager and integrate Redis Sorted Set. | In Progress |
| **Milestone 2** | Update Consola Web dashboard with dynamic Green/Red status badges and WebSocket push events. | Pending |
| **Milestone 2.1** | Review and directly check API console endpoints receiving health check requests to ensure endpoint availability and ingress reliability. | In Progress |
| **Milestone 3** | Deploy Redis schema on xManager production instances and benchmark sub-millisecond lookups. | Pending |
| **Milestone 4** | Release Agent v1.1 firmware with Stage 2 Telemetry Collector and QoS reporting module. | Stage 2 Roadmap |
| **Milestone 5** | Connect InfluxDB time-series backend to Grafana dashboards for fleet QoS analytics. | Stage 2 Roadmap |
| **Milestone 6** | Implement IRD Transponder Ingestion Engine in xManager with Redis schema supporting up to 5 configurable transponders per IRD. | Stage 2 Roadmap |
| **Milestone 7** | Integrate zero-metric alarm trigger engine (Signal Level \== 0 or Bandwidth \== 0\) with Consola Web real-time WebSocket alerts and IRD telemetry widget. | Stage 2 Roadmap |

