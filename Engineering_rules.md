# REGLAS DE ORQUESTACIÓN Y DOCTRINA OPERATIVA - xIPTV / xServer

Este documento define la doctrina de ingeniería de software, el protocolo operacional y las reglas estrictas de pruebas para los agentes de Inteligencia Artificial (Claude Code para programación, Antigravity y Gemini para planificación) que operan sobre el ecosistema **xIPTV** (`xBackend`, `xServerSQL`, `cpanel-admin`, `xManager`, etc.).

---

## 1. Los Cuatro Pilares Doctrinales Fundamentales

Toda propuesta técnica, refactorización, especificación o cambio de código debe superar las cuatro políticas/filtros doctrinales:

### 1.1. Doctrina Británica: *Keep it Plain* (Simplicidad Radical)
- **Principio:** Si la interfaz o el flujo de datos requiere explicaciones extensas, el diseño es defectuoso.
- **Aplicación:** Flujos limpios y directos, interfaces intuitivas en `cpanel-admin`, controladores delgados (*Thin Controllers*) en `xBackend` y ausencia total de complejidad accidental.

### 1.2. Doctrina Alemana: *Gründlichkeit* (Rigor Estructural y Coherencia Sistémica)
- **Principio:** La simplicidad jamás justifica la improvisación ni la ausencia de análisis sistémico previo.
- **Aplicación:** Mapeo exhaustivo de contratos de datos JSONB, integridad referencial en `xServerSQL`, tipos estrictos en TypeScript y análisis de impacto en dependencias antes de editar.

### 1.3. Doctrina Rusa: *Nadezhnost* (Verificación y Resiliencia Exhaustiva - Escenario de Pruebas)
- **Principio:** Lo que no está probado hasta el fallo en condiciones hostiles, no funciona.
- **Aplicación:** Toda solución debe acompañarse de un protocolo de pruebas multinivel riguroso. El agente asume el rol de evaluador hostil del sistema.

### 1.4. Política de Completitud: *Cierre Integral y Cero Pendientes*
- **Principio:** Una tarea propuesta no existe como resuelta hasta que se verifica su terminación total sin cabos sueltos, tanto en la planificación (`Planning-X`) como en la implementación (`Claude Code`).
- **Aplicación:**
  - **Verificación de Cierre Sin Pendientes:** Al finalizar una tarea aquí en `Planning-X` o al enviarla/recibirla de **Claude Code**, se debe auditar punto por punto (RFs, subtareas, pruebas L1/L2/L3 y actualización de `MEMORY.md`) para garantizar que no quede ningún pendiente oculto.
  - **Continuidad Multi-Tarea (2da, 3ra tarea y sucesivas):** Incluso cuando se solicite una segunda o tercera tarea en la misma sesión o en sesiones posteriores, el agente **NO** debe dar por olvidada ni dejar inconclusa la tarea anterior. Antes de cerrar o encadenar tareas, debe verificar el estado de cada tarea propuesta.
  - **Estados Terminales Obligatorios:** Toda tarea propuesta registrada en `MEMORY.md` debe terminar explícitamente en uno de tres estados:
    1. **TERMINADA:** Completada al 100% y verificada sin pendientes.
    2. **ABANDONADA:** Declarada explícitamente como descartada. Obliga a analizar y documentar las implicaciones (código parcial, tablas/endpoints huérfanos, deuda técnica) y ejecutar su limpieza/reversión y cierre formal.
    3. **SUSTITUIDA:** Declarada explícitamente como reemplazada por otra tarea/spec. Obliga a analizar y documentar las implicaciones (qué queda obsoleto, qué se migra o revierte), vincularla con la tarea sustituta y cerrar formalmente el caso original.
  - **Trazabilidad en `MEMORY.md`:** La obligación de **TERMINAR** las tareas propuestas (o cerrarlas formalmente bajo `ABANDONADA` / `SUSTITUIDA` con sus implicaciones) debe quedar plasmada de forma explícita y activa en `MEMORY.md`.

---

## 2. Escenario de Pruebas y Protocolo de Verificación Multinivel (Filtro Ruso)

