# ADR-ARQUITECTURA-OBJETIVO-001 · Arquitectura objetivo mantenible para True Level: Playblast

| Campo                           | Valor                                                                                                                                                       |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Versión del documento           | **v1.1 · propuesta revisada**                                                                                                                               |
| Estado                          | **PROPUESTO · pendiente de aprobación del mantenedor**                                                                                                      |
| Fecha                           | 2026-11-28                                                                                                                                                  |
| Historial                       | v1.0 publicada como propuesta inicial; v1.1 incorpora aclaraciones del mantenedor sobre `object_transform`, playheads, incorporación de escena y versionado |
| Decisión de alcance             | **Opción A revisada**: R1 = Live Sync de pose + Live Sync de transformación de objetos + porcelain para datos definitivos                                   |
| Código funcional de referencia  | `blender-sync` `d62755a` · tag `r1-fase-01`                                                                                                                 |
| Commit documental de referencia | `3f1e9ef` · ADR v1.0 subido por el mantenedor                                                                                                               |
| SRS de referencia               | `workspace-3/consolidated/SRS.md` v2.1.1                                                                                                                    |
| TDD de referencia               | `workspace-3/consolidated/TDD-PLAYBLAST-001.md` v4.7                                                                                                        |
| Fase 1                          | Cerrada y verificada; sus guardas se conservan como red de seguridad inicial                                                                                |
| Modificaciones realizadas       | Solo documentos ADR/propuesta. No se modifica código, SRS ni TDD                                                                                            |

---

## 1. Resumen ejecutivo

R1 queda definido como:

> **Live Sync de pose + Live Sync de transformación de objetos + porcelain para datos definitivos.**

Los adapters live soportados en R1 son:

```text
pose
object_transform
```

No se añaden adapters live para shaders, sliders, luces, modificadores, sculpt, edit mode ni otros continuos. Esos quedan para R2 o posteriores.

La arquitectura debe quedar preparada para que R2 o posteriores puedan añadir un nuevo continuo mediante:

```text
schema declarativo
+ interaction registry
+ adapter por tipo
+ política de lane
+ puerta única de publicación
```

con cambios mínimos y localizados en el núcleo y en los handlers existentes.

La migración será **incremental**, no una reescritura. El comportamiento soportado de pose no debe cambiar. `object_transform` se añade como capacidad live de R1 con contrato análogo al de pose: selección, G/R/S, multi-objeto, keyframes definitivos por porcelain y previsualización reversible sin commits continuos durante el arrastre.

Las correcciones de defectos ya exigidas por SRS/TDD, como fidelidad de visibilidad, siguen siendo obligaciones de R1.

---

## 2. Contexto verificado

### 2.1 El código actual mezcla tres responsabilidades

Hoy conviven:

| Responsabilidad                    | Dónde vive principalmente                                                 | Problema                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Previsualización continua de pose  | `live_sync.py`                                                            | Solo sabe de pose; no hay contrato general de adapter                              |
| Publicación por cambios detectados | `handlers.py`, `timers.py`, `operators.py`                                | Rutas dispersas; riesgo de puntos ciegos                                           |
| Fidelidad de datos                 | `bl_types/`                                                               | Serializadores curados, pero incompletos en varias capas                           |
| Autorización                       | `can_publish`, guardas de fase 1, `Repository.assert_modification_rights` | Política repartida; modelo base simétrico por `RP_COMMON`                          |
| Transporte definitivo              | `replication/porcelain.py`, `replication/repository.py`, servidor         | Commit/push/apply son fundamentales, pero no deben usarse como canal live continuo |
| Presencia / playhead               | `handlers.py`, `timers.py`, `porcelain.update_user_metadata`              | Metadatos de usuario; no son mutación de escena compartida                         |

### 2.2 Porcelain no es el culpable principal de la falta de visibilidad

Porcelain transporta lo que los serializadores producen. Si una propiedad no está en el dump, no viaja.

Ejemplos verificados en `d62755a`:

- `BlCollection.dump()` usa un filtro mínimo con `name` e `instance_offset`; no transporta los flags de visibilidad/restricción de colección.
- `BlObject.dump()` incluye `hide_render`, pero no `hide_viewport` ni `hide_select`.
- `BlArmature.dump()` no transporta correctamente `Bone.hide` ni visibilidad de bone collections; `layers` es mecanismo legado.
- No existe serialización completa de `LayerCollection.exclude`, `LayerCollection.hide_viewport`, `LayerCollection.holdout`, `LayerCollection.indirect_only`.
- `DynamicRightSelectTimer` fuerza `hide_select` desde permisos/readonly, en tensión con SRS §3.8 y TDD INV-3.

Esto ya está reconocido normativamente:

- SRS §3.9 exige fidelidad de visibilidad/restricción.
- TDD DD-03 especifica la corrección.
- TDD AC-06 exige que los flags coincidan con el Host en los tres modos.

### 2.3 Transformaciones de objeto: estado actual y decisión R1

En Multi-User vanilla, mover un objeto puede llegar al Cliente mediante porcelain/commits cuando el cambio queda confirmado y publicado por el handler.

En el código actual existe un gate de arrastre interactivo:

```python
dragging = live_sync.interactive_transform_active() is not None
if dragging:
    return
```

Ese gate está pensado para evitar commits pesados durante transformaciones interactivas, pero el streamer live actual solo implementa pose:

```text
HostPoseStreamer → huesos seleccionados en Modo Pose
```

