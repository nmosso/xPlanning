# Documento de Especificación Técnica: `xRoster` — Fase 1 (Jarvis, Sentinel, Tank, Shikamaru & Quota Ledger)

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-012` |
| **RFC de Referencia:** | [`RFC-2026-003`](file:///d:/Projects/DEV/xIPTV/Planning-X/Cross_System/02_Aprobados/RFC-2026-003_ARQUITECTURA_EQUIPO_AGENTES_XROSTER.md) |
| **Sistema Destino:** | `Cross_System` (`xRoster` [nuevo repo en `d:\Projects\DEV\xIPTV\xRoster`], ajuste mínimo en `xBackend`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-11-24 |
| **Última Actualización:** | 2026-11-24 |
| **Autor(es):** | Lead Architect (`Planning-X`) |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

Implementa la **Fase 1 (MVP)** del ecosistema **`xRoster`**: un microservicio Docker independiente desplegado en el servidor de `xBackend` (portable 100% vía `.env` a una VM dedicada) que conecta un **grupo privado de WhatsApp ("War Room")** con los primeros 4 agentes operativos:

1. **Jarvis** — Orquestador de War Room, autenticación E.164, enrutamiento y flujo de confirmaciones `L1`.
2. **Sentinel** — Monitor proactivo de QoS sobre `monitoring:tenant:*` (Redis) con detección determinista de caídas/degradación, consolidación anti-tormenta y avisos de recuperación.
3. **Tank** — Consultas en vivo de infraestructura/hardware (`tenants`, Proxmox, VMs `xManager`/`Multicast`/`Streamer`, IRD) y acciones operativas seguras bajo confirmación `CONFIRMAR <código>`.
4. **Shikamaru** — Consulta de `https://github.com/nmosso/xPlanning.git` (`main`) y generación de borradores/análisis técnicos coordinados con el plan de **Claude Code**.

Adicionalmente, instrumenta desde el día 1 el **registro de cuotas (`quota_usage_ledger`)** para **Gemini** y **Claude Code**, dejando la base de datos lista para activar **Axelrod** en Fase 2.

---

## 2. Filtro Doctrinal (Británico, Alemán y Ruso)

- **Filtro Británico (*Keep it Plain*):**
  - **Necesidad real:** Recibir alertas filtradas y operar/planificar la plataforma desde un único grupo privado de WhatsApp sin tocar la estabilidad de `xBackend`, dejando una base sólida para el futuro panel web **`cpanel-roster`**.
  - **Descartado:** Frameworks multi-agente pesados (LangGraph/CrewAI), segunda instancia de Evolution API, SQLite limitado para reportes concurrentes o cambios en `cpanel-admin`.
  - **Simplicidad:** Node.js 22 + TypeScript + PostgreSQL dedicado (`xrosterdb` en la instancia PostgreSQL existente) + SDK oficial `@google/genai` + integración CLI/API de Claude Code.
- **Filtro Alemán (*Gründlichkeit*):**
  - Contratos estrictos TypeScript/JSON en webhook, approvals, tools y ledger de cuotas.
  - Aislamiento absoluto de `WHATSAPP_COMMAND_GROUP_ID` en `xBackend` (cero contaminación en soporte/casos/emergencia).
  - Acceso a `postgres-xserver` estrictamente **read-only**.
- **Filtro Ruso (*Nadezhnost*):**
  - Timeout duro de 3 s en el forward de `xBackend` (si `xRoster` cae, `xBackend` ni se entera).
  - Guardrail duro de salida en `xRoster`: solo puede enviar a `WHATSAPP_COMMAND_GROUP_ID`.
  - Confirmación `CONFIRMAR <código>` con TTL de 5 min para toda acción `L1`; bloqueo en código para acciones `L2`.
  - Truncado de salidas SSH/logs (máx. 4 000 chars) antes de pasar al LLM para evitar desbordes de cuota.

---

