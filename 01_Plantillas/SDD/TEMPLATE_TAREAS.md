# TAREAS DE IMPLEMENTACIÓN (tasks.md): [Nombre de la Característica]

> **Reglas:**
> 1. Tareas de duración estimada inferior a 30 minutos (<30 min).
> 2. Ordenadas estrictamente por dependencia.
> 3. Motor Oficial de Programación: **Claude Code** (Tests primero, código mínimo, suite limpia, marcar y PARAR).

---

## Lista de Tareas Secuenciales

- [ ] **T1: Creación de la tabla DDL e índices en PostgreSQL**
  - **RFs Cubiertos:** `RF-01`
  - **Archivos:** `sql/xServerSQL/_Tables/streams_audit.sql`
  - **Hecho cuando:** La tabla e índice se crean en la base de datos de test y la consulta de inserción manual devuelve código 0.

- [ ] **T2: Test L1 y Procedimiento Almacenado PL/pgSQL**
  - **RFs Cubiertos:** `RF-02, RF-04`
  - **Archivos:** `sql/xServerSQL/_Functions/fn_stream_manage.sql`, `test/sql/test_stream_manage.sql`
  - **Hecho cuando:** La suite de pruebas SQL pasa para el caso normal y para el caso de error por parámetros vacíos.

- [ ] **T3: Controlador TypeScript y Marshalling JSONB**
  - **RFs Cubiertos:** `RF-01, RF-03`
  - **Archivos:** `controllers/stream.ts`
  - **Hecho cuando:** `npx tsc --noEmit` finaliza con código 0 y el test unitario HTTP con mock devuelve 200 OK y 503 según el estado del nodo.

- [ ] **T4: Validación Final y Cierre de Memoria**
  - **RFs Cubiertos:** Todos los RFs
  - **Archivos:** `Planning-X/Memory/xManager/Streaming/Multicast_Buffer_Management.md`
  - **Hecho cuando:** Todos los tests de la suite pasan satisfactoriamente y la nota de memoria se encuentra actualizada.
