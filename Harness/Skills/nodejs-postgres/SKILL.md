---
name: nodejs-postgres
description: >-
  Estándares de arquitectura, buenas prácticas y verificación para proyectos Node.js (TypeScript)
  que interactúan con PostgreSQL mediante RPC y procedimientos almacenados en el ecosistema xIPTV.
---

# Node.js + PostgreSQL Engineering Skill

Este skill define las reglas de desarrollo, patrones de diseño y comandos de verificación para aplicaciones Node.js (TypeScript) que consumen bases de datos PostgreSQL en el ecosistema xIPTV (`xBackend`, `xManager`, `xFast`, `xClient`).

---

## 1. Patrón Thin Controller (Controladores Delgados)

1. **Responsabilidad Única del Controlador:**
   - Extraer `data`, `params` y `operator` del objeto de petición HTTP Express (`req.body`, `req.params`, `req.query`, `req.user`).
   - Validar la presencia de campos indispensables (fail fast).
   - Delegar inmediatamente la ejecución al procedimiento almacenado PL/pgSQL mediante el helper `pgsql.sqlresponse` o `pgsql.exec`.
   - Retornar la respuesta HTTP adecuada según el resultado devuelto por la base de datos.

2. **Prohibición de Lógica de Negocio Pesada en Node.js:**
   - No ejecutar bucles anidados intensivos de datos ni ensamblaje complejo de consultas con ORMs.
   - La lógica transaccional y de datos pesados pertenece al procedimiento almacenado en PostgreSQL.

---

## 2. Prevención Crítica del Error PostgreSQL `22P02`

Al enviar arrays u objetos anidados complejos a procedimientos almacenados mediante el driver de PostgreSQL (`pg` / `node-postgres`):

```typescript
// CORRECTO: Serializar explícitamente arrays u objetos anidados
const safeData = typeof data === 'object' ? JSON.stringify(data) : data;
await pgsql.exec('schema.function_name', safeData, params, operator);

// INCORRECTO: Pasar el array de JS directamente sin serializar
await pgsql.exec('schema.function_name', data, params, operator); // Dispara error 22P02 (invalid input syntax for type json)
```

---

## 3. Manejo Asíncrono y Resiliencia de Conexiones

1. **Timeouts en Llamadas Salientes:**
   - Toda petición HTTP a servicios externos o agentes distribuidos (`superagent`, `axios`, `fetch`) debe tener un timeout explícito (máx. 5000ms a 10000ms).
2. **Manejo de Errores de Conexión a Postgres:**
   - Capturar desconexiones del pool (`error` event en `pg.Pool`).
   - Retornar status HTTP 503 con código de error comprensible si la base de datos no está accesible.

---

## 4. Protocolo de Verificación Obligatorio (L2 QA)

Antes de dar por finalizada cualquier tarea en un proyecto Node.js:
```bash
# 1. Compilación estricta sin emisión (garantiza tipado correcto)
npx tsc --noEmit

# 2. Ejecutar suite de pruebas
npm test
```
*Ambos comandos deben finalizar con código de salida 0.*
