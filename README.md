# Planning-X: Espacio Central de Planificación y Requerimientos

Bienvenido a **Planning-X**, el entorno de discusión técnica previa y generación de especificaciones formales para el ecosistema **xIPTV**, cubriendo tanto **`xBackend`** como **`xManager`**.

---

## 1. Misión del Espacio
La misión de **Planning-X** es garantizar que toda nueva funcionalidad, ajuste, corrección de errores o cambio arquitectónico cuente con un análisis sistémico riguroso, validado contra el marco de [Engineering_rules.md](file:///d:/Projects/DEV/xIPTV/Planning-X/Engineering_rules.md), **antes de escribir una sola línea de código en los repositorios de desarrollo**.

> [!IMPORTANT]
> **REGLA FUNDAMENTAL:**  
> **Primero Planificación Aprobada, luego Implementación.**  
> En este sector no se modifica código de producción. El resultado final de cada sesión es un **Documento de Requerimientos Técnico Aprobado**.

---

## 2. Marco Doctrinal de Referencia (Engineering_rules.md)

Toda discusión técnica debe someterse a los cuatro pilares doctrinales fundamentales:

1. **Doctrina Británica (*Keep it Plain* - Simplicidad Radical):**
   - Soluciones directas, interfaces intuitivas, controladores delgados (*Thin Controllers*), cero complejidad accidental y eliminación agresiva de código redundante (*YAGNI*).
2. **Doctrina Alemana (*Gründlichkeit* - Rigor Estructural):**
   - Coherencia sistémica, contratos de datos JSONB explícitos (`_data`, `_params`, `_operator`), integridad referencial DDL en PostgreSQL y tipado TypeScript estricto.
3. **Doctrina Rusa (*Nadezhnost* - Resiliencia y Verificación Hostil L1/L2/L3):**
   - Validación obligatoria de casos borde (L1), integración y contratos (L2), y tolerancia a fallos/estrés de red (L3).
4. **Política de Completitud (*Cierre Integral y Cero Pendientes*):**
   - Verificación estricta de que toda tarea en `Planning-X` o enviada a **Claude Code** quede completada sin pendientes (incluso al encadenar una 2da o 3ra tarea), cerrándose en `MEMORY.md` únicamente como **TERMINADA**, **ABANDONADA** o **SUSTITUIDA** (con análisis de implicaciones y cierre formal en los dos últimos casos).

---

## 3. Estructura del Espacio

```
Planning-X/
│
├── Engineering_rules.md                     # Marco de referencia doctrinal y operativo
├── AGENTS.md                                # Directivas automáticas para agentes Antigravity/Gemini
├── GEMINI.md                                # Contexto jerárquico de modelos Gemini
├── README.md                                # Este documento (índice maestro)
│
├── 00_Procedimientos/                       # Procedimientos y guías operativas
│   ├── PROTOCOLO_DISCUSION_Y_PLANIFICACION.md  # Flujo paso a paso para debatir requerimientos
│   ├── CHECKLIST_DOCTRINAL.md                  # Lista de verificación para aprobar especificaciones
│   └── GUIA_ARQUITECTURA_SISTEMAS.md           # Delimitación y responsabilidades xBackend vs xManager
│
├── 01_Plantillas/                           # Formatos estandarizados de documentos
│   ├── TEMPLATE_REQ_NUEVA_IMPLEMENTACION.md    # Plantilla para nuevas características / módulos
│   ├── TEMPLATE_REQ_AJUSTE_Y_REFACTOR.md       # Plantilla para incidentes, ajustes y refactors
│   └── TEMPLATE_RFC_ARQUITECTURA.md            # Plantilla para propuestas de cambio estructural
│
├── xBackend/                                # Especificaciones técnicas exclusivas de xBackend
│   ├── 01_Borradores/                          # Documentos en discusión técnica activa
│   ├── 02_Aprobados/                           # Documentos cerrados y validados (listos para codificar)
│   └── 03_Historico/                           # Histórico de implementaciones completadas o archivadas
│
├── xManager/                                # Especificaciones técnicas exclusivas de xManager
│   ├── 01_Borradores/
│   ├── 02_Aprobados/
│   └── 03_Historico/
│
└── Cross_System/                            # Especificaciones conjuntas (xBackend + xManager)
    ├── 01_Borradores/
    ├── 02_Aprobados/
    └── 03_Historico/
```

---

## 4. ¿Cómo Iniciar una Nueva Conversación en Planning-X?

Al iniciar una nueva conversación en este espacio de trabajo:

1. **Plantea tu requerimiento o problema en lenguaje natural.**  
   *Ejemplo: "Queremos agregar un mecanismo para pausar streams multicast en xManager cuando no haya clientes activos."*
2. **El agente adoptará automáticamente el rol de Arquitecto Técnico Senior.**  
   - Aplicará el Algoritmo de 5 pasos para cuestionar los supuestos ("Make requirements less dumb").
   - Identificará el sistema correspondiente (`xBackend`, `xManager` o `Cross_System`).
   - Te presentará las dudas clave usando la **Directiva de Comunicación Numerada** (**#1, #2...**).
3. **Responde utilizando el formato numerado (**R1, R2...**).**
4. **El agente redactará un borrador formal en la carpeta `01_Borradores/`** del sistema correspondiente utilizando la plantilla adecuada.
5. **Una vez que des tu aprobación explícita**, el documento se promoverá a la carpeta `02_Aprobados/` y se generará el plan de acción para abrir la sesión de desarrollo en el repositorio correspondiente (`xBackend` o `xManager`).

---

## 5. Reglas Mandatarias de Diseño del Ecosistema

- **NO xProxy en xBackend:** `cpanel-admin` conecta DIRECTAMENTE con `xBackend`. A diferencia de `xManager`, `xBackend` nunca utiliza `xProxy`.
- **Regla UI Branding (No-Astra):** Prohibido el uso de términos como "Astra" o "Astraserver" de cara al usuario. Usar términos neutros como "Servidor", "Encoder" o "Servidor de Streaming".
- **Thin Controller Pattern:** Los controladores en Node.js solo desempaquetan parámetros y delegan la lógica a procedimientos PL/pgSQL en PostgreSQL.
- **Firma RPC Estándar:** `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`.
# xPlanning
