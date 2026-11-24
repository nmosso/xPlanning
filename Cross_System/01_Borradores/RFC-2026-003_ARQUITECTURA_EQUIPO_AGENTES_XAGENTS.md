# Request For Comments (RFC): Arquitectura del Equipo de Agentes Operativos (`xAgents`)

| Metadato | Detalle |
| :--- | :--- |
| **RFC ID:** | `RFC-2026-003` |
| **Sistema(s):** | `Cross_System` (`xAgents` [nuevo], `xBackend`, `xWorker`, `xDevices`, `Planning-X`) |
| **Estado:** | `Borrador` |
| **Fecha:** | 2026-11-24 |
| **Autor(es):** | Lead Architect (`Planning-X`) |

---

## 1. Resumen (Abstract)

Propone la arquitectura, gobernanza de seguridad y plan de puesta en marcha por fases de **`xAgents`**: un microservicio independiente alojado en una VM dedicada que coordina un equipo de **7 agentes IA especializados** accesibles desde un **grupo privado de WhatsApp ("War Room")** donde participa exclusivamente el operador principal.

El sistema reutiliza la instancia Docker existente de **Evolution API v2** (sin abrir una segunda sesión de WhatsApp), opera bajo un esquema **híbrido Gemini + Claude**, impone **confirmación explícita obligatoria para toda acción de escritura en producción** y registra desde el día 1 el consumo de tokens/costos para gobierno FinOps.

---

## 2. Constitución del Proyecto (`xAgents` — SDD Fase 1)

1. **Microservicio aislado:** `xAgents` corre en su propia VM; una falla o saturación de LLMs jamás puede degradar `xBackend`, `xManager` ni la señal IPTV.
2. **Aislamiento total de WhatsApp:** El grupo War Room (`WHATSAPP_COMMAND_GROUP_ID`) está aislado del flujo de soporte a clientes; `xAgents` **nunca** puede enviar mensajes a grupos de clientes.
3. **Identidad verificada:** Solo se procesan mensajes del grupo War Room cuyo remitente coincida con la whitelist E.164 del operador (`OWNER_WHATSAPP_PHONES`).
4. **Regla determinista antes que LLM:** El monitoreo (Sentinel) evalúa umbrales por código determinista; el LLM solo sintetiza, correlaciona o explica anomalías reales (ahorro de tokens y cero alucinaciones).
5. **Seguridad por niveles (`L0`/`L1`/`L2`):** Lectura autónoma (`L0`), escritura/operación solo con `CONFIRMAR <código>` con TTL de 5 min (`L1`), operaciones destructivas prohibidas en código (`L2`).
6. **Observabilidad FinOps nativa:** Cada llamada a Gemini o Claude pasa por un único wrapper que registra tokens, caché, latencia y costo estimado en USD (`llm_usage_ledger`).
7. **Circuit breaker de gasto:** Si el costo diario supera `DAILY_BUDGET_USD`, se bloquean llamadas no críticas y se alerta al operador.
8. **Motor de programación:** Toda implementación de `xAgents` y los ajustes mínimos en `xBackend` serán programados por **Claude Code** bajo TDD (pruebas L1/L2/L3).
9. **Terminología neutra:** Se mantiene la regla UI/mensajes neutros (sin referencias prohibidas en reportes o alertas).

---

## 3. Organigrama Definitivo del Equipo (7 Agentes)

