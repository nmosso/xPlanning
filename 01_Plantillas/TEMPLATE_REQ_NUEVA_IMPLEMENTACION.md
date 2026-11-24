# Documento de Especificación Técnica: [Nombre de la Funcionalidad]

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-[AÑO]-[CORRELATIVO]` (Ej: `REQ-2026-001`) |
| **Sistema Destino:** | `xBackend` / `xManager` / `Cross_System` |
| **Estado:** | `01_Borrador` / `02_Aprobado` / `03_Historico` |
| **Fecha de Creación:** | AAAA-MM-DD |
| **Última Actualización:** | AAAA-MM-DD |
| **Autor(es):** | [Nombre / Rol / Agente] |

---

## 1. Resumen Ejecutivo y Justificación de Negocio
*Breve descripción de la funcionalidad propuesta, el problema que resuelve y el valor que aporta al ecosistema.*

---

## 2. Filtro Doctrinal y Cuestionamiento Inicial (Filtro Británico)
*Aplicación estricta de "Make requirements less dumb" y "Keep it Plain".*

- **¿Cuál es la necesidad real identificada?**
- **¿Qué supuestos iniciales fueron descartados durante la discusión?**
- **¿Por qué esta solución es la más directa y no agrega complejidad accidental?**
- **¿Se eliminaron tablas, endpoints o procesos redundantes?**

---

## 3. Alcance y Límites del Requerimiento

### 3.1. En Alcance (In Scope)
- [ ] Elemento 1
- [ ] Elemento 2

### 3.2. Fuera de Alcance (Out of Scope - *YAGNI*)
- Elemento no prioritario o deliberadamente excluido para no sobrecargar el ciclo.

---

## 4. Arquitectura y Componentes Afectados

*Diagrama de flujo o descripción de cómo interactúan los componentes.*

- **Módulos / Controladores:**
- **Servicios Externos / Dependencias:**
- **Puntos de Interacción de Red:** (Recordar: `cpanel-admin` conecta DIRECTO a `xBackend` sin `xProxy`).

---

## 5. Contratos de Datos y Esquema DDL (Filtro Alemán)

### 5.1. Esquema de Base de Datos (PostgreSQL DDL)
```sql
-- DDL propuesto con integridad referencial, tipos e índices
CREATE TABLE IF NOT EXISTS nombre_tabla (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- campos tipados
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Índices requeridos (especialmente B-Tree o GIN para JSONB)
CREATE INDEX IF NOT EXISTS idx_nombre_tabla_campo ON nombre_tabla(campo);
```

### 5.2. Procedimientos Almacenados (Firma RPC Estándar)
*Ubicación obligatoria: `xServerSQL/_Functions/`*
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
BEGIN
    -- Extracción segura mediante jgk, jgkn, jgkj, jgkb
    -- Lógica transaccional
    RETURN jsonb_build_object('success', true, 'data', v_resultado);
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object('success', false, 'error', SQLERRM, 'code', SQLSTATE);
END;
$$;
```

### 5.3. Contratos de Datos JSONB (Payloads de Entrada y Salida)
- **`_data`:**
```json
{
  "campoObligatorio": "string",
  "valorNumerico": 123
}
```
- **`_params`:**
```json
{
  "limit": 50,
  "offset": 0
}
```
- **`_operator`:**
```json
{
  "userId": "uuid",
  "ipAddress": "192.168.1.10"
}
```
- **Respuesta RPC:**
```json
{
  "success": true,
  "data": { ... }
}
```

---

## 6. Especificación de Endpoints REST / Controladores (Thin Controller Pattern)

### 6.1. Endpoint: `[MÉTODO] /api/v1/ruta`
- **Controlador TypeScript:** Extrae `[data, params, operator]` y delega a `pgsql.sqlresponse`.
- **Códigos de Respuesta:**
  - `200 OK`: Éxito en la operación.
  - `400 Bad Request`: Parámetros inválidos o faltantes.
  - `500 Internal Server Error`: Fallo no controlado.

---

## 7. Requerimientos de Frontend / UI (si aplica)
- **Directiva No-Astra:** Certificar que ninguna etiqueta, modal, mensaje o selector use los términos "Astra" o "Astraserver". Usar términos como "Servidor", "Encoder" o "Servidor de Streaming".
- **Comportamiento Visual:** Estados de carga, mensajes de error y confirmaciones.

---

## 8. Escenario de Pruebas y Protocolo de Verificación (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Pruebas Unitarias y Casos Borde (Edge Cases) │
│ • Casos con campos nulos, strings vacíos y JSON malformado. │
│ • Validaciones de caracteres especiales y límites numéricos.│
│ • Intentos de duplicación de claves únicas.                 │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Pruebas de Integración y Contratos de Datos  │
│ • Backend: npx tsc --noEmit (cero errores de tipos).        │
│ • Frontend: npm run build (compilación sin fallos).         │
│ • Verificación de RPC Postgres y retorno JSONB.             │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia, Estrés y Concurrencia           │
│ • Timeouts y tolerancia a fallos en llamadas de red.        │
│ • Rollback visual y transaccional ante excepciones.         │
│ • Concurrencia y contención de bloqueos.                    │
└─────────────────────────────────────────────────────────────┘
```

### 8.1. Casos Borde a Evaluar (L1)
1. **Caso 1.1:** Payload con `_data` vacío `"{}"`. Resultado esperado: error de validación descriptivo.
2. **Caso 1.2:** Inyección de valores fuera de rango o caracteres acentuados.

### 8.2. Comandos de Validación Obligatorios (L2)
- [ ] `cd xBackend && npx tsc --noEmit`
- [ ] `cd cpanel-admin && npm run build` (si aplica frontend)
- [ ] Ejecución de script de prueba SQL en entorno de desarrollo.

### 8.3. Prueba de Resiliencia y Fallo (L3)
- Simulación de caída del servicio receptor o pérdida de enlace de red.

---

## 9. Criterios de Aceptación (Definition of Done - DoD)
- [ ] Programación e implementación de código realizada mediante **Claude Code**.
- [ ] Esquema DDL y funciones PL/pgSQL desplegadas y validadas en `xServerSQL`.
- [ ] Controladores delgados implementados en TypeScript sin lógica de negocio suelta.
- [ ] Pruebas L1, L2 y L3 ejecutadas satisfactoriamente con evidencia documentada.
- [ ] Regla No-Astra cumplida al 100%.
- [ ] Documentación actualizada y código listo para despliegue.
