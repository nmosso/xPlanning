# Handoff de Implementación para Claude Code — REQ-2026-009

> **Propósito:** Documento operativo para que **Claude Code** ajuste/mantenga `xVodagent` en `D:\Projects\DEV\xIPTV\xVodagent` y despliegue el servicio en el servidor Linux `192.168.7.232`.
> **Especificación Madre Aprobada:** [`REQ-2026-009_TRANSFORMACION_APPLIANCE_TRANSCODING_LINUX_SAMBA_XVODAGENT.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Cross_System/02_Aprobados/REQ-2026-009_TRANSFORMACION_APPLIANCE_TRANSCODING_LINUX_SAMBA_XVODAGENT.md)

---

## 0. Prompt de Ajuste Inmediato para Claude Code (Acciones Bulk Agrupadas en "Requiere Acción")

Ejecuta este prompt en **Claude Code** dentro de `D:\Projects\DEV\xIPTV\xVodagent`:

```text
Ajuste de procesamiento en lote (Bulk) agrupado por tipo de acción en "⚠️ Requiere acción" de xVodagent (REQ-2026-009) y re-despliegue al servidor appliance (100.64.0.5:/opt/xvodagent):

OBJETIVO:
Cuando entran 20–50+ archivos que requieren intervención del operador (por ejemplo, todos necesitan selección de pista de audio o todos esperan aprobación), procesarlos uno por uno es lento. Hay que agrupar los ítems de "⚠️ Requiere acción" por tipo (`paused_audio_review` y `awaiting_approval`) y permitir seleccionarlos y procesarlos todos juntos en una sola acción bulk.

CAMBIOS REQUERIDOS EN D:\Projects\DEV\xIPTV\xVodagent:

1. Endpoint único Bulk en Backend (`src/ingest/ingestService.ts` + `src/ingest/ingestRoutes.ts`):
   - Agregar `POST /api/ingest/bulk` con payload JSON:
     `{ action: 'audio' | 'approve' | 'reject' | 'purge', ids: string[], trackIndex?: number, mode?: 'passthrough' | 'transcode', videoBitrate?: string }`
   - ¿Por qué un solo endpoint?: evita disparar 50 POSTs simultáneos desde el navegador contra el `limit_req burst=40` de Nginx.
   - Validación y ejecución tolerante por ítem:
     * Validar `Array.isArray(ids)` no vacío (máx. 200) y `action` válido (400 si no).
     * Para `action === 'audio'`: validar `trackIndex` entero >= 0. Por cada `id`, verificar que exista y tenga esa pista en `item.audioTracks` antes de invocar la confirmación de audio; si un ítem no existe, está en otro estado o no tiene esa pista, NO abortar el lote: registrar `{ id, fileName, error }` en `failed` y continuar con el resto.
     * Para `action === 'approve'`: aprobar cada ítem (usando `mode` y `videoBitrate` si se especificaron, o su plan recomendado por defecto si no).
     * Para `action === 'reject'` y `action === 'purge'`: rechazar o purgar cada ítem del lote.
     * Responder `200` con `{ ok: number, failed: Array<{ id: string, fileName?: string, error: string }> }`.

2. Agrupación y Barra Bulk en `src/services/ingestUi.ts` dentro de `⚠️ Requiere acción`:
   - Dividir el panel `#ig-pane-attn` en dos grupos claros (mostrando solo los grupos que tengan ítems > 0):
     a) **Grupo 1 — `🗣 Revisión de audio (N)` (`paused_audio_review`):**
        - Barra de herramientas del grupo (`.ig-bulk-bar`):
          * Checkbox / botón **"Seleccionar todos (N)"** / **"Ninguno"** para este grupo.
          * Selector `<select>` con las pistas disponibles en los ítems del grupo (ej. `#1 · aac 2ch (presente en X/N archivos)` — ordenadas por índice de pista; si algún ítem no tiene pistas de audio, indicarlo).
          * Botón **`🗣 Confirmar pista en seleccionados (K)`** → llama a `POST /api/ingest/bulk` con `{ action: 'audio', ids, trackIndex }`.
          * Botón **`🗑 Eliminar seleccionados (K)`** (con `confirm(...)`) → útil para purgar en lote archivos sin audio o malos.
     b) **Grupo 2 — `✔ Esperando aprobación (N)` (`awaiting_approval`):**
        - Barra de herramientas del grupo (`.ig-bulk-bar`):
          * Checkbox / botón **"Seleccionar todos (N)"** / **"Ninguno"** para este grupo.
          * Botón **`✔ Aprobar según plan (K)`** → `POST /api/ingest/bulk` con `{ action: 'approve', ids }`.
          * Input opcional de bitrate + botón **`⚙ Transcodificar seleccionados (K)`** → `POST /api/ingest/bulk` con `{ action: 'approve', ids, mode: 'transcode', videoBitrate }`.
          * Botones **`✖ Rechazar (K)`** y **`🗑 Eliminar (K)`** (con `confirm(...)`).
   - Checkbox en cada `.ig-item` dentro de `attn`:
     * Persistir `selectedIds = {}` en memoria dentro de `ingestUi.ts` (igual que `expandedIds`), limpiando los IDs que ya salieron de `attn`, para que el polling cada 3s **NO borre** la selección del operador.
     * Incluir los `<select>` e `<input>` de la barra bulk en `editing()` (`data-ig-input="1"`) para que el refresco cada 3s no interrumpa mientras el operador elige una pista o escribe un bitrate.
     * Si el endpoint devuelve `failed.length > 0`, mostrar un resumen claro con cuántos se procesaron (`ok`) y qué archivos fallaron y por qué.

