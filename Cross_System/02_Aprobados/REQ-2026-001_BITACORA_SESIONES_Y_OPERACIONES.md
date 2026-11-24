# Documento de Especificación Técnica: Bitácora de Sesiones y Auditoría de Operaciones CRUD

| Metadato | Detalle |
| :--- | :--- |
| **ID Requerimiento:** | `REQ-2026-001` |
| **Sistema Destino:** | `Cross_System` (`xBackend` + `xManager`) |
| **Estado:** | `02_Aprobado` |
| **Fecha de Creación:** | 2026-09-22 |
| **Última Actualización:** | 2026-09-22 |
| **Autor(es):** | Antigravity AI / Arquitectura Técnica de Sistemas |

---

## 1. Resumen Ejecutivo y Justificación de Negocio

El ecosistema xIPTV carece de trazabilidad sobre qué operadores ingresan a la plataforma, mediante qué mecanismo (usuario/contraseña o delegación SSO) y qué alteraciones realizan sobre las entidades críticas de la red (canales, configuraciones de streaming, encoders y hardware).

Este requerimiento formaliza una solución de auditoría distribuida y coordinada:
1. **Control Central de Login y Emisión de Sesión (`xBackend`):** Emisión de un identificador de sesión unívoco (`session_key` UUID v4), registro de auditoría de intentos de login (éxitos y motivos exactos de fallo) y persistencia del ciclo de vida de la sesión en `xServerSQL`. Soporta revocación por logout explícito y expiración por tiempo (TTL).
2. **Propagación y Réplica Local (`xManager`):** Recepción y almacenamiento de la copia del `session_key` en los nodos de streaming `xManager` cuando el operador ingresa vía SSO o autenticación directa.
3. **Auditoría de Mutaciones CRUD con Diferencial Antes/Después (`xManager`):** Trazabilidad de operaciones de mutación (`CREATE`, `UPDATE`, `DELETE`) en las funciones PL/pgSQL de `xClientSQL`, vinculando cada cambio al `session_key` e identidad responsable. En modificaciones (`UPDATE`), se almacena el diferencial estructurado (*antes* y *después*). Las operaciones de lectura (`GET` / consultas) quedan expresamente excluidas para preservar el rendimiento.

---

## 2. Filtro Doctrinal y Cuestionamiento Inicial (Filtro Británico)

- **¿Cuál es la necesidad real identificada?**
  Garantizar trazabilidad de acceso (quién, cuándo, desde qué IP, método de login y causa de rechazo en caso de fallo) y auditoría detallada de cambios operativos en los nodos de red para identificar errores de configuración o acciones no autorizadas.
- **¿Qué supuestos iniciales fueron descartados durante la discusión?**
  - **Descarte de auditoría de consultas (`GET` / `SELECT`):** Generaría una sobrecarga inútil de I/O en PostgreSQL. Se auditan exclusivamente mutaciones CRUD (*YAGNI*).
  - **Descarte de microservicios externos de logging:** Se aprovecha el motor transaccional de PostgreSQL existente (`xServerSQL` y `xClientSQL`).
  - **Descarte de purga automática de logs:** Dado que las mutaciones operativas son de baja frecuencia una vez inicializada la plataforma, los registros se conservan de forma permanente sin rotación destructiva.
- **¿Por qué esta solución es la más directa y no agrega complejidad accidental?**
  Empaqueta el `session_key` dentro del JWT estándar emitido por `xBackend`. Los paneles `cpanel-admin` y `cpanel-client` ya envían `Authorization: Bearer <token>` a través de sus interceptores HTTP nativos.
- **¿Se eliminaron componentes superfluos?**
  Sí. Se reutiliza la firma estándar `fn(_data jsonb, _params jsonb, _operator jsonb)` para propagar `_operator.session_key` hacia los procedimientos SQL sin crear canales de comunicación paralelos.

---

## 3. Alcance y Límites del Requerimiento

### 3.1. En Alcance (In Scope)

