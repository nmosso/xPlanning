# Documento de Especificación Técnica: Asistente WhatsApp de Soporte, Diagnóstico y Analítica de Casos (xBackend)

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-003` |
| **Sistema Destino:** | `xBackend` (con persistencia en `xServerSQL` y estado en `Redis`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-11-11 |
| **Fecha de Aprobación:** | 2026-11-11 |
| **Autor(es):** | Lead Architect (Planning-X) |
| **Motor de Ejecución:** | **Claude Code** |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

El sistema implementa un asistente inteligente en `xBackend` que actúa como sensor, clasificador y recolector de analítica en grupos de WhatsApp de soporte técnico mediante **Evolution API v2** (vinculación por QR):

1. **Ingesta y Analítica Universal:** Todas las conversaciones de los grupos son ingestadas, analizadas mediante **Gemini** y almacenadas en una base histórica de **problemas y soluciones** para minería de datos y estadísticas operativas.
2. **Mapeo Determinista de Tenant:** Extrae el identificador del tenant directamente del nombre del grupo utilizando la convención de corchetes (ej: `"Frentones [43]"` $\to$ `tenant_id = '43'`).
3. **Regla Anti-Spam / Supresión de Tickets (Ventana de 5 minutos):** Cuando un cliente reporta un problema, el sistema inicia una cuenta regresiva de 5 minutos. Si un operador de soporte técnico (registrado en `whatsapp_support_staff`) interviene dentro de ese lapso respondiendo en el grupo, **se suprime la creación del ticket** en el Kanban.
4. **Disparo de Ticket Kanban & Alerta Interna:** Si transcurren 5 minutos desde el reclamo sin respuesta del equipo de soporte, Gemini evalúa el bloque y, de corresponder, crea el ticket en `incident_tickets`. Ante anomalías críticas o fallas masivas de plataforma, envía además una alerta resumida al **Grupo Interno de WhatsApp de Soporte Técnico**.
5. **Cierre y Analítica de Casos (30 minutos de inactividad):** Una vez que el chat permanece en silencio por más de 30 minutos, el caso se cierra y Gemini extrae el diagnóstico final (problema, causa raíz, solución y sentimiento) en `whatsapp_support_cases`.
6. **Seguridad Estricta de Emisión:** El bot **NUNCA** emite respuestas en grupos de clientes.

---

## 2. Requerimientos Funcionales (Sintaxis EARS)

- **RF-01 (Ingesta Webhook):** *Cuando* Evolution API envíe un evento `messages.upsert` de grupo (`@g.us`) y `fromMe == false`, el endpoint `POST /api/v1/whatsapp/webhook` *deberá* validar el token secreto, deduplicar por `message_id` en Redis, encolar el mensaje en la sesión activa del grupo y responder HTTP `200 OK` en menos de 50ms.
- **RF-02 (Extracción de Tenant):** *Cuando* se inicie o procese una sesión de grupo, el sistema *deberá* extraer el `tenant_id` del `group_name` aplicando el patrón `\[([0-9a-zA-Z_-]+)\]` (ej. `"Frentones [43]"` $\to$ `'43'`), asignando `NULL` de forma segura si no existe coincidencia.
- **RF-03 (Ventana de Supresión de 5 Minutos):** *Mientras* una sesión esté dentro de los primeros 5 minutos tras el primer mensaje del cliente, *si* se recibe un mensaje cuyo remitente normalizado pertenezca a `whatsapp_support_staff`, el sistema *deberá* marcar la sesión con `handled_by_staff = true` y cancelar el disparo del ticket en el Kanban.
- **RF-04 (Disparo de Ticket y Alerta Crítica):** *Cuando* transcurran 5 minutos sin intervención del soporte (`handled_by_staff == false`), el worker *deberá* invocar a Gemini con el contexto acumulado; *si* clasifica como `TICKET_SOPORTE` o `INCIDENTE_PLATAFORMA`, *deberá* crear el ticket mediante `fn_whatsapp_create_kanban_ticket` (`pendiente` o `critico`), y en el caso de incidente crítico, enviar una notificación al `WHATSAPP_SUPPORT_ALERT_GROUP_ID`.
- **RF-05 (Cierre y Analítica a los 30 Minutos):** *Cuando* una sesión de grupo cumpla 30 minutos continuos sin nuevos mensajes, el worker *deberá* invocar a Gemini sobre la transcripción completa para extraer `detected_problem`, `case_category`, `root_cause`, `applied_solution` y `case_sentiment`, almacenando el registro definitivo vía `fn_whatsapp_save_closed_case`.
- **RF-06 (Bloqueo de Salida a Clientes):** El servicio de mensajería saliente *deberá* rechazar estrictamente cualquier envío hacia un `remoteJid` que no coincida exactamente con `WHATSAPP_SUPPORT_ALERT_GROUP_ID`.

---

## 3. Filtro Doctrinal (Simplicidad Radical)

- **Pasarela Desacoplada:** Evolution API v2 corre en contenedor Docker independiente para aislar desconexiones del protocolo WebSocket de WhatsApp Web respecto al núcleo Node.js de `xBackend`.
- **Sin tablas redundantes:** Se reutiliza directamente `public.incident_tickets` para el Kanban existente, agregando únicamente la configuración de staff (`whatsapp_support_staff`) y la tabla analítica de cierre (`whatsapp_support_cases`).
- **O(1) en Redis:** Los teléfonos de soporte se sincronizan en un `Set` de Redis (`wa:support_staff_phones`) para evitar consultas SQL en cada webhook entrante.

---

## 4. Máquina de Estados del Caso de Soporte

```
[ Mensaje de Cliente en Grupo ]
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│ Caso Activo en Redis (wa:case:{groupId})                    │
│ - Registra mensaje en transcripción                         │
│ - Reinicia timer de inactividad (30 min)                    │
│ - Si es nuevo caso: programa evaluación ticket en 5 min     │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
¿Interviene Soporte (< 5 min)?       ¿Transcurren 5 min sin soporte?
            │                               │
           [SÍ]                            [NO]
            │                               │
            ▼                               ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│ Marca: handled_by_staff   │   │ Invoca Gemini Evaluador   │
