# MEMORY.md - Tenant `server111` (`Lujan de Cuyo`)

> **Bitácora Individual de Tenant (`xBackend`)** — Creación dinámica bajo demanda.
> **Reglas:** Máximo 200 líneas. No inventar datos (usar `Sin relevar` si no se consultó). No guardar contraseñas ni tokens en texto plano. Nomenclatura neutral (prohibido "Astra"/"Astraserver"; usar "Servidor de Transmisión" / "Encoder").
> **Protocolo e Índice:** [`Tenants/README.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/README.md)

---

## 1. Identidad y Topología Base

| Campo | Valor |
| :--- | :--- |
| **Tenant ID** | `server111` |
| **Posición (`position`)** | `111` |
| **Nombre / Ubicación** | `Lujan de Cuyo` / `Lujan de Cuyo` |
| **Dominio** | `c111.xisrv.xyz` |
| **Estado / Rol** | `enabled` / `distributor` |
| **Última Actualización** | `2026-11-24` |

- **Nodos VPN Estándar (`100.64.111.X`):**
  - Proxmox VE: `100.64.111.2:22` | xManager: `100.64.111.8:22` | Multicast: `100.64.111.2:8522` | Streamer: `100.64.111.2:8222`
  - **Desvíos de red/puertos:** Ninguno detectado en nodos VPN.

---

## 2. Moduladores ISDB-T

| # | Marca / Modelo | Firmware | IP / Puerto | MUX / Canales RF | Estado / Notas |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | `IMP24C-VIRTUAL` (`IPM24C-Virtual`) | `Sin relevar` | `https://192.168.100.136` | 20 MUX (`totalchannels: 20`) | `verified` en `tenants.options` |

---

## 3. IRD(s) y Fuentes de Recepción

| # | Marca / Modelo | URL / Acceso (`ird_url`) | Tuners | Satélites / Perfiles | Estado / Notas |
| :---: | :--- | :--- | :---: | :--- | :--- |
| 1 | `IRD-16` (`IRD`) | `http://192.168.100.135` | `Sin relevar` | `Sin relevar` | `enabled` en `tenants.options` |

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

- La VM Multicast (`100.64.111.2:8522`) fue clonada originalmente desde `server109` y conservaba `CLIENTID=109`/`CLIENT=client109` en `/etc/environment` (normalizado a `111` el `2026-11-24`).
- Multicast reporta heartbeat vía fallback HTTPS público (`https://panel-console.xisrv.xyz:8443/v3/monitoring/heartbeat`, ~242 ms) al no tener ruta directa a `100.64.0.42:4880`.
- El host Proxmox (`100.64.111.2:22`) utiliza una contraseña `root` diferente a la contraseña estándar de `server109`.

---

## 6. Problemas Conocidos y Bitácora de Incidentes

### 6.1. Problemas Activos / Recurrentes

| Fecha | Categoría | Descripción del Problema | Causa Raíz / Workaround | Estado |
| :--- | :--- | :--- | :--- | :--- |
| `2026-11-24` | `Monitoreo` | Sin heartbeat en Multicast (`100.64.111.2:8522`) | Faltaba entrada en `crontab` y `/etc/environment` tenía `CLIENTID=109`. Corregido a `111`, programado cron y verificado con HTTP 200 + disparo cron. | `RESUELTO` |
| `2026-11-24` | `Monitoreo` | Sin heartbeat en Proxmox (`100.64.111.2:22`) | Bloqueado por autenticación SSH (`root` rechazó la contraseña estándar de `c109`). Pendiente de contraseña `root` específica del host para diagnosticar e instalar `heartbeat.sh`. | `ABIERTO` |

### 6.2. Historial de Intervenciones (Bitácora)

- `2026-11-24` | `[Grupal]` | **Acción:** Diagnóstico y remediación de heartbeat en Multicast (`100.64.111.2:8522`) y Proxmox (`100.64.111.2:22`). | **Resultado:** Parcial — Multicast OK (`CLIENTID=111`, cron activo, POST HTTP `200`, disparo cron verificado); Proxmox FAIL por autenticación SSH (`root`).
