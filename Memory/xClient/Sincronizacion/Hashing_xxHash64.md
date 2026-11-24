# Memoria Técnica: Deduplicación y Sincronización mediante xxHash64

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xClient` |
| **Componente:** | `Sincronizacion` |
| **Funcionalidad:** | Deduplicación Rápida de Medios MP4 y Transferencia Remota |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xClient` sincroniza archivos de video `.mp4` hacia almacenamiento central o servidores de streaming. Para evitar transferir archivos idénticos sin saturar memoria ni CPU, se implementó hashing rápido con xxHash64.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión:** Emplear `xxhash-wasm` o binding de streams con algoritmo xxHash64 (64 bits).
- **Alternativa Descartada:** SHA-256 o MD5. Descartados por lentitud y consumo de CPU extremo al procesar archivos de 10GB+ a 50GB+.
- **Rutas Remotas:** Normalización estricta a formato POSIX en minúsculas reemplazando espacios por guiones bajos.
