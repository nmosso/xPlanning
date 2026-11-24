# Documento de Ajuste, Refactor y Corrección Técnica

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `FIX-[AÑO]-[CORRELATIVO]` / `REF-[AÑO]-[CORRELATIVO]` |
| **Sistema Destino:** | `xBackend` / `xManager` / `Cross_System` |
| **Estado:** | `01_Borrador` / `02_Aprobado` / `03_Historico` |
| **Severidad / Prioridad:** | Crítica / Alta / Media / Baja |
| **Fecha de Creación:** | AAAA-MM-DD |
| **Última Actualización:** | AAAA-MM-DD |
| **Autor(es):** | [Nombre / Rol / Agente] |

---

## 1. Diagnóstico del Estado Actual y Problema Técnico
*Descripción detallada del comportamiento anómalo, cuello de botella o necesidad de refactorización observada en el sistema en ejecución.*

---

## 2. Análisis de Causa Raíz (RCA - Root Cause Analysis)
*Explicación técnica precisa de por qué ocurre el problema:*
- **Código / Procedimiento involucrado:** `[Ruta del archivo / función SQL]`
- **Mecanismo del fallo:** (Ej. bloqueo por transacción larga, deserialización JSON errónea `22P02`, fuga de conexiones, falta de índice, timeout no controlado).
- **Evidencia / Trazas de Error:**
```
[Logs, stack traces o mensajes de error capturados]
```

---

## 3. Análisis de Impacto y Dependencias
- **Componentes Afectados:**
- **Riesgo de Regresión:** (¿Qué otras funcionalidades consumen esta misma función SQL o endpoint?)
- **Clientes / Servicios Impactados:** (`cpanel-admin`, nodos `xManager`, workers).

---

## 4. Propuesta de Solución Quirúrgica (Filtro Británico)
*Aplicación estricta de la Simplicidad Radical: resolver de raíz con la menor cantidad de cambios posibles sin parches superficiales.*

- **¿Qué código, capa o proceso se puede eliminar o podar?**
- **Descripción de la solución directa:**
- **Descarte de alternativas:** *¿Por qué no se eligieron otras opciones más complejas?*

---

## 5. Cambios Concretos Propuestos (Filtro Alemán)

### 5.1. Modificaciones en Base de Datos (`xServerSQL/_Functions/`)
```sql
-- DDL o recreación de función PL/pgSQL
CREATE OR REPLACE FUNCTION schema_name.funcion_ajustada(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
-- Cambios quirúrgicos documentados
$$;
```

### 5.2. Modificaciones en Backend / Controladores (TypeScript)
*Detalle de archivos y funciones a editar, preservando el patrón Thin Controller y los tipos estrictos.*

---

## 6. Escenario de Pruebas de Regresión y Resiliencia (Filtro Ruso)

### 6.1. Pruebas Unitarias y Casos Borde (L1)
- [ ] Reproducir el escenario exacto que causaba el fallo original y verificar su resolución.
- [ ] Validar casos con parámetros vacíos o nulos en las funciones alteradas.

### 6.2. Verificación de Integración (L2)
- [ ] Compilación TypeScript: `npx tsc --noEmit` con código de salida 0.
- [ ] Si afecta frontend: `npm run build` en `cpanel-admin` con código de salida 0.
- [ ] Verificación de llamadas dependientes para certificar ausencia de regresión.

### 6.3. Pruebas de Resiliencia y Carga (L3)
- [ ] Simulación de alta concurrencia o pérdida de conectividad durante la operación corregida.

---

## 7. Plan de Contingencia y Rollback
*En caso de fallo imprevisto tras el despliegue:*
- **Procedimiento de reversión:** Pasos para reinstalar la versión anterior de la función SQL o revertir el commit en el repositorio.
- **Tiempo estimado de reversión:** [X minutos].

---

## 8. Criterios de Aceptación (Definition of Done)
- [ ] Programación e implementación de código ejecutada mediante **Claude Code**.
- [ ] El error/cuello de botella original está resuelto y demostrado con evidencia.
- [ ] Ninguna funcionalidad secundaria ha sufrido regresión.
- [ ] Compilación L2 completada con 0 errores.
- [ ] Procedimiento de rollback documentado y verificado.
