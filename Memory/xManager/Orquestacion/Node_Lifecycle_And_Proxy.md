# Memoria Técnica: Ciclo de Vida del Nodo y Enrutamiento con xProxy

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xManager` |
| **Componente:** | `Orquestacion` |
| **Funcionalidad:** | Registro de Nodo, Heartbeats, Failover y Enrutamiento con xProxy |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
Cada instancia de `xManager` opera como el backend principal de un operador de TV (distribución ISDB-T e IPTV). Para unificar la operación, un solo panel de control central interactúa con **`xproxy`**, el cual enruta las peticiones de forma segura hacia la instancia específica de `xManager` correspondiente a cada operador de TV.

---

## 2. Decisiones Técnicas y Arquitectura
- **Topología con xProxy:** Un único panel administrativo consume `xproxy` para conmutar y comunicarse con las múltiples instancias de `xManager` (1 Panel -> N Operadores de TV).
- **Diferencia Fundamental con xBackend:** Recordar que `xBackend` **NO** usa `xProxy` (conecta directo con `cpanel-admin`), mientras que `xproxy` es la pasarela exclusiva hacia las distintas instancias de `xManager`.
- **Heartbeat y Disponibilidad:** Cada operador/nodo reporta su disponibilidad y carga hacia el panel central a través del proxy.

---

## 3. Contratos de Datos
```json
{
  "nodeId": "uuid",
  "hostname": "NODE-EDGE-01",
  "ip": "10.0.1.50",
  "activeChannels": 16,
  "cpuUsage": 34.5,
  "networkOutMbps": 540.2,
  "status": "healthy"
}
```

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- **Tolerancia a Caídas de Red:** Si el nodo pierde comunicación con `xBackend`, la emisión de canales locales debe continuar ininterrumpida utilizando la última configuración persistida en `xClientSQL`.
