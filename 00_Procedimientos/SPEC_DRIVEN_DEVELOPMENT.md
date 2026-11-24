# SPEC-DRIVEN DEVELOPMENT (SDD) - PROTOCOLO MAESTRO DE EJECUCIÓN

Este documento establece el marco riguroso de **Desarrollo Guiado por Especificaciones (Spec-Driven Development)** para el ecosistema xIPTV (`Planning-X`, `xBackend`, `xManager`, etc.).

Su propósito es eliminar la improvisación, garantizar trazabilidad absoluta desde el requerimiento de negocio hasta el test unitario, y blindar el sistema contra la complejidad accidental.

---

## 1. El Ciclo de Vida SDD en 8 Fases

```
[ 1. CONSTITUCIÓN ] ──► [ 2. SPEC (Entrevista) ] ──► [ 3. CLARIFICACIÓN (QA) ]
                                                              │
┌─────────────────────────────────────────────────────────────┘
▼
[ 4. PLAN ARQUITECTURA ] ──► [ 5. TAREAS (<30 min) ] ──► [ 6. IMPLEMENTACIÓN (TDD) ]
                                                              │
┌─────────────────────────────────────────────────────────────┘
▼
[ 7. VALIDACIÓN (RF x RF) ] ──► [ 8. GESTIÓN DE CAMBIOS (Diff Primero) ]
```

---

## 2. Definición Detallada de las 8 Fases y Comandos de Disparo

### FASE 1: Constitución
- **Comando de Disparo:**
  > *"Proponme la constitución de este proyecto: N principios cortos y verificables sobre stack, calidad, tests y límites. Máx. 15 líneas. Espera mi aprobación."*
- **Reglas para el Agente:**
  - Redactar exactamente un decálogo o lista de principios estrictos y verificables en **máximo 15 líneas**.
  - Debe contener: stack tecnológico, reglas de calidad (ej. compilación limpia `strict: true`), límites de arquitectura y criterios de test.
  - **Parada obligatoria:** El agente no debe avanzar a la especificación hasta que el usuario responda *"Aprobado"*.
- **Plantilla:** [`TEMPLATE_CONSTITUCION.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/01_Plantillas/SDD/TEMPLATE_CONSTITUCION.md).

---

### FASE 2: Spec (Entrevista)
- **Comando de Disparo:**
  > *"NO escribas código. Hazme preguntas de una en una (máx. 6) sobre casos límite, errores y alcance, y después genera spec.md con RF numerados en EARS, fuera de alcance y criterios de finalización. Solo el QUÉ y el POR QUÉ."*
- **Reglas para el Agente:**
  - **PROHIBIDO escribir código.**
  - Las preguntas se formulan **de una en una** (hasta un máximo de 6 preguntas), esperando la respuesta del usuario antes de formular la siguiente.
  - Centrarse exclusivamente en: casos borde, fallos tolerables, volúmenes de datos y alcance.
  - Al completar las preguntas, generar `spec.md` redactando cada **Requisito Funcional (RF)** en **Sintaxis EARS** (*Easy Approach to Requirements Syntax*):
    - **Ubicuo:** *El sistema debe [hacer X].*
    - **Respuesta a Evento:** *CUANDO [evento], el sistema debe [hacer X].*
    - **Controlado por Estado:** *MIENTRAS [estado], el sistema debe [hacer X].*
    - **Opcional:** *DONDE [característica esté presente], el sistema debe [hacer X].*
    - **Comportamiento No Deseado:** *SI [condición de error], ENTONCES el sistema debe [hacer X].*
  - Explicar **solo el QUÉ y el POR QUÉ** (nunca el cómo técnico o implementación).
- **Plantilla:** [`TEMPLATE_SPEC_EARS.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/01_Plantillas/SDD/TEMPLATE_SPEC_EARS.md).

---

### FASE 3: Clarificación (Revisión Hostil de QA)
- **Comando de Disparo:**
  > *"Revisa la spec como un QA profesional: ambigüedades, contradicciones, casos límite ausentes, conflictos con la constitución. Solo detecta, no resuelvas."*
- **Reglas para el Agente:**
  - Asumir el rol de un Evaluador Hostil de Calidad (Filtro Ruso).
  - Listar explícitamente:
    1. Ambigüedades detectadas.
    2. Posibles contradicciones internas entre RFs.
    3. Casos borde (Edge Cases) no contemplados.
    4. Conflictos potenciales con los principios de la Constitución.
  - **Restricción:** El agente **solo detecta y expone los problemas**; no los soluciona unilateralmente sin input del usuario.