## 3. Alcance y Límites

### 3.1. En Alcance (In Scope — Fase 1)
- [ ] **Ajuste mínimo en `xBackend`:** Exclusión de `WHATSAPP_COMMAND_GROUP_ID` + forward HTTP POST firmado (`X-Internal-Secret`) hacia `xRoster`.
- [ ] **Nuevo repositorio `xRoster`:** Estructura TypeScript + `Dockerfile` + `docker-compose.yml` con límites (`cpus: "1.0"`, `mem_limit: "1536m"`).
- [ ] **Core `Jarvis`:** Receptor webhook firmado, deduplicación por `message_id`, whitelist E.164 (`OWNER_WHATSAPP_PHONES`), comandos de control (`STATUS`, `SILENCIAR`, `PAUSAR`, `REANUDAR`, `CONFIRMAR`, `CANCELAR`), router hacia especialistas y cliente Evolution API con guardrail de destino.
- [ ] **Motor `ProviderRouter` + `quota_usage_ledger`:** Wrapper unificado para llamadas a **Gemini** y **Claude Code** que persiste tokens, latencia, ventana, errores `429` y cuota estimada en PostgreSQL (`xrosterdb`).
- [ ] **Agente `Sentinel` (QoS):** Worker determinista que lee `monitoring:tenant:*` en Redis cada 60 s, detecta cambios de estado (`ok` → `degraded`/`down` → `ok`), aplica cooldown, agrupa caídas masivas en 1 mensaje y permite consultas bajo demanda (`"¿qué está caído?"`).
- [ ] **Agente `Tank` (Infra & Hardware):**
  - `L0` (autónomo): Buscar tenant en `public.tenants` (por `position`, `tenantid` o `name`), consultar salud de los 4 nodos (`Proxmox`, `xManager`, `Multicast`, `Streamer`) desde Redis + chequeo SSH puntual de solo-lectura (`uptime`, `df -h`, `free -m`, `systemctl is-active`), y consulta de estado IRD (`http://100.64.X.2:20135/`).
  - `L1` (con `CONFIRMAR <código>`): Reinicio controlado de servicios permitidos (`xmanager`, `multicast`, `streamer`) vía whitelist estricta de comandos (sin shell libre arbitrario).
- [ ] **Agente `Shikamaru` (Planificación):**
  - `L0` (autónomo): `git pull --ff-only` sobre `https://github.com/nmosso/xPlanning.git` (`main`), lectura de `Memory/INDEX.md`, `Tenants/<id>/MEMORY.md`, `01_Borradores/` y `02_Aprobados/` para responder consultas de arquitectura/planificación.
  - `L1` (con `CONFIRMAR <código>`): Crear un nuevo archivo borrador en rama `draft/shikamaru-<slug>` y hacer `git push` de la rama (nunca push directo a `main`).

### 3.2. Fuera de Alcance (Out of Scope — Fases 2 y 3)
- Panel web **`cpanel-roster`** (gestor general y reportes en PC — Fase 2; en Fase 1 el esquema `xrosterdb` ya nace diseñado con `JSONB` e índices para alimentarlo sin migraciones traumáticas).
- Agente conversacional **Axelrod** y reporte diario automático de modelos (Fase 2 — aquí solo se construye su tabla `quota_usage_ledger` y resumen básico en `STATUS`).
- Agente **Atlas** y gestión de tickets Kanban (Fase 2).
- Agente **Skipper** y despliegues/migraciones SQL (Fase 3).
- Comandos SSH arbitrarios escritos por el LLM (prohibido por seguridad; Tank solo invoca acciones tipadas de un catálogo cerrado).
- Modificaciones en `cpanel-admin` o `xServerSQL`.

---

## 4. Requerimientos Funcionales (Sintaxis EARS)

