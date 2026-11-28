# ADR-001 · Arquitectura objetivo mantenible para True Level: Playblast

| Campo                     | Valor                                                                                        |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| Estado                    | **PROPUESTO · pendiente de aprobación del mantenedor**                                       |
| Fecha                     | 2026-11-28                                                                                   |
| Decisión de alcance       | **Opción A** aprobada por el mantenedor: R1 no amplía su alcance funcional                   |
| Código de referencia      | `blender-sync` `d62755a` · tag `r1-fase-01`                                                  |
| SRS de referencia         | `workspace-3/consolidated/SRS.md` v2.1.1                                                     |
| TDD de referencia         | `workspace-3/consolidated/TDD-PLAYBLAST-001.md` v4.7                                         |
| Fase 1                    | Cerrada y verificada; sus guardas se conservan como red de seguridad inicial                 |
| Modificaciones realizadas | Ninguna sobre código, SRS ni TDD. Este ADR y la propuesta de enmiendas son documentos nuevos |

---

## 1. Resumen ejecutivo

R1 sigue siendo:

> **Live Sync de pose + porcelain para datos definitivos.**

Pose sigue siendo el **único adapter live soportado en R1**. No se añaden adapters live para objetos, shaders, sliders ni otros continuos.

La arquitectura debe quedar preparada para que R2 o posteriores puedan añadir un nuevo continuo mediante:

```text
schema declarativo
+ interaction registry
+ adapter por tipo
+ política de lane
+ puerta única de publicación
```

con cambios mínimos y localizados en el núcleo y en los handlers existentes.

La migración será **incremental**, no una reescritura. El comportamiento soportado de pose no debe cambiar. Las correcciones de defectos ya exigidas por SRS/TDD, como fidelidad de visibilidad, sí deben implementarse, pero sin ampliar el alcance funcional de R1.

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

### 2.3 El problema de los continuos no está resuelto de forma general

`live_sync.py` resolvió el caso de pose mediante un canal TCP dedicado y overlay temporal reversible.

Pero `handlers.py` todavía puede publicar cambios detectados por depsgraph para nodos rastreados:

```python
if is_live or node.state == UP:
    porcelain.commit(...)
    porcelain.push(...)
```

El gate de arrastre actual está orientado a transformaciones interactivas conocidas, no a todos los cambios RNA continuos. Un slider de shader puede producir muchos cambios sin pasar por ese gate.

Por eso R1 no debe prometer “todo Blender continuo”. Debe mantener pose como único live adapter y dejar la arquitectura preparada para añadir otros después.

### 2.4 La fase 1 cerró rutas, pero no creó una capacidad central

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
| **Live preview**          | Cambios efímeros de interacción continua          | No definitivo; debe restaurarse o ser reemplazado por dato durable | Solo pose                  |
| **Durable transaction**   | Estado confirmado o asentado que debe conservarse | Definitivo; llega por porcelain                                    | Sí, fundamental            |
| **Local-only / presence** | Estado local del peer y metadatos de presencia    | No muta la escena compartida                                       | Sí, con límites explícitos |

Regla rectora:

> **Live preview no es publicación. Porcelain no es canal de arrastre.**

### 3.2 Cuatro contratos

| Contrato                                     | Pregunta que responde                                                        | R1                                                              |
| -------------------------------------------- | ---------------------------------------------------------------------------- | --------------------------------------------------------------- |
| **PublicationPolicy / puerta única**         | ¿Quién puede enviar qué mensaje de replicación?                              | Obligatorio                                                     |
| **ReplicationSchema / registry declarativo** | ¿Qué propiedades se replican y con qué versión?                              | Obligatorio como contrato mínimo; cobertura completa progresiva |
| **InteractionRegistry**                      | ¿Qué interacciones existen y cómo se observan?                               | Obligatorio como contrato; solo pose registrado como live       |
| **Adapter**                                  | ¿Cómo se previsualiza, aplica, restaura y entrega a durable una interacción? | Obligatorio para pose; interfaz preparada para futuros adapters |

---

## 4. Arquitectura objetivo

