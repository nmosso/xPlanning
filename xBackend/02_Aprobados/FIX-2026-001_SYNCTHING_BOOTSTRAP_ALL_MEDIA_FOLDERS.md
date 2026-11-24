# Documento de Ajuste, Refactor y Corrección Técnica

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `FIX-2026-001` |
| **Sistema Destino:** | `xBackend` |
| **Estado:** | `02_Aprobado` |
| **Severidad / Prioridad:** | Alta |
| **Fecha de Creación:** | 2026-11-13 |
| **Última Actualización:** | 2026-11-13 |
| **Autor(es):** | Antigravity (Lead Architect - Planning-X) |
| **Motor de Implementación:** | **Claude Code** |

---

## 1. Diagnóstico del Estado Actual y Problema Técnico
En `xBackend` (`POST /v3/tenants/:tenantid/syncthing/bootstrap-peers`), la función **"Vincular Peers"** utilizada desde la vista de *Gestión de Clúster Syncthing* en `cpanel-admin` presenta las siguientes limitaciones respecto a la configuración productiva de referencia (`http://100.64.89.8:8384/`):

1. **Cobertura parcial de carpetas:** Solo crea y vincula `Live3`. Omite `Live1`, `Live2`, `Movies` (`/mnt/media/vod/movies`), `TvShows` (`/mnt/media/tvshows`) e `Images` (`/mnt/media/images`).
2. **Tipo de carpeta incorrecto en el nodo tenant:** Crea `Live3` con `type: 'sendreceive'` en lugar de `type: 'receiveonly'` (solo recibir).
3. **Riesgo de ruta fuera de `/mnt/media/`:** Configura `autoAcceptFolders: true` en los peers `Pivot` y `Origin-One` antes de asegurar las carpetas locales, lo que puede provocar que Syncthing auto-acepte carpetas en la partición raíz (`/root/Sync/...`) donde no hay espacio en disco, en lugar de `/mnt/media/`.

---