| Nombre Clave | Rol | Fase | Modelo Base (Híbrido) | Fuentes de Datos / Herramientas | Nivel Máx. |
| :--- | :--- | :---: | :--- | :--- | :---: |
| **Jarvis** | Jefe de Gabinete & Orquestador (WhatsApp War Room) | **F1** | Gemini Flash | Evolution API v2, Router de intenciones, Gestor de aprobaciones (`CONFIRMAR`), Consolidación de alertas | `L1` (gate) |
| **Sentinel** | QoS & Monitoreo Proactivo | **F1** | Determinista + Gemini Flash | Redis `monitoring:tenant:{id}` (`xBackend`), InfluxDB local (`100.64.X.8:8086`, bucket `channels`), IRD CGI (`100.64.X.2:20135`) | `L0` |
| **Tank** | Infraestructura & Hardware | **F1** | Gemini Flash *(escala a Claude Sonnet en fallas complejas)* | `postgres-xserver` (`public.tenants`), SSH/scripts `xWorker` (`100.64.N.2/8`, `:8522`, `:8222`), `xDevices` (moduladores ISDB-T) | `L1` |
| **Shikamaru** | Arquitecto & Planificación SDD | **F1** | Claude Sonnet | Repositorio `Planning-X` (Borradores, Aprobados, `Memory/INDEX.md`, `Tenants/<id>/MEMORY.md`), redacción de specs/handoffs | `L1` (solo repo docs) |
| **Axelrod** | FinOps, Contabilidad & Inteligencia de Modelos IA | **F2** | Gemini Flash + búsqueda web/precios | `llm_usage_ledger` (consumo diario Gemini/Claude), límites de presupuesto, análisis de modelos/precios y propuestas de optimización | `L0` |
| **Atlas** | PM & Operaciones | **F2** | Gemini Flash | `incident_tickets` (Kanban vía RPC `xBackend`), métricas del asistente de soporte WhatsApp, seguimiento de pendientes y sprints | `L1` |
| **Skipper** | Despliegues & Release | **F3** | Claude Sonnet | Git/repos, pipelines de despliegue controlados, migraciones SQL, smoke tests post-deploy y rollbacks | `L1` (doble check) |

> [!NOTE]
> **Instrumentación temprana para Axelrod:** Aunque el agente conversacional **Axelrod** se activa en **Fase 2**, su base de datos (`llm_usage_ledger`) y el wrapper de costos se construyen en **Fase 1** para acumular métricas reales desde el primer mensaje de Jarvis, Sentinel, Tank y Shikamaru.

---

## 4. Propuesta Técnica Detallada

### 4.1. Topología y Flujo de Datos