#### Fase 1: Bitácora de Login y Sesiones en `xBackend`
- [ ] Generación de `session_key` (UUID v4) criptográfico en cada login exitoso (`POST /users/login`) y emisión de SSO (`GET /v3/tenants/:tenantid/sso`).
- [ ] Tabla `auth_login_log` en `xServerSQL` para registrar todos los intentos con causas tipificadas de fallo:
  - `INVALID_PASSWORD`: Contraseña incorrecta.
  - `USER_NOT_FOUND`: Email no existe.
  - `USER_DISABLED`: Cuenta bloqueada/inactiva.
  - `CAPTCHA_FAILED`: Fallo o token ausente de Cloudflare Turnstile / reCAPTCHA.
  - `DOMAIN_FORBIDDEN`: Violación de restricción de dominio (ej. `cadmin.xisrv.xyz` sin rol `administrator`).
  - `NO_TENANT_ASSIGNED`: Usuario sin accesos asignados a tenants activos.
  - `SSO_EXPIRED` / `SSO_INVALID`: Token SSO corrupto o expirado.
  - `BACKEND_OFFLINE` / `DATABASE_ERROR`: Fallo de conexión hacia PostgreSQL central.
- [ ] Mecanismo de contingencia en Express: Si PostgreSQL central no responde (*Backend Offline*), el fallo se escribe en un archivo local estructurado JSON (`./logs/auth_failures.log`).
- [ ] Tabla `auth_sessions` en `xServerSQL` para registrar sesiones vivas con IP de origen, dominio, agente, método (`credentials` vs `sso`) y operador generador.
- [ ] Inclusión de `session_key` en el payload JWT y respuesta JSON al cliente.
- [ ] Endpoint `POST /users/logout` y procedimiento `auth_close_session` para revocar explícitamente el `session_key` al cerrar sesión.

#### Fase 2: Replicación de `session-key` en `xManager`
- [ ] Tabla `auth_sessions_cache` en `xClientSQL` para registrar las sesiones reconocidas en el nodo local.
- [ ] Al recibir un acceso SSO (`GET /v3/login?token=...`), `xManager` valida la firma, extrae el `session_key` originado en `xBackend` y lo persiste en `auth_sessions_cache`.
- [ ] Middleware en `xManager` que intercepta `Authorization: Bearer <token>`, valida el JWT y enriquece `_operator` con `{ session_key, identityid, email, role, ip }`.

#### Fase 3: Auditoría CRUD con Diferencial en `xManager`
- [ ] Tabla `audit_operations_log` en `xClientSQL`.
- [ ] Procedimiento auxiliar PL/pgSQL `audit_log_operation(_data, _params, _operator)`.
- [ ] Captura de diferencial (*before* y *after*) en operaciones `UPDATE`:
  - Se consulta el estado actual antes de mutar la fila y se almacena junto al nuevo estado resultante.
- [ ] Intervención de funciones de mutación clave en `xClientSQL` (`newchannel`, `setchanneldata`, `channeldelete`, `setchannelstatus`).

### 3.2. Fuera de Alcance (Out of Scope - *YAGNI*)
- Registro de operaciones `GET` / consultas de lectura.
- Políticas de purga automática periódica de logs.
- Modificación del motor multimedia de bajo nivel (FFmpeg, modulación ISDB-T).

---

## 4. Arquitectura y Flujo de Comunicación

