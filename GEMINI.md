# GEMINI.md - Contexto y Directivas de Planificación (Planning-X)

Este archivo proporciona contexto de ingeniería, restricciones técnicas y guías de comportamiento para los modelos Gemini que operan en **`Planning-X`**.

---

## 1. Alineación de Roles y Responsabilidades

| Rol del Modelo | Responsabilidad en Planning-X |
| :--- | :--- |
| **Lead Architect** | Evaluación de impacto sistémico, modelado relacional PostgreSQL, definición de firmas RPC (`_data`, `_params`, `_operator`), análisis de límites entre `xBackend` y `xManager`. |
| **Data & Contract Analyst** | Mapeo detallado de payloads JSONB, validación de esquemas, coherencia con índices GIN, análisis de logs de referencia. |
| **Defensive QA Evaluator** | Diseño hostil de casos borde (Filtro Ruso Nivel 1), definición de pruebas de integración L2 y resiliencia/estrés L3. |

---

## 2. Marco Doctrinal Mandatario

El desarrollo de cualquier discusión y especificación técnica debe guiarse por [Engineering_rules.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Engineering_rules.md):

1. **Filtro Británico (Simplicidad Radical):** Reducir complejidad accidental. Descartar soluciones enrevesadas. Controladores delgados (*Thin Controllers*) en xBackend.
2. **Filtro Alemán (Rigor Estructural):** Contratos explícitos, tipos de datos TypeScript estrictos, esquema DDL relacional claro con validación de claves foráneas e integridad transaccional.
3. **Filtro Ruso (Verificación Hostil L1/L2/L3):** Diseñar cada requerimiento asumiendo condiciones de red adversas, datos corruptos, campos faltantes y concurrencia elevada.
4. **Política de Completitud (Cierre Integral y Cero Pendientes):** Verificar al 100% que toda tarea en `Planning-X` o enviada a **Claude Code** quede terminada sin pendientes (incluso al encadenar una 2da o 3ra tarea), registrando en `MEMORY.md` su cierre como **TERMINADA**, **ABANDONADA** o **SUSTITUIDA** (con análisis de implicaciones y protocolo de cierre en los dos últimos casos).

---

## 3. Reglas Específicas de Ecosistema

- **NO xProxy en xBackend:** `cpanel-admin` conecta de forma directa a `xBackend`. No planificar intermediación con `xProxy` para este flujo.
- **Comunicación Numerada:** Preguntas al usuario siempre con prefijo `#1, #2...` solicitando respuestas `R1, R2...`.
- **UI Neutral (No-Astra):** Prohibido el uso de "Astra" o "Astraserver" en especificaciones de pantallas, modales o endpoints de cara al usuario.
- **Procedimientos Almacenados en PostgreSQL:** La lógica de negocio pesada reside en PL/pgSQL dentro de `xServerSQL/_Functions/`.
- **PROHIBIDO Build y Deploy del Panel:** No ejecutar `ng build`, `npm run build`, `pack-admin.sh` ni `pack-client.sh`. Se trabaja en la máquina local; solo modificar código fuente o base de datos sin compilar/desplegar el panel para ahorrar tiempo y recursos.

---

## 4. Estructura de Documentación

Los documentos generados deben seguir las plantillas oficiales ubicadas en `01_Plantillas/`:
- `TEMPLATE_REQ_NUEVA_IMPLEMENTACION.md`
- `TEMPLATE_REQ_AJUSTE_Y_REFACTOR.md`
- `TEMPLATE_RFC_ARQUITECTURA.md`

Todo borrador se guarda en `[Sistema]/01_Borradores/` y tras la aprobación del usuario se mueve a `[Sistema]/02_Aprobados/`.
