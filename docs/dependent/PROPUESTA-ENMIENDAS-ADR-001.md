# Propuesta de enmiendas normativas · ADR-ARQUITECTURA-OBJETIVO-001

| Campo                 | Valor                                                                                                                                                                                                                                                                                                                               |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Versión del documento | **v1.3 · APROBADA · consolidada**                                                                                                                                                                                                                                                                                                   |
| Estado                | **APROBADA por el mantenedor · enmiendas aplicadas a SRS v2.2, TDD v4.8, checklist v1.8 y diario 006**                                                                                                                                                                                                                              |
| Documento origen      | `docs/ADR-ARQUITECTURA-OBJETIVO-001.md` v1.2 · APROBADO                                                                                                                                                                                                                                                                             |
| Fecha                 | 2026-11-28                                                                                                                                                                                                                                                                                                                          |
| Historial             | v1.0 borrador commiteado por el mantenedor para generar historial; v1.1 incorpora decisiones aprobadas: fases separadas, retiro de `Reemplazar`, `Combinar` oculto en General settings y casilla única de ajustes de tiempo; v1.2 borrador alineado con ADR v1.2 aprobado; v1.3 aprobación del mantenedor y consolidación normativa |
| SRS afectado          | v2.1.1 → **v2.2 consolidado**                                                                                                                                                                                                                                                                                                       |
| TDD afectado          | v4.7 → **v4.8 consolidado**                                                                                                                                                                                                                                                                                                         |
| Checklist afectado    | v1.7 → **v1.8 consolidado**                                                                                                                                                                                                                                                                                                         |
| Código afectado       | Ninguno en este documento                                                                                                                                                                                                                                                                                                           |

---

## 1. Estado de aprobación

El ADR ya está aprobado:

```text
ADR-ARQUITECTURA-OBJETIVO-001.md v1.2 · APROBADO
```

Esta propuesta de enmiendas **está aprobada por el mantenedor** y consolidada. El ADR v1.2 fue aprobado primero; después el mantenedor aprobó esta propuesta y autorizó aplicar las enmiendas al SRS, TDD, checklist y diario. La consolidación no modifica código ni abre fases 1b/1c.

Decisiones ya tomadas por el mantenedor y reflejadas aquí:

```text
1. Fases separadas: 1b y 1c.
2. Se retira `Reemplazar`.
3. Default: `Nueva escena`.
4. `Combinar` queda oculto en General settings para usuarios avanzados.
5. Casilla única explícita para sincronizar ajustes de tiempo al combinar.
```

---

## 2. Alcance aprobado que debe reflejarse en SRS/TDD

R1 queda como:

```text
Live Sync de pose
+ Live Sync de transformación de objetos
+ porcelain para datos definitivos
+ presencia/playhead como metadata no invasiva
```

Adapters live R1:

```text
pose
object_transform
```

No entran en R1:

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

---

## 3. SRS v2.1.1 → v2.2 propuesta

### 3.1 Enmienda de alcance: §0.2

Añadir a la tabla de alcance:

```markdown
| **Live Sync de transformación de objetos** | **En alcance en R1.** G/R/S de objetos seleccionados se previsualiza en vivo sin commits continuos. Los datos definitivos llegan por porcelain. |
```

Modificar la disposición de Funcionalidad 2:

```markdown
Funcionalidad 2 — Live Sync de pose y transformación de objetos (§5)
```

### 3.2 Nueva sección propuesta: §5.12 Live Sync de transformación de objetos

Texto normativo propuesto:

```markdown
#### 5.12 LIVE SYNC DE TRANSFORMACIÓN DE OBJETOS

R1 soporta previsualización live de transformaciones interactivas de objetos:

- G/R/S sobre objeto(s) seleccionados.
- Multi-objeto.
- Preview reversible en el Cliente.
- Sin porcelain.commit/push continuo durante el arrastre real.
- Keyframes reales y datos definitivos por porcelain.
- El timeline del Cliente no se pausa ni se altera automáticamente.

La entidad live es `object_uuid`.

Reglas observables:

1. Seleccionar objeto(s) transformable(s) abre o actualiza la previsualización en el frame del Host.
2. Arrastrar G/R/S actualiza la previsualización en vivo.
3. Cancelar restaura el estado original.
4. Confirmar sin insertar keyframe no deja transformación fantasma al cerrar/deseleccionar.
5. Insertar un keyframe real produce dato durable por porcelain.
6. Scrub con objeto seleccionado sigue al frame del Host sin mover el playhead del Cliente.
7. Desconexión limpia/restaura previews.
8. Un apply durable durante preview vivo invalida el preview obsoleto sin restaurar valores viejos.
9. La transformación de objeto respeta padres, constraints y animación existente; no hace action swapping ni mute de curvas.
10. Si no hay animación gobernante, el preview es reversible y no crea Actions/keyframes permanentes por accidente.
```