```
  ┌────────────────────────────────────────────────────────┐
  │                 cpanel-admin (Angular)                 │
  └───────────────────────────┬────────────────────────────┘
                              │ 1. POST /users/login o GET /sso
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │                        xBackend                        │
  │   - Genera session_key (UUID v4)                       │
  │   - Evalúa Captcha / Credenciales                      │
  │   - Registra auth_login_log (Éxito o causa de fallo)   │
  │   - Contingencia a log local si DB está Offline        │
  │   - Registra auth_sessions en PostgreSQL Central       │
  │   - Emite JWT con payload: { session_key, yid, ... }   │
  └─────────────┬───────────────────────────┬──────────────┘
                │                           │
                │ RPC SQL                   │ 2. SSO Token Redirect
                ▼                           │    (con session_key)
  ┌───────────────────────────┐             ▼
  │    xServerSQL (Central)   │   ┌────────────────────────────────┐
  │   - auth_login_log        │   │     cpanel-client (Angular)    │
  │   - auth_sessions         │   └──────────────┬─────────────────┘
  └───────────────────────────┘                  │
                                                 │ 3. Peticiones CRUD con
                                                 │    Authorization: Bearer <token>
                                                 ▼
                                  ┌────────────────────────────────┐
                                  │            xManager            │
                                  │   - Middleware extrae JWT      │
                                  │   - Inyecta session_key en     │
                                  │     _operator                  │
                                  └──────────────┬─────────────────┘
                                                 │
                                                 │ RPC SQL (_operator)
                                                 ▼
                                  ┌────────────────────────────────┐
                                  │      xClientSQL (Local)        │
                                  │   - auth_sessions_cache        │
                                  │   - audit_operations_log       │
                                  │     (con diff before/after)    │
                                  │   - Funciones CRUD intervenidas│
                                  └────────────────────────────────┘
```

---

## 5. Contratos de Datos y Esquema DDL (Filtro Alemán)

### 5.1. Esquema DDL en `xServerSQL` (Central)

```sql
-- 1. Bitácora de intentos de login
CREATE TABLE IF NOT EXISTS auth_login_log (
    log_id              BIGSERIAL PRIMARY KEY,
    session_key         UUID NULL,
    email_attempted     VARCHAR(150) NOT NULL,
    identity_id         VARCHAR(100) NULL,
    tenant_id           VARCHAR(100) NULL,
    auth_method         VARCHAR(20) NOT NULL, -- 'credentials' | 'sso'
    sso_operator_id     VARCHAR(100) NULL,
    sso_operator_email  VARCHAR(150) NULL,
    ip_origin           VARCHAR(45) NOT NULL,
    origin_domain       VARCHAR(150) NOT NULL,
    user_agent          TEXT NULL,
    is_success          BOOLEAN NOT NULL,
    failure_reason      VARCHAR(50) NULL,     -- 'INVALID_PASSWORD', 'USER_NOT_FOUND', 'USER_DISABLED', 'CAPTCHA_FAILED', 'DOMAIN_FORBIDDEN', 'NO_TENANT', 'DATABASE_ERROR'
    failure_detail      TEXT NULL,
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_auth_login_log_email ON auth_login_log(email_attempted);
CREATE INDEX IF NOT EXISTS idx_auth_login_log_ip ON auth_login_log(ip_origin);
CREATE INDEX IF NOT EXISTS idx_auth_login_log_session ON auth_login_log(session_key) WHERE session_key IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_auth_login_log_created ON auth_login_log(created_at DESC);

-- 2. Sesiones activas y ciclo de vida
CREATE TABLE IF NOT EXISTS auth_sessions (
    session_key         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    identity_id         VARCHAR(100) NOT NULL,
    tenant_id           VARCHAR(100) NULL,
    auth_method         VARCHAR(20) NOT NULL, -- 'credentials' | 'sso'
    sso_operator_id     VARCHAR(100) NULL,
    sso_operator_email  VARCHAR(150) NULL,
    ip_origin           VARCHAR(45) NOT NULL,
    origin_domain       VARCHAR(150) NOT NULL,
    user_agent          TEXT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active', -- 'active' | 'closed' | 'expired' | 'revoked'
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    expires_at          TIMESTAMP WITH TIME ZONE NOT NULL,
    closed_at           TIMESTAMP WITH TIME ZONE NULL,
    last_activity       TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_auth_sessions_identity ON auth_sessions(identity_id);
CREATE INDEX IF NOT EXISTS idx_auth_sessions_status ON auth_sessions(status);
```

### 5.2. Esquema DDL en `xClientSQL` (Local en cada xManager)

