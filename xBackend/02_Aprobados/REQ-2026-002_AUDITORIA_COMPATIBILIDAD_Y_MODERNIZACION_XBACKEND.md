# REQ-2026-002: Auditoría de Compatibilidad, Modernización de xBackend y Repositorio Maestro (Live + FAST 24/7)

## 1. Resumen Ejecutivo y Diagnóstico

### 1.1. Contexto
`xBackend` actúa como el Orquestador Central del ecosistema xIPTV (`console.xisrv.xyz`), proveyendo la API de catálogos, metadatos y orquestación que consumen `cpanel-admin` y los nodos distribuidos de `xManager`. Debido a la evolución de los formatos multimedia y los nuevos requerimientos de emisión ISDB-T (EPG en vivo vía Gracenote, canales FAST 24/7 y streams de backup), se requiere auditar los contratos actuales, eliminar deuda técnica y modernizar `xBackend` manteniendo su arquitectura basada en *Thin Controllers* y procedimientos PL/pgSQL en `xServerSQL`.

### 1.2. Dictamen de Viabilidad (Modernización vs. xBackend 2.0)
- **Decisión:** **Modernización Quirúrgica Incremental de xBackend**.
- **Fundamento (Doctrina Británica):** La estructura del núcleo (Express + TypeScript + PostgreSQL PL/pgSQL) es robusta y escalable. Construir un backend paralelo desde cero duplicaría el mantenimiento y rompería la compatibilidad de los nodos `xManager` en producción.
- **Acciones Clave:**
  1. Depurar endpoints legados `/v1` y consolidar contratos `/v2` y `/v3`.
  2. Implementar el modelo de **Repositorio Maestro de Canales en Vivo con Streams de Backup**.
  3. Implementar el modelo de **Canales 24/7 (FAST) con Ciclo de Vida de Contenidos y Programación por Franjas Horarias**.
  4. Implementar el **Pipeline EPG (Ingesta Gracenote + EPG Sintética por Franjas + Exportador XMLTV)** para inyección en el motor de transmisión ISDB-T.

---

## 2. Matriz de Auditoría de Endpoints (xManager $\leftrightarrow$ xBackend)

| Endpoint en xBackend | Consumidor en xManager | Procedimiento PL/pgSQL | Estado Actual | Acción de Modernización Requerida |
| :--- | :--- | :--- | :---: | :--- |
| `GET /v2/tenant/categories` | `channelsV2.ts` | `tenantv2getcategories` | **Válido** | Mantener contrato. Optimizar índices de categorías. |
| `GET /v2/tenant/channels` | `channelsV2.ts` | `tenantv2getchannels` | **Parcial** | Soportar filtros por `sourcetype` ('live', 'fast') y paginación determinista. |
| `GET /v2/tenant/movies` | `channelsV2.ts` | `tenantv2getmovies` | **Válido** | Validar rutas de egress y URLs de streaming asociadas. |
| `GET /v2/tenant/tvshows` | `channelsV2.ts` | `tenantv2gettvshows` | **Válido** | Mantener paginación con `datefrom`, `limit` y `offset`. |
| `GET /v2/tenant/episodes` | `channelsV2.ts` | `tenantv2getmovies` (`type=episode`) | **Válido** | Verificar metadatos de temporadas y orden cronológico. |
| `GET /v1/tenant/livechannels` | `admin.ts` (legacy) | N/A (hardcoded v1) | **Obsoleto** | **Migrar a `GET /v2/tenant/livechannels`** integrando fuentes principales y backup. |
| `GET /v3/channels/search/:key`| `channelsV2.ts` | `getchannelssearch` | **Válido** | Mantener soporte de búsqueda indexada. |
| `GET /v2/epg/xmltv` | *Nuevo Requerimiento* | `epg_generate_xmltv` | **Nuevo** | Generar XMLTV normalizado para ingesta en motor de streaming ISDB-T. |
| `GET /v2/epg/grid` | *Nuevo Requerimiento* | `epg_get_grid` | **Nuevo** | Retornar grilla JSON unificada (Gracenote + Sintética de Franjas). |

---

## 3. Especificación Técnica: Repositorio Maestro de Canales en Vivo

```
 ┌─────────────────────────────────────────────────────────────┐
 │                `master_live_channels` (Postgres)            │
 ├─────────────────────────────────────────────────────────────┤
 │ - channel_id (PK)                                           │
 │ - canonical_name (VARCHAR)                                  │
 │ - epg_id / station_id (Gracenote TMS ID)                    │
 │ - logo_url (VARCHAR)                                        │
 │ - category_id (FK)                                          │
 │ - lcn_default (NUMERIC)                                     │
 │ - primary_stream_url (TEXT)                                 │
 │ - backup_stream_urls (JSONB Array: [{url, priority, status}])│
 │ - health_status (VARCHAR: 'healthy', 'degraded', 'offline') │
 │ - last_health_check (TIMESTAMPTZ)                           │
 └─────────────────────────────────────────────────────────────┘
```