Cualquier cambio de código debe ser verificado activamente siguiendo estos 3 niveles de prueba:

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Pruebas Unitarias y Casos Borde (Edge Cases) │
│ • Nulos, strings vacíos, campos inexistentes en JSONB.     │
│ • Longitudes máximas, caracteres especiales, acentos.      │
│ • Validaciones de unicidad (ej. email único, admin único). │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Pruebas de Integración y Contratos de Datos  │
│ • Compilación TypeScript estricta: `npx tsc --noEmit`       │
│ • Compilación de producción Angular: `npm run build`        │
│ • RPC Postgres: marshalling data/params/operator JSONB.     │
│ • Verificación de códigos de error y respuestas HTTP.       │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Pruebas de Resiliencia, Estrés y Concurrencia│
│ • Timeout y tolerancia a fallos en llamadas remotas.        │
│ • Rollback visual y transaccional ante fallos de red.       │
│ • Pruebas de carga y volumen en listas paginadas.           │
└─────────────────────────────────────────────────────────────┘
```

### Reglas de Verificación Obligatorias antes de Concluir una Tarea:
1. **Runner QA Backend:** Ejecutar `npx tsc --noEmit` en `xBackend` y garantizar código de salida 0.
2. **Runner QA Frontend:** Ejecutar `npm run build` en `cpanel-admin` y garantizar código de salida 0.
3. **Verificación SQL:** Validar la sintaxis y comportamiento de toda función PL/pgSQL modificada en `xServerSQL/_Functions/`.
4. **Reporte de Validación:** Documentar en el walkthrough los comandos ejecutados, los resultados obtenidos y los casos borde evaluados.
5. **Auditoría de Completitud (Cero Pendientes):** Verificar que toda tarea propuesta (incluyendo 2da/3ra tarea encadenadas) quede cerrada en `MEMORY.md` como **TERMINADA**, **ABANDONADA** o **SUSTITUIDA** (con análisis de implicaciones y cierre formal en los dos últimos casos).

---

## 3. Algoritmo Operativo de 5 Pasos (Secuencia Estricta)

```
[ Requerimiento Bruto ]
          │
          ▼
[ 1. Cuestionar Requisitos ] ──► Identificar la necesidad real y descartar supuestos
          │
          ▼
[ 2. Eliminar Componentes ]  ──► Eliminar código muerto, capas redundantes o pasos extra
          │
          ▼
[ 3. Simplificar / Optimizar ] ─► Solución más directa; nunca optimizar lo eliminable
          │
          ▼
[ 4. Acelerar el Ciclo ]     ──► Commits atómicos, validación inmediata
          │
          ▼
[ 5. Automatizar ]           ──► CI/CD, scripts de diagnóstico y verificación continua
```

1. **Hacer los requisitos menos tontos (*Make requirements less dumb*):** Cuestionar el requerimiento inicial. Eliminar supuestos "por si acaso" (*YAGNI*).
2. **Eliminar la parte o el proceso (*Delete the part or process*):** Podar agresivamente código innecesario. Menos líneas de código = menor superficie de fallos.
3. **Simplificar y optimizar (*Simplify or optimize*):** No añadir abstracciones innecesarias. Preferir código plano y comprensible.
4. **Acelerar el tiempo de ciclo (*Accelerate cycle time*):** Entregas atómicas, feedback rápido.
5. **Automatizar (*Automate*):** Automatizar verificación de tipos, compilación y scripts repetitivos.

---

## 4. Reglas Críticas del Ecosistema xIPTV / xServer

1. **NO xProxy Rule:** `cpanel-admin` se comunica **DIRECTAMENTE** con `xBackend`. A diferencia de `xManager`, `xServer` **NO USA `xProxy`**.
2. **Numbered Communication Directive:** Las preguntas dirigidas al usuario deben numerarse (**#1, #2...**) y esperar respuestas en formato (**R1, R2...**).
3. **UI Branding Rule:** Prohibido utilizar los términos "Astra" o "Astraserver" en etiquetas, modales, vistas o mensajes de error de cara al usuario. Usar términos neutros como "Servidor de Transmisión", "Servidor", "Encoder" o "Servidor de Streaming".
4. **Thin Controller Pattern:** Los controladores en `xBackend` solo extraen `[data, params, operator]` y delegan la lógica de negocio a procedimientos PL/pgSQL en `xServerSQL`.
5. **JSONB Parameter Pattern:** Firma RPC estándar: `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`. Utilizar `jgk`, `jgkn`, `jgkj`, `jgkb`, `verify_key_exists` para extracción segura.
6. **Symlink Dependency:** El directorio `sql/` de `xBackend` es un enlace simbólico a `xServerSQL`. Todo cambio SQL debe guardarse en `xServerSQL/_Functions/`.
7. **Motor Oficial de Programación (Claude Code):** La programación, creación e implementación de código, pruebas unitarias TDD y refactorizaciones en todos los proyectos se ejecuta mediante **Claude Code**.