```sql
-- 1. Caché de sesiones activas en el nodo local
CREATE TABLE IF NOT EXISTS auth_sessions_cache (
    session_key         UUID PRIMARY KEY,
    identity_id         VARCHAR(100) NOT NULL,
    tenant_id           VARCHAR(100) NULL,
    auth_method         VARCHAR(20) NOT NULL,
    sso_operator_id     VARCHAR(100) NULL,
    sso_operator_email  VARCHAR(150) NULL,
    ip_origin           VARCHAR(45) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    expires_at          TIMESTAMP WITH TIME ZONE NOT NULL
);

-- 2. Bitácora de operaciones CRUD con diferencial before/after
CREATE TABLE IF NOT EXISTS audit_operations_log (
    operation_id        BIGSERIAL PRIMARY KEY,
    session_key         UUID NULL,
    identity_id         VARCHAR(100) NOT NULL,
    operator_email      VARCHAR(150) NULL,
    module              VARCHAR(50) NOT NULL,  -- 'channels', 'devices', 'encoders', 'categories', etc.
    action              VARCHAR(20) NOT NULL,  -- 'CREATE', 'UPDATE', 'DELETE'
    entity_id           VARCHAR(100) NULL,     -- ID de la entidad mutada (ej. channelid)
    ip_origin           VARCHAR(45) NULL,
    before_data         JSONB NULL,            -- Estado previo de la entidad (en UPDATE o DELETE)
    after_data          JSONB NULL,            -- Estado nuevo resultante (en CREATE o UPDATE)
    payload_input       JSONB NULL,            -- Payload crudo recibido en _data
    result_status       VARCHAR(20) NOT NULL DEFAULT 'success', -- 'success' | 'failed'
    error_message       TEXT NULL,
    created_at          TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_audit_ops_session ON audit_operations_log(session_key);
CREATE INDEX IF NOT EXISTS idx_audit_ops_identity ON audit_operations_log(identity_id);
CREATE INDEX IF NOT EXISTS idx_audit_ops_module ON audit_operations_log(module, action);
CREATE INDEX IF NOT EXISTS idx_audit_ops_entity ON audit_operations_log(entity_id);
CREATE INDEX IF NOT EXISTS idx_audit_ops_created ON audit_operations_log(created_at DESC);
```

### 5.3. Procedimientos Almacenados en PostgreSQL

#### xServerSQL: Registro de Intento de Login (`auth_log_login_attempt`)
```sql
CREATE OR REPLACE FUNCTION auth_log_login_attempt(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_session_key UUID;
    v_is_success  BOOLEAN;
BEGIN
    v_is_success := coalesce((_data->>'is_success')::boolean, false);
    IF v_is_success AND (_data->>'session_key') IS NOT NULL THEN
        v_session_key := (_data->>'session_key')::uuid;
    END IF;

    INSERT INTO auth_login_log (
        session_key,
        email_attempted,
        identity_id,
        tenant_id,
        auth_method,
        sso_operator_id,
        sso_operator_email,
        ip_origin,
        origin_domain,
        user_agent,
        is_success,
        failure_reason,
        failure_detail
    ) VALUES (
        v_session_key,
        lower(trim(coalesce(_data->>'email', 'unknown'))),
        _data->>'identity_id',
        _data->>'tenant_id',
        coalesce(_data->>'auth_method', 'credentials'),
        _data->>'sso_operator_id',
        _data->>'sso_operator_email',
        coalesce(_data->>'ip', _operator->>'ip', 'unknown'),
        coalesce(_data->>'origin_domain', _operator->>'domain', 'unknown'),
        _data->>'user_agent',
        v_is_success,
        _data->>'failure_reason',
        _data->>'failure_detail'
    );

    RETURN jsonb_build_object('status', 'success');
EXCEPTION WHEN OTHERS THEN
    RETURN jsonb_build_object('status', 'error', 'message', SQLERRM);
END;
$$;
```

