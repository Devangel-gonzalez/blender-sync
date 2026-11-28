# Propuesta de enmiendas normativas · ADR-001

| Campo              | Valor                                                  |
| ------------------ | ------------------------------------------------------ |
| Estado             | **PROPUESTA SEPARADA · pendiente de aprobación**       |
| Documento origen   | `ADR-001-ARQUITECTURA-OBJETIVO.md`                     |
| Fecha              | 2026-11-28                                             |
| SRS afectado       | v2.1.1 · **sin cambio normativo de alcance propuesto** |
| TDD afectado       | v4.7 · propuesta de enmienda a v4.8 si se aprueba      |
| Checklist afectado | v1.7 · propuesta de actualización derivada             |
| Código afectado    | Ninguno en este documento                              |

---

## 1. Principio de esta propuesta

El ADR-001 no cambia el alcance funcional de R1.

Por tanto:

> **No se propone ampliar el SRS para incluir nuevos continuos.**

R1 sigue siendo:

```text
Live Sync de pose + porcelain para datos definitivos
```

Las enmiendas propuestas son principalmente técnicas y de trazabilidad: hacer que el TDD exprese la arquitectura objetivo, la puerta única de publicación, el enforcement en servidor, el schema/registry mínimo y el contrato de adapters.

---

## 2. SRS v2.1.1

### 2.1 Enmienda normativa propuesta

**Ninguna obligatoria.**

El SRS ya exige:

- Cliente no publica por ninguna vía: §7.12, AC-07.
- Fidelidad de visibilidad/restricción: §3.9.
- Cliente puede seleccionar: §3.8.
- Pose live como núcleo R1: §0.2, §5.
- F1 fuera de soporte: §4.
- Porcelain fundamental para datos definitivos: §0.2, §6.5.

El ADR-001 no añade un requisito funcional nuevo; propone cómo enforcing/arquitecturar los existentes.

### 2.2 Cambio opcional de trazabilidad

Si el mantenedor quiere que el SRS cite el ADR, se propone añadir en §0.3:

```markdown
| `ADR-001-ARQUITECTURA-OBJETIVO.md` | Decisión arquitectónica propuesta/aprobada para mantenibilidad, puerta única de publicación, schema/registry y adapters. No amplía el alcance funcional de R1. |
```

Esto no cambia comportamiento observable.

---

## 3. TDD v4.7 → v4.8 propuesta

### 3.1 Nuevas decisiones de diseño

Se proponen los siguientes IDs, sujetos a comprobación de colisión en la consolidación:

| ID       | Decisión propuesta                                                                                                                                                                                     | Estado propuesto | Justificación                                                                         |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------- | ------------------------------------------------------------------------------------- |
| **D-52** | Toda salida de replicación compartida pasa por una `PublicationPolicy` central. Las guardas de fase 1 se conservan como defensa inicial hasta verificación de la puerta.                               | R1               | INV-1/AC-07 no deben depender de que cada caller recuerde un guard                    |
| **D-53** | El servidor rechaza mutaciones de grafo compartido provenientes de viewers autenticados. La autoridad de Host se liga a la conexión autoritativa, no a `owner`, `ADMIN` o `RP_COMMON` auto-declarados. | R1               | Defensa en profundidad; cliente modificado o ruta futura no debe poder mutar el Host  |
| **D-54** | Se introduce `ReplicationSchema`/registry declarativo mínimo, versionado y curado. R1 no migra todos los tipos de golpe. DD-03 se expresa mediante entries trazables.                                  | R1               | Fidelidad mantenible; evita dump automático de RNA y permite migración incremental    |
| **D-55** | Se introduce `InteractionRegistry` + contrato de adapter. En R1 solo se registra pose como live adapter.                                                                                               | R1               | Preparar R2 sin ampliar alcance R1; preservar comportamiento de pose mediante paridad |
| **D-56** | La generalización a objetos, shaders, sliders y otros continuos queda fuera de R1. R2+ añadirá adapters mediante schema/registry/lane policy con cambios localizados.                                  | R2+              | Decisión explícita de alcance del mantenedor: opción A                                |

