# MAINAGENTS.md - Sistema Maestro de Orquestación y Directivas (Ecosistema xIPTV)

Este documento es la **Constitución y Estándar Maestro** para todos los agentes de Inteligencia Artificial (Claude Code, Antigravity, Gemini) y desarrolladores que operen en el ecosistema **xIPTV** (`Planning-X`, `xBackend`, `xManager`, `xClient`, `xDevices`, `xFast`, `xWorker`, etc.).

Todos los archivos `AGENTS.md` locales de cada componente derivan obligatoriamente de este documento maestro.

---

## 0. División de Roles de IA y Motor Oficial de Programación (Claude Code)

Para optimizar la calidad del ciclo de vida del software, el ecosistema divide el trabajo de agentes de IA con especialización estricta:

1. **Claude Code (Motor Oficial de Programación y Codificación):**
   - **Rol:** Es la herramienta y agente designado para **crear la programación**, implementar features, refactorizar código, escribir pruebas unitarias (TDD) y ejecutar cambios de código en todos los proyectos del ecosistema.
   - **Ámbito:** Cualquier tarea de desarrollo activo de software en `xBackend`, `xManager`, `xVodagent`, `cpanel-admin`, `cpanel-client`, `sql/xServerSQL`, `sql/xClientSQL`, `xClient`, `xDevices`, `xFast`, `xproxy`, `xWorker`, etc.
   - **Ejecución Directa vía WSL (con Confirmación Obligatoria):** Una vez aprobada una especificación o requerimiento, el agente en `Planning-X` puede invocar directamente a **Claude Code** en WSL (`wsl bash -lc 'cd /mnt/d/Projects/DEV/xIPTV/<repo> && claude -p "<prompt>" --permission-mode acceptEdits --allowedTools "Read,Edit,Write,Bash(npx tsc*)"'`) y verificar su salida/compilación, **siempre preguntando previamente al usuario** si desea que la implementación se lance directamente con Claude Code o si el usuario la ejecutará por su cuenta.
2. **Planning-X & Antigravity / Gemini (Arquitectura, Especificación y Orquestación):**
   - **Rol:** Taller de planificación previa, diseño de especificaciones SDD (Fases 1 a 5, 7 y 8), evaluación hostil doctrinal (L1/L2/L3), gestión de infraestructura (`xWorker`) y consultas de inventario central (`postgres-xserver`).
   - **Flujo:** Planning-X produce las specs y tareas atómicas aprobadas, las cuales son entregadas a **Claude Code** para su programación.

---

## 1. Mapa Sistémico del Ecosistema xIPTV

El ecosistema se organiza en dos grandes macro-bloques funcionales y un conjunto de subsistemas especializados:

```
                                  ┌────────────────────────┐
                                  │       Planning-X       │
                                  │ (Taller de Specs y     │
                                  │   Memoria Central)     │
                                  └───────────┬────────────┘
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      ▼                                               ▼
         [ BLOQUE CENTRAL: xBackend ]                    [ BLOQUE DISTRIBUIDO: xManager ]
    ┌───────────────────────────────────┐               ┌─────────────────────────────────────────────────────────┐
    │ • xBackend (Node.js/Express API)  │               │ • xManager (Backend Principal ISDB-T e IPTV)            │
    │ • sql/xServerSQL (PL/pgSQL RPC)   │               │   - Orquestador Canales Live (metadata, fuentes, EPG)   │
    │ • cpanel-admin (Angular Frontend) │               │   - Orquestador Canales 24/7 (metadata, EPG, Syncthing) │
    └───────────────────────────────────┘               │   - Orquestador Películas/Series VOD y Clientes IPTV    │
                                                        │   - Backend de la plataforma / panel de control         │
                                                        │ • sql/xClientSQL (PL/pgSQL Local de Playout)            │
                                                        │ • cpanel-client (Frontend del Panel de Control)         │
                                                        │ • xproxy (Proxy entre panel único y N xManager)         │
                                                        └────────────────────────────┬────────────────────────────┘
                                                                                     │
                                       ┌─────────────────────────────────────────────┴─────────────────────────────┐
                                       ▼                                                                           ▼
                       [ SUBSISTEMAS DISTRIBUIDOS ]                                                [ SUBSISTEMAS DE BROADCAST ]
                       • xClient (Frontend de las APPs de IPTV)                                    • xDevices (Moduladores ISDB-T, Encoders)
                       • xFast / xFast2 (Motor de emisión continua 24/7 para ISDB-T e IPTV)        • xTranscoder (Transcodificación puntual)
                       • xOrigin / xVodagent / xSplicer                                            • Syncthing (Sincronización remota/local)
```

