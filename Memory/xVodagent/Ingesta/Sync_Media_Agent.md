# Nota Técnica: Ingesta y Transferencia Multimedia con xVodagent

- **Fecha:** 2026-10-13
- **Componente:** `xVodagent` / `xBackend`
- **Estado:** Activo / En ampliación (Upgrade FAST Channels)

---

## 1. Contexto y Objetivos
`xVodagent` es el daemon de ingesta distribuido que se instala en máquinas de operadores/encoders para detectar archivos de video `.mp4`, comprobar su estabilidad (evitar subir renders parciales), calcular `xxHash64` para deduplicación centralizada y transferirlos de forma segura hacia el servidor de ingesta `filesin` (`/mnt/media/storage`).

---

## 2. Decisiones de Arquitectura
1. **Separación de Ingesta (`storage`) y Playout (`live3`):**
   - El agente deposita en `filesin:/mnt/media/storage/247/<channel_path>/`.
   - El pipeline en servidor normaliza con FFmpeg a perfiles OTT estandarizados (audio AAC estéreo, video H.264 constrained, moov atom al inicio `+faststart`).
   - Tras normalizar y registrar en la tabla `videos`, el archivo se mueve a `/mnt/media/live3/<channel_path>/` donde el nodo Syncthing `Pivot` lo sincroniza hacia los servidores distribuidos de playout (`xFast` / `xManager`).
2. **Estructura Local en Agente:**
   - Carpetas raíz por defecto: `/247`, `/movies`, `/series`.
   - Fase 1 enfocada exclusivamente en canales lineales 24/7 (`sourcetype = 'fast'`).
3. **Mapeo Relacional N-a-N (`vodagents >--< channels`):**
   - Vinculación explícita entre un agente y uno o más canales FAST.
   - Restricción estricta: si un archivo cae en una carpeta que no coincide con un canal asignado al agente, no se transfiere (`unmapped`).
4. **Normalización Física Local en Disco (Opción A):**
   - `xVodagent` renombra en el disco del operador cualquier archivo con diacríticos, acentos o espacios antes del cálculo de hash y subida (`capitulo_1_el_auto_fantastico_1982.mp4`).
5. **Indexación Automática en Base de Datos de Playout:**
   - Al terminar el stream copy en `filesin`, se extraen metadatos técnicos con `ffprobe` y se da de alta automáticamente en `xBackend` (`fastvideoregister.sql`) en las tablas `videos` y `channelvideos`.
6. **Guardrails de Ingesta y Eficiencia (REQ-2026-007):**
   - Prohibición estricta de subcarpetas en `247/<canal>/`.
   - Validación métrica con `ffprobe` de bitrates (1080p: 2.5-4.5 Mbps, 720p: 1.8-3.0 Mbps, SD: <= 1.8 Mbps; 4K bloqueado) reteniendo archivos no conformes en `paused_efficiency`.
   - Telemetría periódica hacia `xBackend` con snapshot de estado para monitor remoto en `cpanel-admin`.
7. **Estación de Transcoding Local Asistido y Sugerencia IA (REQ-2026-008):**
   - Explorador de archivos directo en disco (sin carga HTTP).
   - Reproductor web HTML5 de audio y video con streaming FFmpeg en vivo (MP4 fragmentado).
   - Sugerencia de perfiles con Gemini 2.0 Flash Lite en `xBackend` (`/api/v1/sync-agent/transcode-suggestion`) con fallback local determinista.
   - Cola FIFO mono-hilo con depósito atómico en la carpeta del canal `247/<canal>/`.
