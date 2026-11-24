# Memoria Técnica: Telemetría en Tiempo Real y Frontend de Nodo

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xManager` |
| **Componente:** | `cpanel-client` |
| **Funcionalidad:** | Visualización de Telemetría en Tiempo Real y Control de Canales |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`cpanel-client` es la interfaz web Angular para la supervisión y operación directa de los nodos de streaming `xManager`. Muestra estados de canales, actividad de encoders y métricas de bitrate en vivo.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión:** Comunicación reactiva mediante WebSockets / SSE con desuscripción estricta (`takeUntilDestroyed()`).
- **Alternativa Descartada:** Polling HTTP cada 500ms. Descartado por saturación innecesaria del servidor y latencia visual en la telemetría.
- **Regla Mandataria de UI Branding (No-Astra):** Prohibido rotundamente el uso de "Astra" o "Astraserver" en vistas, botones, modales o textos. Usar "Servidor", "Encoder" o "Servidor de Streaming".

---

## 3. Contratos de Datos y Dependencias
- Payload de telemetría de canal:
```json
{
  "channelId": "uuid",
  "name": "Canal 1 HD",
  "status": "broadcasting",
  "bitrateKbps": 6500,
  "fps": 30,
  "audioPids": [101, 102],
  "videoPid": 100
}
```

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- **Fugas de Memoria en Angular:** Las suscripciones de RxJS a flujos de datos continuos deben cancelarse al destruir los componentes para evitar saturación de la memoria del navegador.