```text
HOST
────
Sensores existentes
  ├─ depsgraph_update_post
  ├─ contexto de operador modal cuando exista
  ├─ selección / modo / frame para pose
  └─ undo/redo
        │
        ▼
InteractionRegistry + LanePolicy
        │
        ├── LIVE PREVIEW LANE
        │     ├─ R1: pose adapter únicamente
        │     ├─ start / update / end / cancel
        │     ├─ coalescing por entidad/canal cuando aplique
        │     ├─ sin porcelain.commit/push durante arrastre real
        │     └─ overlay reversible según SRS/TDD
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
  └─ Cliente: denegado para mutación de grafo compartido
        │
        ▼
replication / porcelain / servidor
        │
        ▼
Server Authority Policy
  ├─ conexión autoritativa de Host: puede mutar grafo compartido
  └─ viewer autenticado: no puede mutar grafo compartido
        │
        ▼
CLIENTE
───────
  ├─ recibe live preview de pose
  ├─ recibe estado durable por porcelain
  ├─ aplica mediante adapters/serializadores
  ├─ puede seleccionar e inspeccionar
  ├─ puede reproducir su propio timeline
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
| **Cliente local** | Evitar que rutas honestas del addon publiquen desde un viewer                                                                                |
| **Servidor**      | Rechazar mutaciones de grafo compartido aunque un cliente modificado, una ruta futura o un mensaje forjado intente saltarse el cliente local |

El servidor no debe confiar en:

```text
owner declarado en el mensaje
username declarado por el peer
RP_COMMON
auth_type == ADMIN por sí solo
```

Debe existir una noción de **conexión autoritativa de Host** ligada a la sesión autenticada.

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
  live_adapter: None | "pose" | futuro adapter
  lane: durable | live+durable | local-only
  tests: roundtrip / parity / acceptance
```

### 7.3 R1 mínimo obligatorio

En R1 debe existir el contrato y un registry mínimo que permita:

1. Registrar el adapter live de pose.
2. Declarar los campos de fidelidad exigidos por DD-03.
3. Versionar cambios de schema.
4. Ejecutar tests de paridad/round-trip.
5. Mantener los serializadores legacy como adapters envueltos mientras se migran.

R1 no tiene que migrar los 23+ tipos de `bl_types/` de golpe. La migración puede ser por tipo, empezando por los necesarios para:

