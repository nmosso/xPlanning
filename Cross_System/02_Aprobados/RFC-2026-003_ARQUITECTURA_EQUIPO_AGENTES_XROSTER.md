# Request For Comments (RFC): Arquitectura del Equipo de Agentes Operativos (`xRoster`)

| Metadato | Detalle |
| :--- | :--- |
| **RFC ID:** | `RFC-2026-003` |
| **Sistema(s):** | `Cross_System` (`xRoster` [nuevo], `xBackend`, `xWorker`, `xDevices`, `Planning-X`) |
| **Estado:** | `Aprobado` |
| **Fecha:** | 2026-11-24 |
| **Autor(es):** | Lead Architect (`Planning-X`) |

---

## 1. Resumen (Abstract)

Define la arquitectura, gobernanza de seguridad y plan de puesta en marcha por fases de **`xRoster`**: un microservicio independiente en Docker que coordina un equipo de **7 agentes IA especializados** accesibles desde un **grupo privado de WhatsApp ("War Room")** donde participa exclusivamente el operador principal.

- **Despliegue Fase 1 (Camino A):** Contenedor Docker aislado en el servidor de `xBackend` con límites estrictos de recursos (`1.0 vCPU`, `1.5 GB RAM`) y configuración 100% externalizada en `.env` para migrar sin fricción a una VM dedicada en 5 minutos cuando se requiera mayor aislamiento.
- **Pasarela WhatsApp:** Reutiliza la instancia Docker existente de **Evolution API v2** mediante forward firmado y aislamiento estricto del grupo War Room.
- **Stack IA Híbrido:** Combina los dos planes activos del equipo (**Gemini** + **Claude Code**), con telemetría de cuotas y capacidad ociosa (*headroom*) desde el día 1.
- **Seguridad `R4: A`:** Lectura/diagnóstico autónomo (`L0`), escritura/operación con confirmación explícita `CONFIRMAR <código>` (`L1`) y bloqueo duro en código para operaciones destructivas o mensajes a clientes (`L2`).

---

## 2. Constitución del Proyecto (`xRoster` — SDD Fase 1)

1. **Contenedor aislado y portable:** `xRoster` corre en su propio contenedor Docker con límites duros de CPU/RAM y todas las conexiones por `.env` (migra a VM dedicada sin cambios de código).
2. **Aislamiento absoluto de WhatsApp:** El grupo War Room (`WHATSAPP_COMMAND_GROUP_ID`) queda excluido del flujo de soporte en `xBackend`; `xRoster` **jamás** puede enviar mensajes a grupos de clientes.
3. **Identidad verificada:** Solo se procesan mensajes del grupo War Room cuyo remitente coincida con la whitelist E.164 del operador (`OWNER_WHATSAPP_PHONES`).
4. **Determinismo antes que LLM:** Sentinel evalúa umbrales y Heartbeats por código determinista; el LLM solo sintetiza, correlaciona o explica anomalías reales.
5. **Seguridad por niveles (`L0`/`L1`/`L2`):** `L0` autónomo, `L1` exige `CONFIRMAR <código>` (TTL 5 min), `L2` prohibido en código.
6. **Gobierno de cuotas y capacidad (`Axelrod`):** Cada invocación a Gemini o Claude Code registra ventana, tokens/requests y saturación en `quota_usage_ledger` para exprimir al máximo las suscripciones pagas sin exceder límites.
7. **Sincronización segura de Planning:** Shikamaru consume `https://github.com/nmosso/xPlanning.git` (`main`) por `git pull --ff-only` y nunca hace push destructivo sobre `main`.
8. **Motor de programación:** Toda implementación de `xRoster` y el forward en `xBackend` serán programados por **Claude Code** bajo TDD (L1/L2/L3).
9. **Terminología neutra:** Ningún reporte, alerta ni mensaje usa términos prohibidos (regla UI/mensajes neutros).

---

## 3. Organigrama Definitivo del Equipo (7 Agentes)