#### xServerSQL: Registro y Cierre de Sesión (`auth_create_session` y `auth_close_session`)
```sql
CREATE OR REPLACE FUNCTION auth_create_session(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_session_key UUID;
    v_exp_timestamp TIMESTAMP WITH TIME ZONE;
BEGIN
    v_session_key := (_data->>'session_key')::uuid;
    v_exp_timestamp := to_timestamp((_data->>'exp')::numeric);

    INSERT INTO auth_sessions (
        session_key,
        identity_id,
        tenant_id,
        auth_method,
        sso_operator_id,
        sso_operator_email,
        ip_origin,
        origin_domain,
        user_agent,
        status,
        expires_at
    ) VALUES (
        v_session_key,
        _data->>'identity_id',
        _data->>'tenant_id',
        coalesce(_data->>'auth_method', 'credentials'),
        _data->>'sso_operator_id',
        _data->>'sso_operator_email',
        coalesce(_data->>'ip', _operator->>'ip', 'unknown'),
        coalesce(_data->>'origin_domain', _operator->>'domain', 'unknown'),
        _data->>'user_agent',
        'active',
        v_exp_timestamp
    );

    RETURN jsonb_build_object('status', 'success', 'session_key', v_session_key);
END;
$$;

CREATE OR REPLACE FUNCTION auth_close_session(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_session_key UUID;
BEGIN
    v_session_key := coalesce((_data->>'session_key')::uuid, (_operator->>'session_key')::uuid);

    UPDATE auth_sessions
    SET status = 'closed',
        closed_at = CURRENT_TIMESTAMP
    WHERE session_key = v_session_key;

    RETURN jsonb_build_object('status', 'success', 'session_key', v_session_key);
END;
$$;
```

#### xClientSQL: Registro de Auditoría con Diferencial (`audit_log_operation`)
```sql
CREATE OR REPLACE FUNCTION audit_log_operation(
    _data jsonb,
    _params jsonb,
    _operator jsonb
) RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
    v_session_key UUID;
BEGIN
    IF (_operator->>'session_key') IS NOT NULL AND (_operator->>'session_key') != '' THEN
        v_session_key := (_operator->>'session_key')::uuid;
    END IF;

    INSERT INTO audit_operations_log (
        session_key,
        identity_id,
        operator_email,
        module,
        action,
        entity_id,
        ip_origin,
        before_data,
        after_data,
        payload_input,
        result_status,
        error_message
    ) VALUES (
        v_session_key,
        coalesce(_operator->>'identityid', _operator->>'yid', 'system'),
        _operator->>'email',
        coalesce(_data->>'module', 'general'),
        coalesce(_data->>'action', 'UPDATE'),
        _data->>'entity_id',
        coalesce(_operator->>'ip', 'unknown'),
        _data->'before_data',
        _data->'after_data',
        coalesce(_data->'payload_input', '{}'::jsonb),
        coalesce(_data->>'status', 'success'),
        _data->>'error_message'
    );

    RETURN jsonb_build_object('status', 'success');
EXCEPTION WHEN OTHERS THEN
    -- Resiliencia Rusa: Nunca abortar la transacción de negocio por fallo en el log
    RETURN jsonb_build_object('status', 'error', 'message', SQLERRM);
END;
$$;
```

---

## 6. Especificación de Endpoints y Middleware

### 6.1. xBackend: `POST /users/login` (Ajuste)
1. Extrae IP (`req.ip`, cabecera `x-forwarded-for`) y User-Agent.
2. Verifica Captcha. Si falla: invoca `auth_log_login_attempt` con `is_success=false`, `failure_reason='CAPTCHA_FAILED'`.
3. Invoca `identitieslogin` en `xServerSQL`.
   - **Contingencia Backend Offline:** Si la llamada a base de datos rechaza con error de red/conexión, escribe el evento en `./logs/auth_failures.log` con formato JSON y responde HTTP 500 con código `BACKEND_OFFLINE`.
   - **Credenciales inválidas:** Invoca `auth_log_login_attempt` con `is_success=false`, `failure_reason='INVALID_PASSWORD'`.
   - **Éxito:**
     - Genera `session_key = uuidv4()`.
     - Registra la sesión activa mediante `auth_create_session`.
     - Registra el intento en `auth_login_log` con `is_success=true`.
     - Inyecta `session_key` en el payload JWT y en la respuesta JSON.

### 6.2. xBackend: `POST /users/logout` (Nuevo Endpoint)
- Invocado al presionar "Cerrar Sesión" en la interfaz.
- Invoca `auth_close_session` pasando el `session_key` para marcar la sesión como `closed` y registrar `closed_at = CURRENT_TIMESTAMP`.

