# Bitácoras Individuales de Tenants (`xBackend`)

Este directorio almacena de forma **dinámica** el archivo `MEMORY.md` individual de cada tenant operado en el ecosistema **xIPTV** (`xBackend`, `xWorker`, `Planning-X`).

Su objetivo es mantener una bitácora viva y técnica por cabecera/cliente con su hardware (**moduladores ISDB-T**, **IRD**), **cantidad de canales**, **características individuales** (desvíos respecto al estándar) y **problemas conocidos / historial de incidentes**.

---

## 1. Regla Fundamental: Creación Dinámica (NO Pre-crear)

- **PROHIBIDO** pre-generar archivos `MEMORY.md` para todos los tenants de `public.tenants` de forma preventiva.
- Cada archivo se crea **únicamente cuando se opera sobre ese tenant** — ya sea en una **operación individual** o dentro de una **operación grupal / por lote**.
- **Ruta canónica:**
  ```text
  Planning-X/Memory/xBackend/Tenants/<tenantid>/MEMORY.md
  ```
  - `<tenantid>` debe coincidir en minúsculas con `public.tenants.tenantid` en `postgres-xserver` (ej. `server83/MEMORY.md`).
  - Si el operador indica un número de posición (`server 83`, `pos 83`), normalizar a `server83` verificando contra `public.tenants`.
- **Plantilla obligatoria:** [`TEMPLATE_TENANT_MEMORY.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/TEMPLATE_TENANT_MEMORY.md)

---

## 2. Protocolo de Operación (Individual y Grupal)

### 2.1. Antes de Operar (Lectura Obligatoria)
1. Verificar si existe `Tenants/<tenantid>/MEMORY.md` para cada tenant objetivo.
2. Si existe, **leerlo antes de intervenir** para tener presentes sus moduladores, IRD, particularidades de red/hardware y problemas recurrentes previos.

### 2.2. Durante / Al Finalizar una Operación Individual (1 tenant)
1. Si `Tenants/<tenantid>/MEMORY.md` **no existe**, crearlo dinámicamente usando [`TEMPLATE_TENANT_MEMORY.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/TEMPLATE_TENANT_MEMORY.md) con los datos efectivamente relevados durante la sesión.
2. Si **ya existe**, actualizar las secciones que hayan cambiado (modulador, IRD, cantidad de canales, particularidades o problemas) y agregar una entrada en **6.2. Historial de Intervenciones** marcada como `[Individual]`.
3. Actualizar la **Tabla Índice Dinámica** al final de este archivo.

### 2.3. Durante / Al Finalizar una Operación Grupal / Por Lote (N tenants)
1. Por **cada tenant efectivamente intervenido** en el lote:
   - Crear dinámicamente su `Tenants/<tenantid>/MEMORY.md` si no existía, o actualizar el existente.
   - Registrar en **6.2. Historial de Intervenciones** la acción ejecutada marcada como `[Grupal]`, junto con el resultado específico de ese tenant (OK, desvío detectado, versión, error o timeout).
   - Si se relevaron datos de hardware (modulador, IRD), canales o particularidades durante el lote, actualizar las secciones correspondientes.
   - Si un tenant falló o estuvo inalcanzable, registrar el problema en **6.1** o **6.2** (no omitirlo).
2. Actualizar la **Tabla Índice Dinámica** al final de este archivo con todos los tenants creados o actualizados.

---

## 3. Reglas de Calidad y Seguridad

1. **Cero Datos Inventados:** Si una operación no relevó algún dato (ej. firmware del modulador o cantidad de canales), dejar explícitamente `Sin relevar`.
2. **Sin Secretos en Texto Plano:** Prohibido guardar contraseñas, claves SSH, API keys o tokens (`tsdb_token`, `syncthing_apikey`, credenciales de IRD/modulador). Indicar únicamente estado o referencia (ej. `configurado en tenants.options`).
3. **Límite de Longitud:** Máximo **200 líneas** por archivo `MEMORY.md` de tenant. Consolidar hallazgos permanentes en las secciones 2–5 y podar historial antiguo. Los logs crudos de consola pertenecen a `xWorker/logs/`.
4. **Nomenclatura Neutral (No-Astra):** Prohibido usar "Astra" o "Astraserver"; emplear "Servidor de Transmisión", "Encoder" o "Servidor de Streaming".
5. **Estados de Problemas (Política de Completitud):** Todo problema registrado en la sección 6.1 debe mantener un estado claro (`ABIERTO`, `EN SEGUIMIENTO`, `RESUELTO`, `SUSTITUIDO` o `ABANDONADO`).

---

## 4. Índice Dinámico de Tenants con Bitácora

*Esta tabla inicia vacía y se poblará automáticamente a medida que se operen tenants de forma individual o grupal.*

| Tenant ID | Posición | Nombre | Modulador(es) | IRD | Canales | Problemas Abiertos | Última Act. | Bitácora |
| :--- | :---: | :--- | :--- | :--- | :---: | :---: | :---: | :--- |
| `server109` | `109` | Bolivar | `IPM24C` (16 MUX) | `ECD2416` | `Sin relevar` | 0 | `2026-11-24` | [`server109/MEMORY.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/server109/MEMORY.md) |
| `server111` | `111` | Lujan de Cuyo | `IMP24C-VIRTUAL` (20 MUX) | `IRD-16` | `Sin relevar` | 1 | `2026-11-24` | [`server111/MEMORY.md`](file:///d:/Projects/DEV/xIPTV/Planning-X/Memory/xBackend/Tenants/server111/MEMORY.md) |
