# Documento Técnico: Arquitectura y Funcionamiento de `blender-sync`

Este documento detalla la arquitectura interna, los protocolos de red y los mecanismos de sincronización del addon `blender-sync`, basándose exclusivamente en el análisis estático y dinámico de su código fuente. El addon es una bifurcación (fork) altamente modificada del addon "Multi-User" para Blender, diseñada específicamente para soportar flujos de trabajo de revisión de animación en tiempo real (Live Sync) entre múltiples instancias de Blender.

---

## 1. Arquitectura General y Gestión de Estado

El addon opera bajo un modelo Cliente-Servidor con replicación de datos orientada a objetos (`bpy.data`). La gestión del estado global de la sesión se centraliza en un objeto singleton-like para evitar el uso de variables globales dispersas y asegurar el acceso thread-safe a los metadatos de la conexión.

- **`SessionData` (`shared_data.py`):** Es el núcleo del estado compartido. Mantiene referencias al `repository` (el grafo de replicación local), el `remote` (el servidor o cliente conectado) y el diccionario `live_sync_nodes`. Este último es crucial para la Funcionalidad 1 (Toggles Globales), ya que almacena los UUIDs de los datablocks que tienen activado el commit/push automático ante cualquier cambio detectado en el depsgraph.
- **Propiedades de Blender:** El addon inyecta propiedades personalizadas en el API de Blender durante el registro (`__init__.py`):
  - `bpy.types.ID.uuid`: Un `StringProperty` inyectado en cada datablock (Objetos, Mallas, Materiales, etc.) para garantizar una identidad única y estable a través de la red, evitando conflictos de nombres (ej. `Cube.001`).
  - `WindowManager.session`: Almacena los parámetros de la conexión activa.

---

## 2. Capa de Red y Protocolo de Replicación

Aunque la especificación original del proyecto sugería el uso de ZeroMQ (ZMQ), el código empaquetado en las ruedas (`wheels/replication-0.10.3`) y la implementación en `replication/net.py` revelan el uso de **sockets TCP nativos de Python (stdlib)**.

### 2.1. Formato de Mensajes

La comunicación se basa en la serialización de objetos Python mediante `pickle`, precedidos por una cabecera de longitud (length-prefix) para delimitar los frames en el stream TCP continuo:

1.  **Serialización:** Los datos se convierten en chunks de bytes.
2.  **Tipos de Objetos (`replication/objects.py`):**
    - `Node`: Representa un datablock con su UUID, propietario, dependencias y estado (`ADDED`, `UP`, `COMMITED`, `FETCHED`).
    - `Commit`: Contiene un objeto `Delta` (calculado por la librería `deepdiff`) que describe los cambios incrementales respecto a la versión anterior, optimizando el ancho de banda.
    - `Command`: Mensajes de control (`Right`, `Delete`, `Kick`, `Auth`).

### 2.2. Capa de Abstracción (`porcelain.py`)

El módulo `porcelain` expone una API de alto nivel inspirada en Git, abstrayendo la complejidad de los sockets:

- `add()`: Registra un datablock en el área de preparación (staging) del repositorio local.
- `commit()`: Calcula el `Delta` usando `DeepDiff` y actualiza el estado del nodo a `COMMITED`.
- `push()`: Envía los commits al servidor. El servidor actúa como un repetidor (relay) que distribuye los cambios a todos los clientes conectados.
- `apply()`: Reconstruye el estado local del datablock aplicando los datos recibidos del servidor.

---

## 3. Mecanismo de Sincronización Basado en Depsgraph

El motor principal de detección de cambios reside en `handlers.py`, específicamente en el handler `on_scene_update`, que está suscrito al evento `depsgraph_update_post` de Blender.

### 3.1. Filtrado de Ecos (Echo Cancellation)

Para evitar bucles infinitos donde un cliente aplica un cambio recibido y lo vuelve a enviar al servidor, el handler mantiene una lista de `applied_updates`. Cuando el depsgraph reporta un cambio, el handler verifica si el UUID del datablock modificado está en esta lista. Si es así, descarta la actualización por ser un "eco" remoto.

### 3.2. Filtros Inteligentes para Live Sync

Durante la transmisión de datos en tiempo real, el handler implementa bloqueos selectivos para evitar saturar la red:

- **Bloqueo de Armaduras y Objetos:** Si se detecta que el stream de Live Sync está activo (`live_sync.is_streaming()`), se ignoran los commits automáticos para objetos y armaduras. Esto se debe a que enviar matrices de transformación completas cada frame es ineficiente y innecesario si ya se están transmitiendo por el canal de Live Sync.
- **Permisividad para `Action`:** Los cambios en los datablocks de tipo `Action` (que contienen los keyframes) siguen permitidos para asegurar que la animación base se mantenga sincronizada.

### 3.3. Propagación Forzosa de Dependencias (`push_animation_dependencies`)

Blender tiene una limitación en su API: cuando se inserta un keyframe en un objeto, el `depsgraph` reporta que el _Objeto_ cambió, pero no reporta que la _Action_ asociada cambió. Como el sistema de replicación solo envía nodos que han cambiado, los keyframes nuevos quedarían "atrapados" en el cliente local.
La función `push_animation_dependencies` resuelve esto recorriendo el grafo de dependencias (`repository.graph.rbfs_from`) buscando nodos de tipo `Action` asociados al objeto modificado y forzando su `commit()` y `push()` manual.

---

## 4. Motor de Live Sync para Modo Pose (`live_sync.py`)

Esta es la innovación principal del addon, diseñada para permitir que un animador (Host) edite poses interactivamente (usando G, R, S) mientras un revisor (Client) ve los cambios integrados en su reproducción en bucle, sin pausar el timeline del Client.