- **RF-01 (Aislamiento en `xBackend`):** Cuando `xBackend` reciba un evento `messages.upsert` cuyo `group_id` coincida con `WHATSAPP_COMMAND_GROUP_ID`, el sistema deberá omitir completamente `sessions.ingest`, `supportGroup` y `emergency`, y reenviar el payload por HTTP POST asíncrono (timeout 3 000 ms) a `XROSTER_WEBHOOK_URL` con el header `X-Internal-Secret`.
- **RF-02 (Autenticación y Guardrail en `Jarvis`):** Cuando `xRoster` reciba un webhook en `POST /internal/whatsapp/webhook`, deberá validar `X-Internal-Secret` (`401` si no coincide), deduplicar por `message_id`, verificar que `group_id === WHATSAPP_COMMAND_GROUP_ID` y que el remitente normalizado E.164 pertenezca a `OWNER_WHATSAPP_PHONES` (ignorando silenciosamente mensajes propios del bot `fromMe: true` y logueando advertencia si un tercero escribe en el grupo).
- **RF-03 (Guardrail de Salida WhatsApp):** Antes de ejecutar cualquier `POST /message/sendText` contra Evolution API v2, `xRoster` deberá validar que el destinatario sea estrictamente igual a `WHATSAPP_COMMAND_GROUP_ID`; ante cualquier otro destino deberá lanzar `ForbiddenTargetError` y abortar el envío.
- **RF-04 (Comandos de Control de `Jarvis`):** Cuando el operador envíe `STATUS`, `PAUSAR`, `REANUDAR`, `SILENCIAR <tenant|all> <minutos>`, `CONFIRMAR <código>` o `CANCELAR <código>`, `Jarvis` deberá ejecutarlos de forma determinista sin consumir LLM.
- **RF-05 (Enrutamiento de Especialistas):** Cuando el operador envíe un mensaje de texto libre o dirigido (`@Tank ...`, `Sentinel ...`, `Shikamaru ...`), `Jarvis` deberá determinar el agente destino (por mención explícita primero; solo si es ambiguo, clasifica con Gemini Flash) y devolver una respuesta consolidada con firma del agente (ej. `🛡️ *Sentinel*`, `🔧 *Tank*`, `🧠 *Shikamaru*`).
- **RF-06 (Protocolo de Aprobación `L1`):** Cuando un agente proponga una acción de nivel `L1`, el sistema no deberá ejecutarla; en su lugar persistirá un registro en `action_approvals` con código corto (ej. `TK-482`) y expiración a 5 minutos, notificará el detalle exacto al War Room y solo ejecutará la acción si el operador responde `CONFIRMAR <código>` antes de la expiración.
- **RF-07 (Monitoreo Proactivo `Sentinel`):** Cada 60 segundos (configurable por `SENTINEL_POLL_MS`), mientras `xRoster` no esté pausado, `Sentinel` deberá leer las claves `monitoring:tenant:*` de Redis, evaluar caídas/degradaciones de los 4 nodos (`Proxmox`, `xManager`, `Multicast`, `Streamer`) e IRD, respetar ventanas `silenced_until` y cooldown de re-alerta (30 min), consolidar múltiples cabeceras caídas en un único mensaje cuando superen 3 eventos en el mismo ciclo, y emitir aviso `✅ RECUPERADO` al normalizarse.
- **RF-08 (Consultas y Acciones de `Tank`):** Cuando se consulte a `Tank` por un tenant (por número `position`, `tenantid` o nombre parcial), deberá resolverlo desde `public.tenants` (PG read-only), consultar estado en Redis + chequeo SSH read-only con timeout de 8 s por nodo, truncar salidas a 4 000 caracteres y sintetizar el diagnóstico. Para reinicios de servicio (`xmanager`, `multicast`, `streamer`), deberá crear obligatoriamente una solicitud `L1` (RF-06).
- **RF-09 (Consultas y Borradores de `Shikamaru`):** Cuando se consulte a `Shikamaru`, deberá ejecutar `git pull --ff-only` sobre el clone local de `https://github.com/nmosso/xPlanning.git` (`main`), leer los archivos relevantes (`Memory/INDEX.md`, `Tenants/<id>/MEMORY.md`, `01_Borradores/`, `02_Aprobados/`) y responder usando el motor **Claude**. Si el operador pide crear un borrador, deberá generar propuesta y solicitar confirmación `L1` antes de hacer commit/push a una rama `draft/shikamaru-<slug>`.
- **RF-10 (Telemetría de Cuotas `quota_usage_ledger`):** Toda invocación a **Gemini** o **Claude Code** deberá registrar obligatoriamente en PostgreSQL (`xrosterdb`): `provider`, `model`, `agent_name`, `task_type`, `input_tokens`, `output_tokens`, `cached_tokens`, `latency_ms`, `status`, `rate_limited` (`boolean`), permitiendo mostrar en `STATUS` el uso acumulado de las últimas 5h y 24h por proveedor.

