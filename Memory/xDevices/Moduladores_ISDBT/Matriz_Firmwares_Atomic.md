# Memoria Técnica: Control Atómico de Moduladores ISDB-T

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xDevices` |
| **Componente:** | `Moduladores_ISDBT` |
| **Funcionalidad:** | Arquitectura de Control Atómico y Matriz de Modelos/Firmware |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
Control automatizado headless (Playwright) de moduladores broadcast ISDB-T (`ISDBT-06`, `ISDBT-16`, `ISDBT-16-ENC`, `NDS3306I`, `NDS3536S`, `ECD2416`, `PM24T`).

---

## 2. Decisiones Técnicas y Arquitectura
- **Principio de Atomicidad:** 1 acción física o lógica = 1 función irreductible. Macrofunciones construidas exclusivamente componiendo funciones atómicas.
- **Diferenciación por Firmware:** Equipos con distinta versión de firmware se consideran modelos distintos y poseen driver separado en `src/controllers/dvbdevices/<MODELO (VERSION)>/`.
- **Canal de Pruebas:** Las validaciones de configuración deben realizarse en el último TS del equipo (ej. TS16) para no afectar la emisión en vivo.
- **Prohibición de Credenciales Hardcodeadas:** Falla inmediata con `process.exit(1)` si faltan variables de entorno.

---

## 3. Skill de Referencia
- Consultar skill: [`isdbt-modulators`](file:///C:/Users/nmoss/.gemini/config/skills/isdbt-modulators/SKILL.md)
