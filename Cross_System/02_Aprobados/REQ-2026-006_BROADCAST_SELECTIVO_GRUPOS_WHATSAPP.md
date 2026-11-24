# Documento de Especificación Técnica: Difusión Selectiva (Broadcast), Modo Emergencia con Resolución y Botón Maestro del Agente

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-006` (Extensión de `REQ-2026-003`..`005`) |
| **Sistema Destino:** | `Cross_System` (`xBackend` + `cpanel-admin`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-11-12 |
| **Última Actualización:** | 2026-11-12 |
| **Autor(es):** | Lead Architect (Planning-X) |
| **Motor de Ejecución:** | **Claude Code** |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

Amplía el módulo **Asistente WhatsApp** en `xBackend` y `cpanel-admin` con tres mecanismos de control y comunicación hacia los grupos de clientes:

1. **Botón Maestro de Encendido/Apagado del Agente (Outbound Kill-Switch):**
   -Visible en la cabecera principal del módulo en `cpanel-admin` (accesible desde cualquier pestaña).
   - **Modo Mudo (`R1: B`):** Al apagar el agente (`outbound_enabled = false`), se bloquea el **100% de los envíos salientes de WhatsApp** (alertas al grupo interno, respuestas a comandos `Agente:`, respuestas automáticas de emergencia, mensajes de resolución y broadcasts), mientras que la escucha pasiva en segundo plano, la analítica de casos y la creación de tickets en Kanban continúan operando en silencio.
   - **Aborto Inmediato y Purga de Cola (`R2`):** Al apagar el agente, cualquier cola de envío activa (broadcast o cierre de emergencia) se **aborta de inmediato y se purga por completo**. Cuando el agente se vuelve a encender, arranca con la cola limpia (no reanuda envíos viejos; si el operador desea reenviar una difusión, inicia una nueva).
2. **Difusión Selectiva (Broadcast Manual):**
   - Permite seleccionar múltiples grupos desde el panel y enviarles un comunicado masivo con un **intervalo estricto de 2 segundos (2000 ms)** entre cada envío para no saturar WhatsApp.
3. **Respuesta Automática de Emergencia por Incidencia + Mensaje de Cierre/Resolución (`R3`):**
   - Permanece **Apagado por defecto**.
   - Al encenderse con una **"Incidencia"** y un **"Mensaje de Respuesta"**, Gemini analiza los mensajes entrantes de clientes; si están relacionados con la incidencia, responde automáticamente en ese grupo (máximo **1 vez por grupo por incidencia**, espaciado por 2 segundos) y registra ese `group_id` en el conjunto de grupos afectados de la incidencia (`wa:emergency_replied_groups:{incident_id}`), marcando además la sesión `handled_by_staff = true` para no generar tickets repetidos en el Kanban.
   - **Cierre / Resolución al Apagar (`R3`):** Al apagar una emergencia activa desde el panel, el operador dispone de un campo **"Mensaje de resolución / cierre"** y puede elegir entre dos acciones explícitas:
     1. **"Apagar y enviar resolución a afectados"** (envía el mensaje de cierre con intervalo de 2 segundos **exclusivamente** a los grupos registrados en `wa:emergency_replied_groups:{incident_id}` que preguntaron y recibieron la respuesta automática de esa emergencia).
     2. **"Apagar sin enviar mensaje"** (desactiva la emergencia en silencio).

---

## 2. Requerimientos Funcionales (Sintaxis EARS)

### 2.1. Botón Maestro de Encendido/Apagado del Agente (Outbound Kill-Switch)
- **RF-01 (Estado Maestro del Agente):** El sistema *deberá* persistir en Redis (`wa:agent_outbound_enabled`, por defecto `'1'` / `true`) el estado de salida del agente y exponerlo mediante `GET /v3/whatsapp/agent-toggle` (además de incluir `agent_outbound_enabled: boolean` en `GET /v3/whatsapp/status`) y `PUT /v3/whatsapp/agent-toggle` (`{ enabled: boolean }`).
- **RF-02 (Bloqueo Total de Salida en Nivel Gateway — `R1: B`):** *Mientras* `agent_outbound_enabled === false`, `WhatsAppGatewayService` *deberá* bloquear tanto `sendSupportAlert` como `sendGroupMessage` lanzando `AgentMutedError` antes de realizar cualquier petición HTTP a Evolution API, manteniendo activas la ingesta del webhook, el cierre de casos con Gemini y la creación silenciosa de tickets en Kanban.
- **RF-03 (Aborto Inmediato y Purga de Colas — `R2`):** *Cuando* el administrador cambie `agent_outbound_enabled` a `false` (`PUT /v3/whatsapp/agent-toggle`), `xBackend` *deberá*:
  1. Abortar inmediatamente cualquier bucle de envío activo en `WhatsAppBroadcastService` (sea un broadcast manual o un envío de resolución de emergencia), cancelando el timer de espera en curso.
  2. **Purgar todos los ítems pendientes** de la cola (marcándolos como `aborted` en el resumen histórico y vaciando la cola de trabajo) y cancelar cualquier envío pendiente en el espaciador de `WhatsAppEmergencyService`.
  3. *Cuando* el administrador vuelva a cambiar `agent_outbound_enabled` a `true`, el sistema *deberá* iniciar con las colas de envío completamente limpias (sin reanudar ni reenviar mensajes abortados previamente).

### 2.2. Difusión Selectiva (Broadcast)
- **RF-04 (Listado, Ordenamiento por ID y Selección Flexible de Grupos):** *Cuando* el administrador abra la pestaña de Difusión y Emergencia en `cpanel-admin`, el sistema *deberá* consultar `GET /v3/whatsapp/groups`, obteniendo de Evolution API (`/group/fetchAllGroups/{instance}?getParticipants=false`) todos los grupos con `group_id`, `subject`, `tenant_id` (regex `\[([0-9a-zA-Z_-]+)\]`) e indicador `is_support_group` (`group_id === WHATSAPP_SUPPORT_ALERT_GROUP_ID`).
  - **Ordenamiento por ID de Cliente (`tenant_id` numérico ascendente):** Tanto en `xBackend` (`listGroups`) como en `cpanel-admin`, los grupos *deberán* ordenarse por **`tenant_id` numérico ascendente** (ej. `[1]`, `[2]`, ..., `[43]`, `[65]`, `[105]`; si un `tenant_id` es alfanumérico se ordena con `numeric: true` después de los puramente numéricos, luego los grupos sin `tenant_id` ordenados por `subject`, y finalmente el grupo interno de soporte `is_support_group = true` al final). Además, la UI deberá permitir alternar el orden (`ID ↑`, `ID ↓`, `Nombre A-Z`) teniendo **`ID ↑` como predeterminado**.
  - **Cambio y Control de Selección:** La UI *deberá* permitir cambiar libremente la selección en cualquier momento mediante:
    1. Clic individual en cada fila/checkbox (activar/desactivar cualquier cliente después de haber seleccionado todos o un subconjunto).
    2. Checkbox maestro / botones de acción rápida sobre la vista actual (filtrada o completa): **"Seleccionar todos los clientes"**, **"Invertir selección"** (cambia seleccionados por no seleccionados entre los grupos de clientes visibles) y **"Limpiar selección"**, más un filtro rápido **"Ver: Todos / Solo seleccionados ($N$) / No seleccionados"**.
- **RF-05 (Selección, Validación y Despacho Secuencial cada 2 Segundos):** *Cuando* el administrador inicie un broadcast (`POST /v3/whatsapp/broadcast`), `xBackend` *deberá* verificar que el agente esté encendido (`agent_outbound_enabled === true`, o retornar `409 Conflict`), que `text` no esté vacío, que `groups` contenga al menos un JID `@g.us` (deduplicados) y que no exista otro envío masivo en curso (`409 Conflict`). El envío *deberá* esperar exactamente **2000 ms** entre el fin de un mensaje y el inicio del siguiente, reportando el avance en `GET /v3/whatsapp/broadcast/status` (`idle | running | completed | aborted`, `kind: 'manual' | 'emergency_resolution'`, `total`, `sent`, `failed`, `aborted`, `pending`, `items`).

### 2.3. Respuesta Automática de Emergencia y Cierre Selectivo a Afectados
- **RF-06 (Configuración del Modo Emergencia):** El sistema *deberá* exponer `GET /v3/whatsapp/emergency` y `PUT /v3/whatsapp/emergency` para gestionar en Redis (`wa:emergency_config`):
  - `enabled`: `boolean` (**`false` por defecto**).
  - `incident_text`: descripción de la falla común para evaluación de Gemini (obligatorio al encender).
  - `reply_text`: respuesta automática al detectar la incidencia (obligatorio al encender).
  - `resolution_text`: último texto de cierre/resolución configurado (opcional).
  - `incident_id`: UUID regenerado cada vez que la emergencia pasa de `false -> true` o cambia `incident_text`.
  - `replied_groups`: lista de `{ group_id, subject?, replied_at }` correspondientes al `incident_id` actual.
- **RF-07 (Detección con Gemini y Respuesta Automática):** *Mientras* `emergency.enabled === true` y `agent_outbound_enabled === true`, *cuando* ingrese un mensaje de un cliente (`is_staff === false`) en un grupo de clientes (`group_id !== WHATSAPP_SUPPORT_ALERT_GROUP_ID`) que aún no esté en `wa:emergency_replied_groups:{incident_id}`:
  1. El sistema *deberá* consultar a Gemini (`evaluateEmergencyMatch(incident_text, session.messages)`).
  2. *Si* `matches === true`, el sistema *deberá*:
     - Adquirir atómicamente `SET wa:emergency_lock:{incident_id}:{group_id} 1 EX 86400 NX` y agregar el `group_id` al Set `wa:emergency_replied_groups:{incident_id}` (garantizando **1 sola respuesta por grupo por incidencia**).
     - Enviar `reply_text` al `group_id` respetando un espaciado de **2000 ms** entre envíos.
     - Marcar la sesión activa del grupo con `handled_by_staff = true` (`staff_phone = 'emergency_auto'`), evitando generar un ticket duplicado a los 5 minutos.
- **RF-08 (Apagado de Emergencia con Opción de Envío de Resolución sólo a Afectados — `R3`):** *Cuando* el administrador apague una emergencia activa (`PUT /v3/whatsapp/emergency` con `enabled: false`), el payload *deberá* aceptar `{ enabled: false, send_resolution?: boolean, resolution_text?: string }`:
  - **Opción A (`send_resolution: false` o ausente — "Apagar sin enviar mensaje"):** Desactiva inmediatamente `emergency.enabled = false` sin encolar ningún mensaje.
  - **Opción B (`send_resolution: true` — "Apagar y enviar resolución a afectados"):**
    1. Valida que `agent_outbound_enabled === true` (si está en modo mudo retorna `409 Conflict`) y que `resolution_text` no esté vacío (`400 Bad Request`).
    2. Desactiva inmediatamente `emergency.enabled = false` (para no seguir enviando el mensaje de falla entrante).
    3. Obtiene únicamente los grupos almacenados en `wa:emergency_replied_groups:{incident_id}` (los que consultaron por la emergencia y recibieron la respuesta automática).
    4. Si la lista tiene $\ge 1$ grupos, despacha `resolution_text` **exclusivamente a esos grupos** a través de `WhatsAppBroadcastService` (`kind = 'emergency_resolution'`) con el intervalo de **2000 ms** entre envíos. Si ningún grupo había preguntado (`0` afectados), finaliza sin enviar mensajes.
- **RF-09 (Guardrail de Salida Seguro):**
  - `sendSupportAlert` permanece restringido exclusivamente a `WHATSAPP_SUPPORT_ALERT_GROUP_ID`.
  - `sendGroupMessage(groupJid, text)` exige que `agent_outbound_enabled === true` y que `groupJid` termine en `@g.us` (rechazando con `OutboundBlockedError` cualquier JID individual `@s.whatsapp.net`).
- **RF-10 (Plantillas Predeterminadas de Emergencia y Modal de Gestión):**
  - El sistema *deberá* persistir en Redis (`wa:emergency_templates`) una lista de plantillas de emergencia con la estructura `{ id: string, title: string, incident_text: string, reply_text: string, resolution_text: string, updated_at: string }`, inicializando automáticamente un conjunto base de plantillas predeterminadas si la clave aún no existe (ej. *Caída general de señales*, *Corte de proveedor / señal de origen*, *Inconveniente en señales deportivas / eventos*, *Degradación o pixelación de señales*, *Mantenimiento de emergencia*).
  - `xBackend` *deberá* exponer `GET /v3/whatsapp/emergency/templates`, `POST /v3/whatsapp/emergency/templates`, `PUT /v3/whatsapp/emergency/templates/:id` y `DELETE /v3/whatsapp/emergency/templates/:id` (validando `title`, `incident_text`, `reply_text` y `resolution_text` no vacíos).
  - En la tarjeta **Modo Emergencia por Incidencia** de `cpanel-admin`:
    1. Se *deberá* mostrar una barra superior con un **Selector Rápido por Título** (`<select>` *"Seleccionar plantilla predeterminada…"*) y el botón **"Gestionar Plantillas"** (que abre un **Modal CRUD** para crear, editar, eliminar o seleccionar una plantilla de la lista).
    2. Los tres campos (`Incidencia`, `Mensaje de respuesta automática` y `Mensaje de resolución / cierre`) *deberán* estar siempre visibles en el formulario para que, al elegir una plantilla (desde el selector rápido por título o desde el modal con el botón *"Usar plantilla"*), se copien/peguen automáticamente los 3 textos en pantalla permitiendo al operador ajustarlos si lo desea antes de encender la emergencia o enviar el cierre.

---

## 3. Contratos de Datos y Endpoints en `xBackend`

Todos los endpoints en `server/whatsappConsoleRoutes.ts` llevan `[validOrigin('console'), siteApikey()]`:

| Método | Ruta | Payload / Respuesta Clave |
| :--- | :--- | :--- |
| `GET` | `/v3/whatsapp/agent-toggle` | `{ enabled: boolean, updated_at: string \| null }` |
| `PUT` | `/v3/whatsapp/agent-toggle` | Body: `{ enabled: boolean }` $\to$ Si `false`, aborta y purga colas (`{ enabled, aborted_pending_count }`). |
| `GET` | `/v3/whatsapp/groups` | `{ groups: Array<{ group_id, subject, tenant_id, is_support_group }> }` |
| `POST` | `/v3/whatsapp/broadcast` | Body: `{ groups: string[], text: string }` $\to$ `202 Accepted` con estado inicial o `409 Conflict`. |
| `GET` | `/v3/whatsapp/broadcast/status` | `{ state: 'idle'\|'running'\|'completed'\|'aborted', kind: 'manual'\|'emergency_resolution', total, sent, failed, aborted, pending, items }` |
| `GET` | `/v3/whatsapp/emergency` | `{ enabled, incident_id, incident_text, reply_text, resolution_text, replied_groups: string[], replied_groups_count: number, updated_at }` |
| `PUT` | `/v3/whatsapp/emergency` | Body al encender/editar: `{ enabled: true, incident_text, reply_text, resolution_text? }`.<br>Body al apagar: `{ enabled: false, send_resolution?: boolean, resolution_text?: string }` $\to$ retorna `{ emergency, resolution_broadcast? }`. |
| `GET` | `/v3/whatsapp/emergency/templates` | `{ items: Array<{ id, title, incident_text, reply_text, resolution_text, updated_at }> }` (si vacío por primera vez, siembra plantillas base). |
| `POST` | `/v3/whatsapp/emergency/templates` | Body: `{ title, incident_text, reply_text, resolution_text }` $\to$ crea plantilla (`201 Created`). |
| `PUT` | `/v3/whatsapp/emergency/templates/:id` | Body: `{ title, incident_text, reply_text, resolution_text }` $\to$ actualiza plantilla (`200 OK` o `404`). |
| `DELETE` | `/v3/whatsapp/emergency/templates/:id` | Elimina plantilla por `id` (`200 OK` o `404`). |

---

## 4. Especificación UI en `cpanel-admin` (`modules/common/whatsapp-support/`)

### 4.1. Cabecera Global del Módulo (Visible en todas las pestañas)
- Junto al título **"Asistente WhatsApp"**, incluir el **Botón Maestro del Agente**:
  - Cuando `agent_outbound_enabled === true`: Badge verde **`AGENTE ACTIVO`** + botón **`Apagar Agente (Modo Mudo)`**.
  - Cuando `agent_outbound_enabled === false`: Banner de alerta amarillo/rojo visible en la parte superior (**"Agente en Modo Mudo: Todos los envíos de WhatsApp están bloqueados y las colas fueron purgadas"**) + botón verde **`Encender Agente`**.

### 4.2. Orden de las 5 Pestañas
1. **Analítica de Casos** *(default)*
2. **Tickets Generados**
3. **Difusión y Emergencia** *(nueva pestaña)*
4. **Equipo de Soporte**
5. **Conexión & QR** *(última pestaña)*

### 4.3. Diseño de la Pestaña "Difusión y Emergencia"
1. **Tarjeta Superior — Modo Emergencia por Incidencia:**
   - Badge de estado: **`APAGADO`** vs **`ENCENDIDO — RESPONDIENDO AUTOMÁTICAMENTE`** + contador: *"Grupos afectados notificados en esta incidencia: X"*.
   - **Barra de Selección Rápida de Plantilla y Gestión (`RF-10`):**
     - Selector desplegable (`<select>`) con el **Título** de cada plantilla predeterminada (*"Seleccionar problema base / plantilla…"*). Al elegir una plantilla, se copian/pegan automáticamente sus 3 campos en el formulario (`incident_text`, `reply_text` y `resolution_text`), habilitando de inmediato el botón de encendido.
     - Botón **"Gestionar Plantillas"** que abre el **Modal de Plantillas de Emergencia** (listado con botón **"Usar"** para cargar en el formulario, más creación, edición y borrado de plantillas con `Título`, `Incidencia`, `Mensaje de respuesta` y `Mensaje de término/resolución`).
   - Los 3 campos siempre visibles en la tarjeta para que al seleccionar una plantilla se vean rellenos y puedan retocarse:
     1. Campo **"Incidencia (para detección con IA)"**: textarea donde se describe la falla común.
     2. Campo **"Mensaje de respuesta automática"**: textarea con el mensaje que se envía cuando un cliente consulta por la incidencia.
     3. Campo **"Mensaje de resolución / cierre"**: textarea con el mensaje que se enviará al finalizar la incidencia.
   - Botón **"Encender Modo Emergencia"** (cuando está apagado) / **"Actualizar textos"** (cuando está encendido).
   - **Acciones de Cierre / Apagado (habilitadas cuando la emergencia está ENCENDIDA):**
     - Dos botones de acción claros (`R3`):
       - **Botón Primario:** **`Apagar y enviar resolución (X grupos)`** (envía el mensaje de cierre con intervalo de 2s únicamente a los $X$ grupos que preguntaron por la emergencia; deshabilitado si $X = 0$ o si el texto de resolución está vacío).
       - **Botón Secundario:** **`Apagar sin enviar mensaje`** (apaga inmediatamente sin enviar nada).
2. **Tarjeta Inferior — Difusión Selectiva (Broadcast Manual):**
   - **Ordenamiento por ID de Cliente (`tenant_id`):** Los grupos se muestran con su badge `[ID]` a la izquierda del nombre (ej. **`[43]`** `Soporte Frentones [43]`), ordenados por defecto por **`ID ↑` numérico** (`1, 2, ..., 43, 65, 105`, dejando sin ID y Soporte Interno al final), con selector para cambiar orden (`ID ↑` | `ID ↓` | `Nombre A-Z`) y conservando ese mismo orden al despachar el broadcast.
   - **Controles para Cambiar la Selección:**
     - Buscador por ID o nombre + filtro rápido de vista (`Todos` / `Seleccionados` / `No seleccionados`).
     - Botones: *"Seleccionar todos los clientes"*, *"Invertir selección"* (desmarca los marcados y marca los desmarcados sobre los clientes visibles) y *"Limpiar selección"*, además del checkbox individual en cada fila para agregar o quitar clientes puntuales de la selección.
   - Textarea del mensaje masivo + botón **`Enviar Difusión (2s por grupo)`**.
   - Barra de progreso en vivo (compartida para ver tanto el avance de una difusión manual como el envío de una resolución de emergencia o su estado `Abortado`).

---

## 5. Plan de Tareas Secuenciales para Claude Code (<30 min c/u)

- [ ] **T1: Outbound Kill-Switch en `WhatsAppGatewayService`, `listGroups`, `sendGroupMessage` y `evaluateEmergencyMatch` + Tests**
  - **RFs:** `RF-01, RF-02, RF-04, RF-07, RF-09`
  - **Archivos:** `xBackend/services/whatsapp/whatsappGatewayService.ts`, `geminiCaseEvaluator.ts`, `test/whatsapp/gateway.test.ts`, `test/whatsapp/evaluator.test.ts`.
  - **Hecho cuando:** `sendSupportAlert` y `sendGroupMessage` lanzan `AgentMutedError` si el switch maestro está en `false`; `sendGroupMessage` exige `@g.us`; `listGroups` extrae `tenant_id`; `evaluateEmergencyMatch` evalúa con Gemini y falla seguro (`matches: false`).

- [ ] **T2: `WhatsAppBroadcastService` (con `abortAndPurge`) y `WhatsAppEmergencyService` (con cierre selectivo) + Tests**
  - **RFs:** `RF-03, RF-05, RF-06, RF-07, RF-08`
  - **Archivos:** `xBackend/services/whatsapp/broadcastService.ts`, `emergencyService.ts`, `test/whatsapp/broadcast.test.ts`, `test/whatsapp/emergency.test.ts`.
  - **Hecho cuando:** Tests unitarios verifican:
    1. Intervalo de 2000 ms entre mensajes de broadcast.
    2. `abortAndPurge()` detiene inmediatamente un broadcast en curso, marca los restantes como `aborted` y al re-encender el agente no se reenvían.
    3. Modo emergencia responde máximo 1 vez por grupo por `incident_id`, registra el grupo en `wa:emergency_replied_groups:{incident_id}` y marca la sesión `handled_by_staff = true`.
    4. Al apagar con `send_resolution: true`, envía `resolution_text` **únicamente** a los grupos de `wa:emergency_replied_groups:{incident_id}` y no al resto; al apagar con `send_resolution: false`, no envía nada.

- [ ] **T3: Endpoints en `controllers/whatsapp.ts`, `whatsappConsoleRoutes.ts` e Integración en `whatsappAgent.ts` + Tests**
  - **RFs:** `RF-01..RF-09`
  - **Archivos:** `xBackend/controllers/whatsapp.ts`, `xBackend/server/whatsappConsoleRoutes.ts`, `xBackend/services/whatsapp/whatsappAgent.ts`, `test/whatsapp/consoleApi.test.ts`.
  - **Hecho cuando:** `npx tsc --noEmit` y `npm test` pasan al 100%, incluyendo el manejo de `AgentMutedError` sin romper el webhook ni el worker de sesiones cuando el agente está apagado.

- [ ] **T4: Botón Maestro en Cabecera y Pestaña "Difusión y Emergencia" en `cpanel-admin` + Memoria**
  - **RFs:** `RF-01..RF-08`
  - **Archivos:** `cpanel-admin/src/app/core/shared/utils/const.ts`, `modules/common/whatsapp-support/whatsapp-support.*`, `xBackend/MEMORY.md`, `cpanel-admin/MEMORY.md`.
  - **Hecho cuando:** `tsc --noEmit` y `ngc --noEmit` pasan con 0 errores (sin ejecutar `ng build`, `npm run build` ni `pack-admin.sh`) y cero apariciones de "Astra".

---

## 6. Escenario de Pruebas (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Casos Borde                                  │
│ • Apagar emergencia con send_resolution=true pero sin texto │
│   de resolución: 400 Bad Request (emergencia sigue activa). │
│ • Apagar emergencia con send_resolution=true cuando 0       │
│   grupos preguntaron: apaga emergencia y envía 0 mensajes.  │
│ • Intentar iniciar broadcast o enviar resolución con el     │
│   Agente Apagado (Modo Mudo): 409 Conflict.                 │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Contratos e Integración                      │
│ • npx tsc --noEmit y npm test limpios en xBackend.          │
│ • tsc --noEmit y ngc --noEmit limpios en cpanel-admin.      │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia, Kill-Switch y Throttling        │
│ • En medio de un broadcast a 20 grupos (tras enviar 3), el  │
│   operador apaga el Agente: se aborta al instante, los 17   │
│   restantes quedan purgados ('aborted') y al volver a       │
│   encender el Agente NO se reanuda el envío viejo.          │
│ • Con Agente Apagado, un operador escribe "Agente: Ayuda" o │
│   vence una sesión: el ticket Kanban se crea en silencio y  │
│   Evolution API recibe 0 llamadas de salida.                │
│ • De 30 grupos totales, solo 4 reclaman por la incidencia:  │
│   al apagar con resolución, SOLO esos 4 reciben el cierre   │
│   (espaciados por 2000 ms).                                 │
└─────────────────────────────────────────────────────────────┘
```
