---
name: postgres-sql
description: >-
  Estándares de ingeniería de bases de datos relacionales, procedimientos almacenados PL/pgSQL
  y patrones JSONB en PostgreSQL para el ecosistema xIPTV (xServerSQL, xClientSQL).
---

# PostgreSQL & PL/pgSQL Engineering Skill

Este skill define la convención y estándares mandatarios para el diseño de esquemas relacionales (DDL) y procedimientos almacenados (PL/pgSQL) en los componentes de base de datos del ecosistema (`xServerSQL`, `xClientSQL`).

---

## 1. Convención Estándar de Procedimientos Almacenados (RPC)

Toda función que exponga lógica transaccional hacia los backends debe cumplir estrictamente con la siguiente firma:

```sql
CREATE OR REPLACE FUNCTION schema_name.nombre_funcion(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_resultado jsonb;
    v_id uuid;
BEGIN
    -- 1. Extracción segura de parámetros usando helpers del sistema
    -- 2. Validaciones defensivas iniciales (fail-fast)
    -- 3. Lógica transaccional
    -- 4. Retorno estandarizado
    RETURN jsonb_build_object(
        'success', true,
        'data', v_resultado,
        'message', 'Operación completada exitosamente'
    );

EXCEPTION WHEN OTHERS THEN
    -- Manejo defensivo: nunca abortar con excepciones no controladas
    RETURN jsonb_build_object(
        'success', false,
        'error', SQLERRM,
        'code', SQLSTATE
    );
END;
$$;
```

---

## 2. Catálogo de Funciones Auxiliares para Extracción JSONB

Utilizar siempre los helpers nativos del proyecto para evitar errores de casteo:

| Helper | Tipo de Retorno | Uso Típico |
| :--- | :--- | :--- |
| `jgk(_json, 'campo')` | `text` | Nombres, descripciones, identificadores alfanuméricos. |
| `jgkn(_json, 'campo')` | `numeric / integer` | Contadores, IDs numéricos, límites, offsets. |
| `jgkj(_json, 'campo')` | `jsonb` | Sub-objetos o arrays anidados. |
| `jgkb(_json, 'campo')` | `boolean` | Flags, estados booleanos (`true`/`false`). |
| `verify_key_exists(_json, 'campo')` | `boolean` | Comprobar si el payload incluye el campo antes de procesar. |

---

## 3. Integridad Relacional y Optimización

1. **Claves Primarias y Foráneas:**
   - Preferir `UUID` con `gen_random_uuid()` como clave primaria para entidades que se replican o distribuyen entre nodos.
   - Definir siempre la política de eliminación en claves foráneas: `ON DELETE CASCADE` o `ON DELETE SET NULL`.

2. **Índices GIN para JSONB:**
   - Si una columna `jsonb` es consultada frecuentemente por campos internos con operadores `@>`, `?`, `?|`, crear un índice GIN:
     ```sql
     CREATE INDEX IF NOT EXISTS idx_nombre_metadata ON nombre_tabla USING GIN (metadata);
     ```

3. **Ubicación de Archivos:**
   - Toda función central de `xBackend` debe residir en `sql/xServerSQL/_Functions/`.
   - Toda función local de `xManager` debe residir en `sql/xClientSQL/_Functions/`.