### 6.3. xBackend: `GET /v3/tenants/:tenantid/sso` (Ajuste)
- Identifica al operador generador (`_operator.identityid` / `_operator.email`).
- Genera un `session_key` exclusivo para la sesión delegada hacia el tenant.
- Registra la sesión en `auth_sessions` con `auth_method='sso'`, `sso_operator_id` y `sso_operator_email`.
- Registra el evento en `auth_login_log`.
- Retorna el token SSO firmado que incluye el `session_key`.

### 6.4. xManager: Middleware y Recepción de Sesión
- En `GET /v3/login` (SSO): Valida el token, extrae el `session_key` y lo persiste en `auth_sessions_cache`.
- En `credentials.ts`: Para cualquier petición protegida, extrae el Bearer token, valida/decodifica el payload y propaga `{ session_key, identityid, email, role, ip }` a `req.body._auth` y `_operator`.

---

## 7. Requerimientos de Frontend / UI

- **Directiva No-Astra:**
  Queda prohibido el uso de términos como "Astra" o "Astraserver" en cualquier mensaje, pantalla, modal o tabla de bitácora. Se usarán términos neutros: "Servidor de Streaming", "Encoder", "Dispositivo".
- **Transparencia en Frontend:**
  `cpanel-admin` y `cpanel-client` ya gestionan la cabecera `Authorization: Bearer <token>` mediante sus interceptores. Al cerrar sesión, ambos paneles consumirán el endpoint `POST /users/logout` antes de limpiar el `sessionStorage`.

---

## 8. Escenario de Pruebas y Protocolo de Verificación (Filtro Ruso)

```
┌─────────────────────────────────────────────────────────────┐
│ NIVEL 1 (L1) - Pruebas Unitarias y Casos Borde (Edge Cases) │
│ • Contraseña errónea -> Registro con INVALID_PASSWORD.      │
│ • Usuario inexistente -> Registro con USER_NOT_FOUND.       │
│ • Captcha vacío/expirado -> Registro con CAPTCHA_FAILED.    │
│ • Dominio cadmin sin rol administrator -> DOMAIN_FORBIDDEN. │
│ • Cierre de sesión -> Sesión pasa a status 'closed' en DB.  │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 2 (L2) - Pruebas de Integración y Contratos de Datos  │
│ • Compilación TypeScript sin errores: npx tsc --noEmit.     │
│ • Verificación de inserción en auth_login_log y sessions.   │
│ • Flujo SSO completo: Generación en xBackend -> Consumo en │
│   xManager -> Registro del session_key en ambos extremos.    │
│ • Modificación de canal en xManager -> Registro del diff    │
│   (before_data vs after_data) en audit_operations_log.      │
├─────────────────────────────────────────────────────────────┤
│ NIVEL 3 (L3) - Resiliencia y Estrés                         │
│ • Simulación Backend Offline: Caída de PostgreSQL central.  │
│   Verificar captura en Express y escritura en log local     │
│   auth_failures.log.                                        │
│ • Simulación de falla en audit_log_operation no interrumpe  │
│   la actualización real del canal en xClientSQL.            │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Criterios de Aceptación (Definition of Done - DoD)

- [ ] Tablas `auth_login_log` y `auth_sessions` creadas e indexadas en `xServerSQL`.
- [ ] Procedimientos `auth_log_login_attempt`, `auth_create_session` y `auth_close_session` desplegados.
- [ ] `AuthApi.login` en `xBackend` genera `session_key` (UUID v4) y diagnostica causas de fallo.
- [ ] Mecanismo de contingencia a log plano local ante caída de PostgreSQL central.
- [ ] Endpoint `POST /users/logout` operativo para invalidación explícita.
- [ ] Tablas `auth_sessions_cache` y `audit_operations_log` creadas en `xClientSQL`.
- [ ] `xManager` extrae y propaga el `session_key` en `_operator`.
- [ ] Funciones CRUD de canales registran el diferencial (*before* / *after*) en `audit_operations_log`.
- [ ] Compilación TypeScript limpia (`npx tsc --noEmit`) en `xBackend` y `xManager`.
- [ ] Cumplimiento total de la regla No-Astra.