Por tanto, R1 no debe conformarse con que el objeto “llegue después por porcelain”. Debe añadir un adapter live de transformación de objetos para que:

```text
G/R/S de objeto en curso → live preview reversible
confirmación / keyframe real → porcelain durable
```

sin convertir porcelain en canal de arrastre continuo.

### 2.4 Playhead y presencia: estado actual

El frame actual de cada peer se publica hoy como metadato de usuario:

```python
porcelain.update_user_metadata(session.repository, {
    'frame_current': scene.frame_current
})
```

Además, `ClientUpdate` publica metadatos de presencia como escena, modo, vista y color. La UI del Host muestra el frame de los Clientes desde `online_users[...]['metadata']['frame_current']`.

Esto no es una mutación del grafo de escena compartida. Es presencia.

Por tanto:

> **“El Cliente nunca publica” significa “nunca publica datos de escena ni mutaciones del grafo compartido”. No significa “nunca envía metadatos de presencia”.**

R1 conserva los playheads como presencia por porcelain/metadata. No se migran al canal live en R1. Una migración futura a un canal de presencia live es posible, pero no es necesaria para resolver el contrato R1.

### 2.5 La fase 1 cerró rutas, pero no creó una capacidad central

La fase 1 implementó guardas Host-only en operadores, handlers y timers. Eso fue necesario y se conserva.

Pero la regla todavía no vive en un único punto de salida. El modelo base de Multi-User permite `RP_COMMON` para cualquier peer, y el servidor no distingue de forma robusta una conexión autoritativa de Host frente a un viewer autenticado que intenta mutar el grafo compartido.

La arquitectura objetivo debe convertir:

```text
“cada caller recuerda comprobar rol”
```

en:

```text
“ninguna salida de replicación compartida puede saltarse la política”
```

---

## 3. Decisión arquitectónica

Se adopta una arquitectura de **tres carriles** y **cuatro contratos**.

### 3.1 Tres carriles

| Carril                    | Qué transporta                                    | Durabilidad                                                        | R1                         |
| ------------------------- | ------------------------------------------------- | ------------------------------------------------------------------ | -------------------------- |
| **Live preview**          | Cambios efímeros de interacción continua          | No definitivo; debe restaurarse o ser reemplazado por dato durable | Pose y `object_transform`  |
| **Durable transaction**   | Estado confirmado o asentado que debe conservarse | Definitivo; llega por porcelain                                    | Sí, fundamental            |
| **Local-only / presence** | Estado local del peer y metadatos de presencia    | No muta la escena compartida                                       | Sí, con límites explícitos |

Regla rectora:

> **Live preview no es publicación. Porcelain no es canal de arrastre.**

### 3.2 Cuatro contratos

| Contrato                                     | Pregunta que responde                                                        | R1                                                                                     |
| -------------------------------------------- | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **PublicationPolicy / puerta única**         | ¿Quién puede enviar qué mensaje de replicación?                              | Obligatorio                                                                            |
| **ReplicationSchema / registry declarativo** | ¿Qué propiedades se replican y con qué versión?                              | Obligatorio como contrato mínimo; cobertura completa progresiva                        |
| **InteractionRegistry**                      | ¿Qué interacciones existen y cómo se observan?                               | Obligatorio; R1 registra `pose` y `object_transform`                                   |
| **Adapter**                                  | ¿Cómo se previsualiza, aplica, restaura y entrega a durable una interacción? | Obligatorio para `pose` y `object_transform`; interfaz preparada para futuros adapters |

---

## 4. Arquitectura objetivo

```text
HOST
────
Sensores existentes
  ├─ depsgraph_update_post
  ├─ contexto de operador modal cuando exista
  ├─ selección / modo / frame para pose
  ├─ selección / modo / frame para objetos
  └─ undo/redo
        │
        ▼
InteractionRegistry + LanePolicy
        │
        ├── LIVE PREVIEW LANE
        │     ├─ R1: pose adapter
        │     ├─ R1: object_transform adapter
        │     ├─ start / update / end / cancel
        │     ├─ coalescing por entidad/canal cuando aplique
        │     ├─ sin porcelain.commit/push durante arrastre real
        │     └─ overlay/reversible edit según SRS/TDD
        │
        ├── DURABLE TRANSACTION LANE
        │     ├─ cambios confirmados o asentados
        │     ├─ dependencias y orden de aplicación
        │     ├─ dump/delta mediante schema/serializadores
        │     ├─ commit/push por porcelain
        │     └─ invalidación de previews obsoletos
        │
        └── LOCAL-ONLY / PRESENCE
              ├─ selección
              ├─ navegación de viewport
              ├─ playhead local
              ├─ evaluación/físicas locales
              └─ metadatos de presencia permitidos
        │
        ▼
PublicationPolicy local
  ├─ Host: permitido según reglas existentes de ownership/COMMON
  ├─ Cliente: denegado para mutación de grafo compartido
  └─ Presence: permitido dentro de límites explícitos
        │
        ▼
replication / porcelain / servidor
        │
        ▼
Server Authority Policy
  ├─ conexión autoritativa de Host: puede mutar grafo compartido
  ├─ viewer autenticado: no puede mutar grafo compartido
  └─ presence metadata: permitido si no muta escena
        │
        ▼
CLIENTE
───────
  ├─ recibe live preview de pose
  ├─ recibe live preview de object_transform
  ├─ recibe estado durable por porcelain
  ├─ aplica mediante adapters/serializadores
  ├─ puede seleccionar e inspeccionar
  ├─ puede reproducir su propio timeline
  ├─ publica presencia/playhead como metadata, no como escena
  └─ no publica datos de escena
```