### 4.1. Canal de Comunicación Paralelo

Para evitar el overhead de `porcelain.commit` (que genera historial de commits y serializa todo el objeto), Live Sync utiliza un **canal TCP dedicado** en el puerto `remote.port + 1`.

- **Protocolo:** Stream de JSON con prefijo de longitud.
- **Mensajes:** `start`, `update`, `end`.
- **Arquitectura:** `LiveSyncBroadcaster` (Host) gestiona las conexiones entrantes en un hilo secundario, mientras que `HostPoseStreamer` genera los datos.

### 4.2. Lectura del Depsgraph Evaluado

Blender no escribe las transformaciones interactivas (R, G, S) en `bpy.data` hasta que el usuario confirma (click izquierdo/Enter). Para capturar la pose _mientras_ se está arrastrando:

- El `HostPoseStreamer` accede al **Depsgraph Evaluado** mediante `bone.evaluated_get(depsgraph)`.
- Extrae la matriz de transformación visual (`matrix_basis`) directamente de la copia evaluada del hueso.
- Esta matriz se envía inmediatamente al Client a través del canal JSON.

### 4.3. Aplicación en el Cliente (Keyframes Temporales)

El `ClientPoseApplier` recibe las matrices y debe integrarlas visualmente sin detener la animación del Client.

- **Estrategia:** En lugar de modificar la matriz base del hueso (lo cual rompería la animación) o usar Action Swapping, el código inserta o sobrescribe un **keyframe temporal** en el frame exacto donde el Host está trabajando.
- **Optimización:** Se utiliza `fcurve.keyframe_points.insert(frame, value, options={'FAST'})`. La bandera `FAST` evita que Blender recalcule las tangentes y actualice el depsgraph, permitiendo una inserción de keyframes casi instantánea (vital para mantener 60 FPS durante el streaming).
- **Reversibilidad:** El sistema guarda en memoria el valor original del keyframe en ese frame.
  - Si el Host **confirma** (envía señal `end` con `is_cancelled=False`), el commit real llega vía `porcelain` y reemplaza el keyframe temporal con los datos definitivos.
  - Si el Host **cancela** (envía señal `end` con `is_cancelled=True`), el Client restaura el valor original guardado, eliminando cualquier rastro de la edición fallida.

---

## 5. Serialización de Datablocks (`bl_types/`)

El sistema es extensible y modular. Cada tipo de dato de Blender tiene su propia clase de serialización que hereda de `ReplicatedDatablock`.

### 5.1. Dumper y Loader

- **`Dumper`:** Convierte un objeto de Blender en un diccionario Python. Utiliza filtros (`include_filter`) para extraer solo los datos relevantes, ignorando propiedades de solo lectura o metadatos temporales. Para datos masivos (como coordenadas de Shape Keys o Skin Vertices), utiliza `numpy` (`np_dump_collection`) para serializar arrays de forma eficiente.
- **`Loader`:** Toma el diccionario y aplica los valores al objeto Blender local.

### 5.2. Resolución de Referencias

Un desafío crítico en la replicación es mantener las referencias entre objetos (ej. un modificador Armature apuntando a un esqueleto).

- El código prioriza la resolución por **UUID** sobre el nombre.
- Si el UUID no se encuentra (caso raro de desincronización), hace un fallback al **nombre**.
- Funciones como `resolve_modifier_object_refs` y `resolve_datablock_from_uuid` aseguran que, tras una sincronización, los modificadores, constraints y parents sigan apuntando a los objetos correctos, incluso si Blender ha renombrado el objeto localmente para evitar colisiones (ej. `Armature.001`).

---

## 6. Arquitectura de Hilos y Restricciones de Thread-Safety

Blender no es thread-safe para el acceso a `bpy.data`. Cualquier intento de leer o escribir la escena desde un hilo de Python secundario (`threading.Thread`) resulta en un fallo de segmentación (Crash).

El addon respeta estrictamente esta limitación:

1.  **Hilos Secundarios (Background):** Se utilizan _exclusivamente_ para operaciones de E/S de red (leer/escribir sockets, serializar JSON a bytes).
2.  **Hilo Principal (Main Thread):** Toda la interacción con la API de Blender (`bpy.data`, `ops`, `context`) ocurre en el hilo principal.
3.  **Mecanismo de Cola:** Los hilos de red depositan los mensajes entrantes en una cola thread-safe (`queue.Queue` o `collections.deque`). Un timer registrado en `bpy.app.timers` (ej. `LiveSyncTimer`) se ejecuta periódicamente en el hilo principal, vacía la cola y procesa los comandos, aplicando los cambios a la escena de manera segura.

---

## 7. Resumen de Componentes Clave

| Módulo                     | Responsabilidad Principal                                                                       |
| :------------------------- | :---------------------------------------------------------------------------------------------- |
| `shared_data.py`           | Singleton de estado global de la sesión y registro de nodos en Live Sync.                       |
| `handlers.py`              | Intercepción del Depsgraph, filtrado de ecos y propagación de Actions.                          |
| `live_sync.py`             | Motor de streaming de pose (TCP dedicado, lectura de depsgraph evaluado, keyframes temporales). |
| `replication/porcelain.py` | API de alto nivel para operaciones tipo Git (commit, push, apply).                              |
| `replication/objects.py`   | Definición de los paquetes de red (`Node`, `Commit`, `Command`) y serialización Pickle.         |
| `bl_types/bl_object.py`    | Serialización compleja de Objetos (modificadores, física, parenting por UUID).                  |
| `operators.py`             | Operadores de UI para iniciar/hostear sesiones y activar toggles de Live Sync.                  |