---

## 2. Marco Doctrinal Mandatario (Engineering_rules.md)

Toda planificación, arquitectura, especificación o código debe superar sin excepciones los cuatro pilares doctrinales:

### 2.1. Doctrina Británica (*Keep it Plain* - Simplicidad Radical)
- Si una solución o flujo requiere explicaciones extensas, el diseño es defectuoso.
- **Thin Controllers:** En Node.js, los controladores solo extraen `[data, params, operator]` y delegan la lógica a la base de datos o servicios atómicos.
- **YAGNI Estricto:** Eliminar código muerto, capas intermedias redundantes y microservicios innecesarios. Menos código = menor superficie de fallos.

### 2.2. Doctrina Alemana (*Gründlichkeit* - Rigor Estructural)
- **Contratos JSONB Explícitos:** Toda función RPC en PostgreSQL cumple con la firma estándar:
  `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`.
- **Integridad Relacional en PostgreSQL:** Esquemas DDL tipados, claves foráneas estrictas, índices B-Tree y GIN para búsquedas JSONB frecuentes.
- **Tipado TypeScript Estricto:** `strict: true`, interfaces claras y sin `any` arbitrarios.
- **JSON Serialization:** En llamadas `pgsql.exec`, serializar siempre arrays y objetos complejos mediante `JSON.stringify()` para prevenir errores de parsing `22P02` en PostgreSQL.

### 2.3. Doctrina Rusa (*Nadezhnost* - Verificación Hostil L1/L2/L3)
- Lo que no está probado hasta el fallo en condiciones hostiles no funciona.
  - **L1 (Edge Cases):** Nulos, cadenas vacías `""`, payloads `{}` incompletos, longitudes máximas, inyecciones de datos, colisiones de unicidad.
  - **L2 (Integración & Contratos):** Compilación TypeScript (`npx tsc --noEmit`), build Angular (`npm run build`), ejecución limpia de funciones SQL.
  - **L3 (Resiliencia & Estrés):** Timeouts defensivos, reconexión automática ante caídas de red, retroceso exponencial (*exponential backoff*), y rollback transaccional ante excepciones SQL.

### 2.4. Política de Completitud (*Cierre Integral y Cero Pendientes*)
- Ninguna tarea en `Planning-X` o delegada a **Claude Code** se considera resuelta hasta verificar al 100% que no queden pendientes ocultos, incluso al encadenar una 2da o 3ra tarea.
- Toda tarea propuesta debe quedar registrada en `MEMORY.md` con la obligación explícita de **TERMINARLA**, admitiendo únicamente tres estados terminales: **TERMINADA**, **ABANDONADA** o **SUSTITUIDA**.
- Si una tarea es declarada **ABANDONADA** o **SUSTITUIDA**, es obligatorio evaluar y documentar sus implicaciones (código parcial, deuda, objetos huérfanos, impacto cruzado) y ejecutar el cierre/limpieza formal del caso.

---

## 3. Metodología de Desarrollo: Spec-Driven Development (SDD)

En este ecosistema aplicamos el flujo estricto de **Desarrollo Guiado por Especificaciones (SDD)**:

```
1. Constitución ──► 2. Spec (Entrevista) ──► 3. Clarificación (QA Hostil)
                                                        │
┌───────────────────────────────────────────────────────┘
▼
4. Plan Arquitectura ──► 5. Tareas (<30 min) ──► 6. Implementación (TDD)
                                                        │
┌───────────────────────────────────────────────────────┘
▼
7. Validación (RF x RF) ──► 8. Gestión de Cambios (Diff en Spec primero)
```

