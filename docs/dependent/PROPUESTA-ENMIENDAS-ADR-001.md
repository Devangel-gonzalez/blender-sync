# Propuesta de enmiendas normativas · ADR-ARQUITECTURA-OBJETIVO-001

| Campo                 | Valor                                               |
| --------------------- | --------------------------------------------------- |
| Versión del documento | **v1.1 · borrador alineado con ADR v1.1**           |
| Estado                | **PROPUESTA SEPARADA · pendiente de aprobación**    |
| Documento origen      | `docs/ADR-ARQUITECTURA-OBJETIVO-001.md` v1.1        |
| Fecha                 | 2026-11-28                                          |
| SRS afectado          | v2.1.1 · propuesta de enmienda a v2.2 si se aprueba |
| TDD afectado          | v4.7 · propuesta de enmienda a v4.8 si se aprueba   |
| Checklist afectado    | v1.7 · propuesta de actualización derivada          |
| Código afectado       | Ninguno en este documento                           |

---

## 1. Principio de esta propuesta

El ADR v1.1 revisa la opción A inicial con una aclaración del mantenedor:

> **R1 incluye `object_transform` como adapter live, además de pose.**

Por tanto, esta propuesta sí contiene enmiendas normativas de alcance:

```text
R1 = Live Sync de pose + Live Sync de transformación de objetos + porcelain durable
```

No incluye shaders, sliders, luces, modificadores, sculpt, edit mode ni otros continuos.

También propone enmiendas para:

- presencia/playheads como metadata no invasiva;
- incorporación de escena con default `Nueva escena`;
- `Combinar` como opt-in explícito;
- cliente copia como caso límite/reconexión;
- puerta única de publicación y enforcement servidor;
- schema/registry y adapters.

---

## 2. Versionado y commit recomendado

### 2.1 ADR

El ADR v1.0 ya fue subido por el mantenedor. Se conserva como hito histórico de propuesta inicial.

La versión actualizada es:

```text
ADR-ARQUITECTURA-OBJETIVO-001.md v1.1 · PROPUESTO
```

Recomendación:

```text
commitear v1.1 como propuesta revisada, no como aprobada
```

Cuando el mantenedor apruebe, se puede marcar:

```text
v1.2 · APROBADO
```

o la versión que el mantenedor prefiera, pero el estado debe cambiar explícitamente.

### 2.2 Propuesta de enmiendas

No recomiendo marcar la propuesta de enmiendas como `v1.0 definitiva` antes de que el ADR esté aprobado.

Opciones limpias:

| Opción                      | Cuándo usarla                                                                                     |
| --------------------------- | ------------------------------------------------------------------------------------------------- |
| **Esperar**                 | Si no necesitas historial de borradores. Commitear la propuesta solo cuando el ADR esté aprobado. |
| **Commitear como borrador** | Si quieres trazabilidad. Usar `v0.9-draft` o `v1.1-draft`, nunca como aprobada.                   |

Recomendación por defecto:

```text
ADR v1.1 → commit como propuesta revisada
Propuesta de enmiendas v1.1 → commit como borrador si se quiere historial; si no, esperar aprobación del ADR
```

---

## 3. SRS v2.1.1 → v2.2 propuesta

### 3.1 Enmienda de alcance: §0.2

Añadir a la tabla de alcance:

```markdown
| **Live Sync de transformación de objetos** | **En alcance en R1.** G/R/S de objetos seleccionados se previsualiza en vivo sin commits continuos. Los datos definitivos llegan por porcelain. |
```

Modificar la disposición de Funcionalidad 2 para que no se lea como “solo Modo Pose”:

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

Cambiar el default propuesto por D-40a:

```markdown
| **Nueva escena** _(por defecto)_ | Crea una escena dedicada y deja la local intacta. La adopción excluye objetos ya usados por escenas locales. |
| **Combinar** _(opt-in explícito)_ | Conserva lo local y añade contenido del Host. Requiere elección explícita y advertencia visible. Puede ofrecer transferir ajustes de tiempo necesarios para que la animación coincida, como FPS/rango, bajo elección del usuario. |
| **Reemplazar** | Purga la escena activa y datos huérfanos solo con confirmación explícita. Nunca es default. |
```

Añadir:

```markdown
El flujo principal recomendado no exige copiar el `.blend` del Host. Conectar desde una copia se soporta como caso límite y ayuda de reconexión, no como recomendación principal.

La opción de combinar en la misma escena puede vivir en General settings o en un flujo explícito equivalente. Si se ofrece transferir FPS/rango u otros ajustes de tiempo, debe ser una casilla/opción explícita, no una sobrescritura silenciosa.
```

### 3.5 Enmienda de presencia/playhead

Añadir nota normativa, posiblemente en §3.1 o §2.6:

