# Harness: Skills y MCP para el Ecosistema xIPTV

Este directorio contiene el catálogo central de **Skills** y configuraciones **MCP (Model Context Protocol)** diseñadas para estandarizar y potenciar la asistencia de los agentes de IA en cada tecnología del ecosistema.

---

## 1. Catálogo de Skills

| Skill | Tecnologías | Proyectos Destino | Ubicación |
| :--- | :--- | :--- | :--- |
| **`nodejs-postgres`** | Node.js, TypeScript, PostgreSQL | `xBackend`, `xManager`, `xFast`, `xClient` | [`Skills/nodejs-postgres/SKILL.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/Skills/nodejs-postgres/SKILL.md) |
| **`angular-webdev`** | Angular 16+, SCSS, RxJS, No-Astra UI | `cpanel-admin`, `cpanel-client` | [`Skills/angular-webdev/SKILL.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/Skills/angular-webdev/SKILL.md) |
| **`postgres-sql`** | PostgreSQL DDL, PL/pgSQL RPC, JSONB | `sql/xServerSQL`, `sql/xClientSQL` | [`Skills/postgres-sql/SKILL.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/Skills/postgres-sql/SKILL.md) |
| **`isdbt-modulators`** | Playwright, Control Hardware ISDB-T | `xDevices`, `xManager` | Global (`~/.gemini/config/skills/isdbt-modulators/`) |

---

## 2. Servidores MCP

- **PostgreSQL:** [`MCP/postgres_mcp_config.json`](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/MCP/postgres_mcp_config.json) para conexión directa y diagnóstico de bases de datos.
- **Web / Browser:** [`MCP/browser_mcp_config.json`](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/MCP/browser_mcp_config.json) para testing automatizado de los paneles Angular.
- **Guía de Puesta en Marcha:** [`MCP/GUIA_INSTALACION_MCP.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/MCP/GUIA_INSTALACION_MCP.md).