│ SUPRIME creación ticket   │   │ - SOPORTE ➔ Ticket Kanban │
└───────────┬───────────────┘   │ - CRÍTICO ➔ Ticket +      │
            │                   │   Alerta Grupo Interno    │
            │                   └───────────┬───────────────┘
            └───────────────┬───────────────┘
                            │
                            ▼
            [ Chat inactivo por > 30 min ]
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Cierre y Analítica Final (Gemini)                           │
│ - Clasifica Problema, Causa Raíz, Solución y Sentimiento    │
│ - Persiste vía fn_whatsapp_save_closed_case                 │
│ - Limpia sesión activa en Redis                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Esquema DDL y Contratos de Datos (Filtro Alemán)

### 5.1. Esquema de Base de Datos (PostgreSQL DDL)
*Ubicación en `xServerSQL` según estructura del proyecto.*

```sql
-- 1. Miembros de Soporte Técnico (Números que suprimen tickets)
CREATE TABLE IF NOT EXISTS public.whatsapp_support_staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(50) NOT NULL UNIQUE,
    full_name VARCHAR(150) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 2. Histórico y Analítica de Casos (Problemas y Soluciones)
CREATE TABLE IF NOT EXISTS public.whatsapp_support_cases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id VARCHAR(100) NOT NULL,
    group_name VARCHAR(255) NOT NULL,
    tenant_id VARCHAR(50),
    started_at TIMESTAMP WITH TIME ZONE NOT NULL,
    last_message_at TIMESTAMP WITH TIME ZONE NOT NULL,
    closed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Telemetría de atención
    handled_by_staff BOOLEAN DEFAULT FALSE,
    staff_phone VARCHAR(50),
    first_response_seconds INT,
    ticket_created BOOLEAN DEFAULT FALSE,
    ticket_id UUID REFERENCES public.incident_tickets(id) ON DELETE SET NULL,
    
    -- Analítica generada por Gemini
    case_summary TEXT,
    detected_problem VARCHAR(150),
    root_cause TEXT,
    applied_solution TEXT,
    case_sentiment VARCHAR(30), -- 'POSITIVO', 'NEUTRAL', 'FRUSTRADO'
    case_category VARCHAR(50),  -- 'INCIDENTE_PLATAFORMA', 'CAIDA_CANAL', 'ERROR_APP', 'RENOVACION', 'OTRO'
    raw_transcript JSONB NOT NULL,
    gemini_metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX IF NOT EXISTS idx_wa_cases_group_id ON public.whatsapp_support_cases(group_id);
CREATE INDEX IF NOT EXISTS idx_wa_cases_tenant_id ON public.whatsapp_support_cases(tenant_id);
CREATE INDEX IF NOT EXISTS idx_wa_cases_category ON public.whatsapp_support_cases(case_category);
CREATE INDEX IF NOT EXISTS idx_wa_cases_started_at ON public.whatsapp_support_cases(started_at);
```

