# PROTOCOLO DE DISCUSIÓN Y PLANIFICACIÓN TÉCNICA

Este documento describe el flujo de trabajo estandarizado que deben seguir los desarrolladores, arquitectos y agentes de IA en el espacio de trabajo **Planning-X**. Su propósito es transformar una idea o requerimiento inicial en un **Documento de Requerimientos Técnicos** maduro, validado doctrinalmente y listo para su implementación.

---

## 1. Principio Fundamental: Planificación Previa Obligatoria

> [!IMPORTANT]
> **REGLA DE ORO:** Ningún desarrollo de código, migración SQL o alteración de configuración en `xBackend` o `xManager` puede iniciarse sin contar previamente con un **Documento de Requerimientos Aprobado** generado en este espacio.

---

## 2. Flujo Operativo en 6 Etapas

```
[ ETAPA 1: Ingesta del Requerimiento en Bruto ]
                      │
                      ▼
[ ETAPA 2: Cuestionamiento y Descarte (Algoritmo Pasos 1 y 2) ]
                      │
                      ▼
[ ETAPA 3: Formulación de Consultas Numeradas (#1, #2... -> R1, R2...) ]
                      │
                      ▼
[ ETAPA 4: Diseño Técnico y Filtros Doctrinales (Británico, Alemán, Ruso) ]
                      │
                      ▼
[ ETAPA 5: Redacción del Borrador en 01_Borradores/ ]
                      │
                      ▼
[ ETAPA 6: Revisión, Visto Bueno y Promoción a 02_Aprobados/ ]
```

---

### ETAPA 1: Ingesta del Requerimiento en Bruto
1. **Identificación del Sistema:**
   - ¿Afecta a `xBackend`? (API central, base de datos de administración, TMDB, gestión de canales, ingesta general).
   - ¿Afecta a `xManager`? (Nodos distribuidos, ingesta/salida de video, flujos multicast/IPTV, moduladores ISDB-T, hardware local).
   - ¿Es un requerimiento transversal (`Cross_System`)? (Afecta tanto a la API central como a los agentes/nodos distribuidos).

2. **Categorización del Tipo de Trabajo:**
   - **Nueva Implementación:** Nueva funcionalidad, nuevo servicio o nuevo endpoint/módulo. Usar `TEMPLATE_REQ_NUEVA_IMPLEMENTACION.md`.
   - **Ajuste / Refactor / Bug Fix:** Corrección de un fallo existente, optimización de rendimiento o refactorización. Usar `TEMPLATE_REQ_AJUSTE_Y_REFACTOR.md`.
   - **Cambio Arquitectónico Estructural:** Cambio de paradigma o infraestructura. Usar `TEMPLATE_RFC_ARQUITECTURA.md`.

---

### ETAPA 2: Cuestionamiento y Descarte (El Algoritmo en Acción)
Antes de proponer esquemas de base de datos o código:
1. **Hacer el requerimiento menos tonto (*Make requirements less dumb*):**
   - Cuestionar el origen del pedido. ¿Cuál es el problema real que se intenta resolver?
   - Identificar supuestos no comprobados y características "por si acaso" (*YAGNI*).
2. **Eliminar componentes o procesos (*Delete the part or process*):**
   - ¿Se puede resolver reutilizando una función SQL existente?
   - ¿Se puede evitar la creación de una nueva tabla o un nuevo proceso en segundo plano?

---

### ETAPA 3: Formulación de Consultas Numeradas
Si existen ambigüedades, decisiones de diseño o dependencias no resueltas:
- El agente DEBE agrupar las preguntas con una numeración clara (**#1, #2, #3...**).
- El usuario responderá con la nomenclatura correspondiente (**R1, R2, R3...**).
- *Ejemplo:*
  > **#1:** ¿El identificador de canal debe ser un UUID v4 o se vinculará al ID entero histórico?  
  > **#2:** En caso de corte de red con el nodo, ¿cuál es el tiempo máximo de timeout antes de reintentar?  
  > *(Esperando respuesta: R1: ..., R2: ...)*

---

### ETAPA 4: Diseño Técnico y Filtros Doctrinales
Toda solución debe pasar los cuatro filtros de [Engineering_rules.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Engineering_rules.md):
- **Filtro Británico (Simplicidad Radical):**
  - Mantener controladores delgados (*Thin Controllers*) en Node.js.
  - No dispersar la lógica de negocio en múltiples servicios innecesarios.
- **Filtro Alemán (Rigor Estructural):**
  - Definición explícita de contratos JSONB: `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`.
  - Integridad referencial en PostgreSQL: claves foráneas, restricciones de unicidad e índices para JSONB.
  - Tipos e interfaces TypeScript estrictos.
- **Filtro Ruso (Resiliencia y Protocolo de Pruebas L1/L2/L3):**
  - **L1:** Pruebas unitarias y de casos borde (nulos, vacíos, strings largos, caracteres especiales).
  - **L2:** Pruebas de integración, compilación TypeScript (`npx tsc --noEmit`), build frontend Angular (`npm run build`).
  - **L3:** Pruebas de resiliencia ante cortes, timeouts de red y concurrencia.
- **Política de Completitud (Cierre Integral y Cero Pendientes):**
  - Verificar que no queden cabos sueltos ni tareas previas inconclusas (incluso al encadenar una 2da o 3ra tarea).
  - Registrar en `MEMORY.md` el compromiso de **TERMINAR** la tarea, cerrándola únicamente como **TERMINADA**, **ABANDONADA** o **SUSTITUIDA** (con análisis de implicaciones y cierre formal en los dos últimos casos).

---

### ETAPA 5: Redacción del Borrador
- Se crea el documento en la carpeta correspondiente:
  - `xBackend/01_Borradores/REQ-[AÑO]-[CORRELATIVO]_[NOMBRE_CORTO].md`
  - `xManager/01_Borradores/REQ-[AÑO]-[CORRELATIVO]_[NOMBRE_CORTO].md`
  - `Cross_System/01_Borradores/REQ-[AÑO]-[CORRELATIVO]_[NOMBRE_CORTO].md`
- Se completan todas las secciones de la plantilla correspondiente sin dejar campos vacíos ni suposiciones ocultas.

---

### ETAPA 6: Revisión, Visto Bueno y Promoción
1. El usuario revisa el borrador.
2. Si requiere ajustes, se itera sobre el borrador.
3. Al recibir la aprobación explícita del usuario:
   - El archivo se mueve a la carpeta `02_Aprobados/`.
   - Se genera un resumen ejecutivo con los puntos clave y los criterios de aceptación.
   - El requerimiento queda formalmente listo para que el equipo o agente de desarrollo lo tome en el repositorio de destino (`xBackend` o `xManager`).