```markdown
El playhead de cada peer se transmite como metadato de presencia. Mostrar el frame de otro peer en la UI no autoriza mover automáticamente el timeline local. Ningún receptor ejecuta frame_set() ni asigna frame_current automáticamente por presencia. “Snap to user time” sigue siendo acción explícita del usuario.
```

### 3.6 Nuevos criterios de aceptación SRS

Añadir o extender AC:

| ID    | Criterio                                                                                       |
| ----- | ---------------------------------------------------------------------------------------------- |
| AC-15 | G/R/S de objeto en Host produce preview live en Cliente sin commits continuos                  |
| AC-16 | Cancelar/deseleccionar/desconectar no deja transformación de objeto fantasma                   |
| AC-17 | Keyframes reales de objeto llegan por porcelain                                                |
| AC-18 | Playheads/presencia no fuerzan timeline remoto                                                 |
| AC-19 | Default de conexión es Nueva escena; Combinar es opt-in explícito; purga requiere confirmación |

Si se prefiere no aumentar IDs SRS, estos pueden integrarse como subcriterios de AC existentes, pero la propuesta recomienda IDs explícitos.

---

## 4. TDD v4.7 → v4.8 propuesta

### 4.1 Nuevas decisiones de diseño

IDs propuestos, sujetos a comprobación de colisión en consolidación:

| ID       | Decisión propuesta                                                                                                                                                              | Estado propuesto | Justificación                                           |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------- |
| **D-52** | Toda salida de replicación compartida pasa por `PublicationPolicy` central. Guardas fase 1 se conservan como defensa inicial.                                                   | R1               | INV-1/AC-07 por construcción                            |
| **D-53** | Servidor rechaza mutaciones de grafo compartido de viewers. Autoridad Host ligada a conexión autoritativa, no a owner/admin/COMMON auto-declarados.                             | R1               | Defensa en profundidad                                  |
| **D-54** | `ReplicationSchema`/registry declarativo mínimo, versionado y curado. DD-03 y object_transform usan entries trazables.                                                          | R1               | Fidelidad mantenible                                    |
| **D-55** | `InteractionRegistry` + contrato de adapter. R1 registra `pose` y `object_transform`.                                                                                           | R1               | Preparar R2 sin framework universal                     |
| **D-56** | Frontera R1/R2: R1 = pose + object_transform live; shaders/sliders/otros continuos quedan R2+.                                                                                  | R1/R2+           | Alcance acotado                                         |
| **D-57** | Adapter live `object_transform`: G/R/S de objeto, multi-objeto, preview reversible, sin commits continuos, durable por porcelain.                                               | R1               | Aclaración del mantenedor                               |
| **D-58** | Playheads/presencia siguen como `UpdateUserMetadata`/porcelain command channel en R1. No fuerzan timeline remoto. Canal live de presencia queda futuro opcional.                | R1               | Evita migración innecesaria y protege bucle del Cliente |
| **D-59** | Incorporación: default `Nueva escena`; `Combinar` opt-in con advertencia y opción explícita de transferir ajustes de tiempo; copia como caso límite/reconexión. Enmienda D-40a. | R1               | Aclaración del mantenedor                               |

### 4.2 Nueva sección propuesta: §18 Arquitectura objetivo

Añadir sección normativa técnica con:

```text
18.1 Alcance R1: pose + object_transform + porcelain durable
18.2 Carriles: live preview / durable / local-only-presence
18.3 PublicationPolicy
18.4 Server authority enforcement
18.5 ReplicationSchema/registry
18.6 InteractionRegistry y adapters
18.7 Presence/playhead
18.8 Incorporación de escena
18.9 Paridad y migración incremental
```

### 4.3 Enmienda a §10.5 Publicación restringida al Host

Añadir:

```markdown
**D-52/D-53 — cierre arquitectónico.** Las guardas de fase 1 cierran rutas conocidas, pero R1 también requiere puerta central de salida y enforcement en servidor. `porcelain.commit` local no se confunde con publicación. `UpdateUserMetadata` de presencia se permite dentro de límites explícitos y no puede usarse para mutar escena.
```

### 4.4 Enmienda a §10.9 DD-03

Añadir:

```markdown
**D-54 — schema declarativo.** Los campos DD-03 deben expresarse mediante entries versionadas del ReplicationSchema o serializadores legacy trazables. La corrección incluye carga por capa, publicación durante pose seleccionada y reconciliación de hide_select.
```

### 4.5 Enmienda a §10.10 Orden de ejecución

Insertar fases arquitectónicas después de fase 1:

```markdown
| 1b | D-52, D-53, D-54, D-55, D-58 · puerta única, enforcement servidor, schema/registry mínimo, contrato de adapters, pose parity, presence/playhead no invasivo | INV-1, AC-06, AC-07, mantenibilidad |
| 1c | D-57 · adapter live object_transform con tests de no-commits continuos, multi-objeto, reversibilidad y handoff durable | R1 object live |
```