### 5.2. Procedimiento Almacenado RPC 1: `fn_whatsapp_create_kanban_ticket`
*Disparado al minuto 5 si no intervino soporte y Gemini detectó un reclamo/solicitud.*

```sql
CREATE OR REPLACE FUNCTION public.fn_whatsapp_create_kanban_ticket(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_case_id UUID;
    v_group_id VARCHAR;
    v_group_name VARCHAR;
    v_tenant_id VARCHAR;
    v_category VARCHAR;
    v_summary TEXT;
    v_analysis JSONB;
    v_ticket_type VARCHAR;
    v_ticket_status VARCHAR;
    v_ticket_title VARCHAR;
    v_ticket_id UUID;
BEGIN
    v_case_id    := COALESCE((_data->>'case_id')::uuid, gen_random_uuid());
    v_group_id   := _data->>'group_id';
    v_group_name := COALESCE(_data->>'group_name', 'Grupo WhatsApp');
    v_tenant_id  := NULLIF(_data->>'tenant_id', '');
    v_category   := COALESCE(_data->>'category', 'TICKET_SOPORTE');
    v_summary    := _data->>'summary';
    v_analysis   := COALESCE(_data->'analysis', '{}'::jsonb);

    IF v_category = 'INCIDENTE_PLATAFORMA' THEN
        v_ticket_type   := 'system_bug';
        v_ticket_status := 'critico';
        v_ticket_title  := LEFT('[ALERTA PLATAFORMA] ' || COALESCE(v_summary, 'Problema técnico detectado'), 250);
    ELSE
        v_ticket_type   := 'tenant_issue';
        v_ticket_status := 'pendiente';
        v_ticket_title  := LEFT('[SOPORTE WA] ' || COALESCE(v_summary, 'Requerimiento en grupo'), 250);
    END IF;

    INSERT INTO public.incident_tickets (
        tenant_id,
        title,
        description,
        type,
        status,
        deadline,
        is_archived,
        created_at,
        updated_at,
        created_by,
        modified_by
    ) VALUES (
        v_tenant_id,
        v_ticket_title,
        COALESCE(v_summary, '') || E'\n\n--- Contexto Analizado ---\n' || jsonb_pretty(v_analysis),
        v_ticket_type,
        v_ticket_status,
        CASE WHEN v_ticket_status = 'critico' THEN NOW() + INTERVAL '2 hours' ELSE NOW() + INTERVAL '24 hours' END,
        false,
        NOW(),
        NOW(),
        jsonb_build_object(
            'source', 'whatsapp_agent',
            'group_id', v_group_id,
            'group_name', v_group_name,
            'case_id', v_case_id
        ),
        jsonb_build_object('source', 'whatsapp_agent')
    ) RETURNING id INTO v_ticket_id;

    RETURN jsonb_build_object(
        'success', true,
        'ticket_id', v_ticket_id,
        'status', v_ticket_status,
        'type', v_ticket_type
    );
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object(
        'success', false,
        'error', SQLERRM,
        'code', SQLSTATE
    );
END;
$$;
```

### 5.3. Procedimiento Almacenado RPC 2: `fn_whatsapp_save_closed_case`
*Disparado tras 30 minutos de silencio en el chat para guardar la analítica consolidada.*