3. Verificación y Despliegue:
   - Agregar/actualizar tests en `test:ingest` cubriendo `POST /api/ingest/bulk` (audio en lote, aprobación en lote, ítem sin pista que falla parcialmente sin abortar el resto, purge en lote).
   - Ejecutar `npx tsc --noEmit`, `npm run build`, `npm test`, `npm run test:transcoder` y `npm run test:ingest`.
   - Desplegar `dist/` en `100.64.0.5:/opt/xvodagent` (usuario `ubuntu`, llave `D:\Projects\secureXIPTV\Launchpad\launchpad_key.pem` o `/tmp/launchpad_key.pem`) y ejecutar `sudo systemctl restart xvodagent`.
   - Actualizar `MEMORY.md` (<150 líneas).
```

---

## 1. Referencia Completa de Arquitectura (REQ-2026-009)

- **Host:** `192.168.7.232` (Ubuntu 26.04.1 LTS, Kernel 7.0 x86_64, Node.js `v22.22.1`, FFmpeg `8.0.1`, rsync `3.4.1`)
- **GPU Activa:** Intel UHD Graphics 630 en `/dev/dri/renderD128` (`h264_vaapi` probado; `h264_qsv` no soportado por falta de MFX session)
- **Almacenamiento (356 GiB LVM `/mnt/media`):**
  - **Vista Operador (Samba `[xIPTV-Ingest]` + Transcoder UI `:48888`):** `/mnt/media/shared/247/<canal_247>/`
  - **Interno Servicio (oculto al operador):** `/mnt/media/ingest/{processing, completed, failed, archive}`
- **Systemd Unit:** `/etc/systemd/system/xvodagent.service` (`ExecStart=/usr/bin/node /opt/xvodagent/dist/vodagent.js`, user `ubuntu`, grupos `video, render`, env `VAAPI_DEVICE=/dev/dri/renderD128`)

---

## 2. Datos de Acceso y Despliegue al Servidor (`192.168.7.232`)

- **Host:** `192.168.7.232`
- **Usuario:** `ubuntu` (sudo sin contraseña habilitado)
- **Llave SSH:** `D:\Projects\secureXIPTV\Launchpad\launchpad_key.pem` (en WSL copiar a `/tmp/lk.pem` con `chmod 600` si Windows rechaza permisos)
- **Ruta de Despliegue:** `/opt/xvodagent`
- **Servicio Systemd:** `sudo systemctl restart xvodagent && sudo journalctl -u xvodagent -f`
- **Web UI Local del Monitor/Transcoder:** `http://192.168.7.232:48888`
- **Carpeta Compartida Samba para Editores:** `\\192.168.7.232\xIPTV-Ingest\247\<canal>\`