---

## 5. Contratos de Datos y Esquema DDL (Filtro Alemán)

### 5.1. Esquema PostgreSQL Dedicado (`xrosterdb` — `sql/001_init_xrosterdb.sql`)

> [!IMPORTANT]
> Se crea una base de datos aislada **`xrosterdb`** con rol dedicado `xroster_user` dentro de la instancia PostgreSQL existente. Cero tablas nuevas dentro de `xServerSQL`. El esquema ya incluye columnas `JSONB` e índices pensados para los reportes y dashboards del futuro **`cpanel-roster`**.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE IF NOT EXISTS public.quota_usage_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    agent_name TEXT NOT NULL,            -- 'jarvis' | 'sentinel' | 'tank' | 'shikamaru'
    provider TEXT NOT NULL,              -- 'gemini' | 'claude'
    model TEXT NOT NULL,
    task_type TEXT NOT NULL,             -- 'route' | 'summary' | 'diagnose' | 'planning'
    input_tokens INTEGER NOT NULL DEFAULT 0,
    output_tokens INTEGER NOT NULL DEFAULT 0,
    cached_tokens INTEGER NOT NULL DEFAULT 0,
    latency_ms INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL,                -- 'ok' | 'error' | 'rate_limited'
    rate_limited BOOLEAN NOT NULL DEFAULT FALSE,
    error_code TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb
);
CREATE INDEX IF NOT EXISTS idx_quota_provider_created
    ON public.quota_usage_ledger (provider, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_quota_agent_created
    ON public.quota_usage_ledger (agent_name, created_at DESC);

CREATE TABLE IF NOT EXISTS public.action_approvals (
    code TEXT PRIMARY KEY,               -- ej. 'TK-482', 'SH-109'
    agent_name TEXT NOT NULL,
    action_type TEXT NOT NULL,           -- 'restart_service' | 'git_push_draft'
    target_tenant TEXT,
    summary_text TEXT NOT NULL,
    payload JSONB NOT NULL DEFAULT '{}'::jsonb,
    status TEXT NOT NULL DEFAULT 'pending', -- 'pending' | 'confirmed' | 'cancelled' | 'expired' | 'executed' | 'failed'
    requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    decided_at TIMESTAMPTZ,
    result_summary TEXT
);
CREATE INDEX IF NOT EXISTS idx_approvals_status_expires
    ON public.action_approvals (status, expires_at);

CREATE TABLE IF NOT EXISTS public.sentinel_alert_state (
    alert_key TEXT PRIMARY KEY,          -- '{position}:{component}' ej. '43:xmanager', '43:ird'
    tenant_position TEXT NOT NULL,
    tenant_name TEXT,
    component TEXT NOT NULL,
    state TEXT NOT NULL,                 -- 'ok' | 'degraded' | 'down'
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_notified_at TIMESTAMPTZ,
    silenced_until TIMESTAMPTZ,
    last_detail TEXT,
    metrics JSONB NOT NULL DEFAULT '{}'::jsonb
);
CREATE INDEX IF NOT EXISTS idx_sentinel_state_tenant
    ON public.sentinel_alert_state (state, tenant_position);

CREATE TABLE IF NOT EXISTS public.agent_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    message_id TEXT UNIQUE,
    sender_phone TEXT,
    agent_name TEXT,
    input_text TEXT,
    output_text TEXT,
    duration_ms INTEGER,
    status TEXT NOT NULL,                -- 'ok' | 'ignored' | 'error' | 'unauthorized'
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb
);
CREATE INDEX IF NOT EXISTS idx_audit_created
    ON public.agent_audit_log (created_at DESC);
