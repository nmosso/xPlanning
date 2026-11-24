# Documento de Especificación Técnica: Sistema de Heartbeat e Interfaz de Monitoreo de Cabeceras (Monitoring)

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-005` |
| **Sistema Destino:** | `Cross_System` (`xBackend`, `cpanel-admin`, `xManager`, `xWorker`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-10-19 |
| **Última Actualización:** | 2026-10-19 |
| **Autor(es):** | Lead Architect (Antigravity) & Usuario |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

El proyecto **Heartbeat & Monitoring (`REQ-2026-005`)** establece la infraestructura liviana de observabilidad en tiempo real para todas las cabeceras (*tenants*) desplegadas en la plataforma **xIPTV**.

Cada cabecera está compuesta por un nodo físico **Proxmox VE** (`100.64.X.2`) y **3 máquinas virtuales estándar**:
1. **`xManager`** (`100.64.X.8` / `192.168.2.8:22`)
2. **`Multicast`** (`192.168.2.5`, accesible vía SSH NAT en `100.64.X.2:8522`)
3. **`Streamer`** (`192.168.2.4`, accesible vía SSH NAT exclusivamente en `100.64.X.2:8222`)

Además, cada cabecera opera equipamiento físico de recepción satelital (**IRD**, cuya IP local está almacenada en `xManager` en `public.params` bajo `paramid = 'ird.ip'`, ej. `192.168.100.135`) y modulación digital terrestre (**ISDB-T**), junto con instancias de transmisión en las VMs `Multicast` y `Streamer`.

El objetivo es reportar cada **1 minuto** vía `crontab` el latido (*heartbeat*) y las métricas críticas de cada componente hacia **`xBackend`**, almacenando este estado efímero en **Redis** (sin degradar PostgreSQL con escrituras de alta frecuencia), conservando el histórico en el **InfluxDB** local de cada cabecera (`bucket: "channels"`), y exponiendo la matriz consolidada en un nuevo menú **`Monitoring`** dentro de **`cpanel-admin`**.

> [!IMPORTANT]
> **Estrategia de Despliegue Gradual:** Todas las pruebas e implementaciones iniciales se validan únicamente sobre el **Server 93** (`Pampa de los Guanacos`). Una vez certificado el piloto en el `93`, el despliegue al resto de las cabeceras se realizará **por bloques controlados, nunca de forma masiva en una sola ejecución**.

---

## 2. Filtro Doctrinal y Cuestionamiento Inicial (Filtro Británico — *Keep it Plain*)

- **¿Cuál es la necesidad real identificada?**
  Visualizar en una única pantalla centralizada (`cpanel-admin -> Monitoring`) la salud operativa de todos los *tenants*: conectividad del host Proxmox y sus 3 VMs (4 semáforos), estado de recepción del IRD (2 indicadores) y estado de errores de los procesos de transmisión en `Multicast` y `Streamer` (2 indicadores).
- **¿Qué supuestos del borrador móvil preliminar fueron descartados?**
  1. **Descartado MQTT Broker (Mosquitto/EMQX) y agentes Android TV (ADM):** No forman parte de la arquitectura real de cabeceras. Los reportes se realizan mediante scripts ligeros ejecutados cada 1 minuto por `crontab` (carpeta `cron` de cada nodo) hacia la API HTTP de `xBackend`.
  2. **Descartada la creación de tablas SQL redundantes (`tenants` local o `tenant_headend_configs`):** Se reutiliza la tabla maestra existente `public.tenants` en `xServerSQL` (`xBackend`) y `public.params` (`ird.ip`) en `xClientSQL` (`xManager`).
  3. **Descartada la escritura de telemetría o histórico en PostgreSQL (ni en `xManager` ni en `xBackend`):**
     - **Tiempo Real (Semáforos / Estado Actual):** Se guarda exclusivamente en **Redis** de `xBackend`.
     - **Histórico (Series de Tiempo de IRD y Errores de Multicast/Streamer):** Se guarda y consulta directamente en **InfluxDB** (`http://100.64.X.8:8086`, org `xi`, bucket `channels`), que ya almacena las métricas de `stream` y `tuner_status`.
