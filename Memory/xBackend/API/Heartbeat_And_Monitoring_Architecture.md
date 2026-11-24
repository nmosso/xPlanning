# Nota de Memoria: Sistema de Heartbeat (Redis), Telemetría IRD/InfluxDB y Vista Monitoring

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `Cross_System` (`xBackend`, `xManager`, `cpanel-admin`, `xWorker`) |
| **Componente:** | `API & Monitoring` |
| **Funcionalidad:** | Sistema de Heartbeat por Crontab, Estado Efímero en Redis e Histórico en InfluxDB (`REQ-2026-005`) |
| **Fecha:** | 2026-10-19 |
| **Autor/Agente:** | Antigravity |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
Define la arquitectura aprobada (`REQ-2026-005` y `REQ-2026-006`) para monitorear en tiempo real las cabeceras (`tenants`) desde un nuevo menú **`Monitoring`** en **`cpanel-admin`**, mostrando por cada cabecera:
- **4 Semáforos de Infraestructura:** `Proxmox` (`.2:22`), `xManager` (`.8:22`), `Multicast` (`:8522`) y `Streamer` (`:8222`).
- **2 Indicadores de Salud del IRD:** Señal/Lock (`Quality`, `Strength`, `C/N`, `BER`) y Output/Bitrate (`m_en == 1`).
- **2 Indicadores de Errores de Transmisión:** `Multicast` (`192.168.2.5`, `encoder10..13`) y `Streamer` (`192.168.2.4`, `encoder14..19`).

---

## 2. Decisiones Técnicas y Arquitectura
- **Normalización de Identidad (`REQ-2026-006`):** En los 4 nodos de cada cabecera (`Proxmox`, `xManager`, `Multicast`, `Streamer`), `/etc/environment` contiene `CLIENT=X` (numérico puro, ej. `CLIENT=83`, nunca `client83`).
- **Estado en Tiempo Real en Redis (`xBackend`):** Los latidos y el resumen actual de indicadores se guardan en Redis (`monitoring:tenant:{clientid}`) sin escribir telemetría efímera en PostgreSQL.
- **Histórico en InfluxDB Local (`bucket: "channels"` en `100.64.X.8:8086`):** No se importa histórico a PostgreSQL (ni en `xManager` ni en `xBackend`). Ambas VMs (`Multicast` y `Streamer`) ya reportan nativamente la medición `stream` a `http://192.168.2.8:8086` (bucket `channels`), y `xManager` persiste `tuner_status` del IRD en el mismo bucket.
- **Clasificación Multicast vs Streamer sin Tablas Nuevas:** Se reutilizan `public.channels` (`channelid`, `astraserver`) y `public.astraservers` (`encoder10..13` = Multicast `192.168.2.5`; `encoder14..19` = Streamer `192.168.2.4`) en `xClientSQL`.
- **Relevamiento Certificado del IRD (`http://100.64.X.2:20135/`, Basic Auth `admin:admin`):**
  - Menú **Output**: `POST /cgi.php?proctype=ip_stream_out` (`op_code=5&ip_index=0`) filtra tuners activos por `m_en === 1`.
  - Menú **Tuner**: `POST /cgi.php?proctype=tuner` (`op_code=get_all&ch_index=0`) obtiene métricas (`quality`, `strength`, `c_n_100`, `ber`, `lock`, `bitrate_total`) y referencias (`freq`, `lnb_voltage`: `0=18V, 1=13V, 2=0V`, `symbolrate`).
