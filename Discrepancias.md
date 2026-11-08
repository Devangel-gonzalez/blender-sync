Tras un análisis exhaustivo contrastando el código fuente real del repositorio con los documentos `README.MD` (que actúa como la **Especificación Técnica original o "Spec"**) y `README2.MD` (que funciona como un **Post-Mortem o Diario de Desarrollo**), existen **discrepancias críticas y desviaciones arquitectónicas** entre lo que se estipuló en el diseño inicial y lo que el código realmente ejecuta.

El `README.MD` describe el comportamiento *idealizado* del sistema, mientras que el código (y el `README2.MD`) documenta las *parches de supervivencia* necesarios para lidiar con las limitaciones de la API de Python de Blender y los errores de concurrencia.

A continuación, se detallan las principales discrepancias encontradas:

### 1. Protocolo de Red: ZMQ vs. TCP Stdlib
*   **Lo que estipula el .MD:** La especificación (Secciones 1.1, 3.6 y 5.2) exige repetidamente el uso de **ZeroMQ (ZMQ)** para la comunicación de red y para el "canal directo" de Live Sync.
*   **Lo que hace el código:** El addon **abandonó ZeroMQ por completo**. El código empaquetado (`wheels/replication-0.10.3`) y el módulo `live_sync.py` utilizan **sockets TCP nativos de la librería estándar de Python (stdlib)**.
    *   *Evidencia en el código:* El docstring de `live_sync.py` lo admite explícitamente: *"Transport note: the specification this module implements asks for a 'direct ZMQ channel'. The bundled replication 0.10.3 wheel no longer uses ZMQ - it is a stdlib TCP transport... so the direct channel here is a plain length-prefixed JSON stream"*.

### 2. Estructura de Directorios
*   **Lo que estipula el .MD:** La Sección 5.10 y el árbol de directorios (Sección 10) indican que el motor de Live Sync debe residir en `libs/replication/live_sync.py` y que la lógica de red está en `libs/replication/`.
*   **Lo que hace el código:** El archivo `live_sync.py` se encuentra en la **raíz del proyecto**, y el módulo de replicación está en `replication/` (sin el prefijo `libs/`). El propio resumen del commit en GitHub admite esta desviación: *"README.MD specifies... a libs/replication/ layout, but the shipped addon uses... stdlib TCP, so the transport and module location follow the actual code rather than the spec text."*

### 3. Formato de Datos de Transformación (Matrices vs. Canales Crudos)
*   **Lo que estipula el .MD:** La Sección 5.3 y 5.7 dictan que durante el Live Sync, el Host debe extraer y enviar la matriz de transformación (`matrix_basis`) del hueso evaluado.
*   **Lo que hace el código:** El código **rechaza usar `matrix_basis`** y en su lugar transmite canales crudos (`location`, `rotation_mode`, `rotation_quaternion`, `scale`).
    *   *Por qué cambió:* El Post-Mortem (`README2.MD`, Sección 6.9) documenta que `matrix_basis` no puede representar rotaciones mayores a 180° (ej. 270° se normaliza a -90°), lo que destruía las animaciones de múltiples vueltas. El código tuvo que evolucionar para enviar los valores crudos de las propiedades de Blender para preservar la integridad de la rotación.

### 4. Máquina de Estados y el Problema del "Viewer Tardío"
*   **Lo que estipula el .MD:** Define una máquina de estados simple (Sección 5.8): `IDLE` -> `INTERACTING` (vía señal `start`) -> `RESOLVING`. Asume que los clientes están conectados cuando se emite el `start`.
*   **Lo que hace el código:** Tuvo que inventar un sistema de **Heartbeats (Latidos)** y resurrección de overlays que no existe en la Spec.
    *   *Por qué cambió:* Si un Cliente se conectaba *después* de que el Host presionara 'R' (emitiendo el `start`), el Cliente nunca recibía la señal inicial y el overlay jamás aparecía (`README2.MD`, Secciones 4.5, 6.11). El código implementó `HEARTBEAT_INTERVAL` para reenviar la pose en reposo y una función `_revive_overlay()` que permite a un Cliente reconstruir el estado temporal a partir de un mensaje `update` huérfano.

