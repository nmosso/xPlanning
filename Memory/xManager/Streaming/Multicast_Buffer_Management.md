# Memoria Técnica: Gestión de Buffers y Streaming Multicast UDP

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xManager` |
| **Componente:** | `Streaming` |
| **Funcionalidad:** | Ingesta y Emisión Multicast UDP de Alta Densidad (750 Mbps) |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xManager` es el motor central de streaming distribuido en los nodos de xIPTV. Gestiona el enrutamiento de flujos de video MPEG-TS mediante UDP Multicast hacia moduladores, decodificadores y passthrough local, soportando un throughput sostenido de hasta 750 Mbps.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión:** Sockets UDP nativos en Node.js configurados en modo estrictamente no-bloqueante, con tamaño de buffer de recepción (`SO_RCVBUF`) y envío (`SO_SNDBUF`) ajustados a nivel de kernel de Linux (`sysctl`).
- **Alternativa Descartada:** Usar streams HTTP o WebSockets para el transporte interno de emisión TS de alta densidad. Se descartó debido a la sobrecarga de TCP, jitter y latencia excesiva para broadcast televisivo en vivo.
- **Archivos Clave Involucrados:**
  - `src/controllers/stream/`
  - `src/services/multicast/`

---

## 3. Contratos de Datos y Dependencias
- **Dependencias Directas:** Procedimientos en `sql/xClientSQL` para registrar el estado de los canales (running, stopped, error).
- **Parámetros de Configuración Multicast:**
```json
{
  "multicastIp": "239.255.10.1",
  "port": 5000,
  "interface": "eth0",
  "ttl": 16,
  "bitrateKbps": 6500
}
```

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- **Pérdida de Paquetes (Drops):** Si el Event Loop de Node.js se bloquea por cómputo intensivo, los sockets UDP pierden datagramas inmediatamente. Toda operación intensiva debe delegarse a workers o a PostgreSQL.
- **Reconexión:** Implementar timeout de detección de flujo (3 segundos sin paquetes) antes de conmutar a la fuente de backup.
