# Documento de Especificación Técnica: Normalización Masiva de `CLIENT=X` en `/etc/environment` (Proxmox, xManager, Multicast y Streamer)

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-006` |
| **Sistema Destino:** | `xWorker` (`src/normalize_etc_environment_client.ts`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-10-19 |
| **Última Actualización:** | 2026-10-19 |
| **Autor(es):** | Lead Architect (Antigravity) & Usuario |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

Durante el relevamiento en vivo de cabeceras para el proyecto **Heartbeat & Monitoring (`REQ-2026-005`)**, se detectó que la variable de entorno de identificación del cliente en `/etc/environment` presenta inconsistencias entre servidores y entre nodos de una misma cabecera:
- En algunas VMs `xManager` (ej. `100.64.23.8`, `100.64.29.8`) existe como `CLIENT=23` y `CLIENT=29`.
- En otras VMs `xManager` (ej. `100.64.83.8`) existe con prefijo de texto `CLIENT=client83`.
- En otras VMs `xManager` (ej. `100.64.93.8`), así como en los nodos **Proxmox** (`100.64.X.2:22`), **Multicast** (`100.64.X.2:8522`) y **Streamer** (`100.64.X.2:8222`), la variable no está definida en `/etc/environment`.

Este requerimiento define la automatización en **`xWorker`** para normalizar de forma idempotente y segura el archivo `/etc/environment` en los **4 nodos de cada cabecera** (`Proxmox`, `xManager`, `Multicast` y `Streamer`), asegurando que todos contengan estrictamente:
```dotenv
CLIENT=X
```
*(Donde `X` es el número entero `position` del tenant en `public.tenants`, ej. `CLIENT=83`, **nunca** `client83`).*

---

## 2. Filtro Doctrinal (Ingeniería xIPTV)

1. **Filtro Británico (*Keep it Plain*):** Un único script TypeScript en `xWorker` (`src/normalize_etc_environment_client.ts`) que obtiene la lista dinámica desde PostgreSQL (`public.tenants`) y ejecuta la normalización vía SSH sobre los 4 nodos por cabecera.
2. **Filtro Alemán (*Gründlichkeit*):**
   - Idempotencia estricta: Si `/etc/environment` ya tiene `CLIENT=X` exacto, no modifica el archivo (`UNCHANGED`).
   - Si tiene `CLIENT=clientX` o cualquier otro valor distinto de `X`, o si existen líneas duplicadas de `CLIENT=`, las reemplaza limpiamente por una única línea `CLIENT=X` (`UPDATED`), preservando intacto `PATH=...` y cualquier otra variable existente.
   - Si no tiene `CLIENT=`, agrega `CLIENT=X` al final del archivo (`ADDED`).
3. **Filtro Ruso (*Nadezhnost* - L1/L2/L3):**
   - **L1 (Casos Borde):** Creación de backup previo (`/etc/environment.bak`) antes de modificar; preservación de permisos `0644` (`root:root`); verificación post-escritura con `grep -E '^CLIENT=[0-9]+$' /etc/environment`.
   - **L2 (Contratos y Build):** `npx tsc --noEmit` con 0 errores en `xWorker`.
   - **L3 (Resiliencia en Lote y Timeout SSH de 30s):** Timeout explícito de conexión SSH de **`30 segundos` (`30000 ms`)** por nodo para verificar fehacientemente si el nodo no responde; si un nodo o puerto NAT está apagado/inalcanzable, se registra el error en el LOG y el worker continúa con el resto de los nodos sin abortar el bloque.
   - **Estrategia de Despliegue por Bloques:** Se ejecuta primero de forma exclusiva sobre el **Server 93** a modo de piloto. Luego se avanza por bloques explícitos de servidores (`TARGETS_JSON`), **nunca** ejecutando la flota completa de una sola vez.

---

## 3. Matriz de Nodos Objetivo por Cabecera (`position = X`)

| # | Nodo | Host SSH | Puerto SSH | Usuario SSH | Escalado Root | Timeout SSH | Valor Objetivo en `/etc/environment` |
| :--- | :--- | :--- | :---: | :--- | :--- | :---: | :--- |
| **1** | **Proxmox Host** | `100.64.X.2` | `22` | `root` | Directo | `30s` | `CLIENT=X` |
| **2** | **VM `xManager`** | `100.64.X.8` | `22` | `ubuntu` | `sudo` | `30s` | `CLIENT=X` |
| **3** | **VM `Multicast`** | `100.64.X.2` | `8522` | `ubuntu` | `sudo` | `30s` | `CLIENT=X` |
| **4** | **VM `Streamer`** | `100.64.X.2` | `8222` *(exclusivo)* | `ubuntu` | `sudo` | `30s` | `CLIENT=X` |

---

## 4. Comando Idempotente de Normalización Remota

En cada uno de los 4 nodos se ejecutará con privilegios de `root` (`sudo bash -c`):

```bash
TARGET_VAL="X"
cp /etc/environment /etc/environment.bak
if grep -qE "^CLIENT=" /etc/environment; then
  sed -i -E "s/^CLIENT=.*/CLIENT=${TARGET_VAL}/" /etc/environment
else
  echo "CLIENT=${TARGET_VAL}" >> /etc/environment
fi
chmod 644 /etc/environment
grep -E "^CLIENT=" /etc/environment
```

> [!IMPORTANT]
> **Impacto en Scripts Existentes (`/root/cron/sqlbackup.sh` y `/home/ubuntu/cron/syncbackup.sh`):**
> Se verificó en `100.64.23.8` (`CLIENT=23`) que `sqlbackup.sh` y `syncbackup.sh` utilizan `$CLIENT` para la ruta `s3://xisrv/clients/$CLIENT/...`. Normalizar `CLIENT=X` mantiene 100% la estructura estándar `s3://xisrv/clients/X/sql/` y `s3://xisrv/clients/X/sync/`.

---

## 5. Entregables y Criterios de Aceptación (DoD)

- [ ] Script [`xWorker/src/normalize_etc_environment_client.ts`](file:///d:/Projects/DEV/xIPTV/xWorker/src/normalize_etc_environment_client.ts) implementado bajo `strict: true` sin IPs ni credenciales hardcodeadas y con timeout SSH de `30s` (`30000 ms`).
- [ ] Ejecución controlada por bloques vía `TARGETS_JSON` (comenzando exclusivamente con `server93`).
- [ ] Compilación limpia (`npx tsc --noEmit` = 0 errores).
- [ ] Generación automática de archivo de bitácora en `xWorker/logs/YYYY-MM-DD_HHMMSS_normalize_env_server93.log` y reporte Markdown en `xWorker/reports/YYYY-MM-DD_HHMMSS_normalize_env_server93.md` detallando el estado antes/después de los 4 nodos.
- [ ] Actualización de `xWorker/MEMORY.md` al finalizar la ejecución.
