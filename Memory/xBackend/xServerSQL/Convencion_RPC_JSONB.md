# Memoria Técnica: Convención RPC y Funciones Auxiliares JSONB

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xBackend` |
| **Componente:** | `xServerSQL` |
| **Funcionalidad:** | Convención RPC `fn(_data, _params, _operator)` y Helpers `jgk*` |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
Toda la lógica de persistencia y transaccionalidad de `xBackend` reside en procedimientos almacenados en PostgreSQL bajo una firma estándar unificada.

---

## 2. Decisiones Técnicas y Arquitectura
- **Firma:** `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`
- **Helpers de Extracción:**
  - `jgk(json, 'key')` -> Extrae como texto (`text`).
  - `jgkn(json, 'key')` -> Extrae como número (`numeric`).
  - `jgkj(json, 'key')` -> Extrae sub-objeto o array (`jsonb`).
  - `jgkb(json, 'key')` -> Extrae valor booleano (`boolean`).
  - `verify_key_exists(json, 'key')` -> Retorna true si la clave existe.

---

## 3. Manejo de Errores Defensivo
```sql
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'code', SQLSTATE);
```
