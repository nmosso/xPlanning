# GUÍA DE ARQUITECTURA Y FRONTERAS DE SISTEMAS

Este documento define las responsabilidades, fronteras tecnológicas y patrones de comunicación entre **xBackend** y **xManager** dentro del ecosistema xIPTV. Sirve para orientar la planificación y evitar acoplamientos innecesarios.

---

## 1. Mapa de Responsabilidades

| Dimensión | xBackend (Orquestador Central de Catálogo) | xManager (Backend Principal ISDB-T e IPTV) |
| :--- | :--- | :--- |
| **Rol Principal** | API central de administración global, catálogo maestro, TMDB y metadatos. | Backend principal de distribución por ISDB-T e IPTV para operadores de TV. |
| **Orquestación Canales Live** | Catálogo general de fuentes y contratos de datos. | Metadata, fuentes de entrada, EPG, asignación a moduladores ISDB-T y frecuencias. |
| **Orquestación Canales 24/7** | Asignación lógica y metadatos en base de datos central. | Metadata, EPG, sincronizador Syncthing (contenido local/remoto); **`xFast`** emite el stream continuo. |
| **VOD y Clientes IPTV** | Catálogo canónico de títulos, temporadas y episodios. | Orquestador de VOD para IPTV y administrador de clientes IPTV; **`xClient`** es el frontend de las APPs. |
| **Panel de Control** | API para `cpanel-admin` (Directo, **NO usa `xProxy`**). | Backend de la plataforma (`cpanel-client`). **`xproxy`** conecta el panel único con N `xManager`. |
| **Transcoding** | Preparación o validación de assets en ingesta. | Escaso y relativo (no es parte esencial de `xManager`). |
| **Persistencia** | PostgreSQL central (`xServerSQL`). Procedimientos PL/pgSQL intensivos. | PostgreSQL local (`xClientSQL`) para playout, estado y operación ininterrumpida. |

---

## 2. Diagrama de Topología y Comunicación

```
  ┌────────────────────────────────────────────────────────┐
  │                 cpanel-admin (Angular)                 │
  └───────────────────────────┬────────────────────────────┘
                              │ HTTP / REST Directo (NO xProxy)
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │                        xBackend                        │
  │   - API Central / Catálogo Canónico / TMDB             │
  │   - Procedimientos en PostgreSQL (xServerSQL)          │
  └───────────────────────────┬────────────────────────────┘
                              │ Sincronización Syncthing / Manifiestos
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │           Panel de Control Único (Frontend)            │
  └───────────────────────────┬────────────────────────────┘
                              │
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │                         xproxy                         │
  │   - Proxy inverso entre Panel Único y N xManager       │
  └───────────────────────────┬────────────────────────────┘
                              │ Enrutamiento a N Operadores de TV
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │               xManager (Operador de TV)                │
  │   - Backend Principal ISDB-T e IPTV                    │
  │   - Orquestador Canales Live (Moduladores, EPG)        │
  │   - Orquestador Canales 24/7 (Sincronizador Syncthing) │
  │   - Orquestador VOD y Administrador Clientes IPTV      │
  │   - Persistencia Local en xClientSQL                   │
  └─────────────┬───────────────────────────┬──────────────┘
                │                           │
                ▼                           ▼
  ┌───────────────────────────┐   ┌────────────────────────┐
  │           xFast           │   │         xClient        │
  │ Motor Emisión Continua    │   │ Frontend de las APPs   │
  │ 24/7 (ISDB-T e IPTV)      │   │ de IPTV para Usuarios  │
  └─────────────┬─────────────┘   └────────────────────────┘
                │
                ▼
  ┌───────────────────────────┐
  │  Moduladores ISDB-T /     │
  │  Encoders (xDevices)      │
  └───────────────────────────┘
```

---

## 3. Criterio de Decisión: ¿Dónde debe residir un requerimiento?

### Asignar a `xBackend/` cuando:
1. Se trate de creación o alteración de entidades de catálogo (Películas, Series, Canales 24/7, Servidores, Clientes).
2. Se definan contratos de API REST para `cpanel-admin`.
3. Involucre lógica de negocio en procedimientos almacenados en `xServerSQL`.
4. Maneje integración con proveedores externos de metadatos (TMDB, Gracenote).
5. Defina reglas de sincronización centralizada de archivos multimedia (Syncthing).

### Asignar a `xManager/` cuando:
1. Se gestione hardware físico (tarjetas de captura, moduladores ISDB-T, sintonizadores).
2. Se procesen flujos de video en vivo (FFmpeg, transcodificación, empaquetado HLS/TS).
3. Se configure enrutamiento multicast, IGMP o balanceo de carga de streaming.
4. Se optimicen buffers o consumo de red de alta densidad.

### Asignar a `Cross_System/` cuando:
1. Se incorpore un nuevo tipo de canal o evento que requiera configuración en la API central y ejecución especializada en el nodo.
2. Se modifique el protocolo de comunicación o sincronización entre `xBackend` y los nodos `xManager`.
3. Se implemente una funcionalidad integral desde la interfaz administrativa de CPanel hasta la salida modulada de video.
