# Handoff de Implementación para Claude Code — REQ-2026-011 (Etapa 1)

> **Propósito:** Documento operativo para que **Claude Code** implemente el paso intermedio de catalogación y publicación a producción entre `filesin` (`/mnt/media/storage/`), `xBackend` / `xServerSQL` y `cpanel-admin`.
> **Especificación Madre Aprobada:** [`REQ-2026-011_CATALOGACION_Y_PUBLICACION_INTERMEDIA_FILESIN_CPANEL.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Cross_System/02_Aprobados/REQ-2026-011_CATALOGACION_Y_PUBLICACION_INTERMEDIA_FILESIN_CPANEL.md)

---

## Prompt Directo para Claude Code

Copia y pega este bloque en **Claude Code** (trabajando sobre `D:\Projects\DEV\xIPTV\sql\xServerSQL`, `D:\Projects\DEV\xIPTV\xBackend` y `D:\Projects\DEV\xIPTV\cpanel`):

```text
Lee la especificación aprobada en:
D:\Projects\DEV\xIPTV\Planning-X\Cross_System\02_Aprobados\REQ-2026-011_CATALOGACION_Y_PUBLICACION_INTERMEDIA_FILESIN_CPANEL.md

Implementa la Etapa 1 del requerimiento REQ-2026-011 respetando estrictamente el patrón RPC JSONB de xBackend/xServerSQL y las reglas de Planning-X:

1. Base de Datos (D:\Projects\DEV\xIPTV\sql\xServerSQL):
   - Crea el script de migración idempotente (ADD COLUMN IF NOT EXISTS) sobre `media_upload_logs` para agregar:
     `catalog_status varchar(20) NOT NULL DEFAULT 'pending'`, `published_at timestamptz`, `published_by varchar`, `published_path text`, `catalog_ref_id varchar`.
     (Índice recomendado sobre `(catalog_status, content_type)`).
   - Crea las funciones PL/pgSQL con firma estándar `(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`:
     * `catalogstaginglist`: devuelve registros de `media_upload_logs` con `status in ('completed', 'duplicate_skipped')` y `catalog_status = 'pending'`, filtrables por `contentType`.
     * `catalogstagingmark`: actualiza atómicamente `catalog_status` ('published' | 'rejected'), `published_at`, `published_by`, `published_path`, `catalog_ref_id` solo si `catalog_status = 'pending'` (evita doble publicación concurrente).

2. Backend (D:\Projects\DEV\xIPTV\xBackend):
   - Crea un Thin Controller `CatalogStagingController` bajo `/v3/catalog-staging/`:
     * `GET /v3/catalog-staging/pending`: consulta `catalogstaginglist` y adjunta `existsOnDisk` verificando `remote_path` en `/mnt/media/storage/`.
     * `POST /v3/catalog-staging/probe`: valida que `remote_path` esté dentro de `/mnt/media/storage/` (sin path traversal) y ejecuta `ffprobe -v quiet -print_format json -show_format -show_streams` devolviendo duración, tamaño, codec video/audio y resolución.
     * `POST /v3/catalog-staging/publish`:
       - Valida que `catalog_status = 'pending'` y que el archivo exista en `/mnt/media/storage/`.
       - Según `contentType`:
         * `24_7_channel`: ejecuta `ffprobe`, llama a `fastvideoregister` (en PostgreSQL) con `localpath = 'live3/<channel_slug>'`, y mueve localmente (`fs.renameSync` con fallback `copyFileSync + unlinkSync` si cruza mountpoints) hacia `/mnt/media/live3/<channel_slug>/<filename>`. Si falla `fastvideoregister` tras mover, revierte el movimiento a `/mnt/media/storage/`.
         * `movie`: mueve localmente hacia `/mnt/media/movies/<filename>` (sin Syncthing).
         * `tv_show`: mueve localmente hacia `/mnt/media/series/<filename>` (sin Syncthing).
       - Llama a `catalogstagingmark` con `catalog_status = 'published'`.
     * `POST /v3/catalog-staging/reject`: marca `catalog_status = 'rejected'` y opcionalmente elimina o mueve el archivo de staging.
   - Registra las rutas en el router `/v3` de `xBackend`.
   - Ejecuta `npx tsc --noEmit` en `xBackend` para validar compilación limpia.

3. Frontend (D:\Projects\DEV\xIPTV\cpanel / cpanel-admin):
   - Agrega la vista "Catalogación de Ingesta (Staging)" en `cpanel-admin` consumiendo `/v3/catalog-staging/*`, con pestañas para Canales 24/7, Películas y Series, botón de inspección `ffprobe` y modal de confirmación/publicación (recordando UI neutral No-Astra).
   - REGLA ESTRICTA: NO ejecutar `ng build`, `npm run build` ni `pack-admin.sh` en `cpanel-admin` desde la máquina local.

4. Memoria y Cierre:
   - Actualiza los archivos `MEMORY.md` correspondientes (máximo 150 líneas).
```