---

## 5. Contrato de publicación

### 5.1 Principio

La política de publicación debe aplicarse a **toda salida de replicación compartida**, no solo a los botones explícitos ni solo al handler principal.

Debe cubrir, como mínimo:

```text
Node
Commit
Delete
Right
purge/orphan removal que produzca Delete
server repository init que reemplace el grafo
cualquier ruta futura que envíe datos de escena
```

No debe bloquear por error:

```text
Auth
TTL/heartbeat
RequestServerInfo
solicitud/recepción de snapshot
UpdateUserMetadata de presencia permitido
Disconnect
apply/fetch/load local
```

### 5.2 Separación entre commit local y publicación

`porcelain.commit()` es una operación local de repositorio: prepara/dumpea/delta. No es por sí sola una salida de red.

La puerta principal debe estar en la **salida hacia el servidor**, no en convertir cualquier commit local en una operación prohibida. Las guardas de fase 1 sobre operadores explícitos se conservan hasta que la puerta central esté verificada.

Esto evita romper:

- mantenimiento local del grafo;
- apply/fetch;
- resolución de punteros tras undo/redo;
- staging local;
- presencia.

### 5.3 Regla para Cliente

Un Cliente puede:

```text
mirar
seleccionar
entrar en Modo Pose
navegar su viewport
reproducir su timeline
evaluar físicas localmente
tener estado local de vista
publicar metadatos de presencia permitidos
```

Un Cliente no puede:

```text
enviar Node/Commit que mute el grafo compartido
enviar Delete
enviar Right
publicar dependencias externas
publicar escena nueva
publicar anotaciones
publicar overlays
publicar tipos de keyframe
publicar cambios locales de física/evaluación
publicar su playhead como dato de escena que fuerce el timeline de otros
```

### 5.4 Regla para Host

El Host conserva la semántica existente que ya funciona:

```text
owner == username
RP_COMMON
bl_check_common cuando aplique
```

La arquitectura no debe cambiar qué puede publicar el Host. Debe impedir que el Cliente herede esas capacidades por accidente.

### 5.5 Defensa en profundidad

R1 debe tener dos niveles:

| Nivel             | Propósito                                                                                                                                    |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cliente local** | Evitar que rutas honestas del addon publiquen datos de escena desde un viewer                                                                |
| **Servidor**      | Rechazar mutaciones de grafo compartido aunque un cliente modificado, una ruta futura o un mensaje forjado intente saltarse el cliente local |

El servidor no debe confiar en:

```text
owner declarado en el mensaje
username declarado por el peer
RP_COMMON
auth_type == ADMIN por sí solo
```

Debe existir una noción de **conexión autoritativa de Host** ligada a la sesión autenticada.

### 5.6 Presence / playhead no es publicación de escena

Los metadatos de presencia permitidos incluyen, como mínimo:

```text
frame_current del peer
scene_current
mode_current
view_matrix / view_corners
color / identidad visual de presencia
```

Reglas:

1. `frame_current` de un peer describe dónde está ese peer; no ordena a otros peers mover su timeline.
2. El Host puede mostrar el frame de los Clientes en la UI.
3. El Cliente puede mostrar presencia del Host si la UI lo permite.
4. Ningún playhead recibido por presencia debe ejecutar `frame_set()` ni asignar `frame_current` automáticamente en el receptor.
5. “Snap to user time” sigue siendo una acción explícita, voluntaria y reversible del usuario.
6. La puerta de publicación debe permitir `UpdateUserMetadata` de presencia, pero no debe usarlo como caballo de Troya para datos de escena.

R1 conserva este flujo por porcelain/metadata. Migrarlo a un canal live de presencia queda para R2 o posteriores si se decide.

---

## 6. Enforcement en servidor

### 6.1 Entra en R1

El enforcement en servidor **sí entra en R1** para mutaciones del grafo compartido.

R1 debe rechazar en servidor, antes de aplicar y antes de reenviar a otros peers:

```text
viewer → Commit
viewer → Node
viewer → Delete
viewer → Right
viewer → init/reemplazo autoritativo del repositorio compartido
```

Debe permitir:

```text
viewer → Auth
viewer → TTL
viewer → RequestServerInfo
viewer → snapshot fetch
viewer → UpdateUserMetadata de presencia permitido
viewer → Disconnect
Host autoritativo → operaciones de grafo compartidas existentes
```

### 6.2 Requisitos de la autoridad de Host

El mecanismo exacto se detallará en la enmienda del TDD, pero debe satisfacer:

1. La autoridad no puede ser auto-declarada por el cliente mediante un campo `role`, `owner` o `admin`.
2. Debe estar ligada a la conexión autenticada que representa al Host de la sesión.
3. Debe sobrevivir a reconexiones solo si la sesión y el protocolo de reconexión lo permiten explícitamente.
4. No debe conceder capacidad de publicación a un viewer por conocer una contraseña de administración si esa contraseña no representa autoridad de Host.
5. No debe romper el flujo local `localhost` ni el flujo LAN soportado.
6. Si requiere cambiar handshake o payload de Auth, debe versionarse y declararse en el TDD.