### 3.2 Nueva sección propuesta: §18 Arquitectura objetivo

Añadir una sección normativa técnica:

```markdown
## 18. Arquitectura objetivo mantenible (ADR-001)

### 18.1 Alcance

R1 mantiene el contrato SRS v2.1.1: Live Sync de pose + porcelain para datos definitivos.
Pose es el único live adapter soportado en R1.

### 18.2 Carriles

| Carril              | Durabilidad                                   | R1              |
| ------------------- | --------------------------------------------- | --------------- |
| Live preview        | Efímero, reversible o reemplazado por durable | Solo pose       |
| Durable transaction | Definitivo vía porcelain                      | Sí              |
| Local-only/presence | No muta escena compartida                     | Sí, con límites |

### 18.3 Puerta única de publicación

Toda salida que pueda mutar el grafo compartido pasa por PublicationPolicy.
Se clasifican explícitamente mensajes permitidos de control/presencia y mensajes de mutación compartida.

### 18.4 Enforcement en servidor

El servidor distingue conexión autoritativa de Host frente a viewer autenticado.
Rechaza antes de aplicar/reenviar:

- viewer → Commit
- viewer → Node
- viewer → Delete
- viewer → Right
- viewer → init/reemplazo autoritativo del grafo compartido

No confía en owner/username/admin declarados por el peer.

### 18.5 Schema/registry

ReplicationSchema es curado, versionado y por tipo/versión de Blender.
No es dump automático de RNA.
R1 introduce contrato mínimo y entries para pose y DD-03.

### 18.6 InteractionRegistry y adapters

R1 registra únicamente pose.
El contrato de adapter permite a R2 añadir continuos sin lógica por tipo en el núcleo.

### 18.7 Paridad

La extracción del adapter de pose debe preservar mensajes, overlay, restauración, invalidación, handoff durable y cadencia.
```

### 3.3 Enmienda a §10.5 Publicación restringida al Host

Añadir después de la sección actual:

```markdown
**D-52/D-53 — cierre arquitectónico.** Las guardas de fase 1 cierran rutas conocidas, pero R1 también requiere una puerta central de salida y enforcement en servidor. La política local impide publicaciones honestas desde Cliente; el servidor rechaza mutaciones compartidas de viewers aunque el mensaje esté forjado o provenga de una ruta futura. `porcelain.commit` local no se confunde con publicación: la puerta principal gobierna la salida de red.
```

### 3.4 Enmienda a §10.9 DD-03

Añadir:

```markdown
**D-54 — schema declarativo.** Los campos de DD-03 deben expresarse mediante entries versionadas del ReplicationSchema o mediante serializadores legacy trazables a esas entries. La corrección no es solo añadir propiedades al dump: debe incluir carga correcta por capa, publicación durante pose seleccionada y reconciliación de `hide_select`.
```

### 3.5 Enmienda a §10.10 Orden de ejecución

Insertar una fase arquitectónica entre fase 1 y fase 2, sin aprobar todavía las fases 2–13:

```markdown
| 1b | D-52, D-53, D-54, D-55 · puerta única de publicación, enforcement servidor, schema/registry mínimo, adapter de pose con paridad | INV-1, AC-06, AC-07, mantenibilidad |
```

Notas propuestas:

```markdown
> La fase 1b no amplía el alcance funcional de R1. No añade adapters live distintos de pose.
> D-56 queda fuera de R1: nuevos continuos se decidirán en R2 o posteriores.
> DD-03 sigue siendo obligación de R1 y puede implementarse en su fase correspondiente usando el schema introducido en 1b.
```

### 3.6 Enmienda a §14 Modelo de amenazas

Añadir amenaza:

```markdown
| S-04 | Viewer autenticado intenta mutar grafo compartido mediante Commit/Node/Delete/Right forjado o ruta no cubierta por guardas UI | D-53: servidor rechaza antes de aplicar/reenviar; autoridad ligada a conexión de Host |
```

