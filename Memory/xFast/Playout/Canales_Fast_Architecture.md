# Memoria Técnica: Arquitectura de Canales FAST y Playout

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xFast` |
| **Componente:** | `Playout` |
| **Funcionalidad:** | Generación Dinámica de Canales FAST en Tiempo Real |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xFast` proporciona playout programado y dinámico para canales lineales FAST (Free Ad-supported Streaming TV) combinando contenidos VOD pregrabados con inserción de marcadores publicitarios SCTE-35.

---

## 2. Decisiones Técnicas y Arquitectura
- **Decisión:** Desempaquetado delgado en `xfast.ts` y persistencia en procedimientos SQL, acoplando pipelines de transcodificación acelerada con FFmpeg.
- **Alternativa Descartada:** Renderizado continuo de video sin aceleración. Descartado por requerir servidores de GPU sobredimensionados.
