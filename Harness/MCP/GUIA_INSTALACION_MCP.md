# GUÍA DE INSTALACIÓN Y CONFIGURACIÓN DE MCP (MODEL CONTEXT PROTOCOL)

Esta guía explica cómo habilitar e integrar servidores MCP en los distintos proyectos del ecosistema xIPTV para potenciar a los agentes con acceso directo a bases de datos y validaciones web.

---

## 1. Asignación de MCPs por Tipo de Proyecto

| Tipo de Proyecto | Proyectos | Servidor MCP Recomendado | Propósito |
| :--- | :--- | :--- | :--- |
| **Node.js + PostgreSQL** | `xBackend`, `xManager`, `xFast`, `xClient` | `@modelcontextprotocol/server-postgres` | Inspección de esquemas, ejecución de queries de prueba y validación de tipos RPC. |
| **SQL puro** | `sql/xServerSQL`, `sql/xClientSQL` | `@modelcontextprotocol/server-postgres` | Introspección DDL, verificación de funciones PL/pgSQL y planes de ejecución (`EXPLAIN ANALYZE`). |
| **Frontends Web** | `cpanel/v2/cpanel-admin`, `cpanel/v2/cpanel-client` | `@executeautomation/playwright-mcp-server` | Inspección de renderizado UI, validación de consolas, captura de screenshots y testing E2E. |

---

## 2. Configuración en Antigravity / Gemini CLI / Claude

### Opción A: A nivel de Workspace / Proyecto (`.gemini/mcp_config.json` o `.agents/mcp_config.json`)
Cree un archivo `mcp_config.json` en la raíz del proyecto correspondiente (ej. en `xBackend/mcp_config.json`) adaptando la plantilla:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://xiptv_user:mi_password@127.0.0.1:5432/xiptv_db"
      ]
    }
  }
}
```

### Opción B: A nivel Global de Máquina (`~/.gemini/config/mcp_config.json`)
Permite que todos los proyectos compartan los servidores MCP definidos centralmente.

---

## 3. Pruebas de Funcionamiento
Una vez configurado el servidor MCP:
1. Inicie la sesión con el agente.
2. Solicite al agente: *"Lista las tablas de la base de datos"* o *"Verifica la estructura de la función schema.fn_example"*.
3. El agente invocará las herramientas provistas por el servidor MCP de forma transparente.