Una solución candidata es una **capacidad/lease de Host** generada por el servidor o entregada por un canal privado de arranque al proceso Host, y luego ligada a su conexión autenticada. Este ADR no fija todavía el mecanismo criptográfico ni de transporte; fija el requisito.

### 6.3 Qué no entra en R1 dentro del servidor

No entra en R1:

```text
TLS nativo
Internet/WAN
servicio de retransmisión remoto
modelo completo de cuentas/roles administrativos
hardening completo de comandos de administración de sesión no relacionados con mutación de escena
migración de presence/playhead a canal live dedicado
```

`Kick` y otros comandos de administración de sesión deben auditarse, pero no se convierten automáticamente en publicación de escena. Si se quiere cambiar su modelo de permisos, requiere una decisión/enmienda explícita.

---

## 7. Schema y registry declarativo

### 7.1 Qué es

`ReplicationSchema` es un registro curado y versionado de qué datos se replican por tipo de datablock y versión de Blender.

No es:

```text
dump automático de toda la RNA
serialización de todo lo que exista en bpy.types.*
configuración libre de invariantes de seguridad
```

Sí es:

```text
lista blanca curada
overrides por versión de Blender
dependencias explícitas
política de lane
tests de round-trip
trazabilidad hacia SRS/TDD
```

### 7.2 Forma conceptual

Sin fijar todavía la API exacta:

```text
SchemaEntry
  type_id: "Object" | "Collection" | "Armature" | "Scene" | ...
  blender_version_range
  durable_fields
  excluded_fields
  dependencies
  view_layer_extensions
  live_adapter: None | "pose" | "object_transform" | futuro adapter
  lane: durable | live+durable | local-only
  tests: roundtrip / parity / acceptance
```

### 7.3 R1 mínimo obligatorio

En R1 debe existir el contrato y un registry mínimo que permita:

1. Registrar los adapters live de pose y `object_transform`.
2. Declarar los campos de fidelidad exigidos por DD-03.
3. Declarar los campos necesarios para transformaciones de objeto y su handoff durable.
4. Versionar cambios de schema.
5. Ejecutar tests de paridad/round-trip.
6. Mantener los serializadores legacy como adapters envueltos mientras se migran.

R1 no tiene que migrar los 23+ tipos de `bl_types/` de golpe. La migración puede ser por tipo, empezando por los necesarios para:

```text
pose
object_transform
Object
Collection
Armature
Scene / ViewLayer
Action
```

### 7.4 R2 o posteriores

Para añadir un nuevo continuo, por ejemplo un slider de shader, se debería poder hacer:

```text
1. Añadir SchemaEntry o campos live/durable del tipo afectado.
2. Añadir InteractionDescriptor.
3. Añadir Adapter.
4. Añadir LanePolicy.
5. Añadir tests.
```

El núcleo no debería necesitar un nuevo `if isinstance(...)` por cada tipo.

---

## 8. InteractionRegistry y adapters

### 8.1 InteractionDescriptor

Cada interacción soportada debe declarar:

```text
id
entidad afectada
cómo se detecta inicio
cómo se detecta update
cómo se detecta fin/cancelación/settle
lane
adapter
política de coalescing
handoff a durable
restricciones SRS/TDD
```

### 8.2 R1: pose

El adapter live de pose debe preservar:

```text
id: pose
entidad: (armature_uuid, bone_name)
trigger: selección de hueso en Modo Pose sobre armadura rastreada
live: start / update / end
apply: overlay temporal reversible en el frame del Host
durable: keyframes reales por porcelain
cancel: restaurar
end: limpiar/restaurar según contrato R1
```

Debe preservar:

- no pausar ni seekear el timeline del Cliente;
- no mutear curvas;
- no action swapping;
- no keyframes masivos;
- una entidad temporal por hueso/canal animado, no 60 keyframes nuevos por segundo;
- limpieza al deseleccionar/desconectar;
- invalidación segura cuando llega un apply durable;
- procedencia de curvas inventadas;
- multi-hueso y multi-armadura;
- nombres exactos de hueso;
- compatibilidad con físicas locales del Cliente.

### 8.3 R1: object_transform

Se añade como capacidad live de R1:

```text
id: object_transform
entidad: object_uuid
trigger: selección de objeto(s) transformable(s) y transformación interactiva G/R/S
live: start / update / end
apply: previsualización reversible de transformación de objeto
durable: Object / Action / dependencias por porcelain
cancel: restaurar
end: limpiar/restaurar según contrato R1
```

Contrato observable propuesto:

| Acción del Host                          | Qué debe hacer el Cliente                                                                                              |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Selecciona objeto(s) transformable(s)    | Abrir/actualizar preview en el frame del Host, según contrato de selección análogo a pose                              |
| Arrastra G/R/S sin confirmar             | Actualizar preview en vivo; no crear commits continuos                                                                 |
| Confirma G/R/S                           | El preview sigue vivo si la selección/interacción sigue activa; el dato durable llega por porcelain cuando corresponda |
| Cancela con Esc/clic derecho             | Restaurar estado original; no dejar transformación fantasma                                                            |
| Confirma pero no inserta keyframe        | Al cerrar/deseleccionar, restaurar si no hay dato durable que lo reemplace                                             |
| Inserta keyframe real                    | El keyframe viaja por porcelain como dato definitivo                                                                   |
| Scrub con objeto seleccionado            | El preview sigue al frame del Host; el playhead del Cliente no se mueve automáticamente                                |
| Cambia selección                         | Cerrar previews retirados y abrir los nuevos                                                                           |
| Desconecta                               | Limpiar/restaurar previews                                                                                             |
| Llega apply durable durante preview vivo | Invalidar preview obsoleto sin restaurar valores viejos sobre datos recién aplicados                                   |