### 3.3 Enmienda a §7: restricciones extendidas a objetos

Añadir:

```markdown
Las restricciones de no pausar el Cliente, no silenciar curvas, no action swapping, no keyframes masivos y no dejar overlays fantasma aplican también a `object_transform`.

Durante G/R/S real de objeto no se usa porcelain.commit/push como canal continuo.
```

### 3.4 Enmienda a §2.3: incorporación de escena

Sustituir la tabla de opciones por:

```markdown
| Opción                                      | Qué hace                                                                                                                                                                                                                                          |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Nueva escena** _(por defecto)_            | Crea una escena dedicada para el contenido remoto y deja la escena local intacta. La adopción excluye objetos ya usados por escenas locales. Es el flujo recomendado para el usuario novato.                                                      |
| **Combinar** _(oculto en General settings)_ | Conserva lo local y añade contenido del Host en la misma escena. Es un opt-in avanzado explícito, con advertencia visible. Ofrece una casilla única para sincronizar ajustes de tiempo necesarios para que la animación coincida, como FPS/rango. |
```

Eliminar `Reemplazar` como opción de incorporación.

Añadir o sustituir restricciones:

```markdown
El addon no ofrece una opción de incorporación que borre o purgue contenido local. `Reemplazar` queda retirado porque implicaba una operación destructiva.

El sistema no purga contenido por su cuenta, por ningún motivo. Ninguna ruta de conexión puede vaciar la escena del Cliente.

El flujo principal recomendado no exige copiar el `.blend` del Host. Conectar desde una copia se soporta como caso límite y ayuda de reconexión, no como recomendación principal.

La opción `Combinar` vive oculta en General settings para usuarios que saben lo que hacen. Si se activa, debe mostrar advertencia de que puede mezclar contenido local y remoto. La sincronización de FPS/rango u otros ajustes de tiempo se ofrece como una casilla única explícita, no como sobrescritura silenciosa.
```

### 3.5 Enmienda de presencia/playhead

Añadir nota normativa:

```markdown
El playhead de cada peer se transmite como metadato de presencia. Mostrar el frame de otro peer en la UI no autoriza mover automáticamente el timeline local. Ningún receptor ejecuta frame_set() ni asigna frame_current automáticamente por presencia. “Snap to user time” sigue siendo acción explícita del usuario.
```

### 3.6 Nuevos criterios de aceptación SRS

| ID    | Criterio                                                                                                                                     |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| AC-15 | G/R/S de objeto en Host produce preview live en Cliente sin commits continuos                                                                |
| AC-16 | Cancelar/deseleccionar/desconectar no deja transformación de objeto fantasma                                                                 |
| AC-17 | Keyframes reales de objeto llegan por porcelain                                                                                              |
| AC-18 | Playheads/presencia no fuerzan timeline remoto                                                                                               |
| AC-19 | Default de conexión es Nueva escena; Combinar está oculto en General settings y es opt-in avanzado; no existe Reemplazar ni purga automática |

---

## 4. TDD v4.7 → v4.8 propuesta

### 4.1 Nuevas decisiones de diseño

| ID       | Decisión propuesta                                                                                                                                                                                                                                                       | Estado propuesto | Justificación                                           |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------- | ------------------------------------------------------- |
| **D-52** | Toda salida de replicación compartida pasa por `PublicationPolicy` central. Guardas fase 1 se conservan como defensa inicial.                                                                                                                                            | R1               | INV-1/AC-07 por construcción                            |
| **D-53** | Servidor rechaza mutaciones de grafo compartido de viewers. Autoridad Host ligada a conexión autoritativa, no a owner/admin/COMMON auto-declarados.                                                                                                                      | R1               | Defensa en profundidad                                  |
| **D-54** | `ReplicationSchema`/registry declarativo mínimo, versionado y curado. DD-03 y object_transform usan entries trazables.                                                                                                                                                   | R1               | Fidelidad mantenible                                    |
| **D-55** | `InteractionRegistry` + contrato de adapter. R1 registra `pose` y `object_transform`.                                                                                                                                                                                    | R1               | Preparar R2 sin framework universal                     |
| **D-56** | Frontera R1/R2: R1 = pose + object_transform live; shaders/sliders/otros continuos quedan R2+.                                                                                                                                                                           | R1/R2+           | Alcance acotado                                         |
| **D-57** | Adapter live `object_transform`: G/R/S de objeto, multi-objeto, preview reversible, sin commits continuos, durable por porcelain.                                                                                                                                        | R1               | Aclaración aprobada del mantenedor                      |
| **D-58** | Playheads/presencia siguen como `UpdateUserMetadata`/porcelain command channel en R1. No fuerzan timeline remoto. Canal live de presencia queda futuro opcional.                                                                                                         | R1               | Evita migración innecesaria y protege bucle del Cliente |
| **D-59** | Incorporación: default `Nueva escena`; `Combinar` oculto en General settings como opt-in avanzado con advertencia y casilla única de ajustes de tiempo; se retira `Reemplazar`; copia como caso límite/reconexión. Enmienda D-40a y retira D-41 como opción destructiva. | R1               | Decisión aprobada del mantenedor                        |