Ver detalles y protocolos de ejecución en [SPEC_DRIVEN_DEVELOPMENT.md](file:///d:/Projects/DEV/xIPTV/Planning-X/00_Procedimientos/SPEC_DRIVEN_DEVELOPMENT.md).

---

## 4. Memoria y Base de Conocimiento Centralizada

Para evitar la pérdida de contexto entre conversaciones y proyectos:
1. **Memoria Local en Cada Componente (`MEMORY.md`):**
   - **Regla Mandataria:** Cada componente del ecosistema (`xBackend`, `xManager`, `xproxy`, `cpanel-admin`, `cpanel-client`, `xServerSQL`, `xClientSQL`, `xClient`, `xDevices`, `xFast`) DEBE mantener en su raíz su propio archivo **`MEMORY.md`**.
   - **Límite Estricto:** Máximo **150 líneas**.
   - **Gatillo de Actualización Obligatoria:** Tras **crear, actualizar, refactorizar o borrar** cualquier función, endpoint, tabla o componente, el agente DEBE actualizar de inmediato `MEMORY.md` antes de considerar terminada la tarea.
   - **Compromiso de Completitud y Cierre:** `MEMORY.md` debe plasmar la obligación de **TERMINAR** las tareas propuestas sin dejar pendientes (incluso al recibir una 2da o 3ra tarea), registrando su cierre bajo uno de tres estados: **TERMINADA**, **ABANDONADA** o **SUSTITUIDA** (documentando en estos dos últimos casos las implicaciones y la forma de cerrar el caso).
   - **Criterio de Síntesis:** Debe ser un resumen vivo de los avances macro, decisiones de arquitectura y estado general del componente, **conservando las partes generales y el estado actual del proyecto, y descartando lo particular, efímero o minucioso**.
2. **Memoria Técnica Centralizada (`Planning-X/Memory/`):**
   - Alberga el detalle funcional profundo organizado en: `Proyecto -> Componente -> Funcionalidad -> [Notas y Decisiones]`.
3. **Bitácora Individual Dinámica por Tenant (`xBackend`):**
   - **Ruta canónica:** `Planning-X/Memory/xBackend/Tenants/<tenantid>/MEMORY.md` (plantilla y protocolo en [`Planning-X/Memory/xBackend/Tenants/README.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/README.md)).
   - **Creación Dinámica Obligatoria (NO Pre-crear):** Prohibido pre-generar archivos para todos los tenants. Cada `<tenantid>/MEMORY.md` se crea bajo demanda únicamente cuando se opera sobre ese tenant en forma **individual** o **grupal/por lote**.
   - **Contenido Mandatario:** Identidad/topología, modulador(es) ISDB-T (modelo/firmware), IRD(s), cantidad y desglose de canales, características individuales (desvíos del estándar) y problemas conocidos / historial de intervenciones (máx. 200 líneas, sin secretos en texto plano, nomenclatura No-Astra).
   - **Protocolo:** Leer antes de operar (si existe) y crear/actualizar al finalizar cada intervención individual o grupal, manteniendo actualizado el índice dinámico de `Tenants/README.md`.
4. **Índice Maestro:**
   - El archivo [`Planning-X/Memory/INDEX.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/INDEX.md) es el catálogo unificado para que cualquier agente localice antecedentes técnicos de cualquier parte del sistema.

---

## 5. Harness Operativo: Skills y MCP

Los agentes cuentan con un arnés de herramientas especializadas para cada stack:
1. **Node.js & PostgreSQL:** Skill `nodejs-postgres` y MCP Server `@modelcontextprotocol/server-postgres`.
2. **Frontend Angular & Web:** Skill `angular-webdev` y MCP Browser/Playwright para inspección y pruebas UI.
3. **PostgreSQL & PL/pgSQL:** Skill `postgres-sql` con helpers de extracción (`jgk`, `jgkn`, `jgkj`, `jgkb`, `verify_key_exists`).

Ver catálogo y configuración en [Harness/](file:///d:/Projects/DEV/xIPTV/Planning-X/Harness/README.md).

---

## 6. Reglas No Negociables del Ecosistema

1. **Claude Code como Motor Mandatario de Programación:** Toda creación de nuevo código, implementación de funcionalidades, refactorizaciones y desarrollo de software en los repositorios debe ser ejecutada mediante **Claude Code**, alimentado por las especificaciones de Planning-X.
2. **NO xProxy para xBackend:** `cpanel-admin` conecta DIRECTAMENTE con `xBackend`. `xBackend` **NO USA `xProxy`**.
3. **Directiva No-Astra en UI:** Queda terminantemente prohibido utilizar los términos "Astra" o "Astraserver" en interfaces, etiquetas, modales, placeholders o mensajes al usuario. Usar términos neutros: "Servidor de Transmisión", "Servidor", "Encoder" o "Servidor de Streaming".
4. **Directiva de Comunicación Numerada:** Toda pregunta al usuario debe numerarse estrictamente (**#1, #2...**) y esperar respuestas en formato (**R1, R2...**).
5. **Regla de Ubicación SQL en xBackend:** El directorio `sql/` de `xBackend` es un enlace simbólico a `xServerSQL`. Toda función debe crearse en `xServerSQL/_Functions/`.
6. **Primero Planificación Aprobada, luego Implementación:** En ningún caso se escribe código en los repositorios sin haber completado y aprobado la especificación técnica en `Planning-X`.
7. **Mantenimiento Continuo de MEMORY.md:** Toda creación, modificación o eliminación de funciones exige la actualización inmediata de `MEMORY.md` antes de cerrar la tarea, conservando la visión macro y el estado general.
8. **Completitud Obligatoria (Cero Pendientes):** Tanto en `Planning-X` como al enviar tareas a **Claude Code**, se debe verificar que todo quede terminado sin pendientes (incluso al encadenar una 2da o 3ra tarea). Toda tarea propuesta en `MEMORY.md` solo cierra como **TERMINADA**, **ABANDONADA** o **SUSTITUIDA** (con análisis de implicaciones y cierre formal en los dos últimos casos).

---

## 7. Estructura Requerida para los `AGENTS.md` Locales de Cada Componente

Todo archivo `AGENTS.md` ubicado en la raíz de un subproyecto (ej. `xBackend/AGENTS.md`, `xManager/AGENTS.md`) debe seguir la siguiente estructura:

```markdown
# AGENTS.md - [Nombre del Componente]

Este archivo define las reglas locales para este componente y deriva del Estándar Maestro:
-> Referencia Maestra: [MAINAGENTS.md](file:///d:/Projects/DEV/xIPTV/Planning-X/MAINAGENTS.md)

## 1. Rol, Motor de Programación y Stack Tecnológico Local
- **Motor Oficial de Programación:** Claude Code (ejecución de tareas de código, refactorización y TDD).
- Stack: [Node.js / Angular / PL/pgSQL / Python / Bash]
- Repositorio/Ruta: [...]

## 2. Comandos de Verificación L2 (Runner QA)
- Compilación / Verificación estricta: `...`
- Tests: `...`

## 3. Acceso a Memoria
- Memoria específica de este componente: `Planning-X/Memory/[Proyecto]/[Componente]/`
- Índice General: `Planning-X/Memory/INDEX.md`

## 4. Protocolo Obligatorio de Actualización de MEMORY.md
- **Gatillo:** Al crear, editar o borrar cualquier función, endpoint o procedimiento, y al cerrar o encadenar tareas.
- **Acción:** Actualizar `MEMORY.md` reflejando el cambio a nivel macro, el estado general del proyecto y la verificación de que las tareas propuestas queden **TERMINADAS** sin pendientes (o cerradas formalmente como **ABANDONADA** / **SUSTITUIDA** con análisis de implicaciones).
- **Límite:** Máximo 150 líneas. Descartar detalles efímeros.

## 5. Reglas Particulares y Filtros Específicos
- [Reglas locales de red, hardware, timeouts o contratos]
```