Reglas técnicas propuestas:

1. Multi-objeto: cada objeto se indexa por `object_uuid`; no colisionan nombres.
2. La transformación live debe respetar padres, constraints y dependencias; no debe desconectar ni mutear la animación del Cliente.
3. Si el objeto tiene animación gobernante, el preview debe integrarse en el frame del Host sin action swapping.
4. Si el objeto no tiene animación gobernante, el preview debe ser reversible y no debe inventar Actions/keyframes permanentes por accidente.
5. El handoff durable publica `Object`, `Action` y dependencias necesarias por porcelain.
6. No se hace `porcelain.commit/push` por cada tick del arrastre.
7. La UI del Cliente no debe perder selección ni capacidad de inspección.
8. El timeline del Cliente no se pausa ni se seekea automáticamente.
9. La desambiguación pose/objeto debe ser explícita: transformación de hueso en Modo Pose usa `pose`; transformación de objeto a nivel Object usa `object_transform`.

El mecanismo exacto de overlay para objetos —keyframe temporal, conducción directa reversible o híbrido según exista Action gobernante— se especificará en la enmienda del TDD. Este ADR fija el contrato observable y la prohibición de commits continuos.

### 8.4 Contrato de adapter

Un adapter live futuro debe poder implementar, conceptualmente:

```text
observe_host(context) -> change
make_preview(change) -> live_message
apply_preview(client, message) -> reversible_edit
close_preview(client, reason)
invalidate_preview(client, durable_apply)
durable_handoff(change) -> schema/porcelain
```

En R1, el adapter de pose puede ser una envoltura incremental sobre `HostPoseStreamer` y `ClientPoseApplier`. El adapter `object_transform` puede reutilizar la infraestructura de transporte, coalescing, heartbeat, invalidación y handoff durable, pero debe tener su propio sensor/applier de objeto.

### 8.5 Lo que no entra en R1

No se registran ni soportan como live adapters en R1:

```text
shader_slider
light_energy
modifier_value
dope_sheet_drag
sculpt_continuous
edit_mode_continuous
geometry_nodes_slider
playhead_presence_live_channel
```

Notas:

- `object_transform` **sí entra en R1**.
- Los cambios de objetos que no sean transformación interactiva live pueden seguir llegando por porcelain como datos definitivos.
- Los playheads siguen en presencia/metadata por porcelain en R1. Un canal live de presencia es futuro opcional.
- `dope_sheet_drag` no se añade como live adapter general en R1, aunque los keyframes reales insertados/borrados sigan llegando por porcelain.

---

## 9. LanePolicy y transacciones durables

### 9.1 R1

R1 conserva estos caminos:

```text
pose live → live_sync TCP
object_transform live → live_sync TCP
dato definitivo → porcelain
presence/playhead → UpdateUserMetadata / porcelain command channel
```

No se introduce un scheduler genérico de coalescing para todos los cambios RNA.

La política R1 debe expresar:

| Cambio                                          | Lane R1                                                  |
| ----------------------------------------------- | -------------------------------------------------------- |
| Pose seleccionada/arrastrada                    | live preview                                             |
| Objeto seleccionado/transformado con G/R/S      | live preview                                             |
| Keyframe real insertado/borrado                 | durable                                                  |
| Visibilidad/restricción DD-03                   | durable, y debe publicarse aunque haya pose seleccionada |
| Añadir/borrar objeto/colección/material/textura | durable                                                  |
| Undo/redo                                       | reconciliación durable/local según rol                   |
| Selección/vista/playhead local                  | local-only/presence                                      |
| Slider de shader                                | durable por ruta existente; sin nuevo live adapter R1    |

### 9.2 “Un solo commit” se interpreta como lote lógico

Una edición confirmada puede producir varios nodos:

```text
Object + Armature + Action
Material + NodeTree + Image + File
```

El contrato no es “una llamada literal a commit”, sino:

> **Una edición lógica no debe producir cientos de publicaciones intermedias.**

En R1 esto se aplica a pose, `object_transform` y al flujo durable ya soportado. La generalización a sliders y otros continuos queda para R2.

### 9.3 Orden live ↔ durable

Se preserva el contrato actual:

```text
end → restaurar/limpiar overlay → aplicar dato durable
```

y la excepción de invalidación:

```text
si un apply durable reconstruye datos mientras el overlay vive,
descartar overlay obsoleto sin restaurar valores viejos,
conservar procedencia,
reconstruir desde latido live si la selección sigue activa.
```

R2 podrá añadir revisiones/seq más generales. R1 debe preservar los mensajes de pose actuales y añadir solo lo necesario para `object_transform`, versionado y documentado.

---

## 10. Qué entra en R1 y qué queda para después

### 10.1 Entra en R1