```sql
CREATE OR REPLACE FUNCTION public.fn_whatsapp_save_closed_case(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_case_id UUID;
    v_group_id VARCHAR;
    v_group_name VARCHAR;
    v_tenant_id VARCHAR;
    v_started_at TIMESTAMPTZ;
    v_last_message_at TIMESTAMPTZ;
    v_handled_by_staff BOOLEAN;
    v_staff_phone VARCHAR;
    v_first_response_seconds INT;
    v_ticket_id UUID;
    v_summary TEXT;
    v_problem VARCHAR;
    v_root_cause TEXT;
    v_solution TEXT;
    v_sentiment VARCHAR;
    v_category VARCHAR;
    v_raw_transcript JSONB;
    v_metadata JSONB;
BEGIN
    v_case_id                := COALESCE((_data->>'case_id')::uuid, gen_random_uuid());
    v_group_id               := _data->>'group_id';
    v_group_name             := COALESCE(_data->>'group_name', 'Grupo WhatsApp');
    v_tenant_id              := NULLIF(_data->>'tenant_id', '');
    v_started_at             := (_data->>'started_at')::timestamptz;
    v_last_message_at        := (_data->>'last_message_at')::timestamptz;
    v_handled_by_staff       := COALESCE((_data->>'handled_by_staff')::boolean, false);
    v_staff_phone            := NULLIF(_data->>'staff_phone', '');
    v_first_response_seconds := (_data->>'first_response_seconds')::int;
    v_ticket_id              := NULLIF(_data->>'ticket_id', '')::uuid;
    v_summary                := _data->>'summary';
    v_problem                := LEFT(_data->>'detected_problem', 150);
    v_root_cause             := _data->>'root_cause';
    v_solution               := _data->>'applied_solution';
    v_sentiment              := COALESCE(_data->>'case_sentiment', 'NEUTRAL');
    v_category               := COALESCE(_data->>'case_category', 'OTRO');
    v_raw_transcript         := COALESCE(_data->'raw_transcript', '[]'::jsonb);
    v_metadata               := COALESCE(_data->'metadata', '{}'::jsonb);

    INSERT INTO public.whatsapp_support_cases (
        id,
        group_id,
        group_name,
        tenant_id,
        started_at,
        last_message_at,
        closed_at,
        handled_by_staff,
        staff_phone,
        first_response_seconds,
        ticket_created,
        ticket_id,
        case_summary,
        detected_problem,
        root_cause,
        applied_solution,
        case_sentiment,
        case_category,
        raw_transcript,
        gemini_metadata
    ) VALUES (
        v_case_id,
        v_group_id,
        v_group_name,
        v_tenant_id,
        v_started_at,
        v_last_message_at,
        NOW(),
        v_handled_by_staff,
        v_staff_phone,
        v_first_response_seconds,
        (v_ticket_id IS NOT NULL),
        v_ticket_id,
        v_summary,
        v_problem,
        v_root_cause,
        v_solution,
        v_sentiment,
        v_category,
        v_raw_transcript,
        v_metadata
    );

    RETURN jsonb_build_object(
        'success', true,
        'case_id', v_case_id
    );
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object(
        'success', false,
        'error', SQLERRM,
        'code', SQLSTATE
    );
END;
$$;
```

---

## 6. Especificación de Endpoints REST y Variables de Entorno

### 6.1. Endpoints en `xBackend` (Thin Controllers)
- **`POST /api/v1/whatsapp/webhook`**:
  - Verifica header `apikey` contra `EVOLUTION_WEBHOOK_SECRET`.
  - Filtra eventos distintos a `messages.upsert`, mensajes con `fromMe === true` o chats privados (`!remoteJid.endsWith('@g.us')`).
  - Deduplica por `key.id` en Redis (`SET wa:dedup:{msgId} 1 EX 120 NX`).
  - Encola / actualiza el estado de sesión en Redis (`wa:case:{groupId}`) y retorna `200 OK` inmediatamente.
- **`GET /api/v1/whatsapp/status`**:
  - Consulta estado de conexión de la instancia en Evolution API (`open`, `close`, `connecting`).

### 6.2. Variables de Entorno (`.env`)
```bash
EVOLUTION_API_URL=http://localhost:8080
EVOLUTION_API_KEY=xbackend_secret_token
EVOLUTION_INSTANCE_NAME=xbackend_listener
EVOLUTION_WEBHOOK_SECRET=secret_webhook_token
WHATSAPP_SUPPORT_ALERT_GROUP_ID=12036304xxxxxxxx@g.us
GEMINI_API_KEY=AIzaSy...
GEMINI_MODEL=gemini-2.5-flash
```

---

## 7. Salidas Estructuradas de Gemini (JSON Schema)

### 7.1. Evaluación T+5 min (Decisión de Ticket)
```json
{
  "category": "INCIDENTE_PLATAFORMA | TICKET_SOPORTE | IRRELEVANTE",
  "requires_ticket": true,
  "confidence": 0.95,
  "summary": "Reporte reiterado de microcortes en señales deportivas.",
  "urgency": "ALTA | MEDIA | BAJA",
  "affected_services": ["Canal 12", "Servidor Edge 3"]
}
```

