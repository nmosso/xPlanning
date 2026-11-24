# Memoria Técnica: Control, Instalación y Analítica del Canal Local en Streamer

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `Cross_System` (`xFast`, `xManager`, `cpanel-client`) |
| **Componente:** | `Playout / Streamer` |
| **Funcionalidad:** | Instalación, Control de Ciclo de Vida PM2 y Analítica On-Demand del Canal Local |
| **Fecha:** | 2026-11-11 |
| **Estado:** | Aprobado (`REQ-2026-010`) |

---

## 1. Resumen y Contexto
Los nodos de transmisión operan un canal local independiente ejecutado bajo PM2 (`localchannel`), correspondiente a un script bash en `/home/ubuntu/tv/localchannel.sh` que procesa RTMP y emite un stream multicast continuo en `udp://224.2.2.2:5000` (`pkt_size=1316&buffer_size=5000000&localaddr=192.168.3.4`).

---

## 2. Decisiones Técnicas y Arquitectura
- **Puente Ultraliviano (`xFast`):** Reutilización de `xFast` (`:8020`), ya configurado en `params` (`fastwebserver`), para evitar demonios adicionales o accesos SSH.
- **Aprovisionamiento Canónico:** `xFast` inyecta directamente el script en `/home/ubuntu/tv/localchannel.sh` con permisos `0755` y gestiona el ciclo PM2 (`delete` preventivo, `start` y `save`).
- **Analítica On-Demand (ffprobe):** Ejecución de `ffprobe` con timeout defensivo de 4 segundos contra `udp://224.2.2.2:5000` bajo demanda del operador, extrayendo resolución, bitrate, codecs y fps.
- **Frontend `cpanel-client`:** Nueva pestaña de primer nivel "Canal Local" (`/localchannel`) con soporte para estado no instalado, control operativo PM2 (reinicio con modal SweetAlert) y analítica de señal.

---

## Estado de Implementación (2026-11-11)
- Fases 1–3 implementadas (xFast, xManager, cpanel-client). `tsc --noEmit` sin errores en los tres; pruebas locales de autenticación, whitelist, cerrojo 429, ffprobe sin señal y errores 502/504.
- Clave compartida `FASTAPIKEY` en variables de entorno de xFast y xManager (no en `params`, porque `/v3/params` los expone).
- Migración renombrada a `0013_...`; conserva `movies`, `videos`, `monitoring`. **Pendiente: aplicarla en `xserverdb`.**