### 5. Permisos de Publicación y Nodos "COMMON" (El Bug Catastrófico)
*   **Lo que estipula el .MD:** La Sección 4.7 sugiere que si un nodo tiene el Toggle de Live Sync activo (`is_live`), el handler debe forzar el `commit` y `push` *"sin importar otras condiciones de ownership"*.
*   **Lo que hace el código:** Implementa un filtro de seguridad estricto (`can_publish`) que **bloquea** a los Clientes de publicar ciertos nodos, anulando la lógica ingenua de la Spec.
    *   *Por qué cambó:* El Post-Mortem (Sección 6.14) revela que permitir que los Clientes hicieran push de nodos marcados como `RP_COMMON` (como Colecciones o Escenas) causaba que el Host aplicara esos datos, desensamblando objetos y rompiendo los modificadores de armadura (`StructRNA of type Object has been removed`). El código tuvo que hardcodear que solo el Host puede publicar, sin importar si el toggle está armado.

### 6. Reconstrucción de Modificadores (El Bug Silencioso de Blender)
*   **Lo que estipula el .MD:** No entra en detalles sobre cómo se serializan las referencias de objetos dentro de los modificadores (ej. el objetivo de un modificador *Armature* o *Boolean*).
*   **Lo que hace el código:** Contiene una función compleja de rescate llamada `resolve_modifier_object_refs` (`bl_types/bl_object.py`).
    *   *Por qué existe:* El `README2.MD` (Sección 6.18) expone que el `Dumper` genérico serializa las propiedades de tipo ID como simples strings (el nombre del objeto). El `Loader` genérico de Blender intenta asignar ese string a una propiedad de tipo Puntero y **falla en silencio**, dejando el modificador sin objetivo (`object = None`). El código tuvo que inspeccionar `bl_rna` en tiempo de ejecución para resolver estos punteros por UUID o nombre manualmente tras la carga.

### 7. Keyframes "Fantasma" y Curvas F-Curve Inventadas
*   **Lo que estipula el .MD:** Indica insertar un keyframe temporal en el frame del Host y limpiarlo al confirmar o cancelar (Sección 3.5).
*   **Lo que hace el código:** Implementa una lógica agresiva de **podado de curvas (pruning)** en `BlAction.load`.
    *   *Por qué cambió:* Para que el revisor pueda ver una pose no animada (ej. mover un hueso en X) sobre una animación que solo tiene rotación, el overlay del Cliente *inventa* curvas de F-Curve para Location y Scale (`README2.MD`, Sección 4.4). Cuando el Host confirma y `porcelain` aplica el Action real, esas curvas inventadas no existen en el dump. Si el código no las poda explícitamente del *channelbag*, quedan como keyframes "huérfanos" o "fantasmas" permanentes en el Cliente (Sección 6.17).

### 8. Fragilidad Extrema de los Timers (Threading)
*   **Lo que estipula el .MD:** Recomienda correctamente usar `bpy.app.timers` en lugar de `threading` para evitar crashes (Sección 3.7).
*   **Lo que hace el código:** Está lleno de bloques `try/except` defensivos y *guards* de `None` que la Spec no menciona.
    *   *La realidad:* El Post-Mortem (Sección 3.3) confiesa que la arquitectura de `Timer.main()` del addon original convierte **cualquier excepción no capturada** (incluso un `AttributeError` profundo al leer un `action_slot` que es `None` en Blender 5) en una **desconexión inmediata de la sesión**. Esto obligó a los desarrolladores a blindar el código contra fallos silenciosos de la API de Blender que la especificación original no previó.

### Resumen
El `README.MD` es un documento de **intención de diseño** (cómo debería funcionar el sistema en un mundo ideal donde las APIs de Blender son predecibles y ZMQ está disponible). El código real es el resultado de una **guerra de trincheras contra la API de Blender**, donde conceptos como `matrix_basis`, el `Dumper` genérico y los handlers del Depsgraph demostraron ser insuficientes o destructivos, forzando al código a implementar workarounds complejos (Heartbeats, resolución manual de punteros, podado de F-Curves y transporte TCP puro) que contradicen o expanden drásticamente la especificación original.