### 3.1. Funcionalidad de Backup Streams
1. Cada canal en vivo registra una URL principal (`primary_stream_url`) y una lista priorizada de fuentes alternativas (`backup_stream_urls`).
2. `xBackend` expone el listado consolidado a `xManager`, permitiendo al orquestador de playout conmutar automáticamente a la fuente de backup ante caídas de señal.

---

## 4. Especificación Técnica: Canales 24/7 (FAST) y EPG por Franjas Horarias

### 4.1. Programación por Franjas Horarias (Time Slots)
Para la fase inicial, los canales FAST se organizan mediante bloques de franjas horarias fijas (ej. Matinal: 06:00-12:00, Tarde: 12:00-18:00, Prime Time: 18:00-00:00, Trasnoche: 00:00-06:00):
- `fast_channel_slots`:
  - `slot_id` (PK)
  - `channel_id` (FK)
  - `slot_name` (ej. "Cine de Acción", "Maratón Comedia")
  - `start_time` / `end_time` (Horario diario recurrente)
  - `content_filter` (JSONB con reglas: categoría, etiquetas, lista de videos VOD asignados)

### 4.2. Generador de EPG Sintética ("Inventada")
- Para cada canal FAST, el motor proyecta la grilla para una ventana de **24 a 48 horas** (cumpliendo con la directiva de bajo footprint para ISDB-T).
- El evento EPG contiene el título del bloque / contenido, duración, y sinopsis TMDB heredada del archivo VOD.

---

## 5. Especificación Técnica: Pipeline EPG Gracenote y Exportación XMLTV

### 5.1. Ingesta Gracenote
- Worker periódico en `xBackend` (`base/clock.ts`) que consulta `data.tmsapi.com/v1.1/stations/:stationId/airings` para la ventana rodante de 48 horas.
- UPSERT en `programs` / `epg_airings` evitando duplicación mediante clave única `(station_id, start_time)`.

### 5.2. Generación de XMLTV para Transmisión ISDB-T
- `xBackend` genera o expone un endpoint `/v2/epg/xmltv` estructurado con el estándar DVB/XMLTV:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE tv SYSTEM "xmltv.dtd">
<tv>
  <channel id="canal-101">
    <display-name>Canal 101 HD</display-name>
    <icon src="https://assets.xisrv.xyz/images/logo_101.png" />
  </channel>
  <programme start="20260929180000 +0000" stop="20260929193000 +0000" channel="canal-101">
    <title lang="es">SportsCenter</title>
    <desc lang="es">Noticias deportivas en vivo.</desc>
    <category lang="es">Deportes</category>
  </programme>
</tv>
```
- Este archivo XMLTV es alimentado directamente al motor de streaming de `xManager` para la generación automática de tablas **EIT Present/Following y Schedule** en los moduladores ISDB-T.

---

## 6. Plan de Ejecución Segmentado (Fases del ToDo)

### Fase 1: Saneamiento y Contratos de xBackend
- [ ] Purgar rutas legadas `/v1` no utilizadas y formalizar `routerxclient.ts` v2/v3.
- [ ] Validar variables de entorno de paths multimedia (`LOCALMOUNT`, `EGRESSFOLDER`) en `config/config.ts`.
- [ ] Ejecutar compilación TypeScript (`npx tsc --noEmit`) para asegurar cero errores de tipo.

### Fase 2: DDL y Repositorio Maestro en PostgreSQL (`xServerSQL`)
- [ ] Crear DDL para `master_live_channels` (con soporte para stream principal y backups).
- [ ] Crear DDL para `fast_channel_slots` (franjas horarias) y `epg_airings` optimizada.
- [ ] Crear procedimientos PL/pgSQL: `master_get_live_channels`, `master_set_live_channel`, `epg_upsert_airings`.

### Fase 3: Worker Gracenote & Generador EPG Sintética
- [ ] Implementar cliente Gracenote robusto con ventana de 48h y control de rate-limit.
- [ ] Implementar función generadora de grilla sintética basada en franjas horarias para canales FAST.
- [ ] Endpoint `/v2/epg/xmltv` en `xBackend` con caché en memoria o almacenamiento estático rápido.

### Fase 4: Integración en xManager y Motor de Transmisión
- [ ] Actualizar `xManager/src/controllers/admin.ts` y `channelsV2.ts` para consumir los nuevos endpoints v2.
- [ ] Configurar ingesta del XMLTV en el motor de streaming de `xManager` para inyección de tablas EIT.
- [ ] Validar mapeo de Service IDs y nombres de canales contra la configuración de los moduladores ISDB-T.

### Fase 5: Protocolo de Pruebas L1, L2 y L3
- [ ] **L1 (Edge Cases):** Streams de backup caídos, respuestas vacías de Gracenote, eventos de duración nula.
- [ ] **L2 (Integración):** Validación de esquema XMLTV contra validador XML DTD y compilación de ambos proyectos.
- [ ] **L3 (Emisión Real):** Prueba de modulación en TS16 (TS de Pruebas) y verificación de lectura de EPG (Present/Following y grilla de 48h) en televisor / receptor ISDB-T.