| Área                                | Obligación R1                                                                                |
| ----------------------------------- | -------------------------------------------------------------------------------------------- |
| Alcance funcional                   | Pose live + `object_transform` live + porcelain definitivo                                   |
| Adapters live                       | `pose` y `object_transform` únicamente                                                       |
| Puerta local de publicación         | Política central de salida para replicación compartida                                       |
| Guardas fase 1                      | Se conservan hasta que la puerta central esté verificada                                     |
| Enforcement servidor                | Rechazo de mutaciones de grafo compartido por viewers                                        |
| Autoridad Host                      | Conexión autoritativa no auto-declarada por el cliente                                       |
| Presence/playhead                   | Se conserva como metadata; no fuerza timeline remoto                                         |
| Schema/registry                     | Contrato mínimo versionado + registry de pose/object_transform + campos DD-03                |
| Fidelidad visibilidad               | Cumplir SRS §3.9 / TDD DD-03 / AC-06                                                         |
| `hide_select`                       | No gobernarse por readonly/propiedad; reconciliar timer                                      |
| Paridad pose                        | Tests de paridad antes/después de extraer adapter                                            |
| Paridad/aceptación object_transform | Sin commits continuos durante arrastre; preview reversible; durable por porcelain            |
| Incorporación de escena             | Default `Nueva escena`; `Combinar` opt-in con advertencia; copia como caso límite/reconexión |
| Compatibilidad                      | No romper Host actual ni Cliente local/selección/reproducción                                |
| Documentación                       | SRS/TDD/checklist/diario actualizados solo tras aprobación de enmiendas                      |

### 10.2 Queda para R2 o posteriores

| Área                                                                            | Motivo                                                                                  |
| ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Nuevos adapters live: shaders, sliders, luces, modificadores, sculpt, edit mode | Ampliación de alcance; requiere nuevos criterios y mediciones                           |
| Sensor RNA/msgbus general                                                       | No necesario para pose/object_transform R1; tiene limitaciones y requiere diseño propio |
| Coalescing/settle genérico para continuos RNA                                   | Depende de adapters futuros                                                             |
| Schema completo de todos los tipos Blender                                      | Migración progresiva; no big-bang                                                       |
| Transaction scheduler genérico multi-tipo                                       | R1 conserva flujo durable actual                                                        |
| Wire format live universal para todos los continuos                             | R1 solo añade lo necesario para `object_transform`                                      |
| Canal live dedicado para presence/playheads                                     | R1 conserva metadata por porcelain; migración futura opcional                           |
| PKU / D-51 / keyframe azul                                                      | Sigue diferido según TDD                                                                |
| Internet/TLS/relay                                                              | Fuera de R1                                                                             |
| Reescritura de `bl_types/` o `replication/`                                     | Contrario a migración incremental                                                       |

### 10.3 R1 no promete “todo Blender continuo”

R1 puede publicar muchos cambios por porcelain, pero solo promete previsualización live continua para:

```text
pose
object_transform
```

Esta frase debe aparecer en cualquier documento derivado:

> **R1 no es un sistema universal de live editing. Es un sistema de live pose/object review con transporte durable fiel y una arquitectura preparada para futuros adapters.**

---

## 11. Migración incremental propuesta

No se ejecuta todavía. Es el plan que requerirá aprobación y órdenes de fase separadas.

### Paso 0 · Aprobar ADR y enmiendas

Antes de tocar código:

```text
aprobar ADR-ARQUITECTURA-OBJETIVO-001 v1.1 o posterior
aprobar propuesta de enmiendas SRS/TDD
consolidar TDD/checklist/diario
crear tag de inicio de la fase arquitectónica
```

### Paso 1 · Línea de paridad

Congelar evidencia de pose antes de refactorizar:

```text
arnés actual
RG-01…RG-06
pruebas de dos peers
mensajes live actuales
contadores de porcelain
comportamiento de overlay
```

La comparación debe ser contra `r1-fase-01`, no contra una expectativa reescrita después.

### Paso 2 · Puerta local de publicación

Introducir `PublicationPolicy` en la salida de replicación.

Criterio:

```text
Host emite las mismas llamadas que antes.
Cliente no emite mutaciones compartidas por ninguna ruta honesta.
Presencia, apply, fetch, Auth, TTL y snapshot siguen funcionando.
```

### Paso 3 · Enforcement servidor

Añadir autoridad de Host y rechazo server-side.

Criterio:

```text
viewer forjado no puede mutar grafo aunque salte el cliente local.
Host autoritativo sí puede.
Presence metadata sigue permitida.
No se rompe localhost ni LAN.
```

### Paso 4 · Schema/registry mínimo

Introducir contrato declarativo sin migrar todo de golpe.

Criterio:

```text
pose y object_transform registrados como live adapters R1.
DD-03 tiene schema/entries versionadas.
Serializadores legacy siguen funcionando.
Round-trip/parity tests pasan.
```

### Paso 5 · Adapter de pose envuelto

Extraer/registrar pose como adapter sin cambiar semántica.

Criterio:

```text
mismos mensajes live de pose.
mismo overlay.
misma restauración.
misma invalidación.
mismos keyframes definitivos por porcelain.
sin regresión de latencia/cadencia.
```

### Paso 6 · Adapter object_transform

Añadir `object_transform` como adapter live R1.

Criterio:

```text
G/R/S de objeto no produce porcelain.commit/push continuo.
Cliente ve preview reversible.
Multi-objeto funciona por object_uuid.
Confirmación/keyframe real llega por porcelain.
Cancelación/deselección/desconexión limpian.
No se pausa ni seekea el timeline del Cliente.
No se rompe pose.
```

### Paso 7 · Continuar fases R1 existentes

Las fases 2–13 no se aprueban automáticamente por este ADR. Se deciden una a una, usando los nuevos contratos.

