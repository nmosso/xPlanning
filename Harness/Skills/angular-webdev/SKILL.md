---
name: angular-webdev
description: >-
  Estándares, buenas prácticas y reglas mandatarias de diseño UI para aplicaciones frontend
  Angular en el ecosistema xIPTV (cpanel-admin, cpanel-client).
---

# Angular & Frontend Web Development Skill

Este skill establece las reglas de arquitectura, patrones reactivos y verificación técnica para el desarrollo y mantenimiento de los paneles de control web (`cpanel-admin`, `cpanel-client`) construidos con Angular.

---

## 1. Regla de Oro de Branding UI: Prohibición de "Astra"

> [!CAUTION]
> **REGLA ABSOLUTA:** Queda terminantemente prohibido utilizar los términos **"Astra"** o **"Astraserver"** en cualquier componente visual, texto, título, placeholder, botón, modal o mensaje de error de cara al usuario.
> - **Términos Permitidos y Neutros:**
>   - *"Servidor"*
>   - *"Servidor de Transmisión"*
>   - *"Encoder"*
>   - *"Servidor de Streaming"*

---

## 2. Arquitectura de Componentes y Estado Reactivo

1. **Gestión de Memoria y Desuscripción:**
   - Prevenir fugas de memoria (*memory leaks*) en componentes con suscripciones a RxJS.
   - Utilizar el operador `takeUntilDestroyed()` en el contexto del constructor o un `Subject` de destrucción en `ngOnDestroy`.
   - Preferir el pipe `| async` en plantillas HTML cuando sea posible.

2. **Manejo de Errores en Servicios HTTP:**
   - Todo servicio HTTP debe encapsular llamadas con `catchError`.
   - No dejar promesas o observables colgados que provoquen que los indicadores de carga (*spinners*) queden visibles indefinidamente ante un error 500 o timeout de red.

3. **Comunicación con el Backend:**
   - Recordar que `cpanel-admin` conecta **directamente** con `xBackend` sin intermediarios ni `xProxy`.

---

## 3. Protocolo de Verificación Obligatorio (L2 QA)

Antes de concluir cualquier cambio en el frontend:
```bash
# 1. Compilación de producción (Angular AOT y optimización)
npm run build

# 2. Verificación de tipos TypeScript estricta
npx tsc --noEmit
```
*El build debe completarse con código 0 y sin errores de tipado o plantillas.*
