# INDEX.md - Catálogo Central de Memoria Técnica (Planning-X)

Este índice es la fuente de consulta unificada para que cualquier agente o desarrollador que opere en **xBackend**, **xManager** o cualquiera de los subsistemas pueda recuperar antecedentes técnicos, decisiones de arquitectura y notas de diseño previas.

---

## 1. Estructura Jerárquica de la Memoria

La memoria se organiza bajo el principio estricto:
```
Proyecto -> Componente -> Funcionalidad -> [Nota de Memoria]
```

Para crear una nueva nota, utilice la plantilla: [`TEMPLATE_NOTA_MEMORIA.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/TEMPLATE_NOTA_MEMORIA.md).

---

## 2. Mapa de Memoria por Proyecto

### 2.1. Bloque Central: xBackend
| Componente | Funcionalidad | Nota Técnica | Estado |
| :--- | :--- | :--- | :--- |
| **API** | Orquestación Canales & Metadatos | [Orquestacion_Canales.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/API/Orquestacion_Canales.md) | Activo |
| **API** | Orquestación Syncthing Clúster | [Syncthing_Cluster_Orchestration.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/API/Syncthing_Cluster_Orchestration.md) | Activo |
| **API & Monitoring** | Heartbeat (Redis), IRD/InfluxDB & Vista Monitoring (`REQ-2026-005`) | [Heartbeat_And_Monitoring_Architecture.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/API/Heartbeat_And_Monitoring_Architecture.md) | Activo |
| **API & Support** | Asistente WhatsApp de Soporte, Panel QR y Analítica (`REQ-2026-003` / `REQ-2026-004` / `REQ-2026-005`) | [Whatsapp_Support_Assistant.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/API/Whatsapp_Support_Assistant.md) | Activo |
| **xServerSQL** | Convención RPC y Helpers `jgk*` | [Convencion_RPC_JSONB.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/xServerSQL/Convencion_RPC_JSONB.md) | Activo |
| **cpanel-admin** | Directiva UI Branding No-Astra | [Reglas_UI_No_Astra.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/cpanel-admin/Reglas_UI_No_Astra.md) | Activo |
| **Tenants** | Bitácoras Individuales Dinámicas (`<tenantid>/MEMORY.md`: moduladores, IRD, canales, particularidades y problemas) | [Tenants/README.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/README.md) | Activo (Dinámico) |

### 2.2. Bloque Distribuido: xManager
| Componente | Funcionalidad | Nota Técnica | Estado |
| :--- | :--- | :--- | :--- |
| **Streaming** | Gestión de Buffers Multicast UDP | [Multicast_Buffer_Management.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xManager/Streaming/Multicast_Buffer_Management.md) | Activo |
| **Orquestacion** | Ciclo de Vida del Nodo y Failover | [Node_Lifecycle_And_Proxy.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xManager/Orquestacion/Node_Lifecycle_And_Proxy.md) | Activo |
| **xProxy** | Enrutamiento de Borde y Proxy Inverso | [Proxy_Edge_Routing.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xManager/xProxy/Proxy_Edge_Routing.md) | Activo |
| **xClientSQL** | Procedimientos de Playout Local | [Playout_SQL_Procedures.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xManager/xClientSQL/Playout_SQL_Procedures.md) | Activo |
| **Sincronización** | Syncthing Local API & Scheduler | [Syncthing_Local_Sync.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xManager/Sincronizacion/Syncthing_Local_Sync.md) | Activo |
| **cpanel-client** | Telemetría en Tiempo Real | [Realtime_Telemetry.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xManager/cpanel-client/Realtime_Telemetry.md) | Activo |

### 2.3. Subsistemas de Hardware y Playout
| Proyecto | Componente | Funcionalidad | Nota Técnica | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **xDevices** | Moduladores ISDB-T | Control Atómico y Firmware | [Matriz_Firmwares_Atomic.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xDevices/Moduladores_ISDBT/Matriz_Firmwares_Atomic.md) | Activo |
| **xClient** | Sincronización | Hashing xxHash64 y Rsync | [Hashing_xxHash64.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xClient/Sincronizacion/Hashing_xxHash64.md) | Activo |
| **xFast** | Playout | Canales FAST Acelerados | [Canales_Fast_Architecture.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xFast/Playout/Canales_Fast_Architecture.md) | Activo |
| **xFast & xManager** | Playout / Streamer | Control & Analítica Canal Local (`REQ-2026-010`) | [Canal_Local_Control_Analitica.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xFast/Playout/Canal_Local_Control_Analitica.md) | Activo |
| **xVodagent** | Ingesta y Sync | Transferencia Multimedia y Hashing | [Sync_Media_Agent.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xVodagent/Ingesta/Sync_Media_Agent.md) | Activo |
| **xWorker** | Infraestructura | Automatización Proxmox/Ubuntu y MCP Tenants | [Orquestacion_Proxmox_Ubuntu_Tenants.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xWorker/Infraestructura/Orquestacion_Proxmox_Ubuntu_Tenants.md) | Activo |

---

## 3. Guía de Consulta para Agentes
Antes de diseñar o implementar una característica:
1. Abra este `INDEX.md` y busque la sección del proyecto correspondiente.
2. Si la funcionalidad ya tiene una nota de memoria, léala para conocer decisiones previas y alternativas descartadas.
3. Al concluir la implementación o cambiar una decisión de fondo, cree o actualice la nota correspondiente.
