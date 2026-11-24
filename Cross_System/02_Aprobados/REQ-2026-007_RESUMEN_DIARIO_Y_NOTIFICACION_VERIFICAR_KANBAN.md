# Documento de Especificación Técnica: Resumen Diario Automático (Lun-Sáb 08:30 PY) y Notificación al Grupo del Cliente al Pasar Ticket a Verificar

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-007` (Extensión de `REQ-2026-003`..`006`) |
| **Sistema Destino:** | `Cross_System` (`xBackend` + `cpanel-admin`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-11-13 |
| **Última Actualización:** | 2026-11-13 |
| **Autor(es):** | Lead Architect (Planning-X) |
| **Motor de Ejecución:** | **Claude Code** |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

Incorpora dos automatizaciones operativas al **Asistente WhatsApp** e integración con el tablero **Kanban**:

1. **Resumen Diario Automático de Pendientes (Lunes a Sábado, 08:30 AM — Hora de Paraguay):**
   - Todos los días de **lunes a sábado a las 08:30 AM (`America/Asuncion`)**, el agente consulta los tickets pendientes del Kanban (`fn_whatsapp_kanban_summary`) y envía automáticamente el resumen al grupo interno de soporte (`WHATSAPP_SUPPORT_ALERT_GROUP_ID`).
   - Si no hay tickets pendientes (`total === 0`), envía igualmente **`✅ No hay tickets pendientes en el Kanban.`** (`R1`).
2. **Envío Automático al Grupo del Cliente al Pasar un Ticket a `Verificar` en Kanban:**
   - Cuando un operador mueve un ticket a la columna **`Verificar`** en el Kanban (`cpanel-admin`), el modal que solicita el *"Resumen de lo realizado para verificación"* incluye un checkbox marcado por defecto: **`[x] Enviar resumen al grupo de WhatsApp del cliente`** (`R3b`).
   - Si el checkbox está activo, al guardar el cambio de estado (`PUT /v3/incidents/:id/status`), `xBackend` resuelve el grupo de WhatsApp del cliente correspondiente a ese ticket (**nunca el grupo interno de soporte**, `R2`) y le envía el mensaje con el formato `R3a`:
     ```text
     🔧 *Actualización de Soporte*
     <Resumen de lo realizado escrito por el operador>
     ```

---

## 2. Requerimientos Funcionales (Sintaxis EARS)

### 2.1. Resumen Diario Automático (Lunes a Sábado 08:30 AM — `America/Asuncion`)
- **RF-01 (Scheduler Diario en Hora de Paraguay):** *Mientras* el agente de WhatsApp esté activo en `xBackend`, un scheduler interno (`DailySummaryScheduler`) *deberá* evaluar el reloj en la zona horaria **`America/Asuncion`** (configurable vía `WHATSAPP_DAILY_SUMMARY_TZ`, default `'America/Asuncion'`, y hora `WHATSAPP_DAILY_SUMMARY_TIME`, default `'08:30'`).
- **RF-02 (Disparo Lunes a Sábado e Idempotencia Diaria):** *Cuando* la hora local en `America/Asuncion` alcance las **08:30** en un día de **lunes a sábado** (excluido el domingo), el sistema *deberá*:
  1. Adquirir atómicamente en Redis `SET wa:daily_summary_sent:{YYYY-MM-DD} 1 EX 172800 NX` (donde `YYYY-MM-DD` es la fecha local en `America/Asuncion`), garantizando que se envíe **exactamente una vez por día** incluso si `xBackend` se reinicia después de las 08:30.
  2. Invocar `fn_whatsapp_kanban_summary('{}', '{}', '{}')` y formatear el mensaje con `formatKanbanSummary(items)`.
  3. *Si* hay `0` tickets pendientes, enviar **`✅ No hay tickets pendientes en el Kanban.`** (`R1`).
  4. Enviar el mensaje exclusivamente a `WHATSAPP_SUPPORT_ALERT_GROUP_ID` mediante `gateway.sendSupportAlert` (respetando el modo mudo `AgentMutedError` si el agente está apagado).

### 2.2. Notificación al Grupo del Cliente al Pasar Ticket a `Verificar`
- **RF-03 (Checkbox de Confirmación en Modal de Kanban — `R3b`):** *Cuando* el operador mueva un ticket a la columna **`verificar`** en `cpanel-admin` (`kanban-board.component.ts`), el modal de *"Paso a Verificación"* *deberá* mostrar:
  - El `textarea` obligatorio con el resumen de lo realizado (`resolution_note`).
  - Un checkbox **marcado por defecto (`true`)**: **"Enviar resumen al grupo de WhatsApp del cliente"** (`notify_whatsapp`).
  - `KanbanService.updateStatus(id, status, resolution_note, notify_whatsapp)` *deberá* enviar `notify_whatsapp: boolean` en el body de `PUT /v3/incidents/:id/status`.
- **RF-04 (Resolución Segura del Grupo del Cliente — `R2`):** *Cuando* `PUT /v3/incidents/:id/status` actualice un ticket a `status === 'verificar'` con `notify_whatsapp !== false` y `resolution_note` no vacío, `xBackend` *deberá* resolver el `group_id` de WhatsApp del cliente siguiendo este orden estricto:
  1. **Prioridad 1 (`created_by.group_id`):** Si el ticket posee `created_by->>'group_id'` terminado en `@g.us` **Y** ese JID **NO es** `WHATSAPP_SUPPORT_ALERT_GROUP_ID` (ej. tickets creados automáticamente desde el grupo del cliente), usar ese `group_id`. *(Nota: los tickets creados manualmente con `Agente: Generar ID: ...` tienen `created_by.group_id === WHATSAPP_SUPPORT_ALERT_GROUP_ID`, por lo que pasan automáticamente a Prioridad 2).*
  2. **Prioridad 2 (Búsqueda por Tenant `[ID]` en Grupos de WhatsApp):** Resolver el identificador numérico/tag del cliente a partir del ticket (`tenant_server` / `position` devuelto por `incidentupdatestatus`, o `created_by->>'tenant_tag'`, o `tenant_id` quitando el prefijo `'server'`, ej. `'server43'` $\to$ `'43'`). Consultar `gateway.listGroups()` y buscar el grupo de clientes (`!g.is_support_group`) cuyo `g.tenant_id` coincida (case-insensitive) con el tag del cliente.
  3. **Regla Estricta `R2` (Prohibido Grupo Interno):** En ningún caso se enviará el mensaje de verificación al grupo interno de soporte (`WHATSAPP_SUPPORT_ALERT_GROUP_ID`). Si el ticket no tiene tenant o no existe un grupo de WhatsApp para ese cliente, no se envía mensaje por WhatsApp y el cambio de estado en Kanban se completa normalmente informando `whatsapp_notification: { sent: false, reason: 'no_client_group' }`.
- **RF-05 (Formato del Mensaje de Verificación — `R3a`):** El texto enviado mediante `gateway.sendGroupMessage(clientGroupId, text)` *deberá* tener exactamente el formato:
  ```text
  🔧 *Actualización de Soporte*
  <resolution_note.trim()>
  ```
- **RF-06 (Tolerancia a Fallos y Feedback en UI):**
  - El cambio de estado del ticket en PostgreSQL (`incidentupdatestatus`) **nunca** debe fallar ni hacer rollback si WhatsApp / Evolution API está caído, el agente está en modo mudo o el cliente no tiene grupo de WhatsApp.
  - `PUT /v3/incidents/:id/status` devolverá el objeto del ticket actualizado (compatible hacia atrás) incluyendo el campo adicional `whatsapp_notification: { sent: boolean, group_id?: string, group_name?: string, reason?: string }`, para que `cpanel-admin` muestre un toast informativo (ej. *"Enviado a verificar y notificado por WhatsApp al grupo del cliente"* o advertencia si no se pudo enviar por WhatsApp).

### 2.3. Gestión de Acceso por Usuario al Menú `whatsapp-support` en `cpanel-admin`
- **RF-07 (Permiso de Menú `whatsapp-support` por Usuario):**
  - En `cpanel-admin/src/app/core/default-layout/_nav.ts`, el ítem `{ id: 'whatsapp-support', name: 'Asistente WhatsApp', url: '/whatsapp-support' }` **no** debe tener `role: 'admin'` fijo (igual que `kanban` y `monitoring`), de modo que los usuarios `admin` lo vean siempre y los demás usuarios (`operator`, etc.) lo vean cuando tengan `'whatsapp-support'` en `options.menu`.
  - En `cpanel-admin/src/app/components/users/users.component.ts`:
    1. Incluir `'whatsapp-support'` en el catálogo por defecto `availableMenus` (`['dashboard', 'tenants', 'monitoring', 'kanban', 'whatsapp-support', 'livechannels', 'fastchannels', 'videos', 'movies', 'tvshows', 'upcomingmovies', 'syncagents', 'uploads', 'users']`).
    2. En `loadMetadata`, hacer **merge** (`Array.from(new Set([...this.availableMenus, ...metaMenus]))`) en lugar de reemplazar `this.availableMenus`, para que `whatsapp-support` (y `kanban`, `tenants`, `syncagents`, `uploads`, `users`) siempre aparezcan como checkboxes en **Usuarios -> Editar Usuario -> Configurar Permisos de Menú** y como badge en la columna **"Menús Asignados"** de la tabla de usuarios.
  - En PostgreSQL (`public.tenant_menus`), registrar `menuid = 'whatsapp-support'` (`name = 'Asistente WhatsApp'`, `tenantrole = 'administrator'`, `status = 'enabled'`).

---

## 3. Contratos y Archivos Impactados

| Sistema | Archivo | Cambio |
| :--- | :--- | :--- |
| **`xBackend`** | `services/whatsapp/dailySummaryScheduler.ts` *(nuevo)* | Scheduler (tick cada 30s) que verifica hora `08:30` Lun-Sáb en `America/Asuncion`, deduplica con `wa:daily_summary_sent:{YYYY-MM-DD}` y envía el resumen al grupo interno. |
| **`xBackend`** | `services/whatsapp/kanbanVerificationNotifier.ts` *(nuevo)* | Función/servicio que formatea el mensaje `R3a`, resuelve el grupo del cliente (excluyendo siempre `WHATSAPP_SUPPORT_ALERT_GROUP_ID`) y envía con `gateway.sendGroupMessage`. |
| **`xBackend`** | `services/whatsapp/whatsappAgent.ts` | Arranca `DailySummaryScheduler` y expone `notifyTicketVerification(ticket, resolutionNote)` para `IncidentsController`. |
| **`xBackend`** | `controllers/incidents.ts` | En `updateStatus`, ejecuta `pgsql.exec('incidentupdatestatus', ...)` y, si `status === 'verificar'` y `notify_whatsapp !== false`, invoca `notifyTicketVerification` adjuntando `whatsapp_notification` a la respuesta. |
| **`cpanel-admin`** | `modules/common/kanban-board/kanban-board.component.ts` + `kanban.service.ts` | En `promptForStatusNote('verificar')`, agrega el checkbox *"Enviar resumen al grupo de WhatsApp del cliente"* (marcado por defecto) y pasa `notify_whatsapp` a `updateStatus`, mostrando toast con el resultado. |

---

## 4. Plan de Tareas Secuenciales para Claude Code (<30 min c/u)

- [ ] **T1: `DailySummaryScheduler` (Lunes a Sábado 08:30 `America/Asuncion`) + Tests**
  - **RFs:** `RF-01, RF-02`
  - **Archivos:** `xBackend/services/whatsapp/dailySummaryScheduler.ts`, `xBackend/test/whatsapp/dailySummary.test.ts`.
  - **Hecho cuando:** Tests unitarios verifican que dispara de lunes a sábado a las 08:30 (`America/Asuncion`), **no** dispara los domingos ni antes de las 08:30, envía `✅ No hay tickets pendientes en el Kanban.` cuando `items` es vacío, no duplica el envío el mismo día (`wa:daily_summary_sent:{YYYY-MM-DD}`) y captura `AgentMutedError` en silencio si el agente está apagado.

- [ ] **T2: `KanbanVerificationNotifier` + Integración en `IncidentsController.updateStatus` + Tests**
  - **RFs:** `RF-04, RF-05, RF-06`
  - **Archivos:** `xBackend/services/whatsapp/kanbanVerificationNotifier.ts`, `xBackend/services/whatsapp/whatsappAgent.ts`, `xBackend/controllers/incidents.ts`, `xBackend/test/whatsapp/kanbanVerification.test.ts`.
  - **Hecho cuando:** Tests unitarios verifican:
    1. Formato exacto `R3a` (`🔧 *Actualización de Soporte*\n<resumen>`).
    2. Si `created_by.group_id` es el grupo de un cliente, envía directo ahí; si `created_by.group_id` es `WHATSAPP_SUPPORT_ALERT_GROUP_ID` (ej. ticket creado con `Agente: Generar 43: ...`) o no tiene `group_id`, busca el grupo del cliente por `tenant_server` / `tenant_tag` / `tenant_id` (`[43]`).
    3. **Nunca** envía al grupo interno de soporte (`WHATSAPP_SUPPORT_ALERT_GROUP_ID`).
    4. Si `notify_whatsapp === false`, no envía mensaje.

- [ ] **T3: Checkbox en Modal de Verificación en `cpanel-admin` + Memoria**
  - **RFs:** `RF-03, RF-06`
  - **Archivos:** `cpanel-admin/src/app/modules/common/kanban-board/kanban-board.component.ts`, `kanban.service.ts`, `xBackend/MEMORY.md`, `cpanel-admin/MEMORY.md`.
  - **Hecho cuando:** `tsc --noEmit` y `ngc --noEmit` pasan en `cpanel-admin` (sin `ng build` ni `npm run build`), y `npx tsc --noEmit`, `npm test` y `npm run build` pasan en `xBackend`.

---

## 5. Escenario de Pruebas (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Casos Borde                                  │
│ • Domingo 08:30 AM (Hora Paraguay): NO envía resumen.       │
│ • Lunes 08:30 AM con 0 tickets pendientes: envía            │
│   "✅ No hay tickets pendientes en el Kanban." al soporte.  │
│ • Ticket creado con "Agente: Generar 43: ..." tiene         │
│   created_by.group_id = grupo interno: el notificador       │
│   ignora ese JID, resuelve el grupo del cliente [43] y      │
│   NUNCA envía el mensaje de verificación al grupo interno.  │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Contratos e Integración                      │
│ • npx tsc --noEmit, npm test y npm run build en xBackend.   │
│ • tsc --noEmit y ngc --noEmit en cpanel-admin.              │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia e Idempotencia                   │
│ • Si xBackend se reinicia a las 08:35 AM tras haber enviado │
│   el resumen de las 08:30 AM, la clave Redis                │
│   wa:daily_summary_sent:YYYY-MM-DD impide el doble envío.   │
│ • Si Evolution API falla o el agente está en Modo Mudo al   │
│   pasar un ticket a Verificar, el estado en Kanban se       │
│   guarda igual en PostgreSQL y reporta whatsapp_notification│
│   { sent: false, reason } sin romper la operación.          │
└─────────────────────────────────────────────────────────────┘
```