---

### FASE 4: Plan de Arquitectura
- **Comando de Disparo:**
  > *"Lee constitución y spec. Sin código: genera plan.md con módulos, modelo de datos, decisiones justificadas (con la alternativa descartada) y estrategia de tests. Indica qué RF cubre cada parte."*
- **Reglas para el Agente:**
  - **Sin código ejecutable.**
  - Mapear la arquitectura técnica: módulos, interfaces TypeScript, tablas PostgreSQL/DDL y firmas RPC.
  - Para cada decisión técnica relevante, incluir obligatoriamente:
    - **Decisión adoptada.**
    - **Alternativa descartada y justificación de por qué fue rechazada.**
  - Diseñar la matriz de pruebas (L1, L2, L3).
  - Indicar para cada componente qué **RF** específico está cubriendo.
- **Plantilla:** [`TEMPLATE_PLAN.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/01_Plantillas/SDD/TEMPLATE_PLAN.md).

---

### FASE 5: Tareas Atómicas (<30 min)
- **Comando de Disparo:**
  > *"Divide el plan en tareas de <30 min, ordenadas por dependencia, cada una con sus RF y una línea 'Hecho cuando:' verificable. Con checkboxes."*
- **Reglas para el Agente:**
  - Generar `tasks.md` con tareas secuenciales pequeñas y manejables (estimadas en menos de 30 minutos cada una).
  - Cada tarea debe tener:
    - [ ] `Tn: Nombre de la tarea`
    - **RFs cubiertos:** `[RF-01, RF-02]`
    - **Hecho cuando:** Condición binaria e incontestable (ej. *"El test unitario X pasa y npx tsc sale con 0"*).
- **Plantilla:** [`TEMPLATE_TAREAS.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/01_Plantillas/SDD/TEMPLATE_TAREAS.md).

---

### FASE 6: Implementación (TDD Estricto Tarea por Tarea con Claude Code)
- **Motor Designado:** **Claude Code** (agente y herramienta oficial de programación y codificación).
- **Comando de Disparo:**
  > *"Implementa SOLO la tarea Tn. Tests primero. Ejecuta la suite y muéstrame el resultado. Marca Tn como hecha y PÁRATE."*
- **Reglas para Claude Code:**
  - Trabajar **exclusivamente en la tarea solicitada `Tn`** (nunca adelantar tareas futuras).
  - **Tests Primero:** Escribir primero el caso de prueba L1/L2 que valida la condición *"Hecho cuando:"*.
  - Escribir el código mínimo necesario para satisfacer el test.
  - Ejecutar la suite de pruebas y compilación (`npx tsc --noEmit` / `npm test`).
  - Marcar el checkbox `[x]` de `Tn` en `tasks.md`.
  - **PARARSE:** Detener la ejecución inmediatamente y ceder el turno al usuario.

---

### FASE 7: Validación de Conformidad
- **Comando de Disparo:**
  > *"Recorre la spec RF por RF: qué test cubre cada uno y su resultado. Veredicto final: ¿spec cumplida?"*
- **Reglas para el Agente:**
  - Iterar ordenadamente por cada requisito funcional `RF-01`, `RF-02`, etc.
  - Mostrar:
    - `RF-XX`: Nombre del requerimiento.
    - Test asociado: Ruta y nombre del test L1/L2/L3.
    - Resultado de la ejecución: `PASSED` / `FAILED`.
  - Emitir un **Veredicto Final Conclusivo**: `¿SPEC CUMPLIDA AL 100%? (SÍ / NO)`.
  - **Auditoría de Completitud (Cero Pendientes):** Verificar que no quede ningún pendiente de la tarea actual ni de tareas previas (2da/3ra tarea) y actualizar `MEMORY.md` marcando el estado terminal correspondiente (**`TERMINADA`**, **`ABANDONADA`** o **`SUSTITUIDA`**, documentando implicaciones y cierre formal en los dos últimos casos).

---

### FASE 8: Gestión de Cambios
- **Comando de Disparo:**
  > *"Nuevo requisito: [Descripción]. NO toques código: actualiza primero la spec y muéstrame el diff."*
- **Reglas para el Agente:**
  - **PROHIBIDO modificar código fuente.**
  - Modificar únicamente el archivo `spec.md` incorporando los nuevos RFs o ajustando los existentes en sintaxis EARS.
  - Presentar al usuario el **diff claro de la spec** para su revisión y aprobación antes de volver a iterar sobre el plan o las tareas.
