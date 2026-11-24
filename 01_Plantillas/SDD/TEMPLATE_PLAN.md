# PLAN DE ARQUITECTURA (plan.md): [Nombre de la Característica]

| Metadato | Detalle |
| :--- | :--- |
| **Referencia Spec:** | [`spec.md`](./spec.md) |
| **Referencia Constitución:** | [`constitution.md`](./constitution.md) |
| **Fecha:** | AAAA-MM-DD |

---

## 1. Módulos y Componentes Involucrados

| Módulo | Archivos / Componentes | Responsabilidad | RFs Cubiertos |
| :--- | :--- | :--- | :--- |
| **Controlador** | `controllers/stream.ts` | Extracción de parámetros y Thin Controller | RF-01, RF-04 |
| **Procedimiento SQL** | `_Functions/fn_stream_manage.sql` | Transaccionalidad y persistencia | RF-02, RF-03 |

---

## 2. Modelo de Datos y Esquema DDL
```sql
-- DDL propuesto con integridad referencial e índices
CREATE TABLE IF NOT EXISTS streams_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    metadata JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_streams_audit_lookup ON streams_audit(stream_id, event_type);
```

---

## 3. Decisiones Técnicas Justificadas y Alternativas Descartadas

### Decisión 1: [Nombre de la Decisión]
- **Solución Adoptada:** [Detalle técnico de la opción elegida]
- **Justificación:** [Por qué es la más simple y robusta]
- **Alternativa Descartada:** [Qué otra opción se consideró]
- **Motivo de Descarte:** [Por qué no se usó la alternativa (ej. sobrecarga de red, complejidad innecesaria)]

---

## 4. Estrategia de Tests (Filtro Ruso)

- **Pruebas L1 (Casos Borde):**
  - Caso 1: Payload `{}` sin stream_id. Cubre: `RF-04`.
  - Caso 2: Reconexión con timeout forzado. Cubre: `RF-02`.
- **Pruebas L2 (Integración):**
  - Compilación `npx tsc --noEmit` y test de integración RPC Postgres.
- **Pruebas L3 (Resiliencia):**
  - Simulación de pérdida de señal durante 5 segundos y verificación de conmutación.
