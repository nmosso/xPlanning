# Nota de Memoria Técnica: Asistente WhatsApp de Soporte y Analítica (`REQ-2026-003`..`REQ-2026-006`)

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xBackend` |
| **Componente:** | `API & Workers` (Integración Evolution API + Gemini + Redis + xServerSQL) |
| **Funcionalidad:** | Escucha pasiva de grupos WhatsApp, supresión de tickets, alertas internas, analítica de casos, broadcast y modo emergencia |
| **Fecha:** | 2026-11-12 |
| **Autor/Agente:** | Lead Architect (Planning-X) |
| **Estado:** | REQ-2026-003, REQ-2026-004 y REQ-2026-005 en producción. REQ-2026-006 implementado (2026-11-12, xBackend + cpanel-admin, sin SQL; pendiente de despliegue) |

---

## 1. Resumen y Contexto
Sistema de monitoreo pasivo sobre grupos de soporte de WhatsApp que utiliza una instancia **Evolution API v2** (vinculada por QR) para reenviar eventos hacia `xBackend`. Su propósito es capturar reclamos sin saturar el Kanban ni responder en grupos públicos, extrayendo estadísticas de problemas y soluciones mediante **Gemini**.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión Tomada:**
  - Pasarela **Evolution API v2** desacoplada en Docker (evita comprometer la estabilidad de `xBackend` con el WebSocket de WhatsApp Web).
  - Ventana de supresión de tickets de **5 minutos**: si un miembro registrado en `whatsapp_support_staff` responde en < 5 min, no se crea ticket en `incident_tickets`.
  - Ventana de cierre de caso de **30 minutos** de inactividad: cierra la sesión, clasifica problema/causa raíz/solución con Gemini y persiste en `whatsapp_support_cases`.
  - Mapeo de `tenant_id` determinista mediante regex `\[([0-9a-zA-Z_-]+)\]` sobre el nombre del grupo (ej: `Frentones [43]` $\to$ `43`).
  - Bloqueo absoluto de mensajes salientes hacia grupos de clientes; la única salida permitida es hacia `WHATSAPP_SUPPORT_ALERT_GROUP_ID`.
- **Alternativa Descartada:**
  - Embeber Baileys directamente en Node.js de `xBackend` (descartado por acoplamiento de memoria y reconexiones).
  - Evaluación mensaje por mensaje (descartado por alto consumo de tokens y falta de contexto conversacional).
- **Archivos Clave:**
  - `Planning-X/xBackend/02_Aprobados/REQ-2026-003_ASISTENTE_WHATSAPP_SOPORTE_KANBAN.md`
  - `xServerSQL/_Functions/_Structure/0014_whatsapp_support.sql` (tablas e índices)
  - `xServerSQL/_Functions/WhatsApp/fn_whatsapp_create_kanban_ticket.sql`, `fn_whatsapp_save_closed_case.sql`, `fn_whatsapp_support_staff_list.sql`, `fn_whatsapp_kanban_summary.sql`
  - `xBackend/services/whatsapp/*` (gateway, sesión, worker, evaluador Gemini), `controllers/whatsapp.ts`, `server/routerwhatsapp.ts`

---

## 3. Contratos de Datos y Dependencias
- **Dependencias Directas:** `public.incident_tickets`, Redis (`wa:case:{groupId}`, `wa:support_staff_phones`; REQ-2026-006: `wa:agent_outbound_enabled`, `wa:agent_outbound_updated_at`, `wa:broadcast_status`, `wa:emergency_config`, `wa:emergency_replied_groups:{incident_id}`, `wa:emergency_lock:{incident_id}:{group_id}`, `wa:emergency_templates`), Evolution API v2, Google Gemini API (`gemini-2.5-flash`).
- **Procedimientos RPC:**
  - `fn_whatsapp_create_kanban_ticket(_data jsonb, _params jsonb, _operator jsonb)`
  - `fn_whatsapp_save_closed_case(_data jsonb, _params jsonb, _operator jsonb)`
  - `fn_whatsapp_kanban_summary(_data jsonb, _params jsonb, _operator jsonb)` (REQ-2026-005 RF-06; aplicado en producción 2026-11-12)

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- La normalización E.164 de teléfonos en `wa:support_staff_phones` es crítica para detectar correctamente intervenciones de soporte (considerar variantes de código de país como el `9` móvil en Argentina `549...` vs `54...`).
- El reinicio del servicio `xBackend` no debe perder el estado de los casos activos, por lo que los buffers y timestamps residen íntegramente en Redis.
- `549...` vs `54...`: `normalizePhone` canoniza `54` + 10 dígitos agregando el `9`, tanto para staff como para JIDs.

---

## 5. Implementación (2026-11-11)
- **Desvíos respecto a la spec (deliberados):**
  - `fn_whatsapp_create_kanban_ticket` valida el tenant contra `public.tenants`: si el tag del grupo no existe, el ticket se crea con `tenant_id = NULL` (la versión literal fallaba con FK `23503`) y el tag queda en `created_by.tenant_tag`. Es idempotente por `case_id`.
  - `fn_whatsapp_save_closed_case` es idempotente (`ON CONFLICT (id) DO NOTHING`).
  - Nuevo RPC `fn_whatsapp_support_staff_list` para sincronizar el Set Redis cada 5 min.
  - El nombre del grupo no viene en `messages.upsert`: se resuelve con `GET /group/findGroupInfos` y se cachea 1h (`wa:group_name:{jid}`).
  - Si Gemini falla 3 ticks seguidos al evaluar el ticket, se crea un ticket de respaldo `TICKET_SOPORTE`. Si falla en el cierre, el caso se guarda igual con `gemini_metadata.error`.
  - Mensajes que llegan durante el cierre abren un caso nuevo (no se pierden).
- **Pruebas:** `npm test` (node:test, 69 casos L1/L2/L3) y `npm run test:sql` (base efímera `xbackend_req003_test` + E2E worker→PostgreSQL).
- **Limitaciones conocidas:** el lock por grupo es por proceso (una sola instancia de xBackend con el worker activo); con Redis caído el fallback en memoria no se re-sincroniza al volver.

---

## 6. Panel de Gestión, QR y Analítica (`REQ-2026-004`, 2026-11-11)
- **Spec:** `Planning-X/Cross_System/02_Aprobados/REQ-2026-004_GESTION_WHATSAPP_PANEL_Y_ANALITICA.md`.
- **SQL (`xServerSQL/_Functions/WhatsApp/`):**
  - `fn_whatsapp_support_staff_manage` — `action`: `list | create | update | delete`. Teléfono forzado a solo dígitos; duplicado → `{success:false, code:'23505'}`. `delete` es idempotente (`deleted: false` si no existía).
  - `fn_whatsapp_cases_query` — sin `case_id`: `kpis` (`total_cases`, `handled_by_staff_count`, `tickets_created_count`, `avg_first_response_seconds`, `by_category`, `by_sentiment`) + `items` paginados (`_params.limit` 1..200, default 50; `offset`), sin `raw_transcript`/`gemini_metadata`. Con `case_id`: registro completo.
- **Endpoints (`xBackend`, directos desde cpanel-admin, sin xProxy):** `GET /v3/whatsapp/status`, `POST /v3/whatsapp/reconnect`, `GET|POST /v3/whatsapp/staff`, `PUT|DELETE /v3/whatsapp/staff/:id`, `GET /v3/whatsapp/cases`, `GET /v3/whatsapp/cases/:id`. Todos con `[validOrigin('console'), siteApikey()]`. Errores de RPC → 400 (404 si "no encontrado"), excepción de BD → 500, Evolution caída en reconnect → 502.
- **Gateway Evolution:** `getQrCode()`, `logoutInstance()`, `getConnectionAndQr()`, `reconnectInstance()`. Solo tocan `/instance/*` con timeout de 5 s; el bloqueo de salida (`sendSupportAlert` solo a `WHATSAPP_SUPPORT_ALERT_GROUP_ID`) no cambió.
- **cpanel-admin:** módulo lazy `src/app/modules/common/whatsapp-support/` (ruta `/whatsapp-support`, item de menú "Asistente WhatsApp" con `role: 'admin'`). `ENDPOINTS.whatsapp*` en `const.ts`. El servicio usa `HttpClient` directo (mismo header `apikey` que `ApiService`) para recibir el mensaje de error del backend (p. ej. teléfono duplicado). Kanban acepta `?ticket=<id>` para abrir el detalle desde la analítica.
- **Desvíos deliberados respecto al SQL literal de la spec:**
  - Los casts (`uuid`, `boolean`, `timestamptz`, `int`) se hacen en el cuerpo y no en `DECLARE`: en PL/pgSQL un error en `DECLARE` no lo captura el `EXCEPTION` del mismo bloque, así que un id mal formado abortaba la llamada.
  - `jsonb_object_agg` con categoría/sentimiento `NULL` falla ("field name must not be null"): se agrupan como `SIN_CATEGORIA` / `SIN_SENTIMIENTO`.
  - `avg_first_response_seconds` devuelve `0` (no `NULL`) con la tabla vacía, como pide L1.
  - La búsqueda escapa `%`, `_` y `\` (búsqueda literal); un solo CTE alimenta KPIs y página; el orden es `closed_at DESC NULLS LAST, id`.
- **Lecciones:**
  - Polling del QR cada 10 s con un máximo de 30 ciclos; se detiene al conectar (`open`) o al destruir el componente (`takeUntilDestroyed`).
  - `staff_sync: 'failed'` en la respuesta permite avisar en la UI sin perder el guardado; el worker vuelve a sincronizar cada 5 min.
- **Pruebas:** `test/whatsapp/sql/20_req004_tests.sql` (vía `npm run test:sql`), `test/whatsapp/gateway.test.ts` (QR/logout/reconnect + bloqueo saliente), `test/whatsapp/consoleApi.test.ts` (handlers + verificación de guards en las 8 rutas). `npm test`: 99 OK + 1 skip (E2E PG, corre en `test:sql`).

---

## 7. Pestañas, Tickets Generados y Comando Manual (`REQ-2026-005`, 2026-11-11)
- **Spec:** `Planning-X/Cross_System/02_Aprobados/REQ-2026-005_ASISTENTE_WHATSAPP_TABS_TICKETS_Y_COMANDO.md`.
- **SQL:** nuevo `fn_whatsapp_tickets_query` (`_Functions/WhatsApp/`), implementado tal cual el contrato de la spec. Solo `created_by->>'source' = 'whatsapp_agent'`; filtros `tenant_id` (coincide con `tenant_id` real **o** `created_by.tenant_tag`, para tickets sin FK), `status`, `is_archived` (default `false`), `search` literal (escapa `%`, `_`, `\`; título, descripción, `tenants.name`, `group_name`). Orden: `critico → pendiente → testing → verificar → terminado`, luego `created_at DESC`. `limit` 1..200 (default 50). `is_archived` inválido → `{success:false, code:'22P02'}`.
- **Endpoint:** `GET /v3/whatsapp/tickets` (`listTickets` en `createWhatsAppConsoleHandlers`, montado en `whatsappConsoleRoutes.ts` con `[validOrigin('console'), siteApikey()]`). Filtros → `_data`, `limit/offset` → `_params`; el resto de query params se descarta.
- **Grupo interno aislado (RF-04):** en el webhook, si `group_id === WHATSAPP_SUPPORT_ALERT_GROUP_ID` **no** se llama a `sessions.ingest` (no hay sesión 5/30 min, ni caso, ni analítica). Se delega en `SupportGroupCommandHandler` (`services/whatsapp/manualTicketCommand.ts`), inyectado como `supportGroup` en `createWhatsAppHandlers` desde `whatsappAgent.ts`.
- **Comandos `Agente:` (RF-05..RF-07, actualizado 2026-11-12):** el comando sin prefijo (`Generar <ID>: <Datos>`) fue **reemplazado**; ahora el prefijo `Agente:` es obligatorio (case-insensitive). `parseSupportGroupCommand` en `supportCaseRules.ts` devuelve `{ kind: 'generate', tenant_tag, text } | { kind: 'summary' } | { kind: 'help' } | null`:
  - `help`: `/^\s*agente\s*:\s*(?:ayuda|help)\s*$/i`; `summary`: `/^\s*agente\s*:\s*resumen\s*$/i`; `generate`: `/^\s*agente\s*:\s*(?:generar|gererar)\s+([0-9a-zA-Z_-]+)\s*:\s*([\s\S]+)$/i`. Quita comillas envolventes rectas **y tipográficas** (`“…”`, `‘…’`, que WhatsApp iOS inserta solo); problema vacío → `null` (se ignora en silencio). `Resumen por favor`, `Agente ayuda` (sin `:`) o texto sin prefijo → `null`.
  - Flujo común: dedup `wa:dedup:{message_id}` (TTL 120 s) antes de parsear, también para Resumen/Ayuda (un reintento no repite la respuesta).
  - **Generar:** `fn_whatsapp_create_kanban_ticket` con `tenant_id = <ID>`, `summary`, `category: 'TICKET_SOPORTE'`, `group_id` = grupo interno, `group_name: 'Soporte Interno (comando manual)'`, `analysis: { origin: 'manual_command', operator_phone, operator_name, message_id }` → confirmación `✅ Ticket creado` / `Cliente: <tenant_name> [<tenant_position>]` / `Detalle: <Problema>` (sin línea `ID:`; si el tenant no existe en BD → `Cliente: [<tenant_tag>]`; con nombre pero sin `position` → `<name> [<tenant_tag>]`) o `❌ …: <error>`.
  - El RPC resuelve el tag con `tenantid IN (tag, 'server' || tag) OR position::text = tag` (prioridad: exacto > `server<tag>` > `position`) y devuelve `tenant_name` / `tenant_position` (NULL si no existe).
  - **Resumen:** `fn_whatsapp_kanban_summary` (nuevo, `_Functions/WhatsApp/`): **todos** los `incident_tickets` con `is_archived = false AND status <> 'terminado'` (cualquier origen), con `tenant_name`, `tenant_position`, `tenant_tag`, ordenados `critico → pendiente → testing → verificar → otros`, `created_at DESC`. `formatKanbanSummary` (`manualTicketCommand.ts`) arma `📋 Resumen Kanban (N pendientes)` + grupos `🔴 CRÍTICO` / `🟡 PENDIENTE` / `🔵 TESTING` / `🟣 VERIFICAR` (y `⚪ OTROS` para estados desconocidos) con `(n)` por grupo, omitiendo grupos vacíos; cada línea `• <tenant_name> [<tenant_position>] — <title>` (`[<tenant_tag>]` si el ticket no tiene FK, `[Global]` si no tiene tenant). 0 activos → `✅ No hay tickets pendientes en el Kanban.`; fallo del RPC → `❌ No se pudo obtener el resumen del Kanban: <error>`.
  - **Ayuda/Help:** responde `HELP_TEXT` (lista de los 3 comandos con sintaxis), sin RPC.
  - Todas las respuestas van vía `gateway.sendSupportAlert` al mismo grupo interno.
- **Decisiones adicionales:**
  - `case_id` **determinista** (UUID derivado de SHA-1 del `message_id`): si Redis cae y el fallback en memoria no deduplica entre procesos, la idempotencia por `case_id` del RPC evita el doble ticket; ante `duplicate: true` no se reenvía la confirmación.
  - Tenant inexistente (p. ej. `999`): ticket con `tenant_id = NULL` y `tenant_tag = '999'` (comportamiento existente del RPC), visible en el listado filtrando por `999`.
  - Un fallo al enviar la confirmación se loguea y no rompe el flujo.
- **cpanel-admin:** pestañas `cases` (default) · `tickets` · `staff` · `connection`. `ngOnInit` solo carga casos; el estado/QR se consulta al entrar en Conexión & QR. Al salir de esa pestaña se llama `stopPolling()` y `startPolling()` no arranca si la pestaña activa no es `connection` (una respuesta tardía tras cambiar de pestaña no reactiva el polling). Pestaña Tickets: filtros Estado/Tenant/Búsqueda (debounce 400 ms)/switch "Ver archivados", paginación de 25, "Ver en Kanban" (`/kanban?ticket=<id>`) y "Archivar/Restaurar" con confirmación vía `PUT /v3/incidents/:id/archive` (`operator_identity_id` desde `localStorage`, igual que `KanbanService`). `ENDPOINTS.whatsappTickets` en `const.ts`.
- **Pruebas:** `test/whatsapp/sql/25_req005_tests.sql` (filtro por source, archivados, tenant/tenant_tag, búsqueda literal, paginación, entrada inválida, comando manual con tenant 999 e idempotencia), `test/whatsapp/sql/26_req005_agente_tests.sql` (resolución de tenant exacto / `server<tag>` / `position` / inexistente y `fn_whatsapp_kanban_summary`: filtro activos, orden, campos, vacío), `test/whatsapp/manualCommand.test.ts` (29 casos: parser `Agente:`, Generar/Resumen/Ayuda, dedup, aislamiento en webhook) y `consoleApi.test.ts` (`listTickets` + guards en las 9 rutas). `npm test`: 139 OK + 2 skip (los tests con PG real corren en `npm run test:sql`). El schema de test de `tenants` (`00_stubs.sql`) incluye `position INTEGER`. cpanel-admin verificado solo con `tsc --noEmit` y `ngc --noEmit` (sin build).

---

## 8. Broadcast Selectivo, Emergencia por Incidencia y Kill-Switch Maestro (`REQ-2026-006`, 2026-11-12)
- **Spec:** `Planning-X/Cross_System/02_Aprobados/REQ-2026-006_BROADCAST_SELECTIVO_GRUPOS_WHATSAPP.md`.
- **Kill-Switch Maestro (`wa:agent_outbound_enabled`):**
  - Modo Mudo (`R1: B`): cuando es `false`, bloquea en `WhatsAppGatewayService` el 100% de los envíos salientes (`sendSupportAlert` y `sendGroupMessage` lanzan `AgentMutedError`), manteniendo activa la ingesta silenciosa, el cierre de casos con Gemini y la creación de tickets en Kanban.
  - Aborto y Purga (`R2`): al apagar el agente (`PUT /v3/whatsapp/agent-toggle` con `{enabled: false}`), aborta inmediatamente cualquier difusión o envío de resolución en curso y purga los ítems pendientes (`aborted`). Al volver a encender, la cola arranca limpia.
- **Broadcast Manual y Emergencia (`R3`):**
  - Intervalo estricto de **2000 ms** entre envíos a grupos (`sendGroupMessage` exige `@g.us`).
  - Modo Emergencia (`wa:emergency_config`, apagado por defecto): cuando está encendido, Gemini (`evaluateEmergencyMatch`) detecta si el mensaje de un cliente corresponde a la incidencia; si coincide, responde 1 sola vez por grupo por `incident_id`, registra el grupo en `wa:emergency_replied_groups:{incident_id}` y marca la sesión `handled_by_staff = true`.
  - Al apagar la emergencia (`PUT /v3/whatsapp/emergency` con `enabled: false`), el operador elige entre enviar un `resolution_text` (que se despacha con 2s de intervalo **exclusivamente** a los grupos en `wa:emergency_replied_groups:{incident_id}`) o apagar sin enviar mensaje.
- **Implementación (2026-11-12):**
  - **Gateway** (`whatsappGatewayService.ts`): el store se inyecta como 3er parámetro del constructor (por defecto `MemoryWaStore`, compatible con el código existente). `isOutboundEnabled()` es `true` salvo que la clave valga `'0'`; `getOutboundState()` → `{ enabled, updated_at | null }`. En modo mudo, `AgentMutedError` se lanza **antes** del guardrail de destino y de cualquier HTTP. `sendGroupMessage` valida `^[0-9A-Za-z-]+@g\.us$` (rechaza `@s.whatsapp.net`, vacío o sufijos raros). `listGroups()` ordena con `compareWhatsAppGroups` (ver ampliación RF-04), usa el JID como `subject` si falta y no cachea nombres vacíos.
  - **Evaluador** (`geminiCaseEvaluator.ts`): `evaluateEmergencyMatch(incident_text, transcript)` envía solo mensajes de clientes (últimos 20), schema `{ matches: BOOLEAN, reason: STRING }`, **un solo intento** con timeout de 10 s (sin el backoff de 2/4 s de los otros prompts, para no demorar la respuesta). `matches` debe ser exactamente `true`; sin API key, red/JSON inválido ⇒ `{ matches: false, reason: 'eval_error' }`. Se pasa `session.transcript` (la sesión no tiene campo `messages`).
  - **Difusión** (`broadcastService.ts`): `startBroadcast(groups, text, kind)` → 400 (texto vacío / ningún `@g.us` tras deduplicar), 409 (otra en curso / modo mudo); devuelve el estado inicial y despacha en segundo plano; ítems con `subject` desde `wa:group_name:{id}`. Un grupo que falla queda `failed` y sigue el siguiente. `abortAndPurge()` cancela la espera activa (`CancellableDelay`), marca `pending` → `aborted` (el envío ya en vuelo conserva su resultado real) y devuelve la cantidad purgada. Estado persistido en `wa:broadcast_status` tras cada ítem; si el proceso se reinicia con un `running`, `getStatus()` lo informa como `aborted` (jamás se reanuda).
  - **Emergencia** (`emergencyService.ts`): `updateConfig` exige `incident_text` y `reply_text` al encender (400); cambiar solo `reply_text` conserva el `incident_id`. `checkAndReply` relee la config tras Gemini (si se apagó o cambió la incidencia mientras evaluaba, no responde), toma el lock `setNx` y agrega al Set **antes** de enviar; la cola de respuestas garantiza ≥ 2 s entre envíos automáticos. Decisión: si el envío falla o se aborta (`abortPendingReplies`), el grupo se **libera** (srem + del lock) — no cuenta como "notificado" y su próximo mensaje vuelve a evaluarse. Solo tras el envío exitoso se marca la sesión `handled_by_staff = true` (`staff_phone = 'emergency_auto'` si no había respuesta previa de staff). Apagar con resolución: valida texto (400), agente encendido (409) y que no haya otra difusión en curso (409) **antes** de apagar, así un 400/409 deja la emergencia activa; con 0 afectados apaga y responde `sent_to_groups: 0`. Apagar también cancela las respuestas automáticas aún encoladas.
  - **Webhook**: `sessions.ingest` ahora devuelve `{ status, session?, is_staff? }`; tras un `queued` de cliente se dispara `emergency.checkAndReply` sin `await` (errores solo se loguean). Staff, duplicados y el grupo interno no la disparan.
  - **Consola**: `PUT /v3/whatsapp/agent-toggle` con `enabled` no booleano → 400; al apagar responde `aborted_pending_count` = ítems de difusión purgados + respuestas de emergencia descartadas. `PUT /v3/whatsapp/emergency` solo pasa `enabled`, `incident_text`, `reply_text`, `resolution_text`, `send_resolution`. Errores del servicio: `WaRequestError` (400/404/409) → su código; otros → 500 genérico; `GET /groups` con Evolution caída → 502.
  - **cpanel-admin**: cabecera con badge + botón maestro y banner de modo mudo (todas las pestañas); 5 pestañas con `broadcast` en 3er lugar (badge `EMERGENCIA` cuando está encendida). Polling de difusión cada 2 s solo con la pestaña activa y `state === 'running'`. Al cambiar `incident_text` con la emergencia encendida pide confirmación (inicia incidencia nueva y el contador vuelve a 0). También se corrigió el texto de ayuda de la pestaña Tickets (`Agente: Generar <ID>: <problema>`).
  - **Ampliación RF-04 (orden por ID y selección flexible):** `compareWhatsAppGroups(a, b)` (exportada del gateway): grupo interno al final → con `tenant_id` antes que sin → `tenant_id.localeCompare(..., 'es', { numeric: true, sensitivity: 'base' })` (`2 < 10 < 43 < 65 < 105`, alfanuméricos después) → desempate por `subject` con el mismo collator. cpanel-admin replica el criterio (`compareGroups`, modos `id_asc` default / `id_desc` / `name`; el grupo interno y los sin ID quedan siempre al final) y despacha la difusión en el orden elegido. Selección: checkbox por fila, checkbox maestro, `Seleccionar todos los clientes`, `Invertir selección`, `Limpiar selección` (sobre los clientes visibles, nunca el grupo interno) y vista Todos / Solo seleccionados (N) / No seleccionados.
  - **Ampliación RF-10 (plantillas de emergencia):** `WhatsAppEmergencyService.listTemplates/createTemplate/updateTemplate/deleteTemplate` sobre `wa:emergency_templates` (JSON `EmergencyTemplate[]`). Si la clave es `null` se siembran las 5 `DEFAULT_EMERGENCY_TEMPLATES` (ids estables `base-caida-general`, `base-senal-origen`, `base-deportes-eventos`, `base-degradacion`, `base-mantenimiento`; terminología neutra: Servidor de Transmisión / señal de origen / Encoder). Los 4 campos se validan con `trim()` (400); `updateTemplate`/`deleteTemplate` con id inexistente → 404; editar conserva la posición; las nuevas se agregan al final. Borrar todas guarda `'[]'` para no re-sembrar; un JSON ilegible se informa como lista vacía y **no** se sobrescribe. Endpoints `GET/POST /v3/whatsapp/emergency/templates`, `PUT/DELETE /v3/whatsapp/emergency/templates/:id` (solo pasan `title`, `incident_text`, `reply_text`, `resolution_text`). cpanel-admin: selector rápido por título (vuelve al placeholder tras aplicar) + modal `Gestionar Plantillas` (Usar plantilla, nueva, nueva desde el formulario actual, editar, eliminar con confirmación); el campo de resolución pasó a estar siempre visible junto a incidencia y respuesta.
  - **Pruebas**: `test/whatsapp/broadcast.test.ts` (intervalo, dedup, 400/409, fallo parcial, aborto de 20 tras 3 sin reanudar, aborto en vuelo, persistencia/reinicio), `emergency.test.ts` (config, 1 respuesta por grupo, lock concurrente, nueva incidencia, espaciado, abort, 30 grupos → resolución solo a 4, L1 400/409/0 afectados), `evaluator.test.ts`, más casos en `gateway`, `webhook`, `worker` (modo mudo: ticket silencioso, 0 llamadas salientes), `manualCommand` (`Agente: Ayuda` mudo) y `consoleApi` (7 endpoints + guards en 16 rutas). `npm test`: 220 OK + 2 skip tras las ampliaciones RF-04/RF-10 (antes 204); `npm run test:sql` OK. cpanel-admin: `tsc --noEmit -p tsconfig.app.json` y `ngc -p tsconfig.app.json --noEmit` sin errores (sin build).

---

## 9. Resumen Diario Automático y Notificación al Cliente al Pasar a Verificar (`REQ-2026-007`, 2026-11-13)
- **Spec:** `Planning-X/Cross_System/02_Aprobados/REQ-2026-007_RESUMEN_DIARIO_Y_NOTIFICACION_VERIFICAR_KANBAN.md`.
- **Resumen Diario Automático (Lun-Sáb 08:30 AM Hora de Paraguay — `America/Asuncion`):**
  - `DailySummaryScheduler` envía el resumen de pendientes (`fn_whatsapp_kanban_summary` + `formatKanbanSummary`) al grupo interno de soporte (`WHATSAPP_SUPPORT_ALERT_GROUP_ID`).
  - Si hay 0 pendientes, envía igualmente `✅ No hay tickets pendientes en el Kanban.` (`R1`). Deduplica con `SET wa:daily_summary_sent:{YYYY-MM-DD} 1 EX 172800 NX`.
- **Notificación al Grupo del Cliente al Pasar Ticket a `Verificar` en Kanban (`R2`, `R3a`, `R3b`):**
  - El modal de Kanban al pasar a `verificar` incluye el checkbox `[x] Enviar resumen al grupo de WhatsApp del cliente` (marcado por defecto).
  - Resuelve el grupo del cliente usando `created_by.group_id` (siempre que **no** sea `WHATSAPP_SUPPORT_ALERT_GROUP_ID`) o buscando en `gateway.listGroups()` el grupo cuyo `tenant_id` coincida con el cliente del ticket (`tenant_server` / `tenant_tag` / `tenant_id`). **Jamás** envía al grupo interno de soporte (`R2`).
  - Formato (`R3a`): `🔧 *Actualización de Soporte*\n<resumen>`.