### 4.2 Enmienda a D-40a/D-41/D-50

D-40a debe reescribirse:

```markdown
D-40a · Al conectar, el comportamiento por defecto es `Nueva escena`. `Combinar` existe como opt-in avanzado oculto en General settings, con advertencia visible y casilla única explícita para sincronizar ajustes de tiempo. No existe opción `Reemplazar`.
```

D-41 debe marcarse como retirada o reescribirse:

```markdown
D-41 · RETIRADA como opción de incorporación. No se ofrece purga/reemplazo de escena activa desde el flujo de conexión.
```

D-50 se conserva y refuerza:

```markdown
D-50 · Se retira la purga automática de la escena del Cliente. Ninguna contraseña, preset ni ruta de conexión purga contenido local.
```

### 4.3 Nueva sección propuesta: §18 Arquitectura objetivo

Añadir sección normativa técnica con:

```text
18.1 Alcance R1: pose + object_transform + porcelain durable
18.2 Carriles: live preview / durable / local-only-presence
18.3 PublicationPolicy
18.4 Server authority enforcement
18.5 ReplicationSchema/registry
18.6 InteractionRegistry y adapters
18.7 Presence/playhead
18.8 Incorporación de escena sin Reemplazar
18.9 Paridad y migración incremental
```

### 4.4 Enmienda a §10.5 Publicación restringida al Host

Añadir:

```markdown
**D-52/D-53 — cierre arquitectónico.** Las guardas de fase 1 cierran rutas conocidas, pero R1 también requiere puerta central de salida y enforcement en servidor. `porcelain.commit` local no se confunde con publicación. `UpdateUserMetadata` de presencia se permite dentro de límites explícitos y no puede usarse para mutar escena.
```

### 4.5 Enmienda a §10.9 DD-03

Añadir:

```markdown
**D-54 — schema declarativo.** Los campos DD-03 deben expresarse mediante entries versionadas del ReplicationSchema o serializadores legacy trazables. La corrección incluye carga por capa, publicación durante pose seleccionada y reconciliación de hide_select.
```

### 4.6 Enmienda a §10.10 Orden de ejecución

Insertar fases separadas después de fase 1:

```markdown
| 1b | D-52, D-53, D-54, D-55, D-58 · puerta única, enforcement servidor, schema/registry mínimo, contrato de adapters, pose parity, presence/playhead no invasivo | INV-1, AC-06, AC-07, mantenibilidad |
| 1c | D-57 · adapter live object_transform con tests de no-commits continuos, multi-objeto, reversibilidad y handoff durable | R1 object live |
```

Nota:

```markdown
Las fases 1b y 1c son separadas por decisión del mantenedor.
D-59 se implementa en la fase de onboarding correspondiente o en la fase que el mantenedor autorice, pero es obligación R1.
D-56 excluye adapters live de shaders/sliders/otros continuos de R1.
```

### 4.7 Enmienda a §14 Modelo de amenazas

Añadir:

```markdown
| S-05 | Viewer autenticado intenta mutar grafo compartido mediante Commit/Node/Delete/Right forjado o ruta no cubierta por guardas UI | D-53: servidor rechaza antes de aplicar/reenviar |
| S-06 | Presence metadata se usa para forzar timeline remoto o colar datos de escena | D-58: presence no ejecuta frame_set/frame_current automático; schema de metadata permitido |
```

### 4.8 Nuevas verificaciones propuestas

| ID        | Verificación                                           | Método                                                                |
| --------- | ------------------------------------------------------ | --------------------------------------------------------------------- |
| **VP-16** | Servidor rechaza mutación compartida de viewer forjado | Test integración servidor local                                       |
| **VP-17** | Paridad de adapter de pose                             | Comparar contra `r1-fase-01`                                          |
| **VP-18** | Schema DD-03 round-trip                                | Dump/load flags por capa                                              |
| **VP-19** | object_transform no produce porcelain continuo         | Contadores de commit/push durante G/R/S                               |
| **VP-20** | object_transform multi-objeto y reversibilidad         | Dos peers, selección múltiple, cancel/confirm/deselect                |
| **VP-21** | Presence/playhead no invasivo                          | Recibir metadata no cambia frame_current local                        |
| **VP-22** | Incorporación no destructiva                           | Default Nueva escena; Combinar oculto/opt-in; no Reemplazar; no purga |

