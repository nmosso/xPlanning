# Memoria Técnica: Enrutamiento de Borde, CDN y Proxy Inverso con xProxy

| Metadato | Detalle |
| :--- | :--- |
| **Proyecto:** | `xManager` |
| **Componente:** | `xProxy` |
| **Funcionalidad:** | Proxy Inverso de Borde, Seguridad y Balanceo para Nodos xManager |
| **Fecha:** | 2026-10-03 |
| **Estado:** | Activo |

---

## 1. Resumen y Contexto
`xproxy` opera como el proxy inverso entre el frontend del panel de control de la plataforma (un solo panel unificado) y cada una de las instancias de **`xManager`** (correspondientes a N operadores de TV). Su propósito es permitir que un único panel administre múltiples operadores independientes de forma segura y centralizada.

---

## 2. Decisiones Técnicas y Arquitectura
- **Pertenencia Arquitectónica:** `xproxy` es el proxy de acceso para el ecosistema `xManager` (1 Panel -> N Operadores de TV). NO interviene en `xBackend` (el cual conecta directo con `cpanel-admin`).
- **Enrutamiento Multitenant:** Enruta las peticiones HTTP y WebSockets hacia el `xManager` correspondiente al operador seleccionado en el panel.
- **Streaming por Pipes Asíncronos:** Empleo de `http-proxy-middleware` con streams asíncronos para tráfico continuo.
- **Archivos Clave Involucrados:**
  - `src/xproxy.ts`
  - `package.json`

---

## 3. Contratos de Datos y Seguridad
- Validación de IP con `ipaddr.js`.
- Verificación de cabeceras de autorización y tokens JWT antes de realizar proxy al socket de `xManager`.

---

## 4. Notas Operativas y Lecciones Aprendidas (Edge Cases)
- **Timeouts de Upstream:** Si el nodo `xManager` receptor demora más de 5000ms en responder el handshake, `xproxy` corta la conexión y entrega un error limpio 504 Gateway Timeout.
- **Compilación de Verificación (L2):** Ejecutar `npx tsc --noEmit` y `npm run build` para asegurar que las copias de assets y la transpilación TypeScript sean exitosas.
