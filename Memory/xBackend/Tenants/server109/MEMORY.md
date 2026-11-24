# MEMORY.md - Tenant `server109` (`Bolivar`)

> **Bitácora Individual de Tenant (`xBackend`)** — Creación dinámica bajo demanda.
> **Reglas:** Máximo 200 líneas. No inventar datos (usar `Sin relevar` si no se consultó). No guardar contraseñas ni tokens en texto plano. Nomenclatura neutral (prohibido "Astra"/"Astraserver"; usar "Servidor de Transmisión" / "Encoder").
> **Protocolo e Índice:** [`Tenants/README.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/README.md)

---

## 1. Identidad y Topología Base

| Campo | Valor |
| :--- | :--- |
| **Tenant ID** | `server109` |
| **Posición (`position`)** | `109` |
| **Nombre / Ubicación** | `Bolivar` / `Bolivar` |
| **Dominio** | `c109.xisrv.xyz` |
| **Estado / Rol** | `enabled` / `distributor` |
| **Última Actualización** | `2026-11-24` |

- **Nodos VPN Estándar (`100.64.109.X`):**
  - Proxmox VE: `100.64.109.2:22` | xManager: `100.64.109.8:22` | Multicast: `100.64.109.2:8522` | Streamer: `100.64.109.2:8222`
  - **Desvíos de red/puertos:** Ninguno detectado en nodos VPN.

---

## 2. Moduladores ISDB-T

| # | Marca / Modelo | Firmware | IP / Puerto | MUX / Canales RF | Estado / Notas |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | `IPM24C` (`ISDBT-ENC-16`) | `Sin relevar` | `https://192.168.100.136` | 16 MUX (`totalchannels: 16`) | `verified` en `tenants.options` |

---

## 3. IRD(s) y Fuentes de Recepción

| # | Marca / Modelo | URL / Acceso (`ird_url`) | Tuners | Satélites / Perfiles | Estado / Notas |
| :---: | :--- | :--- | :---: | :--- | :--- |
| 1 | `ECD2416` (`IRD 16`, tipo `encoder`) | `http://100.64.47.2:20135` | `Sin relevar` | `Sin relevar` | `enabled` en `tenants.options` (apunta a host `100.64.47.2`) |

---

## 4. Inventario de Canales y Servicios

*Último relevamiento: `Sin relevar` (en esta intervención solo se auditó monitoreo/heartbeat)*

| Categoría | Cantidad | Detalle / Notas |
| :--- | :---: | :--- |
| **Canales Live** | `Sin relevar` | — |
| **Canales 24/7 (FAST)** | `Sin relevar` | — |
| **Canales Locales** | `Sin relevar` | — |
| **Total Canales Activos** | `Sin relevar` | — |

- **Softcam (`ks*`):** `ksc.x1234.xyz` (`KSC`, actualizado `2026-11-09` en `tenants.options`)
- **Syncthing (carpetas / peers):** API key configurada en `tenants.options` (peers `Sin relevar`)
- **VOD (Movies / TvShows):** `Sin relevar`

---

## 5. Características Individuales y Particularidades

- Multicast (`100.64.109.2:8522`) no posee enlace directo a `100.64.0.42:4880` (sale por `192.168.2.1` vía `ens19`), por lo que el heartbeat opera mediante el fallback HTTPS público `https://panel-console.xisrv.xyz:8443/v3/monitoring/heartbeat` (~246 ms).
- Proxmox (`100.64.109.2:22`, host `c109`) reporta heartbeat directamente vía `tailscale0` a `http://100.64.0.42:4880/v3/monitoring/heartbeat`.

---

## 6. Problemas Conocidos y Bitácora de Incidentes

### 6.1. Problemas Activos / Recurrentes

| Fecha | Categoría | Descripción del Problema | Causa Raíz / Workaround | Estado |
| :--- | :--- | :--- | :--- | :--- |
| `2026-11-24` | `Monitoreo` | Sin heartbeat en Proxmox y Multicast | Proxmox sin `/root/heartbeat.sh`, sin `CLIENTID` en `/etc/environment` y sin crontab; Multicast con script instalado pero sin entrada en crontab. Corregido y verificado con HTTP 200 + disparo cron. | `RESUELTO` |

### 6.2. Historial de Intervenciones (Bitácora)

- `2026-11-24` | `[Grupal]` | **Acción:** Diagnóstico y remediación de heartbeat en Proxmox (`100.64.109.2:22`) y Multicast (`100.64.109.2:8522`). | **Resultado:** OK — ambos nodos configurados con `CLIENTID=109`, `crontab` de `root` activo, POST `/v3/monitoring/heartbeat` HTTP `200` y ejecución automática por `cron` verificada.
