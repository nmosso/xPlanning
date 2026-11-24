# Memoria Técnica: Orquestación de Syncthing en Clúster Distribuido

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xBackend` |
| **Componente:** | `API` / `Orquestador de Nodos` |
| **Funcionalidad:** | Gestión Centralizada de Nodos Syncthing y Canales 24/7 |
| **Fecha:** | 2026-10-14 |
| **Estado:** | Activo / En Especificación |

---

## 1. Resumen y Contexto
`xBackend` gestiona la topología de distribución de medios mediante Syncthing a través de toda la red de plataformas edge (`xManager`). Los servidores se direccionan mediante una IP fija calculada a partir de su posición: `http://100.64.{tenant.position}.8:8384`.

---

## 2. Roles del Clúster y Flujo de Contenidos 24/7
1. **`Origin-One` (`GMGVA2R-...`):**  
   - Servidor semilla central. Distribuye las carpetas iniciales `live1`, `live2`, `vod/movies`, `tvshows` e `images`.
   - Actúa como receptor de `live3` proveniente de `Pivot`.
2. **`Pivot` (`AM5VNFY-...`):**  
   - Servidor master exclusivo para la carpeta `Live3`.
   - Ubicado junto a `filesin` compartiendo almacenamiento. Aprovecha su baja latencia y alto ancho de banda para alimentar a todos los nodos remotos y a `Origin-One`.
3. **Plataformas Distribuidas (Nodos Edge):**  
   - Reciben contenido inicial de `Origin-One`.
   - Reciben actualizaciones en tiempo real de `Live3` desde `Pivot`.

---

## 3. Endpoints y Capacidades del Orquestador
- **Telemetría y Estado:** `GET /v3/tenants/:tenantid/syncthing/status` (ping, completitud de carpetas, uptime).
- **Pausa / Reanudación:** `POST /v3/tenants/:tenantid/syncthing/pause` (control de congestión en edge).
- **Bootstrap de Pares:** `POST /v3/tenants/:tenantid/syncthing/bootstrap-peers` (agrega automáticamente `Origin-One` y `Pivot` a la configuración remota si faltan).
- **Conmutación Controlada:** El panel `cpanel-admin` permite al operador verificar la sincronización de `Live3` en todos los nodos antes de cambiar manualmente el canal de `live1`/`live2` a `live3`.