```text
pose
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

### 8.2 R1: solo pose

El único adapter live registrado en R1 es:

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

### 8.3 Contrato de adapter

Un adapter live futuro debe poder implementar, conceptualmente:

```text
observe_host(context) -> change
make_preview(change) -> live_message
apply_preview(client, message) -> reversible_edit
close_preview(client, reason)
invalidate_preview(client, durable_apply)
durable_handoff(change) -> schema/porcelain
```

En R1, el adapter de pose puede ser una envoltura incremental sobre `HostPoseStreamer` y `ClientPoseApplier`. No se exige reescribir esos componentes si la envoltura preserva comportamiento y pasa paridad.

### 8.4 Lo que no entra en R1

No se registran ni soportan como live adapters en R1:

```text
object_transform
shader_slider
light_energy
modifier_value
dope_sheet_drag
sculpt_continuous
edit_mode_continuous
geometry_nodes_slider
```

Algunos de esos cambios pueden seguir llegando por porcelain como datos definitivos si el flujo actual ya los publica. Lo que no se añade es un nuevo canal live continuo para ellos.

---

## 9. LanePolicy y transacciones durables

### 9.1 R1

R1 conserva dos caminos reales:

```text
pose live → live_sync TCP
dato definitivo → porcelain
```

No se introduce un scheduler genérico de coalescing para todos los cambios RNA.

La política R1 debe expresar:

| Cambio                                          | Lane R1                                                  |
| ----------------------------------------------- | -------------------------------------------------------- |
| Pose seleccionada/arrastrada                    | live preview                                             |
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

En R1 esto se aplica al flujo ya soportado. La generalización a sliders y otros continuos queda para R2.

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

R2 podrá añadir revisiones/seq más generales, pero R1 no debe cambiar el formato live de pose sin una enmienda explícita.

---

## 10. Qué entra en R1 y qué queda para después

### 10.1 Entra en R1

| Área                        | Obligación R1                                                 |
| --------------------------- | ------------------------------------------------------------- |
| Alcance funcional           | Sin cambios: pose live + porcelain definitivo                 |
| Adapter live                | Pose únicamente                                               |
| Puerta local de publicación | Política central de salida para replicación compartida        |
| Guardas fase 1              | Se conservan hasta que la puerta central esté verificada      |
| Enforcement servidor        | Rechazo de mutaciones de grafo compartido por viewers         |
| Autoridad Host              | Conexión autoritativa no auto-declarada por el cliente        |
| Schema/registry             | Contrato mínimo versionado + registry de pose + campos DD-03  |
| Fidelidad visibilidad       | Cumplir SRS §3.9 / TDD DD-03 / AC-06                          |
| `hide_select`               | No gobernarse por readonly/propiedad; reconciliar timer       |
| Paridad pose                | Tests de paridad antes/después de extraer adapter             |
| Compatibilidad              | No romper Host actual ni Cliente local/selección/reproducción |
| Documentación               | TDD/checklist/diario actualizados solo tras aprobación        |

### 10.2 Queda para R2 o posteriores

| Área                                            | Motivo                                                                 |
| ----------------------------------------------- | ---------------------------------------------------------------------- |
| Nuevos adapters live: objetos, shaders, sliders | Ampliación de alcance; requiere nuevos criterios y mediciones          |
| Sensor RNA/msgbus general                       | No necesario para pose R1; tiene limitaciones y requiere diseño propio |
| Coalescing/settle genérico para continuos       | Depende de adapters futuros                                            |
| Schema completo de todos los tipos Blender      | Migración progresiva; no big-bang                                      |
| Transaction scheduler genérico multi-tipo       | R1 conserva flujo durable actual                                       |
| Wire format live genérico                       | R1 preserva mensajes de pose actuales                                  |
| PKU / D-51 / keyframe azul                      | Sigue diferido según TDD                                               |
| Internet/TLS/relay                              | Fuera de R1                                                            |
| Reescritura de `bl_types/` o `replication/`     | Contrario a migración incremental                                      |

### 10.3 R1 no promete “todo Blender continuo”

R1 puede publicar muchos cambios por porcelain, pero solo promete previsualización live continua para pose.

Esta frase debe aparecer en cualquier documento derivado:

> **R1 no es un sistema universal de live editing. Es un sistema de live pose review con transporte durable fiel y una arquitectura preparada para futuros adapters.**

---

## 11. Migración incremental propuesta

No se ejecuta todavía. Es el plan que requerirá aprobación y órdenes de fase separadas.

### Paso 0 · Aprobar ADR y enmiendas

Antes de tocar código:

```text
aprobar ADR-001
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
No se rompe localhost ni LAN.
```

### Paso 4 · Schema/registry mínimo

Introducir contrato declarativo sin migrar todo de golpe.

Criterio:

```text
pose registrado como único live adapter.
DD-03 tiene schema/entries versionadas.
Serializadores legacy siguen funcionando.
Round-trip/parity tests pasan.
```

### Paso 5 · Adapter de pose envuelto

Extraer/registrar pose como adapter sin cambiar semántica.

Criterio:

```text
mismos mensajes live.
mismo overlay.
misma restauración.
misma invalidación.
mismos keyframes definitivos por porcelain.
sin regresión de latencia/cadencia.
```

### Paso 6 · Continuar fases R1 existentes

Las fases 2–13 no se aprueban automáticamente por este ADR. Se decides una a una, usando los nuevos contratos.

DD-03 debe implementarse en su fase correspondiente o en la fase que el mantenedor autorice, pero sigue siendo obligación de R1.

---

## 12. Criterios de aceptación del ADR

Estos criterios se proponen para la fase arquitectónica. No sustituyen los AC del SRS/TDD.

### 12.1 Alcance

| ID        | Criterio                                                           |
| --------- | ------------------------------------------------------------------ |
| ADR-AC-01 | El registry live de R1 contiene únicamente pose                    |
| ADR-AC-02 | No se añade ningún mensaje live para objetos/shaders/sliders       |
| ADR-AC-03 | El formato live de pose no cambia sin enmienda explícita           |
| ADR-AC-04 | F1 sigue fuera de soporte; no se convierte en live adapter general |

### 12.2 Publicación

| ID        | Criterio                                                                         |
| --------- | -------------------------------------------------------------------------------- |
| ADR-AC-05 | Toda salida de mutación compartida pasa por política local                       |
| ADR-AC-06 | Cliente no puede enviar Node/Commit/Delete/Right por rutas honestas              |
| ADR-AC-07 | Servidor rechaza mutaciones compartidas de viewer aunque el mensaje esté forjado |
| ADR-AC-08 | Host conserva llamadas y semántica de publicación existentes                     |
| ADR-AC-09 | Presence/Auth/TTL/snapshot/apply/fetch no se bloquean por error                  |
| ADR-AC-10 | Las guardas de fase 1 siguen pasando hasta ser sustituidas formalmente           |

### 12.3 Schema y fidelidad

| ID        | Criterio                                                                                                 |
| --------- | -------------------------------------------------------------------------------------------------------- |
| ADR-AC-11 | Existe schema/registry versionado mínimo                                                                 |
| ADR-AC-12 | DD-03 se expresa declarativamente o mediante entries trazables                                           |
| ADR-AC-13 | Object/Collection/Armature/ViewLayer restauran flags exigidos por SRS §3.9                               |
| ADR-AC-14 | `hide_select` no se fuerza desde readonly/propiedad                                                      |
| ADR-AC-15 | Cambios DD-03 llegan al Cliente incluso si hay hueso seleccionado, sin retirar protección de rendimiento |

### 12.4 Paridad de pose

| ID        | Criterio                                                                                    |
| --------- | ------------------------------------------------------------------------------------------- |
| ADR-AC-16 | Selección sin mover → deseleccionar no deja rastro                                          |
| ADR-AC-17 | Transformar sin keyear → deseleccionar restaura                                             |
| ADR-AC-18 | Keyframe real llega por porcelain y no queda fantasma                                       |
| ADR-AC-19 | Scrub con hueso seleccionado mueve overlay al frame del Host sin mover playhead del Cliente |
| ADR-AC-20 | Desconexión/reconexión limpia y reconstruye solo lo activo                                  |
| ADR-AC-21 | Apply durable durante overlay vivo no restaura valores obsoletos                            |
| ADR-AC-22 | Multi-hueso/multi-armadura no colisiona                                                     |
| ADR-AC-23 | Cadencia/latencia no empeora respecto a la línea de paridad                                 |

---

## 13. Riesgos y mitigaciones

| Riesgo                                                | Mitigación                                                                 |
| ----------------------------------------------------- | -------------------------------------------------------------------------- |
| Sobre-ingeniería: convertir R1 en framework universal | R1 solo registra pose; registry mínimo; sin sensores RNA generales         |
| Romper pose al extraer adapter                        | Paridad antes/después; envoltura incremental; feature flag si hace falta   |
| Puerta local rompe presencia o snapshot               | Clasificación explícita de mensajes permitidos; tests de control positivo  |
| Autoridad servidor basada en `admin` o `owner`        | Requisito explícito: autoridad ligada a conexión, no auto-declarada        |
| Schema automático serializa datos peligrosos          | Listas blancas curadas, no dump total de RNA                               |
| `bl_types/` y wheel divergen                          | Cambios en `replication/` deben actualizar fuente runtime y wheel juntos   |
| Nuevos continuos se cuelan en R1                      | ADR-AC-01/02/03 y revisión de alcance por fase                             |
| Visibilidad se serializa pero no se publica           | DD-03 incluye publicación durante pose seleccionada                        |
| Crash preexistente de FCurve se mezcla con refactor   | Mantener OBS-F1-05 separado; no atribuirlo a la arquitectura sin evidencia |

---

## 14. Compatibilidad

### 14.1 Host/Cliente de la misma versión R1

Debe funcionar el flujo actual:

```text
Host localhost/LAN
Cliente copia / escena nueva / contenido propio según SRS
Pose Live Sync ON
F1 OFF por defecto
Cliente reproduce en bucle
Host edita pose
Keyframes reales via porcelain
```

### 14.2 Versiones mixtas

Este ADR no promete compatibilidad entre versiones mixtas más allá de lo que el handshake actual ya garantice.

Si la autoridad de Host o el schema requieren cambiar payload de Auth/replicación, debe declararse en la enmienda del TDD y tratarse como cambio versionado.

### 14.3 Wire format live de pose

R1 debe preservar los mensajes actuales:

```text
start
update
end
```

con los campos planos documentados en TDD §17.F, salvo enmienda explícita.

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
nuevos adapters live
cambio de alcance R1
reescritura de porcelain
cambio de modelo de administración de sesión
```

Tampoco modifica por sí mismo SRS ni TDD. Las enmiendas necesarias se presentan en documento separado.

---

## 16. Resultado esperado tras aprobar este ADR

Si el mantenedor aprueba ADR-001:

1. Se consolidan las enmiendas propuestas al TDD/checklist/diario.
2. Se decide la ubicación de la fase arquitectónica respecto a las fases existentes.
3. Se crea el tag de inicio correspondiente.
4. Se implementa primero la puerta de publicación y el enforcement servidor con criterios de aceptación previos.
5. Se introduce schema/registry mínimo.
6. Se envuelve pose como adapter con tests de paridad.
7. Se continúa con las fases R1 ya definidas, sin ampliar alcance.

La frase de control para todo el trabajo posterior es:

> **R1 mantiene su contrato. La arquitectura cambia para que ese contrato sea enforced por construcción y para que R2 pueda añadir continuos sin arqueología.**
