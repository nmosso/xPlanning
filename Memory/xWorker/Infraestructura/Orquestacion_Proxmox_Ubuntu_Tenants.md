# Nota de Memoria: Orquestación y Automatización de Servidores con MCP Tenants

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xWorker` |
| **Componente:** | `Infraestructura` |
| **Funcionalidad:** | Automatización Proxmox/Ubuntu e Inventario Dinámico vía MCP `tenants` |
| **Fecha:** | 2026-10-17 |
| **Autor/Agente:** | Antigravity |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xWorker` es el proyecto encargado de ejecutar acciones específicas y automatizar tareas operativas en los servidores **Proxmox VE** y **Ubuntu** del ecosistema xIPTV que manualmente requieren mucho tiempo. Hereda íntegramente los principios de ingeniería y gobernanza de memoria de `xManager` y `MAINAGENTS.md`.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión Tomada:** Utilizar el servidor MCP **`postgres-xserver`** (`xBackend` DB) y su tabla `public.tenants` como única fuente de verdad (Single Source of Truth) para descubrir y filtrar la flota de servidores.
- **Alternativa Descartada:** Mantener archivos de inventario estáticos (`.ini` o `.json` manuales) que se desactualizan cuando se dan de alta, bloquean o reconfiguran cabeceras en `xBackend`.
- **Archivos Clave Involucrados:**
  - `d:\Projects\DEV\xIPTV\xWorker\AGENTS.md`
  - `d:\Projects\DEV\xIPTV\xWorker\.antigravity_rules.md`
  - `d:\Projects\DEV\xIPTV\xWorker\GEMINI.md`
  - `d:\Projects\DEV\xIPTV\xWorker\MEMORY.md`

---

## 3. Contratos de Datos y Dependencias
- **Dependencias Directas:** MCP Server `postgres-xserver` -> tabla `public.tenants`.
- **Estructura de Datos Relevante (`tenants`):**
```json
{
  "tenantid": "server83",
  "name": "Fontana",
  "localtion": "Fontana",
  "status": "enabled",
  "position": "83",
  "domain": "c83.xisrv.xyz",
  "role": "distributor",
  "options": {
    "devices": [],
    "rustdeskid": "...",
    "tsdb_token": "..."
  }
}
```
- **Regla Mandataria de Alcance por `position` (`server N`):**
  - Cuando se solicite operar sobre el `server 19, 23, 29...` (`position = N`), los 4 nodos de la cabecera son:
    1. **`100.64.N.2:22`**: Servidor **Proxmox VE** (Host hipervisor).
    2. **`100.64.N.8:22`**: VM **Ubuntu `xManager`** (`192.168.2.8`).
    3. **`100.64.N.2:8522`**: VM **Ubuntu `Multicast`** (`192.168.2.5`, SSH vía NAT).
    4. **`100.64.N.2:8222`** *(fallback `:8422`)*: VM **Ubuntu `Streamer`** (`192.168.2.4`, SSH vía NAT).
  - Dominio público/túnel: `c{position}.xisrv.xyz`.
  - **Estándar `/etc/environment` (`REQ-2026-006`):** En los 4 nodos (`Proxmox`, `xManager`, `Multicast`, `Streamer`), `/etc/environment` debe contener estrictamente **`CLIENT=N`** (numérico puro, ej. `CLIENT=83`, nunca `client83`).

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- **Trazabilidad Mandataria (LOG y Reporte):** Toda tarea ejecutada debe registrar su archivo de log en `xWorker/logs/` y generar su reporte consolidado en `xWorker/reports/` (además de presentarlo al operador).
- **Filtrado L1:** En ejecuciones por lote general, filtrar por defecto `status = 'enabled'` y `role = 'distributor'`, salvo cuando el operador indique explícitamente los números de servidor a intervenir.
- **Resiliencia L3:** Al iterar sobre múltiples servidores vía SSH/HTTP, aplicar siempre timeouts estrictos y capturar errores por nodo (`continue`) para que un nodo desconectado no bloquee la ejecución del lote restante.