Nota:

```markdown
D-59 se implementa en la fase de onboarding correspondiente o en la fase que el mantenedor autorice, pero es obligación R1.
D-56 excluye adapters live de shaders/sliders/otros continuos de R1.
```

Si el mantenedor prefiere una sola fase, `1b` y `1c` pueden fusionarse, pero la recomendación es separarlas para aislar el riesgo de `object_transform`.

### 4.6 Enmienda a §14 Modelo de amenazas

Añadir:

```markdown
| S-04 | Viewer autenticado intenta mutar grafo compartido mediante Commit/Node/Delete/Right forjado o ruta no cubierta por guardas UI | D-53: servidor rechaza antes de aplicar/reenviar |
| S-05 | Presence metadata se usa para forzar timeline remoto o colar datos de escena | D-58: presence no ejecuta frame_set/frame_current automático; schema de metadata permitido |
```

### 4.7 Nuevas verificaciones propuestas

| ID        | Verificación                                           | Método                                                 |
| --------- | ------------------------------------------------------ | ------------------------------------------------------ |
| **VP-16** | Servidor rechaza mutación compartida de viewer forjado | Test integración servidor local                        |
| **VP-17** | Paridad de adapter de pose                             | Comparar contra `r1-fase-01`                           |
| **VP-18** | Schema DD-03 round-trip                                | Dump/load flags por capa                               |
| **VP-19** | object_transform no produce porcelain continuo         | Contadores de commit/push durante G/R/S                |
| **VP-20** | object_transform multi-objeto y reversibilidad         | Dos peers, selección múltiple, cancel/confirm/deselect |
| **VP-21** | Presence/playhead no invasivo                          | Recibir metadata no cambia frame_current local         |

### 4.8 Nuevos criterios de aceptación TDD

Añadir a §16.1:

| ID        | Criterio                                                                  |
| --------- | ------------------------------------------------------------------------- |
| **AC-15** | Servidor rechaza mutaciones compartidas de viewers aunque mensaje forjado |
| **AC-16** | Registry live R1 contiene solo pose y object_transform                    |
| **AC-17** | Pose adapter pasa paridad contra r1-fase-01                               |
| **AC-18** | object_transform live no produce commits continuos y es reversible        |
| **AC-19** | Playheads/presencia no fuerzan timeline remoto                            |
| **AC-20** | Default Nueva escena; Combinar opt-in; purga confirmada                   |

### 4.9 Enmienda a §17.F Esquema de mensajes live

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

### 4.10 Checklist derivado

Actualizar `CHECKLIST-IMPLEMENTACION.md`:

```text
- añadir D-52…D-59
- añadir fases 1b y 1c en §19
- añadir VP-16…VP-21 o pruebas equivalentes
- añadir AC-15…AC-20
- marcar R1 pendientes hasta evidencia
- marcar D-56 frontera R2+ para continuos excluidos
- recuento re-medido por verificar_docs.py
```

### 4.11 Diario derivado

Añadir entrada si se aprueba:

```text
006 · ADR-ARQUITECTURA-OBJETIVO-001 v1.1 aprobado
```

Debe registrar:

```text
- object_transform entra en R1
- pose parity obligatoria
- puerta única + servidor enforcement
- schema/registry mínimo
- presence/playhead sigue metadata
- default Nueva escena / Combinar opt-in
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
aprueba fases 1b/1c automáticamente
añade adapters live de shaders/sliders/luces/modificadores/sculpt/edit mode
migra playheads a canal live
aprueba P-01 adelantado
aprueba D-51
aprueba PKU
```

---

## 6. Orden de aprobación sugerido

1. Mantenedor revisa ADR v1.1.
2. Mantenedor aprueba o pide cambios al ADR.
3. Mantenedor revisa esta propuesta de enmiendas.
4. Si ambas se aprueban, se consolidan SRS/TDD/checklist/diario.
5. Se ejecuta `verificar_docs.py`.
6. Se decide si abrir fase 1b y 1c, o una fase fusionada.
7. Solo entonces se crea tag de inicio y se preparan parches.

---

## 7. Preguntas abiertas para el mantenedor

1. ¿Fases separadas `1b` y `1c`, o una sola fase arquitectónica?
   - Recomendación: separadas.
2. ¿`Reemplazar` se conserva como opción explícita confirmada, o se retira del diálogo objetivo?
   - Recomendación: conservar solo si es explícito y confirmado; nunca default.
3. ¿La opción de transferir FPS/rango al combinar debe ser una casilla única “sincronizar ajustes de tiempo” o opciones separadas?
   - Recomendación: empezar con casilla única explícita y documentada.
4. ¿Quieres commitear esta propuesta como borrador versionado, o esperar a que el ADR esté aprobado?
   - Recomendación: esperar si no necesitas historial; si lo commiteas, marcarlo como borrador.
