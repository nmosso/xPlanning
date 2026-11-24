# CONSTITUCIÓN: [Nombre del Proyecto / Módulo]

> **Regla de Formato:** Máximo 15 líneas. Principios cortos, no negociables y verificables.

1. **Stack:** Node.js 20+ con TypeScript en modo estricto (`strict: true`), PostgreSQL y Angular 16+.
2. **Arquitectura:** Thin Controllers en backend; lógica de datos pesada delegada a procedimientos PL/pgSQL en PostgreSQL.
3. **Contratos RPC:** Firma estándar `fn(_data jsonb, _params jsonb, _operator jsonb) returns jsonb` con helpers `jgk*`.
4. **Serialización Segura:** En llamadas `pgsql.exec`, serializar siempre arrays y objetos complejos con `JSON.stringify()`.
5. **UI Branding:** Prohibido el uso de "Astra" o "Astraserver" en vistas, modales o textos de cara al usuario.
6. **Red:** Comunicación directa entre `cpanel-admin` y `xBackend` sin intermediación de `xProxy`.
7. **Límite de Dependencias:** Cero bibliotecas externas superfluas; preferir módulos nativos de Node.js y PostgreSQL.
8. **Pruebas L1:** Validación obligatoria de casos borde (nulos, vacíos, longitudes, inyecciones de datos).
9. **Pruebas L2:** Compilación TypeScript sin errores (`npx tsc --noEmit`) y build de Angular exitoso (`npm run build`).
10. **Pruebas L3:** Timeouts defensivos en llamadas remotas y tolerancia a caídas temporales de nodos.
11. **Flujo SDD y Programación:** Ningún cambio de código sin spec aprobada en Planning-X. La programación e implementación de código es ejecutada obligatoriamente con **Claude Code**.
12. **Definición de Terminado:** Todo cambio debe estar testeado, tipado y reflejado en `MEMORY.md` (máx. 150 líneas).