### 3.7 Nuevas verificaciones propuestas

Añadir a §15:

| ID        | Verificación                                           | Método                                                                                                               |
| --------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| **VP-16** | Servidor rechaza mutación compartida de viewer forjado | Test de integración con servidor local; enviar Commit/Node/Delete/Right con identidad viewer y owner/COMMON forjados |
| **VP-17** | Paridad de adapter de pose                             | Comparar mensajes live, porcelain calls, overlay y restauración contra `r1-fase-01`                                  |
| **VP-18** | Schema DD-03 round-trip                                | Dump/load de Object/Collection/Armature/ViewLayer en versiones soportadas; flags coinciden                           |

Si el mantenedor prefiere no crear nuevas VP, estas pueden registrarse como pruebas de fase 1b en §15.3/§15.4.

### 3.8 Nuevos criterios de aceptación propuestos

Añadir a §16.1:

| ID        | Criterio                                                                                                                               |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **AC-15** | El servidor rechaza mutaciones de grafo compartido de viewers aunque el mensaje esté forjado o no provenga de la UI                    |
| **AC-16** | El registry live de R1 contiene únicamente pose; no se emiten mensajes live de objetos/shaders/sliders                                 |
| **AC-17** | La extracción del adapter de pose pasa paridad contra `r1-fase-01` sin regresión de overlay, restauración, handoff durable ni cadencia |

### 3.9 Checklist derivado

Si se aprueba la enmienda del TDD, actualizar `CHECKLIST-IMPLEMENTACION.md`:

```text
- añadir filas D-52…D-56
- añadir fase 1b en §19
- marcar D-52/D-53/D-54/D-55 como R1 pendientes hasta implementación
- marcar D-56 como R2+/diferido
- añadir VP-16…VP-18 o pruebas equivalentes
- añadir AC-15…AC-17
- recuento de estados re-medido por verificar_docs.py
```

No se propone marcar nada como ✅ hasta tener evidencia del mantenedor.

### 3.10 Diario derivado

Si se aprueba, añadir entrada:

```text
006 · ADR-001 aprobado · arquitectura objetivo sin ampliación de R1
```

Debe registrar:

```text
- opción A elegida por el mantenedor
- R1 sin nuevos adapters live
- puerta única + servidor enforcement entran en R1
- schema/registry mínimo entra en R1
- paridad de pose obligatoria
- enmiendas TDD/checklist aplicadas
- código todavía sin tocar hasta orden de fase
```

---

## 4. Lo que esta propuesta no hace

Esta propuesta no:

```text
modifica código
modifica SRS/TDD/checklist por sí misma
aprueba fase 2
aprueba fase 1b automáticamente
añade adapters live de objetos/shaders/sliders
promete compatibilidad mixta nueva
aprueba P-01 adelantado
aprueba D-51
aprueba PKU
```

---

## 5. Orden de aprobación sugerido

1. Mantenedor aprueba o pide cambios al **ADR-001**.
2. Mantenedor aprueba o pide cambios a esta **propuesta de enmiendas**.
3. Si ambas se aprueban, se consolidan TDD/checklist/diario.
4. Se ejecuta `verificar_docs.py`.
5. Se decide si abrir fase 1b.
6. Solo entonces se crea tag de inicio y se preparan parches.

---

## 6. Pregunta abierta para el mantenedor

La única decisión de ubicación que conviene cerrar antes de consolidar es:

> ¿La fase arquitectónica debe insertarse como **fase 1b** entre fase 1 y fase 2, o prefieres distribuirla como prerequisitos dentro de fases existentes?

Recomendación por defecto:

```text
fase 1b separada
```

Motivo:

- mantiene fase 1 cerrada como hito histórico;
- no mezcla enforcement/schema/adapter con identidad/adopción de fase 2;
- permite tag y criterios propios;
- hace visible que la arquitectura se está consolidando antes de seguir avanzando.
