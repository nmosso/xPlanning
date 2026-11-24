# MEMORY.md - Tenant `<tenantid>` (`<name>`)

> **Bitácora Individual de Tenant (`xBackend`)** — Creación dinámica bajo demanda.
> **Reglas:** Máximo 200 líneas. No inventar datos (usar `Sin relevar` si no se consultó). No guardar contraseñas ni tokens en texto plano. Nomenclatura neutral (prohibido "Astra"/"Astraserver"; usar "Servidor de Transmisión" / "Encoder").
> **Protocolo e Índice:** [`Tenants/README.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/README.md)

---

## 1. Identidad y Topología Base

| Campo | Valor |
| :--- | :--- |
| **Tenant ID** | `<tenantid>` (ej. `server83`) |
| **Posición (`position`)** | `<N>` |
| **Nombre / Ubicación** | `<name>` / `<location>` |
| **Dominio** | `c<N>.xisrv.xyz` |
| **Estado / Rol** | `enabled` / `distributor` |
| **Última Actualización** | `AAAA-MM-DD` |

- **Nodos VPN Estándar (`100.64.<N>.X`):**
  - Proxmox VE: `100.64.<N>.2:22` | xManager: `100.64.<N>.8:22` | Multicast: `100.64.<N>.2:8522` | Streamer: `100.64.<N>.2:8222` *(fallback `:8422`)*
  - **Desvíos de red/puertos:** `Ninguno` *(o detallar excepciones)*

---

## 2. Moduladores ISDB-T

| # | Marca / Modelo | Firmware | IP / Puerto | MUX / Canales RF | Estado / Notas |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | `Sin relevar` | `Sin relevar` | `Sin relevar` | `Sin relevar` | — |

---

## 3. IRD(s) y Fuentes de Recepción

| # | Marca / Modelo | URL / Acceso (`ird_url`) | Tuners | Satélites / Perfiles | Estado / Notas |
| :---: | :--- | :--- | :---: | :--- | :--- |
| 1 | `Sin relevar` | `Sin relevar` | `Sin relevar` | `Sin relevar` | — |

---

## 4. Inventario de Canales y Servicios

*Último relevamiento: `AAAA-MM-DD`*

| Categoría | Cantidad | Detalle / Notas |
| :--- | :---: | :--- |
| **Canales Live** | `Sin relevar` | *(Satélite / IP / perfiles)* |
| **Canales 24/7 (FAST)** | `Sin relevar` | *(Live1 / Live2 / Live3)* |
| **Canales Locales** | `Sin relevar` | *(Señales propias del cableoperador)* |
| **Total Canales Activos** | `Sin relevar` | — |

- **Softcam (`ks*`):** `Sin relevar`
- **Syncthing (carpetas / peers):** `Sin relevar`
- **VOD (Movies / TvShows):** `Sin relevar`

---

## 5. Características Individuales y Particularidades

*Documentar aquí todo lo que diferencia a este tenant del estándar de flota (ISP/enlace, NAT/firewall, hardware dedicado, discos/storage, excepciones de monitoreo, reglas comerciales/técnicas del cliente).*

- `Sin particularidades registradas aún.`

---

## 6. Problemas Conocidos y Bitácora de Incidentes

### 6.1. Problemas Activos / Recurrentes

| Fecha | Categoría | Descripción del Problema | Causa Raíz / Workaround | Estado |
| :--- | :--- | :--- | :--- | :--- |
| `AAAA-MM-DD` | `Modulador / IRD / Red / Canales / Syncthing / HW` | *Descripción concisa* | *Causa o paliativo vigente* | `ABIERTO` / `EN SEGUIMIENTO` / `RESUELTO` / `SUSTITUIDO` / `ABANDONADO` |

### 6.2. Historial de Intervenciones (Bitácora)

*Mantener las últimas ~15 intervenciones relevantes; consolidar hallazgos permanentes en las secciones 2 a 6.1.*

- `AAAA-MM-DD` | `[Individual | Grupal]` | **Acción:** Descripción breve | **Resultado:** OK / Parcial / Fallo — observación clave.