### 7.2. Cierre de Caso T+30 min inactivo (Analítica Final)
```json
{
  "detected_problem": "Falla de reproducción en canal premium ESPN",
  "case_category": "CAIDA_CANAL",
  "root_cause": "Buffer saturado en transcodificador regional",
  "applied_solution": "Operador reinició PID de la señal y cliente confirmó funcionamiento",
  "case_summary": "Cliente reportó pantalla en negro; soporte intervino a los 3 minutos y solucionó reseteando el stream.",
  "case_sentiment": "POSITIVO"
}
```

---

## 8. Plan de Tareas Secuenciales para Claude Code (<30 min c/u)

- [ ] **T1: Despliegue DDL y Procedimientos PL/pgSQL en `xServerSQL`**
  - **RFs:** `RF-04, RF-05`
  - **Entregable:** Tablas `whatsapp_support_staff`, `whatsapp_support_cases` y funciones `fn_whatsapp_create_kanban_ticket`, `fn_whatsapp_save_closed_case`.
  - **Hecho cuando:** Las tablas e índices existen y las pruebas unitarias SQL de ambos RPC retornan `success: true`.

- [ ] **T2: Cliente Evolution API & Guardrails de Salida (`xBackend`)**
  - **RFs:** `RF-06`
  - **Entregable:** Servicio HTTP `whatsappGatewayService.ts` con método de consulta de estado y método `sendSupportAlert` bloqueado exclusivamente al `WHATSAPP_SUPPORT_ALERT_GROUP_ID`.
  - **Hecho cuando:** El unit test confirma que cualquier intento de enviar mensaje a otro `remoteJid` lanza error y es bloqueado.

- [ ] **T3: Webhook Controller y Deduplicación Redis (`xBackend`)**
  - **RFs:** `RF-01, RF-02, RF-03`
  - **Entregable:** Controlador delgado `POST /api/v1/whatsapp/webhook` + gestor de sesión en Redis (`wa:case:{groupId}`), extracción regex de `tenant_id` y verificación O(1) contra `wa:support_staff_phones`.
  - **Hecho cuando:** `npx tsc --noEmit` pasa sin errores y el test de integración simula mensajes de cliente y soporte marcando `handled_by_staff` en < 5 min.

- [ ] **T4: Servicio Evaluador Gemini y Scheduler Temporal (5 min / 30 min)**
  - **RFs:** `RF-03, RF-04, RF-05`
  - **Entregable:** Worker periódico en `xBackend` que verifica expiración de los 5 min (creación de ticket si no hay respuesta de soporte + alerta crítica) y expiración de 30 min de inactividad (cierre y guardado analítico).
  - **Hecho cuando:** Tests L1/L2 verifican supresión de ticket a los 4m58s, creación de ticket a los 5m02s y cierre analítico a los 30m de silencio.

- [ ] **T5: Verificación L1/L2/L3 y Actualización de Memoria**
  - **RFs:** Todos los RFs
  - **Entregable:** Ejecución completa de suite de pruebas, actualización de `xBackend/MEMORY.md` y nota en `Planning-X/Memory`.
  - **Hecho cuando:** `npx tsc --noEmit` limpio y suite de tests L1/L2 pasando al 100%.

---

## 9. Escenario de Pruebas y Protocolo de Verificación (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Casos Borde y Validaciones Temporales        │
│ • El soporte responde a los 4m 58s: Se suprime el ticket.   │
│ • El soporte responde a los 5m 02s: Ticket creado; caso     │
│   posterior asocia ticket_id sin duplicar.                  │
│ • Mensaje llega a los 29 min: El timer de 30 min se         │
│   reinicia automáticamente extendiendo la sesión.           │
│ • Nombre sin corchetes ("Clientes"): tenant_id = NULL.      │
│ • Normalización estricta de teléfonos en Redis (sin '+',    │
│   espacios ni prefijos inconsistentes).                     │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Integración y Contratos de Datos             │
│ • npx tsc --noEmit en xBackend sin errores de tipos.        │
│ • Ejecución RPC fn_whatsapp_create_kanban_ticket y          │
│   fn_whatsapp_save_closed_case en PostgreSQL.               │
│ • Ticket visible en incident_tickets con tenant_id exacto.  │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia y Seguridad                      │
│ • Reinicio del proceso xBackend durante ventana activa:     │
│   Redis conserva estado y timers.                           │
│ • Caída temporal de Gemini API: Reintento exponencial.      │
│ • 0 mensajes salientes hacia grupos de clientes (garantizado).│
└─────────────────────────────────────────────────────────────┘
```
