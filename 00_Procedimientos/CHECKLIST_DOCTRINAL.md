# CHECKLIST DE VALIDACIÓN DOCTRINAL

Toda especificación técnica redactada en **Planning-X** debe someterse a esta lista de verificación antes de ser promovida de `01_Borradores/` a `02_Aprobados/`.

---

## 1. Filtro Británico: *Keep it Plain* (Simplicidad Radical)
- [ ] **Necesidad probada:** ¿Se ha cuestionado el requerimiento inicial y descartado lo superfluo (*YAGNI*)?
- [ ] **Ausencia de capas redundantes:** ¿Se evitó crear microservicios, middlewares o capas intermedias que no aportan valor real?
- [ ] **Thin Controller Pattern (xBackend):** Si involucra endpoints de `xBackend`, ¿el controlador se limita a desempaquetar `[data, params, operator]` y delegar la lógica a la base de datos sin lógica de negocio dispersa en TypeScript?
- [ ] **Claridad expositiva:** ¿La solución puede explicarse y entenderse en menos de dos párrafos?

---

## 2. Filtro Alemán: *Gründlichkeit* (Rigor Estructural y Coherencia Sistémica)
- [ ] **Contratos de Datos RPC:** ¿Están detallados los payloads de entrada y salida con la firma estándar `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`?
- [ ] **Manejo Seguro de JSONB:** ¿Se contempla el uso de funciones seguras de extracción (`jgk`, `jgkn`, `jgkj`, `jgkb`, `verify_key_exists`)?
- [ ] **Integridad Relacional (DDL):**
  - [ ] Claves primarias claras (UUID / BIGSERIAL).
  - [ ] Claves foráneas con restricciones explícitas (`ON DELETE CASCADE` o `ON DELETE SET NULL`).
  - [ ] Índices específicos creados (especialmente índices B-Tree o GIN para búsquedas JSONB frecuentes).
- [ ] **Análisis de Impacto:** ¿Se evaluaron los efectos colaterales en clientes existentes (`cpanel-admin`, nodos `xManager`, agentes externos)?
- [ ] **Tipado Estricto:** ¿Están especificadas las interfaces TypeScript (`strict: true`) para controladores y modelos?

---

## 3. Filtro Ruso: *Nadezhnost* (Resiliencia y Escenario de Pruebas L1/L2/L3)
- [ ] **NIVEL 1 - Pruebas Unitarias y Casos Borde (Edge Cases):**
  - [ ] ¿Cómo responde el sistema si un campo requerido viene `null`, vacío `""` o ausente en el JSONB?
  - [ ] ¿Qué ocurre con cadenas excesivamente largas, acentos o caracteres especiales?
  - [ ] ¿Están previstas las violaciones de restricciones únicas (ej. duplicados)?
- [ ] **NIVEL 2 - Pruebas de Integración y Contratos:**
  - [ ] Verificación de compilación: `npx tsc --noEmit` sin errores en el backend correspondiente.
  - [ ] Verificación de build del frontend: `npm run build` sin fallos.
  - [ ] Validación de serialización de arrays y objetos en `pgsql.exec` (`JSON.stringify` preventivo contra error `22P02`).
  - [ ] Códigos de estado HTTP y respuestas de error estandarizadas.
- [ ] **NIVEL 3 - Resiliencia, Estrés y Concurrencia:**
  - [ ] Timeouts configurados en todas las llamadas de red o HTTP salientes.
  - [ ] Comportamiento ante desconexión o indisponibilidad temporal de un nodo o servicio.
  - [ ] Mecanismos de reintento con retroceso defensivo (*exponential backoff*).
  - [ ] Rollback transaccional en PostgreSQL ante excepciones durante la ejecución del procedimiento.

---

## 4. Política de Completitud: *Cierre Integral y Cero Pendientes*
- [ ] **Verificación sin Pendientes:** ¿Se verificó que todos los RFs, subtareas y criterios de aceptación queden cubiertos tanto en la spec como en el handoff a **Claude Code**?
- [ ] **Continuidad Multi-Tarea:** Si existen tareas previas activas (1ra, 2da, 3ra tarea), ¿se auditó su estado antes de iniciar o cerrar la actual para no dejar pendientes ocultos?
- [ ] **Estado Terminal y Registro en `MEMORY.md`:** ¿La tarea queda registrada en `MEMORY.md` con compromiso de **TERMINAR** o cerrada formalmente como:
  - [ ] **`TERMINADA`** (100% verificada sin pendientes), o
  - [ ] **`ABANDONADA`** / **`SUSTITUIDA`** (con análisis de implicaciones, limpieza de artefactos/código parcial y cierre formal documentado)?

---

## 5. Reglas Especiales del Ecosistema
- [ ] **Regla NO xProxy:** Si el requerimiento es para `xBackend` o `cpanel-admin`, ¿se garantiza comunicación directa sin pasar por `xProxy`?
- [ ] **Regla UI Branding (No-Astra):** Si el requerimiento incluye pantallas, modales o mensajes dirigidos al usuario, ¿se certifica la total ausencia de los términos "Astra" o "Astraserver"?
- [ ] **Ubicación de Archivos SQL:** ¿Se indica que los scripts SQL deben residir en `xServerSQL/_Functions/` (debido al symlink `sql/` de `xBackend`)?