DD-03 debe implementarse en su fase correspondiente o en la fase que el mantenedor autorice, pero sigue siendo obligación de R1.

La incorporación de escena con default `Nueva escena` y `Combinar` opt-in debe implementarse en la fase de onboarding que corresponda, o en la fase que el mantenedor autorice, pero es objetivo R1.

---

## 12. Criterios de aceptación del ADR

Estos criterios se proponen para la fase arquitectónica. No sustituyen los AC del SRS/TDD.

### 12.1 Alcance

| ID        | Criterio                                                                                  |
| --------- | ----------------------------------------------------------------------------------------- |
| ADR-AC-01 | El registry live de R1 contiene únicamente `pose` y `object_transform`                    |
| ADR-AC-02 | No se añade ningún mensaje live para shaders/sliders/luces/modificadores/sculpt/edit mode |
| ADR-AC-03 | Los mensajes live de pose no cambian sin enmienda explícita                               |
| ADR-AC-04 | F1 sigue fuera de soporte; no se convierte en live adapter general                        |
| ADR-AC-05 | Playheads/presencia siguen como metadata; no fuerzan timeline remoto                      |

### 12.2 Publicación

| ID        | Criterio                                                                         |
| --------- | -------------------------------------------------------------------------------- |
| ADR-AC-06 | Toda salida de mutación compartida pasa por política local                       |
| ADR-AC-07 | Cliente no puede enviar Node/Commit/Delete/Right por rutas honestas              |
| ADR-AC-08 | Servidor rechaza mutaciones compartidas de viewer aunque el mensaje esté forjado |
| ADR-AC-09 | Host conserva llamadas y semántica de publicación existentes                     |
| ADR-AC-10 | Presence/Auth/TTL/snapshot/apply/fetch no se bloquean por error                  |
| ADR-AC-11 | Las guardas de fase 1 siguen pasando hasta ser sustituidas formalmente           |

### 12.3 Schema y fidelidad

| ID        | Criterio                                                                                                 |
| --------- | -------------------------------------------------------------------------------------------------------- |
| ADR-AC-12 | Existe schema/registry versionado mínimo                                                                 |
| ADR-AC-13 | DD-03 se expresa declarativamente o mediante entries trazables                                           |
| ADR-AC-14 | Object/Collection/Armature/ViewLayer restauran flags exigidos por SRS §3.9                               |
| ADR-AC-15 | `hide_select` no se fuerza desde readonly/propiedad                                                      |
| ADR-AC-16 | Cambios DD-03 llegan al Cliente incluso si hay hueso seleccionado, sin retirar protección de rendimiento |

### 12.4 Paridad de pose

| ID        | Criterio                                                                                    |
| --------- | ------------------------------------------------------------------------------------------- |
| ADR-AC-17 | Selección sin mover → deseleccionar no deja rastro                                          |
| ADR-AC-18 | Transformar sin keyear → deseleccionar restaura                                             |
| ADR-AC-19 | Keyframe real llega por porcelain y no queda fantasma                                       |
| ADR-AC-20 | Scrub con hueso seleccionado mueve overlay al frame del Host sin mover playhead del Cliente |
| ADR-AC-21 | Desconexión/reconexión limpia y reconstruye solo lo activo                                  |
| ADR-AC-22 | Apply durable durante overlay vivo no restaura valores obsoletos                            |
| ADR-AC-23 | Multi-hueso/multi-armadura no colisiona                                                     |
| ADR-AC-24 | Cadencia/latencia no empeora respecto a la línea de paridad                                 |

### 12.5 object_transform

| ID        | Criterio                                                                             |
| --------- | ------------------------------------------------------------------------------------ |
| ADR-AC-25 | G/R/S de objeto en Host produce preview live en Cliente sin commits continuos        |
| ADR-AC-26 | Multi-objeto seleccionado produce previews independientes por `object_uuid`          |
| ADR-AC-27 | Cancelar G/R/S restaura el estado original del Cliente                               |
| ADR-AC-28 | Confirmar sin keyframe no deja transformación fantasma al cerrar/deseleccionar       |
| ADR-AC-29 | Keyframe real de objeto llega por porcelain como dato definitivo                     |
| ADR-AC-30 | Scrub con objeto seleccionado sigue al frame del Host sin mover playhead del Cliente |
| ADR-AC-31 | Apply durable durante preview vivo no restaura valores obsoletos                     |
| ADR-AC-32 | Desconexión limpia/restaura previews de objeto                                       |
| ADR-AC-33 | object_transform no rompe pose ni la cadencia del canal live                         |

### 12.6 Incorporación de escena

| ID        | Criterio                                                                                                                                                        |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ADR-AC-34 | El comportamiento por defecto al conectar es `Nueva escena`                                                                                                     |
| ADR-AC-35 | `Combinar` en la misma escena es opt-in explícito, con advertencia de que puede mezclar contenido local y remoto                                                |
| ADR-AC-36 | La opción de combinar puede ofrecer transferir ajustes de tiempo necesarios para que la animación coincida, como FPS/rango, bajo elección explícita del usuario |
| ADR-AC-37 | Conectar desde una copia del `.blend` se trata como caso límite soportado/reconexión, no como flujo principal recomendado                                       |
| ADR-AC-38 | Ninguna ruta purga contenido local sin elección explícita y confirmada                                                                                          |

---

## 13. Riesgos y mitigaciones