## 2. Análisis de Causa Raíz (RCA - Root Cause Analysis)
- **Código involucrado:** [`SyncthingController.bootstrapPeers`](file:///d:/Projects/DEV/xIPTV/xBackend/controllers/syncthing.ts#L304-L588)
- **Mecanismo del fallo:**
  - Las líneas 398-440 buscan y crean únicamente `id.toLowerCase() === 'live3'`, hardcodeando `type: 'sendreceive'`.
  - Las líneas 489-497 (`Pivot`) y 549-557 (`Origin-One`) filtran estrictamente `pf.id.toLowerCase() === 'live3'` y `of.id.toLowerCase() === 'live3'`, impidiendo que `Origin-One` comparta `Live1`, `Live2`, `Movies`, `TvShows` e `Images`.
  - `autoAcceptFolders: true` en `Pivot` y `Origin-One` permite a Syncthing auto-crear carpetas en la ruta por defecto del sistema (`/root/Sync/`) si el peer remoto las ofrece antes de que existan localmente.

### Configuración Canónica Aprobada (Referencia `100.64.89.8:8384`)
**Topología unidireccional estricta:**
- `Pivot` (`100.64.0.7`) $\to$ Tenants (solo `Live3`) y `Pivot` $\to$ `Origin-One` (`Live3`).
- `Origin-One` (`100.64.0.6`) $\to$ Tenants (las 6 carpetas).
- **Nunca** desde Tenant hacia `Pivot` ni hacia `Origin-One`. En consecuencia, **las 6 carpetas (incluida `Live3`) en todo tenant deben ser estrictamente `receiveonly`**, y `autoAcceptFolders: false` tanto en el tenant como al registrar el tenant en `Pivot` y `Origin-One`.

| Folder ID | Label | Path obligatorio (`/mnt/media/`) | Type | Versioning | Order | Peers en Tenant |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `Images` | `Images` | `/mnt/media/images` | `receiveonly` | `trashcan` (`cleanoutDays: "1"`) | `random` | Tenant + `Origin-One` |
| `Live1` | `Live1` | `/mnt/media/live1` | `receiveonly` | `trashcan` (`cleanoutDays: "1"`) | `random` | Tenant + `Origin-One` |
| `Live2` | `Live2` | `/mnt/media/live2` | `receiveonly` | `trashcan` (`cleanoutDays: "1"`) | `random` | Tenant + `Origin-One` |
| `Live3` | `Live3` | `/mnt/media/live3` | `receiveonly` | `trashcan` (`cleanoutDays: "1"`) | `random` | Tenant + `Origin-One` + `Pivot` |
| `Movies` | `Movies` | `/mnt/media/vod/movies` | `receiveonly` | `trashcan` (`cleanoutDays: "1"`) | `random` | Tenant + `Origin-One` |
| `TvShows` | `TvShows` | `/mnt/media/tvshows` | `receiveonly` | `trashcan` (`cleanoutDays: "1"`) | `random` | Tenant + `Origin-One` |

---

## 3. Análisis de Impacto y Dependencias
- **Componentes Afectados:** `xBackend/controllers/syncthing.ts` (`bootstrapPeers`).
- **Riesgo de Regresión:** Bajo; se preserva la firma y el contrato HTTP de `POST /v3/tenants/:tenantid/syncthing/bootstrap-peers`. `cpanel-admin` invoca este endpoint sin parámetros de carpetas.
- **Clientes / Servicios Impactados:** Nodos `xManager` (Syncthing local), `Origin-One` (`100.64.0.6:8384`), `Pivot` (`100.64.0.7:8384`) y `cpanel-admin`.

---

## 4. Propuesta de Solución Quirúrgica (Filtro Británico)
- **Tabla declarativa única (`MEDIA_FOLDERS_SPEC`):** Definir las 6 carpetas canónicas con `id`, `label`, `path` bajo `/mnt/media/` y lista de peers (`ORIGIN_ONE`, y `PIVOT` únicamente para `Live3`).
- **Helper único para handshake bilateral (`upsertClusterPeerOnRemote`):** Eliminar la duplicación de código entre `Pivot` y `Origin-One`.
- **Salvaguardas estrictas para que ninguna carpeta caiga fuera de `/mnt/media/`:**
  1. **Desactivar `autoAcceptFolders` (`false`)** en los dispositivos `Pivot` y `Origin-One` dentro de la configuración del tenant (y al actualizar dichos peers si ya existían con `true`), impidiendo que Syncthing auto-cree carpetas en `/root/Sync/`.
  2. **Guardia de ruta explícita (`assertMediaPath`):** Verificar antes de persistir que cada `spec.path` comience exactamente con `/mnt/media/`.
  3. **Orden estricto de ejecución:**
     - Paso 1: Registrar/actualizar `Pivot` y `Origin-One` en el tenant con `autoAcceptFolders: false`.
     - Paso 2: Crear o normalizar las 6 carpetas en el tenant (`PUT /rest/config/folders`) **antes** de compartirlas desde `Origin-One` o `Pivot`. Si una carpeta ya existe (ej. `Live3` en `sendreceive` o con otra ruta/versionado), forzar actualización de `path` al canónico `/mnt/media/...`, `type: 'receiveonly'`, `order: 'random'`, `versioning: { type: 'trashcan', params: { cleanoutDays: '1' }, cleanupIntervalS: 3600, fsPath: '', fsType: 'basic' }` y los peers correspondientes sin duplicar `deviceID`.
     - Paso 3: Recién después, ejecutar el handshake bilateral en `Pivot` (solo `Live3`) y en `Origin-One` (las 6 carpetas de `MEDIA_FOLDERS_SPEC`).
- **Descarte de alternativas:** No se utiliza `autoAcceptFolders: true` porque Syncthing crea carpetas entrantes en el directorio por defecto del usuario (`/root/Sync/`), saturando el disco raíz.

---

## 5. Cambios Concretos Propuestos (Filtro Alemán)

### 5.1. Modificaciones en Base de Datos (`xServerSQL/_Functions/`)
No se requieren cambios en base de datos ni nuevas tablas ni procedimientos almacenados.

### 5.2. Modificaciones en Backend / Controladores (TypeScript)
- Archivo: `d:\Projects\DEV\xIPTV\xBackend\controllers\syncthing.ts`
- Definir `MEDIA_FOLDERS_SPEC` y `DEFAULT_TRASHCAN_VERSIONING`.
- Validar que cada ruta inicie con `/mnt/media/`.
- Actualizar `SyncthingController.bootstrapPeers`:
  - `autoAcceptFolders: false` para `Pivot` y `Origin-One` en el tenant.
  - Iterar `MEDIA_FOLDERS_SPEC` para crear/normalizar las 6 carpetas en el tenant (`receiveonly`, `trashcan` 1 día, `order: 'random'`, ruta `/mnt/media/...`).
  - Usar helper compartido para registrar el tenant en `Pivot` (compartiendo `Live3`) y en `Origin-One` (compartiendo `Images`, `Live1`, `Live2`, `Live3`, `Movies`, `TvShows`).
  - Retornar en la respuesta JSON `foldersConfigured` con los IDs y paths configurados.

---

## 6. Escenario de Pruebas de Regresión y Resiliencia (Filtro Ruso)

### 6.1. Pruebas Unitarias y Casos Borde (L1)
- [ ] Tenant nuevo sin carpetas de medios: crea las 6 carpetas bajo `/mnt/media/`, todas con `type: 'receiveonly'`, `order: 'random'`, `versioning.type: 'trashcan'` (`cleanoutDays: '1'`) y `autoAcceptFolders: false` en peers.
- [ ] Tenant existente con `Live3` en `sendreceive`, sin versionado o con ruta fuera de `/mnt/media/`: normaliza `path`, `type: 'receiveonly'` y `versioning` sin duplicar `deviceID`.
- [ ] Idempotencia: una segunda llamada consecutiva a `bootstrapPeers` reporta `devicesUpdated: false` y `foldersUpdated: false`.

### 6.2. Verificación de Integración (L2)
- [ ] Compilación TypeScript en `xBackend`: `npx tsc --noEmit` con código de salida 0.
- [ ] Verificación de respuesta `POST /v3/tenants/:tenantid/syncthing/bootstrap-peers` contra un tenant de prueba y lectura posterior de `GET /rest/config/folders` en el tenant, `Origin-One` y `Pivot`.

### 6.3. Pruebas de Resiliencia y Carga (L3)
- [ ] Caída o timeout de `Pivot` u `Origin-One`: la configuración local en `/mnt/media/` del tenant no falla y `bilateralHandshake` informa con precisión qué peer falló (`true`/`false`).

---

## 7. Plan de Contingencia y Rollback
- **Procedimiento de reversión:** Revertir el commit en `xBackend/controllers/syncthing.ts` y reiniciar el servicio `xBackend`.
- **Tiempo estimado de reversión:** < 5 minutos.

---

## 8. Criterios de Aceptación (Definition of Done)
- [ ] Programación e implementación ejecutada mediante **Claude Code** en `xBackend`.
- [ ] `npx tsc --noEmit` sin errores.
- [ ] Validado en un nodo tenant que las 6 carpetas (`Images`, `Live1`, `Live2`, `Live3`, `Movies`, `TvShows`) queden bajo `/mnt/media/`, en modo `receiveonly` y con versionado `trashcan` (1 día).
- [ ] Actualizar `xBackend/MEMORY.md` tras la implementación.
