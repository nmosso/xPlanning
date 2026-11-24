# Documento de Especificación Técnica: Reordenamiento de Panel, Tickets Generados y Comando Manual en Grupo de Soporte

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-005` (Extensión de `REQ-2026-003` y `REQ-2026-004`) |
| **Sistema Destino:** | `Cross_System` (`xBackend` + `xServerSQL` + `cpanel-admin`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-11-11 |
| **Fecha de Aprobación:** | 2026-11-11 |
| **Autor(es):** | Lead Architect (Planning-X) |
| **Motor de Ejecución:** | **Claude Code** |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

Ajuste operativo sobre el asistente WhatsApp (`REQ-2026-003` / `REQ-2026-004`) para priorizar la visibilidad analítica en el panel de control, habilitar la gestión directa de tickets originados por el agente y permitir la creación inmediata de tickets desde el grupo interno de soporte mediante comando de texto:

1. **Nuevo orden de pestañas en `cpanel-admin` (`whatsapp-support`):**
   1. **Analítica de Casos** *(pestaña inicial por defecto)*
   2. **Tickets Generados** *(nueva pestaña)*
   3. **Equipo de Soporte**
   4. **Conexión & QR** *(última pestaña)*
2. **Pestaña "Tickets Generados":** Listado filtrado de `incident_tickets` creados por `whatsapp_agent`, con filtros de estado/búsqueda/archivados, botón **"Ver en Kanban"** (`?ticket=<id>`) y botón **"Archivar / Restaurar"** reutilizando el endpoint existente `/v3/incidents/:id/archive`.
3. **Comandos con Prefijo `Agente:` en Grupo Interno de Soporte (`WHATSAPP_SUPPORT_ALERT_GROUP_ID`):**
   - `Agente: Generar <tenant_id>: <Problema>` (tolerante a `Gererar`, mayúsculas/minúsculas y comillas): crea el ticket de inmediato y confirma `Cliente: <name> [<position>]`.
   - `Agente: Resumen`: devuelve en el grupo todos los tickets pendientes del Kanban (`incident_tickets` con `is_archived = false AND status <> 'terminado'`).
   - `Agente: Ayuda` / `Agente: Help`: lista los comandos disponibles del asistente.

---

## 2. Requerimientos Funcionales (Sintaxis EARS)

- **RF-01 (Orden de Pestañas en `cpanel-admin`):** *Cuando* un administrador ingrese a `/whatsapp-support`, el componente *deberá* mostrar las pestañas en el orden exacto: (1) `Analítica de Casos` (activa por defecto), (2) `Tickets Generados`, (3) `Equipo de Soporte`, (4) `Conexión & QR`. El polling de QR *solo* deberá ejecutarse mientras la pestaña `Conexión & QR` esté activa y desconectada.
- **RF-02 (Listado de Tickets del Agente):** *Cuando* se seleccione la pestaña `Tickets Generados`, `cpanel-admin` *deberá* consultar `GET /v3/whatsapp/tickets` y listar únicamente los registros de `incident_tickets` donde `created_by->>'source' = 'whatsapp_agent'`, filtrables por `status`, `tenant_id`, `is_archived` (por defecto `false`) y texto `search`.
- **RF-03 (Acciones sobre Ticket):** *Cuando* el administrador haga clic en **Ver en Kanban**, el sistema *deberá* navegar a `/kanban` con `?ticket=<id>`. *Cuando* haga clic en **Archivar** (o **Restaurar** si visualiza archivados), el sistema *deberá* invocar `PUT /v3/incidents/:id/archive` con `operator_identity_id` y refrescar el listado.
- **RF-04 (Aislamiento del Grupo Interno de Soporte):** *Cuando* llegue un webhook `messages.upsert` cuyo `group_id` sea igual a `WHATSAPP_SUPPORT_ALERT_GROUP_ID`, el sistema *no deberá* abrir sesiones de cliente (5 min / 30 min) ni evaluar analítica de conversación; únicamente deberá inspeccionar si el texto coincide con un comando `Agente:`.
- **RF-05 (Comando `Agente: Generar <ID>: <Problema>`):** *Cuando* un participante (`fromMe == false`) dentro de `WHATSAPP_SUPPORT_ALERT_GROUP_ID` envíe un mensaje que coincida con `^\s*agente\s*:\s*(?:generar|gererar)\s+([0-9a-zA-Z_-]+)\s*:\s*([\s\S]+)$` (case-insensitive), `xBackend` *deberá* responder en el acto (sin espera de 5 min):
  1. Deduplicar por `message_id` en Redis (`wa:dedup:{message_id}`).
  2. Quitar comillas envolventes opcionales (`"..."`, `'...'`, `“...”`, `‘...’`) y espacios sobrantes.
  3. Invocar inmediatamente `fn_whatsapp_create_kanban_ticket` con `tenant_id = <ID>`, `summary = <Problema>`, `category = 'TICKET_SOPORTE'` y metadatos `origin: 'manual_command'`, `operator_phone`, `operator_name`.
  4. Enviar confirmación al mismo `WHATSAPP_SUPPORT_ALERT_GROUP_ID` vía `gateway.sendSupportAlert` con `Cliente: <tenant_name> [<tenant_position>]` y `Detalle: <Problema>` (sin `ID:`).
- **RF-06 (Comando `Agente: Resumen`):** *Cuando* un participante (`fromMe == false`) dentro de `WHATSAPP_SUPPORT_ALERT_GROUP_ID` envíe `^\s*agente\s*:\s*resumen\s*$` (case-insensitive), `xBackend` *deberá* invocar en el acto `fn_whatsapp_kanban_summary` para obtener todos los tickets de `public.incident_tickets` con `is_archived = false AND status <> 'terminado'` y enviar el listado agrupado por estado (`critico`, `pendiente`, `testing`, `verificar`) al mismo grupo interno.
- **RF-07 (Comando `Agente: Ayuda` / `Agente: Help`):** *Cuando* un participante (`fromMe == false`) dentro de `WHATSAPP_SUPPORT_ALERT_GROUP_ID` envíe `^\s*agente\s*:\s*(?:ayuda|help)\s*$` (case-insensitive), `xBackend` *deberá* responder en el acto en el mismo grupo interno con la lista de comandos disponibles y ejemplos de uso.

---

## 3. Filtro Doctrinal (Simplicidad Radical)

- **Reutilización de `incidentarchive` y `fn_whatsapp_create_kanban_ticket`:**
  - Para archivar tickets desde la nueva pestaña se reutiliza el endpoint existente `PUT /v3/incidents/:id/archive` (`IncidentsController.archive`), evitando duplicar lógica de archivado y auditoría en `incident_ticket_logs`.
  - Para el comando manual en el grupo de soporte se reutiliza `fn_whatsapp_create_kanban_ticket` existente (que ya valida el `tenant_id` contra `tenants` sin fallar por FK `23503` si el ID no existe, guardando el `tenant_tag` en `created_by`).
- **Único RPC nuevo:** `fn_whatsapp_tickets_query` para consultar exclusivamente tickets con `created_by->>'source' = 'whatsapp_agent'` sin alterar el `incidentlist` del Kanban general.

---

## 4. Contrato SQL (`xServerSQL`)

### 4.1. Nuevo RPC: `fn_whatsapp_tickets_query`
*Ubicación: `sql/xServerSQL/_Functions/WhatsApp/fn_whatsapp_tickets_query.sql`*

```sql
CREATE OR REPLACE FUNCTION public.fn_whatsapp_tickets_query(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_tenant_id VARCHAR;
    v_status VARCHAR;
    v_is_archived BOOLEAN;
    v_search VARCHAR;
    v_limit INT;
    v_offset INT;
    v_items JSONB;
    v_total INT;
BEGIN
    v_tenant_id   := NULLIF(TRIM(COALESCE(_data->>'tenant_id', '')), '');
    v_status      := NULLIF(TRIM(COALESCE(_data->>'status', '')), '');
    v_is_archived := COALESCE(NULLIF(_data->>'is_archived', '')::boolean, false);
    v_search      := NULLIF(TRIM(COALESCE(_data->>'search', '')), '');
    v_limit       := LEAST(GREATEST(COALESCE(NULLIF(_params->>'limit', '')::int, 50), 1), 200);
    v_offset      := GREATEST(COALESCE(NULLIF(_params->>'offset', '')::int, 0), 0);

    IF v_search IS NOT NULL THEN
        v_search := regexp_replace(v_search, '([%_\\])', '\\\1', 'g');
    END IF;

    SELECT COUNT(*)::int INTO v_total
    FROM public.incident_tickets it
    LEFT JOIN public.tenants t ON it.tenant_id = t.tenantid
    WHERE it.created_by->>'source' = 'whatsapp_agent'
      AND it.is_archived = v_is_archived
      AND (v_tenant_id IS NULL OR it.tenant_id = v_tenant_id OR it.created_by->>'tenant_tag' = v_tenant_id)
      AND (v_status IS NULL OR it.status = v_status)
      AND (v_search IS NULL OR (
          it.title ILIKE '%' || v_search || '%' ESCAPE '\' OR
          COALESCE(it.description, '') ILIKE '%' || v_search || '%' ESCAPE '\' OR
          COALESCE(t.name, '') ILIKE '%' || v_search || '%' ESCAPE '\' OR
          COALESCE(it.created_by->>'group_name', '') ILIKE '%' || v_search || '%' ESCAPE '\'
      ));

    SELECT COALESCE(jsonb_agg(row_to_json(q)::jsonb), '[]'::jsonb) INTO v_items
    FROM (
        SELECT
            it.id,
            it.tenant_id,
            t.name AS tenant_name,
            it.title,
            it.description,
            it.type,
            it.status,
            it.deadline,
            it.is_archived,
            it.created_at,
            it.updated_at,
            it.created_by->>'group_name' AS group_name,
            it.created_by->>'tenant_tag' AS tenant_tag,
            it.created_by->>'case_id' AS case_id
        FROM public.incident_tickets it
        LEFT JOIN public.tenants t ON it.tenant_id = t.tenantid
        WHERE it.created_by->>'source' = 'whatsapp_agent'
          AND it.is_archived = v_is_archived
          AND (v_tenant_id IS NULL OR it.tenant_id = v_tenant_id OR it.created_by->>'tenant_tag' = v_tenant_id)
          AND (v_status IS NULL OR it.status = v_status)
          AND (v_search IS NULL OR (
              it.title ILIKE '%' || v_search || '%' ESCAPE '\' OR
              COALESCE(it.description, '') ILIKE '%' || v_search || '%' ESCAPE '\' OR
              COALESCE(t.name, '') ILIKE '%' || v_search || '%' ESCAPE '\' OR
              COALESCE(it.created_by->>'group_name', '') ILIKE '%' || v_search || '%' ESCAPE '\'
          ))
        ORDER BY
            CASE WHEN it.status = 'critico' THEN 1
                 WHEN it.status = 'pendiente' THEN 2
                 WHEN it.status = 'testing' THEN 3
                 WHEN it.status = 'verificar' THEN 4
                 WHEN it.status = 'terminado' THEN 5
                 ELSE 6 END ASC,
            it.created_at DESC
        LIMIT v_limit OFFSET v_offset
    ) q;

    RETURN jsonb_build_object(
        'success', true,
        'total', v_total,
        'items', v_items,
        'limit', v_limit,
        'offset', v_offset
    );
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'code', SQLSTATE);
END;
$$;
```

### 4.2. Nuevo RPC: `fn_whatsapp_kanban_summary`
*Ubicación: `sql/xServerSQL/_Functions/WhatsApp/fn_whatsapp_kanban_summary.sql`*
*(Devuelve todos los tickets activos del Kanban con `is_archived = false AND status <> 'terminado'`, independientemente de su origen).*

```sql
CREATE OR REPLACE FUNCTION public.fn_whatsapp_kanban_summary(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_items JSONB;
BEGIN
    SELECT COALESCE(jsonb_agg(row_to_json(q)::jsonb), '[]'::jsonb) INTO v_items
    FROM (
        SELECT
            it.id,
            it.status,
            it.type,
            it.title,
            it.tenant_id,
            t.name AS tenant_name,
            t.position AS tenant_position,
            it.created_by->>'tenant_tag' AS tenant_tag,
            it.created_at
        FROM public.incident_tickets it
        LEFT JOIN public.tenants t ON it.tenant_id = t.tenantid
        WHERE it.is_archived = false
          AND it.status <> 'terminado'
        ORDER BY
            CASE WHEN it.status = 'critico' THEN 1
                 WHEN it.status = 'pendiente' THEN 2
                 WHEN it.status = 'testing' THEN 3
                 WHEN it.status = 'verificar' THEN 4
                 ELSE 5 END ASC,
            it.created_at DESC
    ) q;

    RETURN jsonb_build_object(
        'success', true,
        'total', jsonb_array_length(v_items),
        'items', v_items
    );
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'code', SQLSTATE);
END;
$$;
```

---

## 5. Cambios en `xBackend`

### 5.1. Endpoint Nuevo (`server/whatsappConsoleRoutes.ts` y `controllers/whatsapp.ts`)
- **`GET /v3/whatsapp/tickets`** protegido con `[validOrigin('console'), siteApikey()]`:
  - Query params: `tenant_id`, `status`, `is_archived` (default `false`), `search`, `limit`, `offset`.
  - Llama a `fn_whatsapp_tickets_query`.

### 5.2. Procesamiento de Comandos `Agente:` en Grupo Interno de Soporte
- **Parser en `services/whatsapp/supportCaseRules.ts`:**
  ```ts
  export type SupportGroupCommand =
      | { kind: 'generate'; tenant_tag: string; text: string }
      | { kind: 'summary' }
      | { kind: 'help' };

  export function parseSupportGroupCommand(rawText: string): SupportGroupCommand | null
  ```
  - **`help`:** `/^\s*agente\s*:\s*(?:ayuda|help)\s*$/i`
  - **`summary`:** `/^\s*agente\s*:\s*resumen\s*$/i`
  - **`generate`:** `/^\s*agente\s*:\s*(?:generar|gererar)\s+([0-9a-zA-Z_-]+)\s*:\s*([\s\S]+)$/i` (limpia comillas envolventes `"..."`, `'...'`, `“...”`, `‘...’` y `.trim()`; si el problema queda vacío retorna `null`).
- **Flujo en `SupportGroupCommandHandler` (`services/whatsapp/manualTicketCommand.ts`):**
  1. Deduplica con `store.setNx('wa:dedup:' + message_id, '1', 120)`.
  2. Evalúa `parseSupportGroupCommand(msg.text)`. Si es `null`, retorna `{ status: 'ignored' }`.
  3. **`kind === 'help'`:** envía por `gateway.sendSupportAlert`:
     ```text
     🤖 Comandos del Asistente
     • Agente: Resumen — Lista los tickets pendientes del Kanban (excepto terminados y archivados).
     • Agente: Generar <ID>: <Problema> — Crea un ticket inmediato para el cliente <ID>.
     • Agente: Ayuda (o Agente: Help) — Muestra esta ayuda.
     ```
  4. **`kind === 'summary'`:** invoca `rpc('fn_whatsapp_kanban_summary', {}, {}, OPERATOR)` y formatea la lista agrupada por estado (`🔴 CRÍTICO`, `🟡 PENDIENTE`, `🔵 TESTING`, `🟣 VERIFICAR`) mostrando `• <tenant_name> [<tenant_position>]: <title>` (o `[Global]` si no tiene tenant). Si hay 0 activos, responde `✅ No hay tickets pendientes en el Kanban.`.
  5. **`kind === 'generate'`:** invoca `fn_whatsapp_create_kanban_ticket` (que busca el tenant por `tenantid IN (v_tenant_tag, 'server' || v_tenant_tag) OR position::text = v_tenant_tag` y devuelve `tenant_name` y `tenant_position`) y confirma:
     ```text
     ✅ Ticket creado
     Cliente: <tenant_name> [<tenant_position>]
     Detalle: <cmd.text>
     ```
     *(Si el tenant no existe en `public.tenants`, muestra `Cliente: [<cmd.tenant_tag>]`).*

---

## 6. Cambios en `cpanel-admin`

- **`src/app/core/shared/utils/const.ts`**: Agregar `whatsappTickets: '/v3/whatsapp/tickets'` a `ENDPOINTS`.
- **`whatsapp-support.service.ts`**:
  - `listGeneratedTickets(filters)` $\to$ `GET /v3/whatsapp/tickets`.
  - `archiveTicket(id, is_archived)` $\to$ `PUT /v3/incidents/:id/archive` (incluyendo `operator_identity_id` desde `localStorage` igual que `KanbanService`).
- **`whatsapp-support.component.ts` y `.html`**:
  - `type WaTab = 'cases' | 'tickets' | 'staff' | 'connection';`
  - `activeTab: WaTab = 'cases';`
  - `ngOnInit()` carga `this.casesReload$.next()` inicialmente (no llama a `loadStatus()` hasta que se entre a la pestaña `connection`).
  - Al salir de `connection` (`selectTab`), detener el polling de QR (`stopPolling()`).
  - Pestaña **Tickets Generados**: filtros (`Estado`, `Tenant`, `Archivados` checkbox/select, `Búsqueda`), tabla con Tenant, Título/Descripción, Estado/Tipo, Fecha, botón **"Ver en Kanban"** (`[routerLink]="['/kanban']" [queryParams]="{ ticket: t.id }"`) y botón **"Archivar"** / **"Restaurar"**.
  - Regla **No-Astra** estricta (cero menciones a "Astra"/"Astraserver").
  - **Prohibido ejecutar `ng build`, `npm run build` o `pack-admin.sh`** (solo verificar con `tsc --noEmit` / `ngc --noEmit`).

---

## 7. Plan de Tareas Secuenciales para Claude Code (<30 min c/u)

- [ ] **T1: RPC `fn_whatsapp_tickets_query` + Tests SQL**
  - **RFs:** `RF-02`
  - **Archivos:** `sql/xServerSQL/_Functions/WhatsApp/fn_whatsapp_tickets_query.sql`, `test/whatsapp/sql/25_req005_tests.sql`.
  - **Hecho cuando:** `npm run test:sql` pasa validando filtro estricto `source = 'whatsapp_agent'`, archivados, búsqueda literal y paginación.

- [ ] **T2: Comando manual `Generar <ID>: <Datos>` en grupo interno de soporte + Tests**
  - **RFs:** `RF-04, RF-05`
  - **Archivos:** `xBackend/services/whatsapp/supportCaseRules.ts`, `xBackend/services/whatsapp/manualTicketCommand.ts` (o dentro de `whatsappAgent.ts`), `xBackend/controllers/whatsapp.ts`, `test/whatsapp/manualCommand.test.ts`.
  - **Hecho cuando:** Tests unitarios cubren `Generar 43: "..."`, `gererar 43: ...`, sin comillas, texto vacío ignorado, deduplicación por `message_id`, aislamiento de `WHATSAPP_SUPPORT_ALERT_GROUP_ID` (no abre caso 5/30 min) y envío de confirmación al mismo grupo.

- [ ] **T3: Endpoint `GET /v3/whatsapp/tickets` en `xBackend`**
  - **RFs:** `RF-02`
  - **Archivos:** `xBackend/controllers/whatsapp.ts`, `xBackend/server/whatsappConsoleRoutes.ts`, `test/whatsapp/consoleApi.test.ts`.
  - **Hecho cuando:** `npx tsc --noEmit` y `npm test` pasan al 100% incluyendo el guard `[validOrigin('console'), siteApikey()]`.

- [ ] **T4: Reordenamiento de pestañas y pestaña `Tickets Generados` en `cpanel-admin`**
  - **RFs:** `RF-01, RF-02, RF-03`
  - **Archivos:** `cpanel-admin/src/app/core/shared/utils/const.ts`, `modules/common/whatsapp-support/whatsapp-support.*`.
  - **Hecho cuando:** `tsc --noEmit` y `ngc --noEmit` pasan con 0 errores (SIN ejecutar `ng build`, `npm run build` ni `pack-admin.sh`) y cero apariciones de "Astra".

---

## 8. Escenario de Pruebas (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Casos Borde                                  │
│ • "Generar 43: Se cayó ESPN" y "gererar 43: \"Se cayó\""    │
│   crean ticket inmediato; "Generar 43:   " se ignora.       │
│ • Charla normal en grupo interno de soporte ("ok gracias")  │
│   NO abre sesión en Redis ni crea casos ni tickets.         │
│ • Tenant "999" inexistente en comando manual: crea ticket   │
│   con tenant_id=NULL y tenant_tag="999" sin romper FK.      │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Contratos e Integración                      │
│ • npx tsc --noEmit, npm test y npm run test:sql limpios.    │
│ • cpanel-admin: tsc --noEmit y ngc --noEmit sin errores.    │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Concurrencia y Resiliencia                   │
│ • Reintento de webhook del mismo message_id en comando:     │
│   dedup en Redis evita doble creación y doble confirmación. │
│ • Cambio rápido de pestañas en Angular no deja polling QR   │
│   corriendo en segundo plano.                               │
└─────────────────────────────────────────────────────────────┘
```