| Riesgo                                                  | Mitigación                                                                            |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Sobre-ingeniería: convertir R1 en framework universal   | R1 solo registra pose y object_transform; registry mínimo; sin sensores RNA generales |
| Romper pose al extraer adapter                          | Paridad antes/después; envoltura incremental; feature flag si hace falta              |
| object_transform introduce conflictos con pose          | Desambiguación explícita por modo/entidad; tests cruzados pose+objeto                 |
| object_transform reintroduce commits continuos          | Gate de arrastre + lane live + contador de porcelain en tests                         |
| Puerta local rompe presencia o snapshot                 | Clasificación explícita de mensajes permitidos; tests de control positivo             |
| Autoridad servidor basada en `admin` o `owner`          | Requisito explícito: autoridad ligada a conexión, no auto-declarada                   |
| Schema automático serializa datos peligrosos            | Listas blancas curadas, no dump total de RNA                                          |
| `bl_types/` y wheel divergen                            | Cambios en `replication/` deben actualizar fuente runtime y wheel juntos              |
| Nuevos continuos se cuelan en R1                        | ADR-AC-01/02 y revisión de alcance por fase                                           |
| Visibilidad se serializa pero no se publica             | DD-03 incluye publicación durante pose seleccionada                                   |
| Playhead presence se confunde con publicación de escena | Sección 5.6 y tests de no-frame-set automático                                        |
| Onboarding destructivo por defecto                      | Default Nueva escena; Combinar opt-in; purga solo confirmada                          |
| Crash preexistente de FCurve se mezcla con refactor     | Mantener OBS-F1-05 separado; no atribuirlo a la arquitectura sin evidencia            |

---

## 14. Compatibilidad e incorporación de escena

### 14.1 Host/Cliente de la misma versión R1

Debe funcionar el flujo objetivo:

```text
Host localhost/LAN
Cliente conecta por defecto en Nueva escena
Combinar en misma escena solo si el usuario lo elige explícitamente
Cliente copia se soporta como caso límite / ayuda de reconexión
Pose Live Sync ON
Object Transform Live Sync ON dentro del contrato R1
F1 OFF por defecto
Cliente reproduce en bucle
Host edita pose y transforma objetos
Keyframes reales via porcelain
Playheads/presencia via metadata, sin forzar timeline remoto
```

### 14.2 Política de incorporación propuesta

La política objetivo de R1 es:

| Caso                         | Tratamiento                                                                                                                                                                |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Nueva escena**             | Default. Crea una escena dedicada para el contenido remoto y deja la escena local intacta                                                                                  |
| **Combinar en misma escena** | Opt-in desde General settings o flujo explícito equivalente. Advertencia visible. Puede ofrecer transferir ajustes de tiempo como FPS/rango para que la animación coincida |
| **Reemplazar**               | Si se conserva, debe ser explícito y confirmado; nunca default ni automático                                                                                               |
| **Cliente copia**            | Caso límite soportado y ayuda de reconexión; no es el flujo principal recomendado                                                                                          |
| **Reconexión**               | No re-adopta ni purga automáticamente; resincroniza contenido ya adoptado                                                                                                  |

Esta política requiere enmienda normativa porque el SRS/TDD actuales describen `Combinar` como default en D-40a. No se aplica hasta aprobación.

### 14.3 Versiones mixtas

Este ADR no promete compatibilidad entre versiones mixtas más allá de lo que el handshake actual ya garantice.

Si la autoridad de Host, el schema o los mensajes live de `object_transform` requieren cambiar payload de Auth/replicación/live, debe declararse en la enmienda del TDD y tratarse como cambio versionado.

### 14.4 Wire format live

R1 debe preservar los mensajes de pose actuales:

```text
start
update
end
```

con los campos planos documentados en TDD §17.F, salvo enmienda explícita.

`object_transform` puede añadir mensajes/canales live nuevos, pero deben estar versionados y documentados. No se introduce un wire format universal para todos los continuos futuros en R1.

---

## 15. Decisiones que este ADR no toma

Este ADR no aprueba:

```text
fase 2
fase 3…13
P-01 adelantado
H-04 / VP-10
D-51
PKU
adapters live de shaders/sliders/luces/modificadores/sculpt/edit mode
canal live de presence/playheads
cambio de modelo de administración de sesión
reescritura de porcelain
```

Tampoco modifica por sí mismo SRS ni TDD. Las enmiendas necesarias se presentan en documento separado.

---

## 16. Resultado esperado tras aprobar este ADR

Si el mantenedor aprueba ADR-ARQUITECTURA-OBJETIVO-001 v1.1 o posterior:

1. Se consolidan las enmiendas propuestas al SRS/TDD/checklist/diario.
2. Se decide la ubicación de la fase arquitectónica respecto a las fases existentes.
3. Se crea el tag de inicio correspondiente.
4. Se implementa primero la puerta de publicación y el enforcement servidor con criterios de aceptación previos.
5. Se introduce schema/registry mínimo.
6. Se envuelve pose como adapter con tests de paridad.
7. Se añade `object_transform` como adapter live R1 con tests propios.
8. Se continúa con las fases R1 ya definidas, incluyendo fidelidad de visibilidad e incorporación de escena según las enmiendas aprobadas.

La frase de control para todo el trabajo posterior es:

> **R1 mantiene un alcance acotado: pose + object_transform live, porcelain durable y presencia no invasiva. La arquitectura cambia para que ese contrato sea enforced por construcción y para que R2 pueda añadir continuos sin arqueología.**
