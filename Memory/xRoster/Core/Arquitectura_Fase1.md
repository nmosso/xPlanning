# Nota de Memoria Técnica: `xRoster` — Equipo de Agentes Operativos (Fase 1)

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xRoster` (`Cross_System` con `xBackend`) |
| **Componente:** | `Core / Agents / Providers / PostgreSQL (xrosterdb)` |
| **Funcionalidad:** | War Room WhatsApp (`Jarvis`), QoS (`Sentinel`), Infra/Hardware (`Tank`), Planificación (`Shikamaru`) y `quota_usage_ledger` |
| **Fecha:** | 2026-11-24 |
| **Autor/Agente:** | Lead Architect (`Planning-X`) + Claude Code |
| **Estado:** | Fase 1 implementada y verificada (`REQ-2026-012`); pendiente de provisión de `xrosterdb` y despliegue Docker en host `xBackend` |

---

## 1. Resumen y Contexto
Microservicio independiente en Docker (Node.js + TypeScript + PostgreSQL dedicado `xrosterdb`) que opera un equipo de agentes especializados desde un grupo privado de WhatsApp ("War Room") reutilizando la instancia Docker existente de **Evolution API v2** mediante un forward aislado en `xBackend`.

- **RFC Aprobado:** `Planning-X/Cross_System/02_Aprobados/RFC-2026-003_ARQUITECTURA_EQUIPO_AGENTES_XROSTER.md`
- **Spec Fase 1:** `Planning-X/Cross_System/02_Aprobados/REQ-2026-012_XROSTER_FASE1_JARVIS_SENTINEL_TANK_SHIKAMARU.md`

---

## 2. Decisiones Técnicas y Arquitectura
- **Despliegue Fase 1 (Camino A):** Contenedor Docker aislado en el servidor de `xBackend` (`cpus: "1.0"`, `mem_limit: "1536m"`), 100% portable vía `.env` a una VM dedicada.
- **Base de Datos Dedicada (`xrosterdb`):** Base PostgreSQL separada dentro de la instancia existente (rol dedicado `xroster_user`), elegida en lugar de SQLite para dar soporte concurrente y `JSONB`/GIN al futuro panel web **`cpanel-roster`** (Fase 2).
- **Aislamiento WhatsApp (`xBackend` `T-01`):** `WHATSAPP_COMMAND_GROUP_ID` se intercepta antes de `sessions.ingest`, `supportGroup` y `emergency` y se reenvía en modo *fire-and-forget* (timeout 3 s, `X-Internal-Secret`) a `XROSTER_WEBHOOK_URL`. `xRoster` aplica guardrail duro de salida (`ForbiddenTargetError` si `target !== WHATSAPP_COMMAND_GROUP_ID`).
- **Seguridad `R4: A` (`L0`/`L1`/`L2`):** Lectura autónoma (`L0`), escritura/reinicios/ramas Git solo con `CONFIRMAR <código>` (TTL 5 min, whitelist E.164 `OWNER_WHATSAPP_PHONES`), acciones destructivas y shell libre prohibidos en código (`L2`).
- **Telemetría de Cuotas (`Axelrod` base):** `ProviderRouter` (híbrido Gemini + Claude Code) registra toda invocación en `quota_usage_ledger` (tokens, latencia, `rate_limited`, ventanas 5h/24h).

---

## 3. Verificación y Estado de Pruebas (2026-11-24)
- **`xBackend` (`T-01`):** `npx tsc --noEmit` + `tsc -p tsconfig.test.json` OK; suite WhatsApp **262 pass, 0 fail, 1 skipped** (incluye 20 tests nuevos en `test/whatsapp/commandGroup.test.ts`).
- **`xRoster` (`T-02..T-08`):** `npx tsc --noEmit` OK; `npm run build` OK; `npm test` **143 pass, 0 fail** (incluye E2E webhook→Jarvis→Evolution y test estricto No-Astra).
- **Pendiente para puesta en producción:**
  1. Crear BD `xrosterdb` y rol `xroster_user` + rol read-only para `public.tenants` en PostgreSQL.
  2. Correr `TEST_PG_URL=... npm run test:sql` y `docker build` en el host Linux de despliegue.
  3. Confirmar nombres exactos de unidades systemd (`TANK_SERVICE_UNITS`) en una cabecera real.
  4. Configurar `../.env.roster` (fuera del repo Git para evitar sobrescritura en `git pull`) y las 3 variables en `xBackend` (`WHATSAPP_COMMAND_GROUP_ID`, `XROSTER_WEBHOOK_URL`, `XROSTER_WEBHOOK_SECRET`), y levantar `docker compose --env-file ../.env.roster up -d`.