```
                   ┌──────────────────────────────────────┐
                   │   WhatsApp (Grupo Privado War Room)  │
                   │      [Solo Operador Principal]       │
                   └──────────────────┬───────────────────┘
                                      │
                                      ▼
                   ┌──────────────────────────────────────┐
                   │   Evolution API v2 (Docker actual)   │
                   └────────┬────────────────────▲────────┘
                 Webhook    │                    │ HTTP POST sendText
                (existente) │                    │ (solo WHATSAPP_COMMAND_GROUP_ID)
                            ▼                    │
        ┌───────────────────────────┐            │
        │        xBackend           │            │
        │  Si group === COMMAND_ID: │            │
        │  1) Ignora en Soporte     │            │
        │  2) Forward POST firmado  │            │
        └─────────────┬─────────────┘            │
                      │ HTTPS/VPN + X-Internal-Secret
                      ▼                          │
┌────────────────────────────────────────────────┴────────────────────────┐
│                     VM DEDICADA: xAgents (Docker)                       │
│                                                                         │
│  ┌────────────────┐    ┌──────────────────┐    ┌─────────────────────┐  │
│  │  Jarvis Core   │───▶│  LLM Router +    │───▶│  Gemini API (Flash) │  │
│  │ (WhatsApp API, │    │  FinOps Wrapper  │───▶│  Claude API (Sonnet)│  │
│  │  Auth, Dedup,  │    │ (llm_usage_ledger│    └─────────────────────┘  │
│  │  Approvals L1) │    │  + Budget Guard) │                             │
│  └───────┬────────┘    └──────────────────┘                             │
│          │                                                              │
│    ┌─────┴──────────┬──────────────┬───────────────┬──────────────┐     │
│    ▼                ▼              ▼               ▼              ▼     │
│ [Sentinel]       [Tank]      [Shikamaru]       [Axelrod]    [Atlas/Skip]│
│ QoS Worker     SSH/xWorker   Planning-X       FinOps Cron    (F2 / F3)  │
│ (Cron/Poll)    & xDevices    Git/Docs         (Diario)                  │
│    │                │              │                                    │
│    ▼                ▼              ▼                                    │
│ Redis xBackend   100.64.N.*    Planning-X      SQLite/PG Local          │
│ InfluxDB/IRD     PG Tenants    (clone/sync)    (ledger + audit)         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2. Integración Limpia con Evolution API v2 (Sin Romper `xBackend`)
Para cumplir **R1** sin duplicar sesiones de WhatsApp Web ni tocar la estabilidad de `xBackend`:
1. **Cambio mínimo en `xBackend` (~25 líneas):**
   - Nueva variable de entorno opcional `WHATSAPP_COMMAND_GROUP_ID` y `XAGENTS_WEBHOOK_URL` + `XAGENTS_WEBHOOK_SECRET`.
   - En el webhook de WhatsApp de `xBackend`, **antes** de `sessions.ingest` y de `supportGroup`, si `group_id === WHATSAPP_COMMAND_GROUP_ID`:
     - No crea sesión, no dispara emergencia ni analítica de soporte.
     - Reenvía el payload en modo *fire-and-forget* (timeout 3 s) hacia `XAGENTS_WEBHOOK_URL` con header `X-Internal-Secret`. Si `xAgents` está caído, solo loguea warning y responde `200 OK` a Evolution API.
2. **Envío saliente desde `xAgents`:**
   - `xAgents` llama directamente a `EVOLUTION_API_URL/message/sendText/{instance}` con un guardrail duro en código: **si `remoteJid !== WHATSAPP_COMMAND_GROUP_ID`, lanza excepción y aborta**.

### 4.3. Especificación de la VM Dedicada (`xAgents`)
Para preparar la VM mencionada en **R2**:
- **SO:** Ubuntu Server 24.04 LTS.
- **Recursos:** 2 vCPU, 4 GB RAM, 30 GB SSD (suficiente para Node.js + SQLite/Postgres + Redis + clone de `Planning-X`/`xWorker`).
- **Software base:** Docker + Docker Compose, `git`, `openssh-client`.
- **Conectividad de red requerida:**
  1. Red privada/VPN hacia `xBackend` (Redis de monitoreo, PostgreSQL `postgres-xserver` con usuario **read-only** para consultas de inventario, Evolution API HTTP) y recepción del forward HTTP desde `xBackend`.
  2. Acceso VPN `100.64.0.0/16` hacia cabeceras (`Proxmox .2:22`, `xManager .8:22`, `Multicast :8522`, `Streamer :8222`, InfluxDB `:8086`, IRD `:20135`).
  3. Salida HTTPS a `generativelanguage.googleapis.com` y `api.anthropic.com` (y Git remoto para `Planning-X`).

### 4.4. Contratos de Datos y Esquema Local (`xAgents`)
Base de datos local en la VM (PostgreSQL liviano en Docker o SQLite con WAL; se recomienda **PostgreSQL 16** en el mismo `docker-compose.yml` para mantener 100% paridad SQL con el ecosistema xIPTV):

1. **`llm_usage_ledger`** *(Base de Axelrod — activa desde Fase 1)*:
   - `id`, `created_at`, `agent_name`, `provider` (`gemini` | `anthropic`), `model`, `task_type`, `input_tokens`, `output_tokens`, `cached_tokens`, `cost_usd` (`numeric(10,6)`), `latency_ms`, `status`, `error_code`.
2. **`action_approvals`** *(Protocolo `R4: A`)*:
   - `code` (ej. `TK-482`), `agent_name`, `action_type`, `target_tenant`, `summary_text`, `payload_jsonb`, `status` (`pending` | `confirmed` | `rejected` | `expired` | `executed` | `failed`), `requested_at`, `expires_at` (`+5 min`), `decided_at`, `result_summary`.
3. **`agent_audit_log`**:
   - Registro inmutable de cada orden recibida por WhatsApp, agente invocado, herramientas ejecutadas, duración y resultado.
4. **`sentinel_alert_state`**:
   - Estado por `tenant_id + metric_key` (`ok` | `degraded` | `down`), `first_seen_at`, `last_notified_at`, `silenced_until` (anti-spam / cooldown + detección de recuperación `✅ RECUPERADO`).

### 4.5. Protocolo de Interacción en WhatsApp (UX de War Room)
- **Conversación natural o directa por nombre:**
  - `"@Tank revisá el server 43, cómo está el disco y el IRD"` → **Jarvis** rutea directo a **Tank**.
  - `"¿Qué tenemos caído ahora?"` → **Jarvis** consulta a **Sentinel**.
  - `"Shikamaru, armá borrador de idea para alerta de bitrate bajo"` → **Jarvis** rutea a **Shikamaru**.
- **Flujo de aprobación `L1` (`R4: A`):**
  1. Operador: `"Tank, reiniciá xManager en el server 43"`
  2. **Tank** verifica estado actual (`L0`) y responde vía **Jarvis**:
     > ⚠️ **Confirmación requerida (`TK-482`)**  
     > **Agente:** Tank  
     > **Acción:** `systemctl restart xmanager` en `100.64.43.8` (Frentones [43])  
     > **Expira en:** 5 minutos  
     > Respondé **`CONFIRMAR TK-482`** o **`CANCELAR TK-482`**.
  3. Operador: `"CONFIRMAR TK-482"` → **Tank** ejecuta, valida post-check y reporta resultado.
- **Comandos de control de Jarvis:**
  - `STATUS` → Salud de los 7 agentes, conexión a `xBackend`/Evolution y gasto IA del día.
  - `SILENCIAR <tenant|all> <minutos>` → Pausa alertas de Sentinel durante mantenimientos.
  - `PAUSAR` / `REANUDAR` → Kill-switch rápido del grupo War Room.

### 4.6. Diseño Específico de **Axelrod** (FinOps & Inteligencia de Modelos — Fase 2)
- **Evaluación diaria automática** (ej. 09:00 `America/Asuncion`) enviada al War Room:
  - Gasto de ayer y acumulado del mes desglosado por proveedor (**Gemini** vs **Claude**) y por agente (`Jarvis`, `Sentinel`, `Tank`, `Shikamaru`, etc.).
  - Ratio de eficiencia (tokens de entrada vs salida, uso de prompt caching, llamadas fallidas o costosas).
  - Alertas de anomalías (ej. *"Tank subió 300% el consumo por logs SSH sin truncar"*).
- **Radar de Modelos y Optimización (semanal o bajo demanda `"Axelrod, revisá modelos"`):**
  - Compara catálogo vigente de precios/capacidades de Gemini y Claude contra los patrones reales del `llm_usage_ledger`.
  - Propone cambios concretos de routing o fórmulas de uso (ej. *"Mover clasificación de logs de Tank de Sonnet a Gemini Flash-Lite ahorra ~$X/mes sin pérdida de precisión"*; *"Activar context caching para el índice de Planning-X en Shikamaru"*).

### 4.7. Impacto en Doctrinas Fundamentales
- **Filtro Británico:** Stack idéntico al ecosistema actual (Node.js + TypeScript + PostgreSQL + Docker). Sin frameworks pesados ni abstracciones multi-agente opacas (nada de cadenas complejas innecesarias): un router limpio + herramientas tipadas + wrappers directos a SDK oficial de Google Gemini y Anthropic Claude.
- **Filtro Alemán:** Contratos estrictos en JSON/TypeScript, separación física en VM dedicada, aislamiento del grupo WhatsApp en `xBackend`, usuario SQL read-only para consultas de inventario y tabla de auditoría inmutable.
- **Filtro Ruso (Matriz L1/L2/L3):**
  - **L1 (Edge Cases):** Mensajes de números no autorizados en el grupo (ignorados y alertados), `CONFIRMAR` expirado o duplicado, texto vacío/audios/stickers, salida SSH gigante (truncado estricto antes de pasar al LLM para no explotar tokens).
  - **L2 (Integración):** `tsc --noEmit`, tests unitarios de router/approvals/guardrail saliente (`remoteJid !== COMMAND_GROUP` siempre falla), test del forward en `xBackend` sin afectar `npm test` existente.
  - **L3 (Resiliencia):** `xAgents` caído no afecta `xBackend`; Redis/InfluxDB de una cabecera caído dispara timeout de 5 s y continúa con el resto de la flota; tormenta de caídas (ej. corte masivo de internet) agrupa alertas en un solo mensaje consolidado en vez de enviar 50 WhatsApps.

---

## 5. Alternativas Consideradas y Rechazadas

- **Alternativa A — Integrar los 7 agentes dentro de `xBackend`:** Rechazada por **R2** y Filtro Ruso (tareas SSH largas, sesiones LLM o fallos de dependencias externas no deben compartir proceso ni memoria con la API crítica de producción).
- **Alternativa B — Segunda instancia de Evolution API con otro número:** Rechazada por **R1** (duplica mantenimiento de sesiones WhatsApp Web/QR y recursos Docker cuando el aislamiento por JID ya está probado en `REQ-2026-005`).
- **Alternativa C — Frameworks multi-agente pesados (CrewAI / AutoGen / LangGraph):** Rechazada por Filtro Británico (introducen Python/dependencias pesadas, prompts ocultos que disparan consumo de tokens y dificultan auditar el guardrail `L0`/`L1`/`L2`).

---

## 6. Plan de Implementación por Fases

| Fase | Alcance | Entregables Clave |
| :--- | :--- | :--- |
| **Fase 1 (MVP — Prioridad Inmediata)** | **Core `xAgents` + Jarvis + Sentinel + Tank + Shikamaru** (+ `llm_usage_ledger` activo) | 1) Setup VM Docker + forward aislado en `xBackend`.<br>2) **Jarvis:** War Room, auth E.164, routing, aprobaciones `CONFIRMAR`.<br>3) **Sentinel:** Polling de `monitoring:tenant:*` (Redis) + alertas consolidadas con cooldown y recuperación.<br>4) **Tank:** Consultas en vivo de `tenants`, salud de Proxmox/VMs/IRD y reinicios controlados (`L1`).<br>5) **Shikamaru:** Lectura de `Planning-X`, consulta de specs/memoria y creación de borradores/ideas desde WhatsApp. |
| **Fase 2 (Gestión & FinOps)** | **Axelrod + Atlas** | 1) **Axelrod:** Reporte diario de costos/tokens Gemini+Claude, detección de desperdicios y radar de modelos.<br>2) **Atlas:** Gestión de Kanban (`incident_tickets`), prioridades y resúmenes ejecutivos. |
| **Fase 3 (Releases)** | **Skipper** | Despliegues controlados, migraciones SQL supervisadas, smoke tests y rollbacks con doble confirmación. |

---

## 7. Preguntas Abiertas para Cerrar el RFC y Pasar a Spec de Fase 1

Para congelar este RFC y redactar la especificación detallada de **Fase 1** (`REQ-2026-012`) lista para que **Claude Code** genere el proyecto, confirmame estos 3 puntos:

- **#1 (Nombre del repositorio y carpeta):** ¿Te parece bien llamar al nuevo proyecto **`xAgents`** (en `d:\Projects\DEV\xIPTV\xAgents`)?
- **#2 (Acceso de Shikamaru a `Planning-X` desde la VM):** `Planning-X` vive hoy en tu PC Windows (`d:\Projects\DEV\xIPTV\Planning-X`). Para que **Shikamaru** pueda leer specs/memoria y proponerte borradores desde la VM Linux, ¿`Planning-X` está sincronizado en un repositorio **Git remoto** (GitHub/GitLab/Gitea) al que la VM pueda hacer `git pull` / crear ramas de borrador?
- **#3 (Presupuesto inicial / Circuit Breaker):** ¿Qué tope diario en USD quieres configurar por defecto en `DAILY_BUDGET_USD` para el circuit breaker de Fase 1 (ej. `5.00` USD/día o `10.00` USD/día) mientras calibramos el consumo real?