| Nombre Clave | Rol | Fase | Motor IA | Fuentes de Datos / Herramientas | Nivel Máx. |
| :--- | :--- | :---: | :--- | :--- | :---: |
| **Jarvis** | Jefe de Gabinete & Orquestador (WhatsApp War Room) | **F1** | Gemini Flash | Evolution API v2, Router de intenciones, Gestor de aprobaciones (`CONFIRMAR`), Consolidación de alertas | `L1` (gate) |
| **Sentinel** | QoS & Monitoreo Proactivo | **F1** | Determinista + Gemini Flash | Redis `monitoring:tenant:{id}` (`xBackend`), InfluxDB local (`100.64.X.8:8086`, bucket `channels`), IRD CGI (`100.64.X.2:20135`) | `L0` |
| **Tank** | Infraestructura & Hardware | **F1** | Gemini Flash *(escala a Claude en diagnósticos profundos)* | `postgres-xserver` (`public.tenants`, usuario RO), SSH/scripts `xWorker` (`100.64.N.2/8`, `:8522`, `:8222`), `xDevices` (ISDB-T) | `L1` |
| **Shikamaru** | Arquitecto & Planificación SDD | **F1** | Claude Code / Claude | Git `https://github.com/nmosso/xPlanning.git` (`main`), lectura de specs/memoria, borradores SDD y planificación de backlog | `L1` (ramas docs) |
| **Axelrod** | Quota, Capacity & Model Intelligence Manager | **F2** | Gemini Flash + telemetría de planes | `quota_usage_ledger` (cuota usada vs disponible en Gemini y Claude Code), coordinación con Shikamaru para aprovechar margen ocioso, evaluación diaria de modelos y fórmulas de uso | `L0` |
| **Atlas** | PM & Operaciones | **F2** | Gemini Flash | `incident_tickets` (Kanban vía RPC `xBackend`), analítica WhatsApp de soporte, seguimiento de tareas y prioridades | `L1` |
| **Skipper** | Despliegues & Release | **F3** | Claude Code / Claude | Git/repos, despliegues controlados, migraciones SQL, smoke tests post-deploy y rollbacks | `L1` (doble check) |

---

## 4. Propuesta Técnica Detallada

### 4.1. Topología de Red y Despliegue (Camino A — Portable a VM)

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
                (existente) │                    │ (guardrail: solo COMMAND_GROUP_ID)
                            ▼                    │
        ┌───────────────────────────┐            │
        │        xBackend           │            │
        │  Si group === COMMAND_ID: │            │
        │  1) Ignora en Soporte     │            │
        │  2) Forward POST firmado  │            │
        └─────────────┬─────────────┘            │
                      │ HTTP interno Docker + X-Internal-Secret
                      ▼                          │
┌────────────────────────────────────────────────┴────────────────────────┐
│       CONTENEDOR DOCKER AISLADO: xRoster (1 vCPU / 1.5 GB RAM)         │
│         [Host actual: xBackend | Portable a VM dedicada vía .env]       │
│                                                                         │
│  ┌────────────────┐    ┌──────────────────────┐    ┌─────────────────┐  │
│  │  Jarvis Core   │───▶│  Provider Router +   │───▶│ Plan Gemini     │  │
│  │ (Webhook, Auth │    │  Quota Ledger        │───▶│ Plan Claude Code│  │
│  │  Dedup, L1)    │    │ (quota_usage_ledger) │    └─────────────────┘  │
│  └───────┬────────┘    └──────────────────────┘                         │
│          │                                                              │
│    ┌─────┴──────────┬──────────────┬───────────────┬──────────────┐     │
│    ▼                ▼              ▼               ▼              ▼     │
│ [Sentinel]       [Tank]      [Shikamaru]       [Axelrod]    [Atlas/Skip]│
│ QoS Worker     SSH/xWorker   xPlanning.git    Quota/Models   (F2 / F3)  │
│    │                │              │               │                    │
│    ▼                ▼              ▼               ▼                    │
│ Redis xBackend   100.64.N.*    GitHub Repo     PostgreSQL (xrosterdb)  │
│ InfluxDB/IRD     PG RO Tenants (branch main)   (cuotas, approvals, log) │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2. Enfoque de Axelrod: Cuotas y Capacidad Ociosa (R3)
En lugar de enfocarse solo en gasto pay-as-you-go, el motor de **Axelrod** mide y optimiza el aprovechamiento de los **2 planes contratados** (**Gemini** y **Claude Code**):
1. **Tracking por ventana de cuota:** Registra por proveedor y modelo el conteo de invocaciones, tokens de entrada/salida/caché, duración, errores `429 / rate_limit` y estimación de % de cuota consumida en ventanas móviles (5h / diaria / semanal).
2. **Detección de capacidad ociosa (*Headroom*):** Calcula diariamente cuánto margen libre queda antes del reset de cuota de cada plan.
3. **Coordinación con Shikamaru:** Cuando existe margen ocioso significativo, propone despachar tareas planificadas de alto valor (ej. auditoría de logs de cabeceras, revisión de specs en `01_Borradores/`, actualización de bitácoras de tenants) para **exprimir al máximo las suscripciones**. Cuando un plan se acerca al límite (>80%), recomienda priorizar tareas críticas o desviar triage liviano al otro plan.
4. **Evaluación diaria de modelos y fórmulas:** Reporta diariamente estado de cuotas/recursos y propone mejoras de modelos, prompt caching y asignación agente→modelo.

