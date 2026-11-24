# Documento de Especificación Técnica: Panel de Gestión, Vinculación QR y Analítica del Asistente WhatsApp

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-004` (Extensión de `REQ-2026-003`) |
| **Sistema Destino:** | `Cross_System` (`xBackend` + `xServerSQL` + `cpanel-admin`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-11-11 |
| **Fecha de Aprobación:** | 2026-11-11 |
| **Autor(es):** | Lead Architect (Planning-X) |
| **Motor de Ejecución:** | **Claude Code** |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

`REQ-2026-003` implementó el motor de escucha pasiva, supresión de tickets (5 min), alertas internas y cierre analítico (30 min) en `xBackend`. Sin embargo, hoy la operación depende de editar tablas SQL manualmente y consultar Evolution API por fuera del sistema.

Esta especificación cubre la capa de administración operativa en **`xBackend`**, **`xServerSQL`** y **`cpanel-admin`** para:
1. **Vinculación y Salud de Sesión WhatsApp:** Visualizar estado en tiempo real y escanear el código QR directamente desde `cpanel-admin` (cuando la instancia esté desconectada).
2. **Administración de Staff de Soporte (`whatsapp_support_staff`):** Alta, baja, edición y activación de números telefónicos del equipo técnico con sincronización inmediata hacia Redis (`wa:support_staff_phones`).
3. **Tablero de Analítica de Casos (`whatsapp_support_cases`):** Consultar KPIs (casos totales, % atendidos sin ticket, tiempo promedio de primera respuesta, distribución por categoría/sentimiento) y auditar la transcripción y solución aplicada de cada caso cerrado.

---

## 2. Requerimientos Funcionales (Sintaxis EARS)

- **RF-01 (Estado y QR):** *Cuando* un administrador abra la vista de Asistente WhatsApp en `cpanel-admin`, el sistema *deberá* consultar `GET /v3/whatsapp/status` y mostrar el estado (`open`, `connecting`, `close`), el estado del worker y, *si* la sesión no está conectada (`close` o `connecting`), obtener y renderizar el código QR en base64 desde Evolution API.
- **RF-02 (Reconexión / Desvinculación):** *Cuando* el administrador solicite reiniciar la vinculación WhatsApp, `POST /v3/whatsapp/reconnect` *deberá* invocar el cierre/reinicio de instancia en Evolution API y devolver el nuevo código QR sin reiniciar `xBackend`.
- **RF-03 (CRUD Staff + Sync Redis):** *Cuando* el administrador cree, edite, active/desactive o elimine un teléfono en `whatsapp_support_staff` mediante `/v3/whatsapp/staff`, el backend *deberá* normalizar el número (solo dígitos, soporte variantes Argentina `54 11` / `54 9 11`), persistir en PostgreSQL vía `fn_whatsapp_support_staff_manage` y ejecutar `worker.syncStaff()` de inmediato para actualizar el `Set` en Redis.
- **RF-04 (Consulta y KPIs de Casos):** *Cuando* el administrador consulte `/v3/whatsapp/cases`, el backend *deberá* retornar los KPIs agregados y el listado paginado de `whatsapp_support_cases` filtrable por `tenant_id`, `case_category`, `case_sentiment`, `handled_by_staff`, rango de fechas y texto de búsqueda, sin incluir el arreglo pesado `raw_transcript` en el listado.
- **RF-05 (Detalle de Transcripción):** *Cuando* el administrador seleccione un caso específico (`GET /v3/whatsapp/cases/:id`), el sistema *deberá* devolver el registro completo incluyendo `raw_transcript` y `gemini_metadata`, con enlace directo al ticket Kanban si `ticket_id` existe.
- **RF-06 (Seguridad de Consola):** Todos los endpoints `/v3/whatsapp/*` *deberán* estar protegidos con `[validOrigin('console'), siteApikey()]` y conectar directo de `cpanel-admin` a `xBackend` (sin `xProxy`).

---

## 3. Filtro Doctrinal (Simplicidad Radical)

- **Sin nuevas tablas:** Se reutilizan al 100% `whatsapp_support_staff` y `whatsapp_support_cases` creadas en `0014_whatsapp_support.sql`.
- **Dos RPC unificados en lugar de seis funciones sueltas:**
  - `fn_whatsapp_support_staff_manage` resuelve `list | create | update | delete`.
  - `fn_whatsapp_cases_query` resuelve listado + KPIs + detalle individual según presencia de `case_id`.
- **Un solo componente Angular modular:** Una vista unificada con pestañas (`Conexión & QR`, `Equipo de Soporte`, `Analítica de Casos`) dentro de `cpanel-admin`, reutilizando la estética y servicios existentes (`ApiService`, `ENDPOINTS`).
- **Directiva UI No-Astra:** Prohibido usar "Astra" o "Astraserver" en textos, tooltips, etiquetas o errores.

---

## 4. Contratos de Datos y Procedimientos RPC (`xServerSQL`)

### 4.1. RPC 1: `fn_whatsapp_support_staff_manage`
*Ubicación: `sql/xServerSQL/_Functions/WhatsApp/fn_whatsapp_support_staff_manage.sql`*

```sql
CREATE OR REPLACE FUNCTION public.fn_whatsapp_support_staff_manage(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_action VARCHAR := LOWER(COALESCE(_data->>'action', 'list'));
    v_id UUID := NULLIF(_data->>'id', '')::uuid;
    v_phone VARCHAR := regexp_replace(COALESCE(_data->>'phone_number', ''), '[^0-9]', '', 'g');
    v_name VARCHAR := TRIM(COALESCE(_data->>'full_name', ''));
    v_active BOOLEAN := COALESCE((_data->>'is_active')::boolean, true);
    v_row JSONB;
BEGIN
    IF v_action = 'list' THEN
        RETURN jsonb_build_object(
            'success', true,
            'items', COALESCE((
                SELECT jsonb_agg(jsonb_build_object(
                    'id', id,
                    'phone_number', phone_number,
                    'full_name', full_name,
                    'is_active', is_active,
                    'created_at', created_at
                ) ORDER BY full_name, phone_number)
                FROM public.whatsapp_support_staff
            ), '[]'::jsonb)
        );
    ELSIF v_action = 'create' THEN
        IF v_phone = '' OR v_name = '' THEN
            RETURN jsonb_build_object('success', false, 'error', 'phone_number y full_name son obligatorios');
        END IF;

        INSERT INTO public.whatsapp_support_staff (phone_number, full_name, is_active)
        VALUES (v_phone, v_name, v_active)
        RETURNING jsonb_build_object(
            'id', id, 'phone_number', phone_number, 'full_name', full_name, 'is_active', is_active, 'created_at', created_at
        ) INTO v_row;

        RETURN jsonb_build_object('success', true, 'item', v_row);
    ELSIF v_action = 'update' THEN
        IF v_id IS NULL THEN
            RETURN jsonb_build_object('success', false, 'error', 'id es obligatorio');
        END IF;

        UPDATE public.whatsapp_support_staff
        SET phone_number = COALESCE(NULLIF(v_phone, ''), phone_number),
            full_name    = COALESCE(NULLIF(v_name, ''), full_name),
            is_active    = CASE WHEN _data ? 'is_active' THEN (_data->>'is_active')::boolean ELSE is_active END
        WHERE id = v_id
        RETURNING jsonb_build_object(
            'id', id, 'phone_number', phone_number, 'full_name', full_name, 'is_active', is_active, 'created_at', created_at
        ) INTO v_row;

        IF v_row IS NULL THEN
            RETURN jsonb_build_object('success', false, 'error', 'Registro no encontrado');
        END IF;

        RETURN jsonb_build_object('success', true, 'item', v_row);
    ELSIF v_action = 'delete' THEN
        IF v_id IS NULL THEN
            RETURN jsonb_build_object('success', false, 'error', 'id es obligatorio');
        END IF;

        DELETE FROM public.whatsapp_support_staff WHERE id = v_id;
        RETURN jsonb_build_object('success', true, 'deleted', FOUND);
    ELSE
        RETURN jsonb_build_object('success', false, 'error', 'Acción no soportada: ' || v_action);
    END IF;
EXCEPTION WHEN unique_violation THEN
    RETURN jsonb_build_object('success', false, 'error', 'El número telefónico ya existe en el equipo de soporte', 'code', SQLSTATE);
WHEN OTHERS THEN
    RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'code', SQLSTATE);
END;
$$;
```

### 4.2. RPC 2: `fn_whatsapp_cases_query`
*Ubicación: `sql/xServerSQL/_Functions/WhatsApp/fn_whatsapp_cases_query.sql`*

```sql
CREATE OR REPLACE FUNCTION public.fn_whatsapp_cases_query(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_case_id UUID := NULLIF(_data->>'case_id', '')::uuid;
    v_tenant_id VARCHAR := NULLIF(_data->>'tenant_id', '');
    v_category VARCHAR := NULLIF(_data->>'category', '');
    v_sentiment VARCHAR := NULLIF(_data->>'sentiment', '');
    v_handled BOOLEAN := CASE WHEN _data ? 'handled_by_staff' AND NULLIF(_data->>'handled_by_staff', '') IS NOT NULL
                              THEN (_data->>'handled_by_staff')::boolean ELSE NULL END;
    v_from TIMESTAMPTZ := NULLIF(_data->>'from', '')::timestamptz;
    v_to TIMESTAMPTZ := NULLIF(_data->>'to', '')::timestamptz;
    v_search VARCHAR := NULLIF(TRIM(COALESCE(_data->>'search', '')), '');
    v_limit INT := LEAST(GREATEST(COALESCE((_params->>'limit')::int, 50), 1), 200);
    v_offset INT := GREATEST(COALESCE((_params->>'offset')::int, 0), 0);
    v_detail JSONB;
    v_items JSONB;
    v_kpis JSONB;
BEGIN
    -- Modo detalle de caso individual
    IF v_case_id IS NOT NULL THEN
        SELECT to_jsonb(c.*) INTO v_detail
        FROM public.whatsapp_support_cases c
        WHERE c.id = v_case_id;

        IF v_detail IS NULL THEN
            RETURN jsonb_build_object('success', false, 'error', 'Caso no encontrado');
        END IF;

        RETURN jsonb_build_object('success', true, 'item', v_detail);
    END IF;

    -- Modo listado + KPIs
    WITH filtered AS (
        SELECT
            id, group_id, group_name, tenant_id,
            started_at, last_message_at, closed_at,
            handled_by_staff, staff_phone, first_response_seconds,
            ticket_created, ticket_id, case_summary,
            detected_problem, root_cause, applied_solution,
            case_sentiment, case_category
        FROM public.whatsapp_support_cases
        WHERE (v_tenant_id IS NULL OR tenant_id = v_tenant_id)
          AND (v_category IS NULL OR case_category = v_category)
          AND (v_sentiment IS NULL OR case_sentiment = v_sentiment)
          AND (v_handled IS NULL OR handled_by_staff = v_handled)
          AND (v_from IS NULL OR started_at >= v_from)
          AND (v_to IS NULL OR started_at <= v_to)
          AND (v_search IS NULL OR (
              group_name ILIKE '%' || v_search || '%' OR
              COALESCE(case_summary, '') ILIKE '%' || v_search || '%' OR
              COALESCE(detected_problem, '') ILIKE '%' || v_search || '%' OR
              COALESCE(applied_solution, '') ILIKE '%' || v_search || '%'
          ))
    )
    SELECT jsonb_build_object(
        'total_cases', COUNT(*),
        'handled_by_staff_count', COUNT(*) FILTER (WHERE handled_by_staff),
        'tickets_created_count', COUNT(*) FILTER (WHERE ticket_created),
        'avg_first_response_seconds', ROUND(AVG(first_response_seconds) FILTER (WHERE first_response_seconds IS NOT NULL)),
        'by_category', COALESCE((
            SELECT jsonb_object_agg(cat, cnt)
            FROM (SELECT case_category AS cat, COUNT(*) AS cnt FROM filtered GROUP BY case_category) t1
        ), '{}'::jsonb),
        'by_sentiment', COALESCE((
            SELECT jsonb_object_agg(sent, cnt)
            FROM (SELECT case_sentiment AS sent, COUNT(*) AS cnt FROM filtered GROUP BY case_sentiment) t2
        ), '{}'::jsonb)
    ) INTO v_kpis
    FROM filtered;

    SELECT COALESCE(jsonb_agg(row_to_json(p)::jsonb), '[]'::jsonb) INTO v_items
    FROM (
        SELECT
            id, group_id, group_name, tenant_id,
            started_at, last_message_at, closed_at,
            handled_by_staff, staff_phone, first_response_seconds,
            ticket_created, ticket_id, case_summary,
            detected_problem, root_cause, applied_solution,
            case_sentiment, case_category
        FROM public.whatsapp_support_cases
        WHERE (v_tenant_id IS NULL OR tenant_id = v_tenant_id)
          AND (v_category IS NULL OR case_category = v_category)
          AND (v_sentiment IS NULL OR case_sentiment = v_sentiment)
          AND (v_handled IS NULL OR handled_by_staff = v_handled)
          AND (v_from IS NULL OR started_at >= v_from)
          AND (v_to IS NULL OR started_at <= v_to)
          AND (v_search IS NULL OR (
              group_name ILIKE '%' || v_search || '%' OR
              COALESCE(case_summary, '') ILIKE '%' || v_search || '%' OR
              COALESCE(detected_problem, '') ILIKE '%' || v_search || '%' OR
              COALESCE(applied_solution, '') ILIKE '%' || v_search || '%'
          ))
        ORDER BY closed_at DESC
        LIMIT v_limit OFFSET v_offset
    ) p;

    RETURN jsonb_build_object(
        'success', true,
        'kpis', v_kpis,
        'items', v_items,
        'limit', v_limit,
        'offset', v_offset
    );
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'code', SQLSTATE);
END;
$$;
```

---

## 5. Especificación de Endpoints REST en `xBackend` (`routerwhatsapp.ts`)

Se mantienen los endpoints de `REQ-2026-003` (`/api/v1/whatsapp/webhook` y `/api/v1/whatsapp/status`) y se agregan bajo el estándar `/v3/` para `cpanel-admin` con `[validOrigin('console'), siteApikey()]`:

| Método | Ruta | Controlador / Servicio | Descripción |
| :--- | :--- | :--- | :--- |
| `GET` | `/v3/whatsapp/status` | `whatsappGatewayService.getConnectionAndQr()` | Estado de instancia + estado worker + QR base64 si está desconectado. |
| `POST` | `/v3/whatsapp/reconnect` | `whatsappGatewayService.reconnectInstance()` | Reinicia/reconecta instancia en Evolution API y devuelve nuevo QR. |
| `GET` | `/v3/whatsapp/staff` | `fn_whatsapp_support_staff_manage` (`list`) | Lista miembros de soporte técnico. |
| `POST` | `/v3/whatsapp/staff` | `fn_whatsapp_support_staff_manage` (`create`) + `worker.syncStaff()` | Alta de miembro y recarga inmediata del Set Redis. |
| `PUT` | `/v3/whatsapp/staff/:id` | `fn_whatsapp_support_staff_manage` (`update`) + `worker.syncStaff()` | Edición / toggle `is_active` y recarga del Set Redis. |
| `DELETE` | `/v3/whatsapp/staff/:id` | `fn_whatsapp_support_staff_manage` (`delete`) + `worker.syncStaff()` | Baja y recarga del Set Redis. |
| `GET` | `/v3/whatsapp/cases` | `fn_whatsapp_cases_query` | KPIs agregados y listado paginado con query params (`tenant_id`, `category`, `sentiment`, `handled_by_staff`, `from`, `to`, `search`, `limit`, `offset`). |
| `GET` | `/v3/whatsapp/cases/:id` | `fn_whatsapp_cases_query` (`case_id`) | Detalle completo del caso incluyendo `raw_transcript`. |

### Extensión necesaria en `whatsappGatewayService.ts`:
- `getQrCode()`: `GET {apiUrl}/instance/connect/{instanceName}` con header `apikey` -> extrae `base64` / `code` (QR) cuando la instancia no está `open`.
- `logoutInstance()`: `DELETE {apiUrl}/instance/logout/{instanceName}` con header `apikey` para forzar desvinculación limpia antes de pedir un nuevo QR.

---

## 6. Requerimientos de Frontend Angular (`cpanel-admin`)

- **Conexión Directa:** `cpanel-admin` llama directamente a `xBackend` (`ENDPOINTS` en `src/app/core/shared/utils/const.ts`), **sin `xProxy`**.
- **Reglas `angular-webdev`:**
  - Gestión de suscripciones con `takeUntilDestroyed()` o `Subject` + `ngOnDestroy()`.
  - Manejo de errores con `catchError` para apagar spinners ante HTTP 4xx/5xx/timeouts.
  - **Prohibido "Astra" / "Astraserver":** Usar términos neutros ("Servidor", "Asistente WhatsApp", "Pasarela WhatsApp").
- **Ubicación sugerida:** `cpanel/v2/cpanel-admin/src/app/modules/common/whatsapp-support/` (integrado al menú lateral bajo Operaciones/Soporte junto al Tablero Kanban).
- **3 Pestañas en la Vista:**
  1. **Conexión y QR:** Semáforo de estado (`Conectado` / `Desconectado`), estado del worker (activo/apagado según config), botón "Generar Nuevo QR / Reconectar", visor de imagen base64 del QR con polling controlado mientras esté en `close`/`connecting`.
  2. **Equipo de Soporte (Supresión 5 min):** Tabla CRUD de `whatsapp_support_staff` con input normalizado (ej: `54911...`), nombre completo y switch `is_active`.
  3. **Analítica de Casos:** Tarjetas KPI superiores (`Total Casos`, `Resueltos por Staff <5m`, `Tickets Disparados`, `Tiempo Promedio Respuesta`), filtros (`Tenant`, `Categoría`, `Sentimiento`, `Búsqueda`), tabla paginada y modal de detalle con la transcripción cronológica (`raw_transcript`) y link al ticket en Kanban.

---

## 7. Plan de Tareas Secuenciales para Claude Code (<30 min c/u)

- [ ] **T1: Procedimientos RPC en `xServerSQL` + Tests SQL**
  - **RFs:** `RF-03, RF-04, RF-05`
  - **Archivos:** `sql/xServerSQL/_Functions/WhatsApp/fn_whatsapp_support_staff_manage.sql`, `fn_whatsapp_cases_query.sql` y suite `test:sql`.
  - **Hecho cuando:** `npm run test:sql` pasa incluyendo CRUD completo de staff, duplicado `23505`, listado paginado de casos, cálculo de KPIs y detalle individual.

- [ ] **T2: Métodos QR/Reconexión en `whatsappGatewayService.ts` + Tests**
  - **RFs:** `RF-01, RF-02`
  - **Archivos:** `xBackend/services/whatsapp/whatsappGatewayService.ts`, `test/whatsapp/gateway.test.ts`.
  - **Hecho cuando:** Tests unitarios validan obtención de QR base64, manejo de timeout/error de Evolution API y mantenimiento estricto de bloqueo de envíos salientes.

- [ ] **T3: Endpoints `/v3/whatsapp/*` en `xBackend` + Controlador Delgado**
  - **RFs:** `RF-01..RF-06`
  - **Archivos:** `xBackend/controllers/whatsapp.ts`, `xBackend/server/routerwhatsapp.ts`, `test/whatsapp/consoleApi.test.ts`.
  - **Hecho cuando:** `npx tsc --noEmit` y `npm test` pasan; cada alta/edición/baja de staff dispara `syncStaff()` sin romper si Redis está degradado.

- [ ] **T4: Módulo Angular `whatsapp-support` en `cpanel-admin`**
  - **RFs:** `RF-01..RF-06`
  - **Archivos:** `cpanel-admin/src/app/core/shared/utils/const.ts`, `modules/common/whatsapp-support/*`, routing y menú.
  - **Hecho cuando:** `npx tsc --noEmit` en `cpanel-admin` pasa sin errores (NO ejecutar `ng build`, `npm run build` ni `pack-admin.sh`) y hay cero apariciones de "Astra".

---

## 8. Escenario de Pruebas y Protocolo de Verificación (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Casos Borde y Validaciones                   │
│ • Alta de staff con "+", espacios o guiones: se guarda solo │
│   dígitos y no falla si ya existe (mensaje claro 400).      │
│ • Consulta de casos con tabla vacía: KPIs retornan 0 sin    │
│   errores de división por cero o nulls en agregaciones.     │
│ • Evolution API apagada al pedir /v3/whatsapp/status:       │
│   devuelve estado "unreachable" en < 5s sin colgar UI.      │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Integración y Contratos de Datos             │
│ • npx tsc --noEmit en xBackend y npm run test:sql limpios.  │
│ • Validación middleware [validOrigin('console'),            │
│   siteApikey()] en todas las rutas /v3/whatsapp/*.          │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia                                  │
│ • Doble clic en "Reconectar" o guardado de staff:           │
│   idempotencia / manejo de unicidad sin crash del proceso.  │
└─────────────────────────────────────────────────────────────┘
```