CREATE INDEX IF NOT EXISTS idx_audit_metadata_gin
    ON public.agent_audit_log USING GIN (metadata);

CREATE TABLE IF NOT EXISTS public.system_flags (
    key TEXT PRIMARY KEY,                -- 'paused' | 'sentinel_silenced_all_until'
    value TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5.2. Contrato de Forward HTTP (`xBackend` → `xRoster`)

- **Endpoint receptor en `xRoster`:** `POST /internal/whatsapp/webhook`
- **Headers obligatorios:**
  - `Content-Type: application/json`
  - `X-Internal-Secret: <XROSTER_WEBHOOK_SECRET>`
- **Body JSON:**
```json
{
  "message_id": "3EB0A1B2C3...",
  "group_id": "120363000000000000@g.us",
  "sender_phone": "54911XXXXXXXX",
  "sender_name": "Operador",
  "from_me": false,
  "text": "@Tank revisá el server 43",
  "timestamp": 1763980000
}
```
- **Respuestas:** `200 OK` (`{ "accepted": true }`), `401 Unauthorized` (secret inválido), `400 Bad Request` (payload incompleto).

### 5.3. Variables de Entorno (`xRoster/.env.example`)
- **HTTP & Seguridad:** `PORT=3900`, `XROSTER_WEBHOOK_SECRET`, `WHATSAPP_COMMAND_GROUP_ID`, `OWNER_WHATSAPP_PHONES` (CSV E.164).
- **Evolution API:** `EVOLUTION_API_URL`, `EVOLUTION_API_KEY`, `EVOLUTION_INSTANCE_NAME`.
- **Fuentes del Ecosistema (portables a VM):** `PG_XROSTER_URL` (pool lectura/escritura hacia `xrosterdb`), `PG_XSERVER_RO_URL` (pool **read-only** hacia `public.tenants` de `xBackend`), `REDIS_URL`, `SSH_KEY_PATH`, `SSH_USER`, `SSH_TIMEOUT_MS=8000`.
- **Git / Shikamaru:** `XPLANNING_REPO_URL=https://github.com/nmosso/xPlanning.git`, `XPLANNING_BRANCH=main`, `XPLANNING_GIT_TOKEN` (opcional para push de ramas `draft/*`), `XPLANNING_LOCAL_DIR=/data/xPlanning`.
- **Motores IA:** `GEMINI_API_KEY`, `GEMINI_MODEL_FAST=gemini-2.5-flash`, `CLAUDE_PROVIDER=cli|api`, `ANTHROPIC_API_KEY` (si `api`), `CLAUDE_MODEL=claude-sonnet-4-5`.
- **Sentinel:** `SENTINEL_POLL_MS=60000`, `SENTINEL_ALERT_COOLDOWN_MS=1800000`, `SENTINEL_STORM_THRESHOLD=3`.

---

## 6. Escenario de Pruebas y Protocolo de Verificación (Filtro Ruso)

### 6.1. Nivel 1 (L1) — Unitarias y Casos Borde
1. **Guardrail WhatsApp:** Intentar enviar mensaje a `@s.whatsapp.net` o a un `@g.us` de cliente distinto de `WHATSAPP_COMMAND_GROUP_ID` → debe lanzar `ForbiddenTargetError` sin disparar HTTP.
2. **Auth E.164:** Mensaje con `sender_phone` fuera de `OWNER_WHATSAPP_PHONES` → `status = 'unauthorized'`, cero ejecución de agentes; normalización `54` vs `549` igual que `xBackend`.
3. **Approvals `L1`:**
   - `CONFIRMAR TK-482` después de 5 min → marca `expired` y rechaza ejecución.
   - `CONFIRMAR TK-482` enviado dos veces → la segunda vez informa que ya fue ejecutado (idempotente).
   - Comando fuera de whitelist en Tank (ej. `rm -rf /` o servicio desconocido) → rechazado antes de crear approval.
4. **Truncado SSH:** Salida simulada de 50 KB → truncada a ≤ 4 000 chars antes de invocar al LLM.
5. **Sentinel Anti-Tormenta:** 10 cabeceras caen en el mismo tick → genera **1 solo mensaje consolidado**, no 10.
6. **Ledger de Cuotas:** Llamada exitosa y llamada con `429` registran correctamente en `quota_usage_ledger`.

### 6.2. Nivel 2 (L2) — Integración y Contratos
- [ ] `cd xRoster && npx tsc --noEmit` (cero errores).
- [ ] `cd xRoster && npm test` (suite completa `node:test` L1/L2/L3).
- [ ] `cd xBackend && npx tsc --noEmit` y `npm test` (verificar que las ~220 pruebas existentes de WhatsApp siguen 100% en verde + nuevo test de forward/aislamiento de `WHATSAPP_COMMAND_GROUP_ID`).
- [ ] Validación de `docker build` de `xRoster`.

### 6.3. Nivel 3 (L3) — Resiliencia y Fallos
1. **`xRoster` caído:** `xBackend` recibe mensaje en `WHATSAPP_COMMAND_GROUP_ID`, el forward hace timeout/ECONNREFUSED en ≤ 3 s, `xBackend` responde `200 OK` a Evolution API y **no** crea sesión/ticket de soporte.
2. **Cabecera sin red:** `Tank` consulta un tenant apagado (`100.64.X.8` inalcanzable) → timeout SSH a 8 s, devuelve estado parcial de Redis + aviso claro de timeout sin colgar el proceso.
3. **Git sin conexión:** `Shikamaru` falla `git pull` → responde usando la copia local existente con advertencia `⚠️ copia local sin sincronizar`.
4. **Error `429` en proveedor IA:** Se registra `status='rate_limited'` en `quota_usage_ledger` y `Jarvis` avisa claramente qué plan alcanzó límite.

---

## 7. Desglose de Tareas Atómicas para Claude Code (SDD Fase 5)

### Bloque A — Ajuste Mínimo en `xBackend`
- [ ] **T-01:** En `xBackend`, agregar variables opcionales `WHATSAPP_COMMAND_GROUP_ID`, `XROSTER_WEBHOOK_URL` y `XROSTER_WEBHOOK_SECRET`, e interceptar el grupo War Room antes de `sessions.ingest` con forward HTTP *fire-and-forget* (timeout 3 s).  
  *Hecho cuando:* `test/whatsapp/` incluye tests de aislamiento + forward firmado + tolerancia a caída de `xRoster`, y `npx tsc --noEmit && npm test` pasan en verde.

### Bloque B — Scaffolding y Core de `xRoster`
- [ ] **T-02:** Inicializar proyecto `d:\Projects\DEV\xIPTV\xRoster` (`package.json`, `tsconfig.json`, `.env.example`, `Dockerfile`, `docker-compose.yml` con límites `1.0 vCPU / 1.5 GB RAM` y `MEMORY.md`).  
  *Hecho cuando:* `npx tsc --noEmit` compila limpio y existe estructura `src/{core,agents,providers,db,tools}` + `test/`.
- [ ] **T-03:** Implementar capa PostgreSQL (`src/db/database.ts`, pool `pg` + `sql/001_init_xrosterdb.sql` idempotente + interfaz `RosterStore` con implementación PostgreSQL y store in-memory para tests unitarios rápidos) cubriendo las 5 tablas (`quota_usage_ledger`, `action_approvals`, `sentinel_alert_state`, `agent_audit_log`, `system_flags`).  
  *Hecho cuando:* `npm test` valida lógica con `MemoryRosterStore` y `npm run test:sql` aplica `001_init_xrosterdb.sql` en una BD PostgreSQL efímera validando índices, `JSONB` y TTL de approvals.
- [ ] **T-04:** Implementar `ProviderRouter` (`src/providers/`) para **Gemini** (`@google/genai`) y **Claude** (`cli` / `api`) con registro automático obligatorio en `quota_usage_ledger` y resumen de uso 5h/24h.  
  *Hecho cuando:* Tests unitarios verifican registro de `ok`, `error` y `rate_limited` con mocks de ambos proveedores.
- [ ] **T-05:** Implementar `EvolutionClient` con guardrail duro (`ForbiddenTargetError`) + `Jarvis` (`src/core/jarvis.ts`, webhook Express/HTTP, auth E.164, dedup, comandos deterministas `STATUS`/`PAUSAR`/`REANUDAR`/`SILENCIAR`/`CONFIRMAR`/`CANCELAR`, router a especialistas).  
  *Hecho cuando:* Tests L1/L2 cubren auth, dedup, guardrail de destino, flujo completo `L1` y expiración.

### Bloque C — Agentes Fase 1 (`Sentinel`, `Tank`, `Shikamaru`)
- [ ] **T-06:** Implementar **Sentinel** (`src/agents/sentinel.ts`): polling de Redis `monitoring:tenant:*`, máquina de estados `ok/degraded/down`, cooldown 30 min, silenciamiento, consolidación anti-tormenta (>3 caídas) y consulta bajo demanda.  
  *Hecho cuando:* Tests L1/L3 validan transición caída→alerta→cooldown→recuperación y consolidación de 10 caídas en 1 mensaje.
- [ ] **T-07:** Implementar **Tank** (`src/agents/tank.ts`): consulta read-only a `public.tenants`, lectura de salud en Redis, herramienta SSH read-only con timeout 8 s y truncado a 4 000 chars, consulta IRD CGI y catálogo cerrado de acciones `L1` (`restart_service` solo sobre `xmanager|multicast|streamer`).  
  *Hecho cuando:* Tests L1/L3 validan resolución de tenant, rechazo de comandos fuera de whitelist, truncado SSH y creación de approval `L1`.
- [ ] **T-08:** Implementar **Shikamaru** (`src/agents/shikamaru.ts`): sincronización `git pull --ff-only` de `xPlanning.git` (`main`), lector indexado de `Memory/INDEX.md`, `Tenants/<id>/MEMORY.md` y carpetas `01_Borradores`/`02_Aprobados`, síntesis con motor Claude y creación de borradores en rama `draft/shikamaru-<slug>` bajo confirmación `L1`.  
  *Hecho cuando:* Tests L1/L3 validan lectura de repo local, fallback ante error de red Git y bloqueo de push directo a `main`.

---

## 8. Criterios de Aceptación (Definition of Done - DoD)
- [ ] Programación e implementación realizada por **Claude Code** en `xRoster` y `xBackend`.
- [ ] Suite completa de pruebas L1, L2 y L3 en verde tanto en `xRoster` como en `xBackend`.
- [ ] Cero impacto en el flujo actual de soporte a clientes de WhatsApp en `xBackend`.
- [ ] Regla No-Astra cumplida al 100% en todos los prompts y mensajes.
- [ ] `MEMORY.md` creado en `xRoster` y `Planning-X/Memory/INDEX.md` actualizado tras la implementación.