### 4.3. Almacenamiento de `xRoster` (`xrosterdb` en PostgreSQL) y Futuro `cpanel-roster`
Aprovechando el servidor **PostgreSQL** ya operativo en el host de `xBackend`, `xRoster` utiliza una **base de datos dedicada y aislada: `xrosterdb`** (con usuario propio `xroster_user`, sin permisos de escritura sobre la BD principal de `xBackend`).
- **Ventajas técnicas:** Cero consumo extra de motor, concurrencia real para lecturas/escrituras simultáneas de los agentes y el futuro panel web **`cpanel-roster`**, tipos nativos (`UUID`, `TIMESTAMPTZ`, `JSONB` con índices GIN) y agregaciones analíticas rápidas (`date_trunc`, window functions) para reportes de cuotas de **Axelrod**, auditoría y QoS.
- **Panel futuro (`cpanel-roster`):** Panel web de gestión general y reportes accesibles desde PC (análogo a `cpanel-admin`), conectado **directamente** a la API REST de `xRoster` (sin `xProxy`), que consumirá `xrosterdb` para dashboards de cuotas/modelos, alertas de Sentinel, aprobaciones `L1` y bitácora de agentes.
- **Inventario de cabeceras:** `xRoster` consulta `public.tenants` en la BD de `xBackend` mediante un pool separado con credenciales estrictamente **read-only** (`PG_XSERVER_RO_URL`).

### 4.4. Impacto en Doctrinas Fundamentales
- **Filtro Británico:** Stack Node.js 22 + TypeScript + Docker + PostgreSQL (`xrosterdb` en instancia existente). Sin frameworks multi-agente pesados ni segunda instancia de Evolution API.
- **Filtro Alemán:** Contratos estrictos TypeScript/JSONB, aislamiento de `WHATSAPP_COMMAND_GROUP_ID` en `xBackend`, usuario PG read-only para `xBackend`, base separada `xrosterdb` con DDL tipado e índices GIN/B-Tree.
- **Filtro Ruso:** Límites de recursos Docker, timeout de 3 s en el forward de `xBackend` (un fallo de `xRoster` jamás afecta `xBackend`), confirmación `CONFIRMAR <código>` con TTL de 5 min para `L1`, truncado de salidas SSH antes de enviar al LLM y agrupación anti-tormenta en alertas de Sentinel.

---

## 5. Alternativas Consideradas y Rechazadas

- **Alternativa A — Integrar los agentes dentro del código de `xBackend`:** Rechazada por acoplamiento de ciclo de vida y riesgo operativo.
- **Alternativa B — Segunda instancia Docker de Evolution API:** Rechazada por **R1** (duplica mantenimiento de sesiones WhatsApp Web).
- **Alternativa C — SQLite embebido (WAL):** Descartada al confirmarse el futuro panel web **`cpanel-roster`** y reportes analíticos concurrentes; una base dedicada **`xrosterdb`** en el PostgreSQL existente entrega mayor potencia (`JSONB`, agregaciones, concurrencia) sin costo de infraestructura adicional.

---

## 6. Plan de Implementación por Fases

| Fase | Alcance | Spec / Documento |
| :--- | :--- | :--- |
| **Fase 1 (MVP)** | Core `xRoster` (Docker + PostgreSQL `xrosterdb`) + Forward `xBackend` + **Jarvis** + **Sentinel** + **Tank** + **Shikamaru** + `quota_usage_ledger` activo | `REQ-2026-012` |
| **Fase 2** | **Axelrod** (reporte diario de cuotas/headroom + radar de modelos + coordinación con Shikamaru) + **Atlas** (Kanban & PM) + MVP de **`cpanel-roster`** (reportes y gestión) | `REQ-2026-013` (futuro) |
| **Fase 3** | **Skipper** (despliegues controlados, migraciones y rollbacks con doble confirmación) + panel completo en **`cpanel-roster`** | `REQ-2026-014` (futuro) |
