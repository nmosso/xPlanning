# MEMORY.md - Planning-X

> **Regla de Gobernanza:** Memoria sintética del taller central de planificación (`Planning-X`). Máximo 150 líneas. Resume el marco metodológico vigente, herramientas de arnés y estado de las especificaciones del ecosistema xIPTV.
> - **Catálogo Central:** [`Planning-X/Memory/INDEX.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/INDEX.md)
> - **Estándar Maestro:** [`Planning-X/MAINAGENTS.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/MAINAGENTS.md)

---

## 1. Propósito y Rol Arquitectónico
Taller de discusión previa, debate hostil y generación formal de documentos de requerimientos técnicos para `xBackend`, `xManager` y todos los subsistemas.
- **Regla Fundamental:** En este espacio NO se toca código de producción. Primero planificación aprobada, luego implementación en repositorios de destino.
- **Marco Doctrinal:** [Engineering_rules.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Engineering_rules.md) (Filtro Británico, Alemán, Ruso y **Política de Completitud**).

---

## 2. Metodología y Herramientas Vigentes
1. **Spec-Driven Development (SDD):** Marco de 8 etapas (Constitución, Spec EARS, Clarificación QA, Plan, Tareas <30 min, Implementación TDD, Validación RFxRF, Gestión de Cambios con diff).
2. **Memoria Jerárquica:** Estructura `Proyecto -> Componente -> Funcionalidad` en `Memory/` complementada con `MEMORY.md` ($\le$ 150 líneas) en la raíz de cada componente.
3. **Harness Operativo:** Skills estandarizados (`nodejs-postgres`, `angular-webdev`, `postgres-sql`, `isdbt-modulators`) y configuraciones MCP.
4. **Directiva de Comunicación:** Preguntas numeradas `#1, #2...` y respuestas `R1, R2...`.

---

## 3. Política de Completitud y Cierre de Tareas (OBLIGATORIO)
- **Compromiso de TERMINAR:** Toda tarea propuesta en `Planning-X` o enviada a **Claude Code** DEBE verificarse y completarse al 100% sin dejar ningún pendiente oculto, **incluso cuando se solicite una segunda o tercera tarea encadenada**.
- **Auditoría Pre-Cierre:** Antes de dar por concluida una tarea (o pasar a la siguiente sin cierre), verificar punto por punto: RFs, subtareas, tests L1/L2/L3 y actualización de `MEMORY.md`.
- **Estados Terminales Válidos (Sin Limbo):** Toda tarea propuesta solo puede cerrarse bajo uno de estos 3 estados:
  1. **`TERMINADA`:** Completada al 100% y verificada sin pendientes.
  2. **`ABANDONADA`:** Declarada explícitamente como descartada. Obliga a analizar y documentar implicaciones (código parcial, objetos huérfanos, deuda) y ejecutar limpieza/reversión para cerrar el caso.
  3. **`SUSTITUIDA`:** Declarada explícitamente como reemplazada por otra tarea/spec. Obliga a analizar y documentar implicaciones (qué queda obsoleto, qué se hereda o revierte), vincular a la nueva tarea y cerrar formalmente el caso original.

---

## 4. Hitos y Avances Generales
- [x] **Establecimiento de MAINAGENTS.md:** Estandarización de `AGENTS.md` y `MEMORY.md` en los 10 componentes del ecosistema.
- [x] **Base de Conocimiento Centralizada:** Creación del árbol de memoria técnica e índice maestro.
- [x] **Procedimientos Operativos:** Documentación de SDD, filtros doctrinales y delimitación de fronteras de sistemas.
- [x] **Plantillas Formales:** Modelos para Nuevas Implementaciones, Ajustes/Refactors, RFCs de Arquitectura y artefactos SDD.
- [x] **Política de Completitud (4ta Política):** Incorporación del cierre integral y control de estados (`TERMINADA` / `ABANDONADA` / `SUSTITUIDA`) en toda la doctrina y en `MEMORY.md`.
- [x] **Bitácoras Dinámicas por Tenant (`xBackend`):** Protocolo y plantilla en [`Planning-X/Memory/xBackend/Tenants/`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/README.md) para crear dinámicamente `<tenantid>/MEMORY.md` (moduladores, IRD, canales, particularidades y problemas) solo al operar cada tenant de forma individual o grupal (propagado a `MAINAGENTS.md`, `Planning-X/AGENTS.md`, `xBackend/{AGENTS,CLAUDE,GEMINI,MEMORY}.md` y `xWorker/AGENTS.md`). — **Estado: TERMINADA**.

---

## 5. Estado Actual y Foco Activo
- **Estado General:** Infraestructura de planificación completamente operativa (4 pilares doctrinales vigentes).
- **Foco Activo:** Recepción, maduración y cierre integral sin pendientes de especificaciones para el ecosistema xIPTV.
