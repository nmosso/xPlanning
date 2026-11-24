# Memoria Técnica: Orquestación de Canales y Metadatos

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xBackend` |
| **Componente:** | `API` |
| **Funcionalidad:** | Orquestación Central de Canales Live y 24/7 |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xBackend` actúa como la fuente canónica de la lista de canales para todo el ecosistema. `cpanel-admin` gestiona la asignación de feeds, configuraciones EPG y orígenes VOD directamente contra `xBackend`.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión:** Thin Controller en Express delegando a procedimientos PL/pgSQL en `xServerSQL`.
- **Alternativa Descartada:** Ejecutar lógica de negocio o joins complejos en TypeScript con ORMs tipo Prisma o TypeORM. Se descartó por rendimiento y consistencia transaccional con la doctrina del proyecto.
- **Archivos Clave:**
  - `xBackend/controllers/channels.ts`
  - `sql/xServerSQL/_Functions/fn_channels_manage.sql`

---

## 3. Contratos de Datos y Dependencias
- Comunicación directa desde `cpanel-admin` (NO usa `xProxy`).
- Firma RPC estándar: `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`.

---

## 4. Notas Operativas y Lecciones Aprendidas
- **Serialización en `pgsql.exec`:** Al pasar arreglos o payloads anidados, usar siempre `JSON.stringify()` para evitar el error `22P02` en Postgres.