### 4.9 Nuevos criterios de aceptación TDD

| ID        | Criterio                                                                         |
| --------- | -------------------------------------------------------------------------------- |
| **AC-15** | Servidor rechaza mutaciones compartidas de viewers aunque mensaje forjado        |
| **AC-16** | Registry live R1 contiene solo pose y object_transform                           |
| **AC-17** | Pose adapter pasa paridad contra r1-fase-01                                      |
| **AC-18** | object_transform live no produce commits continuos y es reversible               |
| **AC-19** | Playheads/presencia no fuerzan timeline remoto                                   |
| **AC-20** | Default Nueva escena; Combinar oculto/opt-in; no Reemplazar; no purga automática |

### 4.10 Enmienda a §17.F Esquema de mensajes live

Preservar mensajes de pose actuales.

Añadir subsección propuesta:

```markdown
#### 17.F.2 Mensajes live de object_transform

Pendiente de especificación exacta en fase 1c. Debe incluir como mínimo:

- channel/kind: object_transform
- action: start / update / end
- object_uuid
- frame del Host
- canales de transformación necesarios
- rotation_mode cuando aplique
- información de cancelación/restauración

No debe romper mensajes de pose existentes.
```

### 4.11 Checklist derivado

Actualizar `CHECKLIST-IMPLEMENTACION.md`:

```text
- añadir D-52…D-59
- añadir fases 1b y 1c en §19
- añadir VP-16…VP-22 o pruebas equivalentes
- añadir AC-15…AC-20
- marcar R1 pendientes hasta evidencia
- marcar D-56 frontera R2+ para continuos excluidos
- marcar D-41 retirada/reescrita según consolidación
- recuento re-medido por verificar_docs.py
```

### 4.12 Diario derivado

Añadir entrada si se aprueba esta propuesta:

```text
006 · ADR-ARQUITECTURA-OBJETIVO-001 v1.2 aprobado y enmiendas normativas consolidadas
```

Debe registrar:

```text
- object_transform entra en R1
- pose parity obligatoria
- puerta única + servidor enforcement
- schema/registry mínimo
- presence/playhead sigue metadata
- default Nueva escena
- Combinar oculto en General settings
- Reemplazar retirado
- casilla única de ajustes de tiempo
- fases 1b y 1c separadas
- enmiendas SRS/TDD/checklist aplicadas
- código sin tocar hasta orden de fase
```

---

## 5. Lo que esta propuesta no hace

Esta propuesta no:

```text
modifica código
modifica SRS/TDD/checklist por sí misma
aprueba fase 2
abre fases 1b/1c automáticamente
añade adapters live de shaders/sliders/luces/modificadores/sculpt/edit mode
migra playheads a canal live
aprueba P-01 adelantado
aprueba D-51
aprueba PKU
```

---

## 6. Orden de aprobación ejecutado

1. ADR v1.2 aprobado por el mantenedor.
2. Propuesta v1.2 leída y aprobada por el mantenedor.
3. Enmiendas consolidadas en SRS v2.2, TDD v4.8, checklist v1.8 y diario 006.
4. `verificar_docs.py` ejecutado como comprobación mecánica.
5. Quedan pendientes de autorización separada las órdenes de fase `1b` y `1c`.
6. No se crean tags de inicio ni parches hasta que el mantenedor autorice cada fase.

**Nota de consolidación.** Las amenazas nuevas se consolidan como `S-05` y `S-06` porque `S-04` ya existía en el TDD (traversal de NAT). Los criterios de aceptación añadidos se mantienen en dos series documentales: `SRS AC-15…AC-19` son criterios de requisito; `TDD AC-15…AC-20` son criterios técnicos de diseño/verificación. Las referencias cruzadas deben usar el prefijo del documento para evitar ambigüedad.

---

## 7. Decisiones cerradas por el mantenedor

| Pregunta                       | Decisión                                                     |
| ------------------------------ | ------------------------------------------------------------ |
| ¿Fases separadas o fusionadas? | **Separadas: 1b y 1c**                                       |
| ¿Se conserva `Reemplazar`?     | **No. Se retira por peligroso/destructivo**                  |
| ¿Default de incorporación?     | **Nueva escena**                                             |
| ¿Dónde vive `Combinar`?        | **Oculto en General settings, para usuarios avanzados**      |
| ¿Transferencia de FPS/rango?   | **Casilla única explícita de sincronizar ajustes de tiempo** |
