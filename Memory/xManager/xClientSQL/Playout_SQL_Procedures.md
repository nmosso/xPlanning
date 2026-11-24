# Memoria Técnica: Procedimientos de Playout y Persistencia Local

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xManager` |
| **Componente:** | `xClientSQL` |
| **Funcionalidad:** | Procedimientos Almacenados de Playout, Estados y Métricas |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xClientSQL` almacena la persistencia local de cada nodo `xManager`. Los bucles de monitoreo y telemetría de canales invocan estos procedimientos con alta frecuencia para registrar bitrates, pérdidas de señal y cambios de estado.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión:** Funciones PL/pgSQL RPC ultraligeras con firma `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb`.
- **Alternativa Descartada:** Escribir métricas en archivos de texto locales o SQLite. Se descartó para mantener consistencia transaccional con la API de telemetría y consultas concurrentes desde el panel `cpanel-client`.
- **Archivos Clave Involucrados:**
  - `sql/xClientSQL/_Functions/`
  - `sql/xClientSQL/_structure/instructions.txt`

---

## 3. Contratos de Datos y Dependencias
- Retorno JSONB estandarizado:
```json
{
  "success": true,
  "data": {
    "channelId": "uuid",
    "status": "online",
    "measuredBitrate": 6420
  }
}
```

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- **Bloqueos de Tabla:** No realizar transacciones de larga duración en `xClientSQL` porque pueden retrasar el loop de telemetría de video en tiempo real de `xManager`.
- **Serialización Segura:** En `pgsql.exec`, serializar siempre arrays y objetos con `JSON.stringify()` para evitar el error `22P02`.
