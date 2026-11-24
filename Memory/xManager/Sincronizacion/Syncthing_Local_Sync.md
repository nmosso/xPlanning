# Nota de Memoria Técnica: Sincronización Syncthing Local API para Canales 24/7 (REQ-2026-004)

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xManager` / `Cross_System` |
| **Componentes:** | `xManager`, `xClientSQL`, `xBackend` |
| **Fecha:** | 2026-10-17 |
| **Estado:** | Implementado y Verificado |
| **Ref SDD:** | [`REQ-2026-004`](file:///d:/Projects/DEV/xIPTV/Planning-X/Cross_System/02_Aprobados/REQ-2026-004_SYNCTHING_LOCAL_SYNC_247_CHANNELS_XMANAGER.md) |

---

## 1. Contexto y Problema Resuelto
- Antes: `xManager` realizaba un escaneo físico recursivo de disco (`fs.readdirSync`) sobre `/mnt/media/live/` para inventariar archivos de canales 24/7. Además, la descarga de metadata desde `xBackend` se realizaba con un bucle secuencial de `limit = 1` por canal.
- Ahora:
  1. Se consulta la **API REST local de Syncthing** (`127.0.0.1:8384`) consumiendo `/rest/db/browse?folder=Live3` y cruzando con `/rest/db/need?folder=Live3` para excluir descargas en curso.
  2. Los archivos validados se concilian en lote atómico en PostgreSQL local (`xclientdb`) vía el procedimiento `fastchannelsyncsyncthing`.
  3. Archivos presentes se marcan `downloadstatus = 'downloaded'`, `status = 'enabled'`, `channelvideos.status = 'present'`.
  4. Archivos borrados o removidos de Syncthing reciben **soft-delete** defensivo (R2): `downloadstatus = 'forDelete'`, `status = 'disabled'`.
  5. Playout y caché Redis se refrescan inmediatamente mediante `Channels.updateListInternal(0)`.
  6. La sincronización se ejecuta diariamente a las **07:00:00 UTC (04:00 AM hora local)** mediante `Clock.ts`.

---

## 2. Parámetro de Configuración y Circuit-Breaker (R1)
- Tabla: `params` (`paramid = 'updates.vod.files'`)
- Estructura JSON:
  ```json
  {
    "syncthing_apikey": "API_KEY",
    "update_fast": true,
    "update_movies": false,
    "update_series": false
  }
  ```
- **Circuit-Breaker (R1):** Si el parámetro no existe o `syncthing_apikey` es nula/vacía, `Clock.ts` aborta silenciosamente sin realizar ninguna operación, protegiendo servidores en etapa de comisionamiento.

---

## 3. Endpoints de Control Remoto
- **xManager:** `GET` / `POST /v2/admin/updates/vod-files` (soporta `run_now: true` para forzar ejecución manual en background con mutex anti-colisión).
- **xBackend:** `GET` / `POST /v3/tenants/:tenantid/updates/vod-files` (resuelve el nodo del tenant y reenvía la configuración y órdenes de sincronización).