- **¿Qué componentes existentes se reutilizan y mejoran?**
  - `public.tenants` y `tenantlist.sql` en `xServerSQL` (para catálogo de tenants y `tsdb_token`).
  - Parámetro local **`ird.ip`** en `public.params` de `xManager` (`Server.params.get('ird.ip')`, ej. `192.168.100.135`), utilizado por `xManager` para consultar al IRD en la LAN de cabecera.
  - Variable `/etc/environment` **`CLIENT=X`** (numérico puro, ej. `CLIENT=93`, **no** `client93`), que se normalizará vía `xWorker` (`REQ-2026-006`) en `Proxmox`, `xManager`, `Multicast` y `Streamer`.
  - Función existente **`InfluxDB.monitorIRDInternal`** en [`xManager/src/controllers/influxdb.ts`](file:///d:/Projects/DEV/xIPTV/xManager/src/controllers/influxdb.ts#L154-L230): se mejorará para tomar `Server.params.get('ird.ip')`, descubrir automáticamente los tuners activos desde el menú **Output** (`proctype=ip_stream_out`, `op_code=5`) y obtener estado + parámetros de referencia desde **Tuner** (`proctype=tuner`, `op_code=get_all`), guardando en InfluxDB (`tuner_status`) y retornando el resumen para el heartbeat.
  - Consultas Flux en [`xManager/src/controllers/influxdb.ts`](file:///d:/Projects/DEV/xIPTV/xManager/src/controllers/influxdb.ts#L241-L313) sobre el bucket `channels` (`_measurement == "stream"`) cruzadas con `public.channels` y `public.astraservers` en `xClientSQL`.

---

## 3. Alcance y Límites del Requerimiento

### 3.1. En Alcance (In Scope)
- [ ] **Normalización de `/etc/environment` (`CLIENT=X`) vía `xWorker` (`REQ-2026-006`):** Garantizar (primero en Server 93 y luego por bloques) que en los nodos `Proxmox` (`100.64.X.2:22`), `xManager` (`100.64.X.8:22`), `Multicast` (`100.64.X.2:8522`) y `Streamer` (`100.64.X.2:8222`), el archivo `/etc/environment` contenga exactamente `CLIENT=X` (con timeout SSH de `30s`).
- [ ] **Mejora de `InfluxDB.monitorIRDInternal` en `xManager` (usando IP local `ird.ip`):**
  - Obtener la IP local del IRD desde `Server.params.get('ird.ip')` (`public.params` en `xManager`, ej. `192.168.100.135`).
  - Cruzar `POST http://${irdIp}/cgi.php?proctype=ip_stream_out` (`op_code=5&ip_index=0`) para identificar automáticamente los tuners habilitados en **Output** (`m_en === 1`).
  - Consultar `POST http://${irdIp}/cgi.php?proctype=tuner` (`op_code=get_all&ch_index=0`) para extraer las métricas clave (**`Quality`**, **`Strength`**, **`C/N`**, **`BER`**, **`Bitrate`**, **`Lock`**) y los datos de referencia (**`Frequency`**, **`LNB Voltage`**, **`Symbolrate`**).
  - Persistir la serie histórica en InfluxDB (`bucket: "channels"`, `_measurement: "tuner_status"`).
- [ ] **Scripts de Heartbeat en Carpeta `cron` (`* * * * *`):**
  - En **Proxmox (`.2:22`)**, **Multicast (`:8522`)** y **Streamer (`:8222`)**: script ligero `/root/cron/heartbeat.sh` (Proxmox) y `/home/ubuntu/cron/heartbeat.sh` (Multicast/Streamer) que lee `CLIENT` de `/etc/environment` y envía cada 1 minuto vía `curl` su latido de vida (`clientid`, `node`, `uptime`) a `POST /v3/monitoring/heartbeat` en `xBackend`.
  - En **`xManager` (`.8:22`)**: script `/root/cron/heartbeat.sh` que dispara cada 1 minuto `curl -sS -m 10 -X POST http://localhost:8040/v3/monitoring/collect`, el cual:
    1. Consulta el IRD por su IP local (`ird.ip`) (`Output` + `Tuner`) y escribe la serie en InfluxDB (`bucket: "channels"`).
    2. Lee desde InfluxDB (`bucket: "channels"`) el resumen actual de errores de transmisión clasificados en **Multicast** (`encoder10..13` / `192.168.2.5`) y **Streamer** (`encoder14..19` / `192.168.2.4`).
    3. Envía a **Redis de `xBackend`** (`POST /v3/monitoring/heartbeat`) su propio latido + los 2 indicadores del IRD + los 2 indicadores de errores (`Multicast` y `Streamer`).
- [ ] **Capa Redis en `xBackend`:** Almacenamiento en memoria del estado efímero de los 4 nodos, los 2 indicadores del IRD y los 2 indicadores de transmisión por cada `CLIENT` (`position`).
- [ ] **Consulta Histórica Directa a InfluxDB:** Para gráficas o detalle histórico de señal IRD y errores de Multicast/Streamer, consulta directa al InfluxDB del tenant (`100.64.X.8:8086`, bucket `channels`) sin importar datos a PostgreSQL.
- [ ] **Nuevo Menú `Monitoring` en `cpanel-admin`:** Vista matricial de todos los tenants con el nombre del tenant, **4 semáforos de infraestructura** (Proxmox + 3 VMs), **2 indicadores de salud del IRD** y **2 indicadores de errores en Multicast y Streamer**.

### 3.2. Fuera de Alcance (Out of Scope - *YAGNI*)
- Monitoreo de Set-Top Boxes Android TV por MQTT.
- Importación o duplicación de series históricas de telemetría (IRD o errores de stream) en PostgreSQL (`xServerSQL` o `xClientSQL`).

---

## 4. Arquitectura, Flujo Operativo y Relevamiento Certificado

```mermaid
flowchart LR
    subgraph Cabecera_Tenant ["Cabecera Tenant (Position = X)"]
        PVE["Proxmox Host<br/>100.64.X.2:22<br/>/root/cron/heartbeat.sh<br/>CLIENT=X"]
        VM_XM["VM xManager<br/>100.64.X.8:22<br/>/root/cron/heartbeat.sh<br/>CLIENT=X"]
        VM_MC["VM Multicast<br/>100.64.X.2:8522 (192.168.2.5)<br/>/home/ubuntu/cron/heartbeat.sh<br/>CLIENT=X"]
        VM_ST["VM Streamer<br/>100.64.X.2:8222 (192.168.2.4)<br/>/home/ubuntu/cron/heartbeat.sh<br/>CLIENT=X"]
        IRD["Hardware IRD<br/>IP Local: ird.ip en xManager<br/>(ej. 192.168.100.135)"]
        INFLUX[("InfluxDB Local<br/>100.64.X.8:8086<br/>bucket: channels")]

        IRD -->|LAN CGI: tuner & ip_stream_out| VM_XM
        VM_XM -->|tuner_status| INFLUX
        VM_MC -->|stream (encoder10..13)| INFLUX
        VM_ST -->|stream (encoder14..19)| INFLUX
    end

    subgraph Bloque_Central ["Bloque Central (xBackend)"]
        API["xBackend API<br/>POST /v3/monitoring/heartbeat<br/>GET /v3/monitoring/status"]
        REDIS[("Redis Cache<br/>Estado Efímero Tiempo Real")]
        PG[("PostgreSQL (xServerSQL)<br/>Catálogo public.tenants")]

        API --> REDIS
        API --> PG
    end

    subgraph Frontend_Admin ["Consola Central"]
        CPANEL["cpanel-admin<br/>Menú: Monitoring"]
    end

    PVE -->|Cron 1m Heartbeat| API
    VM_MC -->|Cron 1m Heartbeat| API
    VM_ST -->|Cron 1m Heartbeat| API
    VM_XM -->|Cron 1m Heartbeat + IRD + Errores MC/ST| API
    API -.->|Consulta Histórica Directa| INFLUX
    CPANEL -->|HTTP Directo (Sin xProxy)| API
```

### 4.1. Relevamiento de las VMs, Carpetas `cron` y `/etc/environment` (Server 93 y Cabeceras)

| Nodo | Acceso SSH | Carpeta `cron` | Estándar `/etc/environment` | Comportamiento de `heartbeat.sh` (`* * * * *`) |
| :--- | :--- | :--- | :--- | :--- |
| **Proxmox Host** | `100.64.X.2:22` | `/root/cron` *(a crear por `xWorker`)* | `CLIENT=X` | Envía `{ clientid: X, node: "proxmox", uptime }` a `xBackend`. |
| **VM `xManager`** | `100.64.X.8:22` | `/root/cron` *(existente)* | `CLIENT=X` | Invoca `http://localhost:8040/v3/monitoring/collect`, que recolecta IRD (IP local `ird.ip`) + InfluxDB y publica en `xBackend`. |
| **VM `Multicast`** | `100.64.X.2:8522` (`192.168.2.5`) | `/home/ubuntu/cron` *(existente)* | `CLIENT=X` | Envía `{ clientid: X, node: "multicast", uptime }` a `xBackend`. |
| **VM `Streamer`** | `100.64.X.2:8222` (`192.168.2.4`) | `/home/ubuntu/cron` *(existente)* | `CLIENT=X` | Envía `{ clientid: X, node: "streamer", uptime }` a `xBackend`. |

### 4.2. Relevamiento Técnico Certificado del IRD (`http://100.64.83.2:20135/`)
El equipo IRD utiliza servidor embebido `Embedthis-Appweb/3.4.2` con autenticación HTTP Basic (`admin:admin`, sin Captcha) y expone dos endpoints JSON directos vía `/cgi.php`:

#### A. Detección de Tuners Activos en Menú `Output` (`proctype=ip_stream_out`)
- **Request:** `POST http://100.64.X.2:20135/cgi.php?proctype=ip_stream_out`
  - **Body:** `op_code=5&ip_index=0`
- **Estructura de Respuesta:**
  - `mpts[]`: Configuración de los canales IP de salida. Filtrando `m_en === 1` se obtienen los **`tuner_index`** realmente utilizados en producción (ej. en Server 83: `tuner_index` `0, 1, 2, 3, 7` correspondientes a los Tuners `1, 2, 3, 4, 8` saliendo por `239.2.2.2:2000..2016`).
  - `status`: Incluye `alarm_count`, `active_count` y el arreglo `status.mpts[]` con `{ m_status, bitrate_valid, bitrate_total, tuner_index }`.

#### B. Métricas de Señal y Parámetros de Referencia en Menú `Tuner` (`proctype=tuner`)
- **Request:** `POST http://100.64.X.2:20135/cgi.php?proctype=tuner`
  - **Body:** `op_code=get_all&ch_index=0` *(devuelve en 1 sola llamada `status` y `config` de los 16 tuners)*.
- **Mapeo Exacto de Campos (Ejemplo real Tuner 1 — `index: 0` en Server 83):**

| Categoría | Campo en UI | Ruta JSON (`tuner[i]`) | Valor en Server 83 (Tuner 1) | Regla de Mapeo / Unidad |
| :--- | :--- | :--- | :--- | :--- |
| **Estado / Tráfico** | **TS Lock & Bitrate** | `status.lock` / `status.bitrate_total` | `lock: 1`, `59.419 Mbps` (`valid: 51.82`) | `lock === 1` (Verde `label-success`), `0` (Rojo/Gris) |
| **Valor Importante** | **Quality** | `status.quality` | `59` (`59%`) | Porcentaje `0..100%` |
| **Valor Importante** | **Strength** | `status.strength` | `62` (`62%`) | Porcentaje `0..100%` |
| **Valor Importante** | **C/N** | `status.c_n_100` | `14.75` (`14.75 dB`) | Relación Portadora/Ruido (float) |
| **Valor Importante** | **BER** | `status.ber` | `"0.0E+00"` | String científico (`"0.0E+00"` = sin errores; `"1.0E-09"` en unlock) |
| *Métrica Auxiliar* | *Power* | `status.power` | `-37.03` (`dBm`) | Nivel RF en dBm |
| **Referencia** | **Frequency** | `config.freq` | `11559` (`11559.000 MHz`) | Frecuencia portadora en MHz |
| **Referencia** | **LNB Voltage** | `config.lnb_voltage` | `0` $\to$ **`18V`** | `0` = **`18V`**, `1` = **`13V`**, `2` = **`0V`** |
| **Referencia** | **Symbolrate** | `config.symbolrate` | `30000` (`30000 Ksps`) | Tasa de símbolos en Ksps |
| *Referencia Aux.* | *LNB Freq / 22K / Sat* | `config.lnb_freq` / `dvbs_22k` / `sate` | `10600 MHz` / `1` (`ON`) / `1` | `dvbs_22k`: `0` = `OFF`, `1` = `ON` |

---

## 5. Contratos de Datos: Redis (Tiempo Real) e InfluxDB (Histórico)

### 5.1. Estructura en Redis (`xBackend` — Estado Efímero en Tiempo Real)
- **Clave por Tenant (`HSET`):** `monitoring:tenant:{clientid}` (donde `{clientid}` es el entero `CLIENT` / `position`, ej. `83`).
- **Campos del Hash en Redis:**
  - `proxmox_ts`: Epoch timestamp (segundos) del último heartbeat de Proxmox (`.2`).
  - `xmanager_ts`: Epoch timestamp (segundos) del último heartbeat de la VM `xManager` (`.8`).
  - `multicast_ts`: Epoch timestamp (segundos) del último heartbeat de la VM `Multicast` (`:8522`).
  - `streamer_ts`: Epoch timestamp (segundos) del último heartbeat de la VM `Streamer` (`:8222` / `:8422`).
  - `ird_summary`: JSON stringificado con los **2 indicadores de salud del IRD** y el detalle de los tuners activos (filtrados por `m_en === 1` en Output).
  - `multicast_errors`: JSON stringificado con el **indicador de errores en Multicast** (canales asignados a `192.168.2.5` / `encoder10..13`).
  - `streamer_errors`: JSON stringificado con el **indicador de errores en Streamer** (canales asignados a `192.168.2.4` / `encoder14..19`).

### 5.2. Estructura en InfluxDB (`100.64.X.8:8086` — Repositorio Histórico en Bucket `channels`)
En lugar de importar datos históricos a PostgreSQL, toda consulta histórica desde `cpanel-admin` (vía `xBackend`) lee del bucket `channels` en el InfluxDB del tenant:
1. **Medición `tuner_status` (escrita por `InfluxDB.monitorIRDInternal` en `xManager`):**
   - **Tags:** `ird_ip`, `index` (0-based), `tuner_num` (1-based), `freq`, `symbolrate`, `lnb_voltage` (`18V`/`13V`/`0V`), `m_ip`, `m_port`.
   - **Fields:** `lock` (int), `quality` (int), `strength` (int), `cn` (float), `ber` (string/float), `power` (float), `bitrate_valid` (float), `bitrate_total` (float), `m_status` (int).
2. **Medición `stream` (escrita nativamente por ambas VMs `Multicast` y `Streamer` en `192.168.2.8:8086`):**
   - **Tags:** `id` (`channelid`), `name`.
   - **Fields:** `active`, `onair`, `bitrate`, `cc_error`, `pes_error`, `sc_error`, `sync_error`.
   - **Clasificación Multicast vs Streamer (Reutilizando `public.channels` y `public.astraservers` en `xManager`):**
     - Ambos nodos (`192.168.2.5` y `192.168.2.4`) inyectan directamente en el mismo bucket `channels` (`tsdb_url: "http://192.168.2.8:8086"`).
     - En `xClientSQL`, la tabla `public.astraservers` mapea `encoder10..13` a `192.168.2.5:8010..8013` (**Multicast**) y `encoder14..19` a `192.168.2.4:8014..8019` (**Streamer**), mientras que `public.channels` vincula cada `channelid` con su `astraserver`.
     - Por lo tanto, `xManager` clasifica sin crear tablas nuevas qué errores de InfluxDB corresponden al indicador **Multicast** y cuáles al indicador **Streamer**.

### 5.3. Reglas Deterministas de Evaluación de Semáforos e Indicadores (`R#1` y `R#2` Aprobadas)

#### A. 4 Semáforos de Infraestructura (`Proxmox`, `xManager`, `Multicast`, `Streamer`)
Al ejecutar `GET /v3/monitoring/status` desde `cpanel-admin`, `xBackend` compara `now_sec - {nodo}_ts`:
- **VERDE (`ONLINE`):** $\Delta t \le 90\text{ segundos}$ (otorga 30s de gracia sobre el ciclo de `crontab` de 60s).
- **ROJO (`OFFLINE`):** $\Delta t > 90\text{ segundos}$ o clave ausente en Redis.

#### B. 2 Indicadores de Salud del IRD
1. **IRD Indicador 1 (`Señal / Lock`):**
   - **VERDE (`OK`):** Todos los tuners activos en Output (`m_en == 1`) tienen `lock == 1`, `C/N >= 8.0 dB` y `Quality >= 40%`.
   - **AMARILLO (`WARNING`):** Todos los tuners activos tienen `lock == 1`, pero alguno presenta señal degradada (`C/N < 8.0 dB`, `Quality < 40%` o `BER != "0.0E+00"`).
   - **ROJO (`ERROR`):** Algún tuner activo en Output (`m_en == 1`) está desenganchado (`lock == 0`) o el IRD no responde.
   - **GRIS (`N/A`):** El tenant no posee IRD desplegado.
2. **IRD Indicador 2 (`Output / Bitrate`):**
   - **VERDE (`OK`):** `alarm_count == 0` y todos los MPTS habilitados (`m_en == 1`) presentan `bitrate_total > 0 Mbps` y `m_status == 1`.
   - **ROJO (`ERROR`):** `alarm_count > 0` o algún MPTS habilitado cae a `0 Mbps` (`m_status != 1`).
   - **GRIS (`N/A`):** El tenant no posee IRD desplegado.

#### C. 2 Indicadores de Salud de Transmisión (`Multicast` y `Streamer`)
Evaluados por separado para los canales habilitados en **Multicast** (`192.168.2.5` / `encoder10..13`) y en **Streamer** (`192.168.2.4` / `encoder14..19`):
- **VERDE (`OK`):** Todos los canales habilitados tienen `bitrate > 0` y sin errores de continuidad (`cc_error == 0`).
- **AMARILLO (`WARNING`):** Todos los canales habilitados tienen `bitrate > 0`, pero existen canales con `cc_error > 0` / `pes_error > 0` en la ventana actual.
- **ROJO (`ERROR`):** Existe $\ge 1$ canal habilitado caído en `0 kbps` (mostrando badge resumen, ej. `2 caídos / 35 activos`).

### 5.4. Contrato JSON del Payload de Heartbeat (`POST /v3/monitoring/heartbeat`)
```json
{
  "clientid": 83,
  "node": "xmanager",
  "timestamp": 1760896800,
  "metrics": {
    "uptime": 96116,
    "ird": {
      "ird_ip": "192.168.100.135",
      "active_count": 5,
      "alarm_count": 0,
      "signal_status": "green",
      "bitrate_status": "green",
      "tuners": [
        {
          "tuner": 1,
          "index": 0,
          "lock": 1,
          "bitrate_total": 59.419,
          "bitrate_valid": 51.819,
          "quality": 59,
          "strength": 62,
          "cn": 14.75,
          "ber": "0.0E+00",
          "power": -37.03,
          "freq": 11559,
          "lnb_freq": 10600,
          "symbolrate": 30000,
          "lnb_voltage": "18V",
          "dvbs_22k": "ON",
          "sate": 1,
          "out_ip": "239.2.2.2",
          "out_port": 2000,
          "out_status": 1
        }
      ]
    },
    "multicast": {
      "status": "green",
      "active_channels": 35,
      "down_channels": 0,
      "cc_error_channels": 0
    },
    "streamer": {
      "status": "green",
      "active_channels": 18,
      "down_channels": 0,
      "cc_error_channels": 0
    }
  }
}
```

### 5.5. Persistencia de Metadatos de Equipamiento de Cabecera en PostgreSQL (`public.tenants`)
Sin crear nuevas tablas, cualquier dato de inventario de cabecera (IP del IRD, IP del ISDB-T, estado de provisión de Switch/IRD/ISDB-T, notas) se almacena dentro de la columna JSONB existente `public.tenants.options` bajo la clave `headend`:

```json
{
  "headend": {
    "ird_ip": "192.168.100.135",
    "ird_status": "Deployed",
    "isdbt_ip": "192.168.100.136",
    "isdbt_model": "Dexin 16p",
    "isdbt_status": "Deployed",
    "switch_status": "Proveído",
    "notes": "Antena Arsat 2 (1 LNB quad)"
  }
}
```

---

## 6. Requerimientos de Frontend (`cpanel-admin` -> Menú `Monitoring`)

### 6.1. Cumplimiento de Directiva No-Astra
- Prohibido mostrar las palabras `"Astra"` o `"Astraserver"` en columnas, encabezados, tooltips o modales.
- Los dos indicadores de errores de transmisión se denominarán **"Multicast"** y **"Streamer"**.

### 6.2. Estructura Visual de la Tabla `Monitoring`
Cada fila representa un Tenant activo (`role = 'distributor'`, ordenado por `position` / Server Nro) con las siguientes columnas:

1. **Tenant (`Server # - Nombre`):** Ej. `[83] Cabecera 83` (`100.64.83.2`).
2. **Semáforos de Infraestructura (4 Luces Verde/Rojo):**
   - `Proxmox` (`.2`)
   - `xManager` (`.8`)
   - `Multicast` (`:8522`)
   - `Streamer` (`:8222`)
   *(Con tooltip indicando "Último reporte: hace Xs" o "Sin reporte").*
3. **Salud del IRD (2 Indicadores):**
   - **IRD Señal (Lock / Nivel):** Semáforo Verde/Amarillo/Rojo/Gris con tooltip o modal de detalle de los tuners activos (`Quality`, `Strength`, `C/N`, `BER` y referencia `Frequency`, `LNB Voltage`, `Symbolrate`).
   - **IRD Output (Bitrate):** Semáforo Verde/Rojo/Gris mostrando la suma o detalle de Mbps de salida IP.
4. **Salud de Transmisión (2 Indicadores de Errores):**
   - **Multicast:** Semáforo Verde/Amarillo/Rojo con conteo de canales activos / con errores CC / caídos.
   - **Streamer:** Semáforo Verde/Amarillo/Rojo con conteo de canales activos / con errores CC / caídos.

---

## 7. Escenario de Pruebas y Protocolo de Verificación (Filtro Ruso — *Nadezhnost*)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Pruebas Unitarias y Casos Borde (Edge Cases) │
│ • Payload sin clientid o con node inválido -> HTTP 400.     │
│ • Normalización de CLIENT=X numérico en /etc/environment.   │
│ • Tenant sin IRD configurado -> Indicador Gris (N/A), no    │
│   falsa alarma roja.                                        │
│ • Valores nulos o respuestas HTML de error desde el IRD.    │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Pruebas de Integración y Contratos de Datos  │
│ • xBackend: npx tsc --noEmit (0 errores).                   │
│ • xManager: npx tsc --noEmit (0 errores).                   │
│ • cpanel-admin: npm run build (0 errores).                  │
│ • xWorker: npx tsc --noEmit (0 errores).                    │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia, Estrés y Concurrencia           │
│ • Caída de Redis en xBackend: degradación elegante (retorna │
│   lista de tenants con estado 'unknown' sin tirar HTTP 500).│
│ • Timeouts estrictos (<= 5s) en scripts de cron para evitar │
│   solapamiento de procesos cada 1 minuto.                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Criterios de Aceptación (Definition of Done - DoD)
- [x] Especificación aprobada y promovida a `Cross_System/02_Aprobados/`.
- [x] Parámetro `CLIENT=X` (numérico, no `clientX`) normalizado en `/etc/environment` de Proxmox, `xManager`, `Multicast` y `Streamer` (validado y certificado en nodo piloto Server 75 `Las Lomitas` vía `xWorker` con timeout 30s).
- [x] Scripts en carpeta `cron` ejecutándose cada 1 minuto sin logs ruidosos ni acumular procesos colgados.
- [x] Ingesta en Redis funcionando en `xBackend` sin escrituras periódicas innecesarias en PostgreSQL.
- [x] IRD parametrizado vía `ird.tuners` en `params` de `xManager` con mapeo satelital (`Arsat-2` tuners 1..3 y `Arsat-1` tuners 4..6).
- [x] Consultas históricas de Grafana (3 series: Ancho de banda verde, Streams activos amarillo, Errores CC azul) implementadas y servidas en `xBackend` desde InfluxDB (`bucket: channels`).
- [x] Vista `Monitoring` operativa en `cpanel-admin` con columnas de semáforos en cabecera (luz roja destacada de 16px), visualización de tuners IRD con satélites y gráfico time-series tipo Grafana cumpliendo 100% la directiva **No-Astra**.
