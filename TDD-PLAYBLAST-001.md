# Technical Design Document

# True Level: Playblast — Sistema de Live Sync para revisión de animación en Blender

| Campo                        | Valor                                                                                                                                                                                                                            |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Documento                    | TDD-PLAYBLAST-001                                                                                                                                                                                                                |
| Versión del documento        | 4.0                                                                                                                                                                                                                              |
| Producto                     | True Level: Playblast                                                                                                                                                                                                            |
| Identificador de extensión   | `true_level_playblast`                                                                                                                                                                                                           |
| Versión objetivo             | 1.0.0                                                                                                                                                                                                                            |
| Maintainer                   | Soilent                                                                                                                                                                                                                          |
| Repositorio                  | `github.com/Devangel-gonzalez/blender-sync` · rama `main`                                                                                                                                                                        |
| Revisión de código auditada  | `00b90a2` (12 nov 2026) · 28 commits                                                                                                                                                                                             |
| Revisión de código vigente   | **`f00d655`** (16 nov 2026) · 31 commits · **código Python idéntico al auditado** (§17.J)                                                                                                                                        |
| Especificación de requisitos | `SRS.md`                                                                                                                                                                                                                         |
| Plataforma de referencia     | Blender 5.2.2 · Windows 10                                                                                                                                                                                                       |
| Plataforma mínima declarada  | Blender 4.5.0                                                                                                                                                                                                                    |
| Licencia                     | `SPDX:GPL-3.0-or-later`                                                                                                                                                                                                          |
| Estado                       | Borrador consolidado v4.0 — coherencia verificada contra la auditoría y contra el código vigente `f00d655` (§17.J); X-01 definido y resuelto; identidad/`tagline`/`LICENSE` verificados en `main`; pendiente de aprobación final |
| Fecha de emisión             | 16 nov 2026                                                                                                                                                                                                                      |

---

## 0. Control del documento

### 0.1 Propósito y alcance

Este documento especifica la arquitectura, los defectos identificados, las decisiones de diseño y los requisitos de implementación de la release 1.0.0 (R1) de True Level: Playblast, así como el diseño preservado para la Phantom Keyframe Update (PKU).

El alcance cubre:

- Descripción del sistema tal como existe en la revisión auditada `00b90a2`.
- Auditoría técnica completa con clasificación de severidad.
- Análisis de causa raíz de los tres defectos funcionales visibles, con evidencia numérica.
- Registro de hipótesis evaluadas y descartadas, con la evidencia que las invalida.
- Protocolo de verificación y errores de medición conocidos que deben evitarse.
- Registro de decisiones de diseño con justificación y alternativas consideradas.
- Especificación de implementación, criterios de aceptación y estrategia de validación.
- Contraste de las decisiones tardías de la sesión de auditoría contra las decisiones fundacionales (§17.H).

Queda fuera del alcance de R1 la operación a través de internet, el modo Phantom Keyframe y el soporte de auto-keying, todos especificados como trabajo diferido.

### 0.2 Audiencia

- **Mantenedor**: responsable de implementación, orden de ejecución y aceptación.
- **Colaboradores futuros**: requieren mapa de arquitectura, registro de decisiones y registro de hipótesis descartadas.
- **Ingeniería de soporte**: requiere el protocolo de verificación y el operador de diagnóstico para reportes de rigs no contemplados.
- **Revisores técnicos**: requieren trazabilidad entre defecto, evidencia, decisión y criterio de aceptación.

El documento asume conocimiento de `bpy`, del depsgraph y de Python. Los apartados de seguridad y de mecánica de deformación se desarrollan sin asumir experiencia previa en esas áreas.

### 0.3 Relación con el SRS

`SRS.md` constituye la especificación de requisitos y define doce secciones: información general, escenario de trabajo, restricciones fundamentales, Funcionalidad 1, Funcionalidad 2, interacción entre funcionalidades, prohibiciones explícitas, consideraciones de rendimiento, casos límite, estructura de archivos, dependencias y resumen ejecutivo.

**El comportamiento esperado descrito en el SRS es normativo. El pseudocódigo, las rutas de archivo y los nombres de función del SRS son ilustrativos, no normativos.** Cuando la implementación difiere del texto del SRS pero satisface el comportamiento esperado, el texto se enmienda (E-xx). Cuando la implementación no satisface el comportamiento esperado, se registra como defecto (A-xx, DEF-xx) o como defecto conocido aceptado (KI-xx).

**Estado de cumplimiento en la revisión auditada.** Las dos funcionalidades del SRS están implementadas y operan conforme al comportamiento observable esperado en el flujo nominal:

- **Funcionalidad 1** (toggle global de commit/push armado): los botones individuales se convierten en toggles; el commit/push automático se ejecuta sobre los cambios confirmados detectados por el depsgraph; el armado inicial hace commit/push inmediato.
- **Funcionalidad 2** (Live Sync de Modo Pose): la edición interactiva del Host se manifiesta en el Cliente como un keyframe temporal en el frame de edición, integrado en el bucle; el timeline del Cliente no se pausa ni altera; el hueso no se silencia ni se congela; no hay Action swapping; un único keyframe temporal por hueso seleccionado, actualizado y no recreado; inserción con `options={'FAST'}`; canal TCP directo sin porcelain durante la transformación; acceso a `bpy.data` exclusivamente en el hilo principal.

Este cumplimiento del flujo nominal **no es un cumplimiento global del SRS**. El sistema presenta incumplimientos conductuales registrados que R1 corrige, y que este TDD declara explícitamente para no incurrir en sobreafirmación:

| Incumplimiento                                                                   | Requisito SRS afectado                                                                                                    | Registro       | Corrección                |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------- | ------------------------- |
| Keyframes fantasma persisten tras confirmar sin push que los reemplace           | §3.5, §7.9 (ninguna modificación temporal sobrevive fuera del flujo commit/push; no dejar huérfanos, con PKU desactivado) | A-02           | R1 · D-02/03/04           |
| Flicker al confirmar+keyear+deseleccionar en rápida sucesión                     | §5.4 (retención hasta reemplazo) vs §6.4 (orden end→datos); trade-off aceptado                                            | KI-01 · E-12   | Grace-period · FUT · D-05 |
| Vaciado del canal live a ~10 Hz en lugar de 30–60                                | §8.2 (frecuencia razonable 30–60 FPS)                                                                                     | P-01           | R1 · D-10                 |
| Publicación demorada tras confirmar hasta el siguiente operador (auto-key/panel) | §4.4 (commit/push por cada cambio), §6.2 Fase 3a                                                                          | A-01 · KI-02   | PKU · D-28                |
| El Cliente publica dependencias de imagen y muta el `.blend` del Host            | §1.4 (el Cliente es receptor pasivo); INV-1                                                                               | DEF-03         | R1 · DD-02                |
| Mallas escaladas/desplazadas al construir la escena del Cliente                  | §2.2 (construcción fiel de la escena con los mismos UUIDs)                                                                | DEF-01, DEF-02 | R1 · D-39, DD-19/20       |
| Asignación de UUIDs idénticos al Host no satisfecha                              | §2.2 punto 3 (incumplimiento literal, causa raíz de DEF-02)                                                               | §2.4           | R1 · D-39b                |

**El objeto de R1 no es implementar el SRS** —las dos funcionalidades ya existen— sino:

1. Corregir los defectos y casos límite que impiden la operación fiable (los incumplimientos anteriores y los defectos de fidelidad de §6.4).
2. Incorporar capacidades no contempladas en el SRS: tres modos de conexión, política de identidad y adopción, incorporación de escena con elección de usuario, diagnóstico integrado, reparación de archivos dañados, handshake de LAN y empaquetado con identidad propia.

Este TDD complementa y enmienda el SRS. Las enmiendas se registran en §2.6 y son normativas por sí mismas. El SRS prevalece en todo lo no enmendado explícitamente.

**Nota sobre la línea base del SRS.** El archivo `SRS.md` entregado contiene, tras la sección 12, material no normativo (roadmap, diario de decisiones en crudo y texto ajeno al proyecto) y presenta contradicciones internas (§1.4 «no altera el sistema de permisos» frente al gate host-only de §4.7; §5.4 frente a §6.4 sobre el momento de limpieza del keyframe temporal; §5.10 frente a §10 sobre la ubicación del motor). Este TDD toma como normativo el comportamiento esperado, resuelve las contradicciones mediante enmiendas explícitas (E-03, E-12, DD-02) y no depende del apéndice. Se recomienda limpiar y versionar el SRS como línea base estable (SRS v2); ver §17.H.4.

Documentación legada sin valor normativo: `README2.MD` (duplicado), `test.MD` (marcador de posición). Documentos de apoyo con valor informativo: `Arquitectura.md`, `DesicionesDeDiseno.md` (diario de desarrollo), `Discrepancias.md` (desviaciones especificación↔código).

### 0.4 Convenciones y notación

**Estados**: `R1` · `PKU` · `FUT` · `ESTABLECIDO` · `PENDIENTE` · `PROCESO` · `RETIRADA`.

**Esquema de identificadores**:

| Prefijo          | Dominio                                                |
| ---------------- | ------------------------------------------------------ |
| `INV-x`          | Invariante del sistema                                 |
| `A-xx`           | Defecto crítico de la auditoría                        |
| `P-xx`           | Hallazgo de rendimiento                                |
| `S-xx`           | Hallazgo de seguridad (único dominio de este prefijo)  |
| `M-xx`           | Hallazgo de empaquetado o mantenimiento                |
| `R-xx`           | Defecto menor                                          |
| `DEF-xx`         | Defecto funcional visible con causa raíz determinada   |
| `HI-xx`          | Hipótesis de diagnóstico evaluada y descartada         |
| `VE-xx`          | Verificación empírica ejecutada                        |
| `PM-xx`          | Error de medición conocido                             |
| `PV-xx`          | Condición del protocolo de verificación                |
| `D-xx` / `DD-xx` | Decisión de diseño (serie histórica / serie extendida) |
| `KI-xx`          | Defecto conocido aceptado                              |
| `AC-xx`          | Criterio de aceptación                                 |
| `E-xx`           | Enmienda al SRS                                        |
| `T-xx`           | Prueba de regresión                                    |
| `X-xx`           | Cuestión de diseño bloqueante, definida y resuelta     |
| `SRS-x.x`        | Requisito del SRS                                      |

Las series nocturnas `C-xx` (contradicciones internas), `F-xx` (errores factuales) y las verificaciones `V-xx`/`C-02`/`X-07`/`T-03` se registran como resueltas en §0.6 y §17.I; no constituyen una serie normativa activa. El prefijo `S-xx` queda reservado a seguridad; los huecos de cobertura del SRS identificados durante la auditoría se resolvieron como enmiendas (E-08…E-12, DD-37) y como mapeo (§2.7), no como hallazgos de seguridad.

**Código**: los bloques marcados _(ilustrativo)_ requieren adaptación de helpers al código real. Los bloques sin marcar son modificaciones verificadas contra `00b90a2`.

**Referencias**: `archivo:línea` sobre `00b90a2`. Anclas, no contrato.

**Tolerancia numérica, por dominio**: comparación de matrices con tolerancia absoluta `1e-4` (`_MATRIX_EPSILON`). Comparación de valores de canal exacta, como hace el código (`payload != record["sent"]`). Diferencias inferiores a `1e-5` en magnitudes geométricas se clasifican como ruido de punto flotante. La constante `_VALUE_EPSILON = 1e-5` está definida y no se usa; se registra como constante muerta. No existe banda intermedia sin clasificar: el dominio (matriz vs canal) determina el criterio.

### 0.5 Trazabilidad

Cada defecto funcional visible (§6) referencia su evidencia (§6, §17), sus hipótesis descartadas (§7), las decisiones que lo corrigen (§9) y sus criterios de aceptación (§16). El registro de decisiones (§9) constituye la fuente normativa única para las decisiones `D-xx`/`DD-xx`; ninguna decisión de esa serie se considera vigente si no figura en él. Las enmiendas `E-xx` son normativas por sí mismas (decisión E.2(a) de la auditoría); no requieren un `D-xx` de respaldo cuando solo documentan una divergencia ya registrada en `Discrepancias.md` o un hecho verificado en código.

### 0.6 Correcciones aplicadas en esta consolidación (v4.0 respecto a v3.0)

Esta versión consolida la totalidad de la auditoría y **contrasta las decisiones tardías (último 40% de la sesión) contra las fundacionales (primer 60%)**, corrigiendo las incoherencias introducidas por aprobaciones nocturnas precipitadas. El detalle del contraste está en §17.H. Correcciones materiales:

1. **DD-03 / `hide_select`**: se restituye la inclusión de `hide_select` en la sincronización, conforme a la aprobación fundacional y a la retractación nocturna explícita. v3.0 lo había excluido indebidamente (§6.4.2, §9.4, §10.9).
2. **X-01**: se define y resuelve explícitamente la cuestión poda↔D-44. La poda se conserva como limpieza post-invalidación acotada por procedencia; nunca con el overlay vivo; nunca sobre curvas legítimas (§4.8, §9.3, §10.8).
3. **§0.3 / §17.G**: se acota la afirmación de cumplimiento al flujo nominal de F1/F2 y se registran honestamente los incumplimientos conductuales que R1 corrige; se elimina la sobreafirmación «satisface el comportamiento observable» sin matices.
4. **KI-01 / E-12**: se corrige la redacción de KI-01 y se enmienda explícitamente la ambigüedad SRS §5.4↔§6.4; R1 sigue el orden de §6.4, la retención de §5.4 se difiere al grace-period.
5. **DD-35 / `slots[0]`**: se conserva el comportamiento existente en R1 (sin cambio, conforme a la retractación) y se registra el hueco defensivo como limitación diferida KI-09 con la recomendación preservada.
6. **DD-06**: se añade que la purga del cache alcanza solo los ficheros escritos por la propia instancia (protección INV-3 ante peers homónimos en la misma máquina). **Además se corrige la base del cache**: se conserva `extension_path_user`, que ya es el mecanismo embarcado, en lugar de moverlo a `__file__/cache` (§17.J.3).
7. **Trazabilidad**: se incorporan a §9 las decisiones PKU D-29…D-32 y D-34; se registran las decisiones retiradas (E-02, DD-04, DD-07, DD-09 reformulada, DD-15/17/21); se definen X-01 y las verificaciones de código resueltas (§17.I); se elimina la referencia colgante `X-01` del encabezado sin definición.
8. **Esquema de identificadores**: `S-xx` queda reservado a seguridad; se documenta la colisión nocturna y su resolución.
9. **Verificación contra el código vigente `f00d655`** (§17.J): se comprobó que el código Python es idéntico al auditado, de modo que todos los hallazgos siguen presentes; se aporta la causa raíz de DD-03 a nivel de `include_filter` con el punto exacto de intervención y la asimetría `Bone`/`EditBone` de la armadura; se acota la ruta `AnnotationUpdates` a Blender < 5.0.0; se precisa el registro dinámico de `on_scene_update` y el recuento de timers activos; se documenta la poda incondicional vigente y la destrucción de `created_curves`; se corrige la base de DD-06; se registra que el renombrado a `SRS.md` se ejecutó como copia; se añaden la causa estructural de la jerarquía de colecciones y el test inalcanzable de `sanitize_deps_graph`; se corrigen tamaños y una referencia de línea.

Correcciones factuales y de completitud heredadas de v3.0 y verificadas: restauración de §4.9–§4.12; eliminación de la referencia colgante `S-06`; requisito de orden SRS §6.4 (E-12); tabla orientativa SRS §8.3 (DD-37); mapeo completo de casos límite SRS §9 (§2.7); `_operator_stack_error_reported` (§4.4); cambio de frame a mitad de edición (§4.5); nota `tagline`/`website`/`LICENSE` resuelta contra `main` (§10.7); contexto de la escena ligera (§15.2); recuento `bl_types` (15 tipos adicionales + `__init__.py`); nota al pie de `SRS.md` como denominación objetivo; acotación de la clasificación de invariantes (A-02/03/04 vs A-01/05/06); tolerancia por dominio; `PV-xx`/`T-xx` en el esquema; SRS-3.2 completo (prohibición de congelar/desconectar).

### 0.7 Método de consolidación

La sesión de auditoría (13–16 nov 2026, 65 intercambios) se consolidó con la siguiente regla, fijada durante la propia sesión y reafirmada por el mantenedor:

- **Primer 60% (intercambios #1–#42)**: verdad absoluta. Contiene la filosofía, las invariantes, el diagnóstico con evidencia medida, las decisiones fundacionales y sus aprobaciones explícitas.
- **Último 40% (#43–#65, sesión nocturna)**: contrastado contra el primer 60%. El mantenedor declaró haber aprobado precipitadamente por fatiga. Se validan los elementos coherentes (§17.H.2) y se corrigen los que contradicen lo fundacional (§17.H.1).
- **Regla de no-cambio**: ninguna propuesta de cambio de comportamiento sobre una ruta que el mantenedor declaró funcionando, salvo evidencia medida de fallo. Las discrepancias documento↔código se corrigen como hechos (p. ej. E-02 retirada), no se convierten en decisiones nuevas. Esta regla explica la retractación de las propuestas nocturnas de poda, `slots[0]` y división de DD-03, **y la corrección de la base del cache de DD-06** (§17.J.3), y gobierna la redacción de §4.8, §4.5 y §9.4.

---

## 1. Información general del proyecto

### 1.1 Addon base

| Atributo                   | Valor                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------------- |
| Nombre                     | Multi-User                                                                                      |
| Versión base               | 0.8.2                                                                                           |
| Autor original             | Swann Martinez (Slumber)                                                                        |
| Repositorio upstream       | `gitlab.com/slumber/multi-user`                                                                 |
| Extensión                  | `extensions.blender.org/add-ons/multi-user/`                                                    |
| Documentación              | `slumber.gitlab.io/multi-user/index.html` · `multi-user.readthedocs.io`                         |
| Estado declarado upstream  | «Under development, use it at your own risks. Currently tested on Windows platform.»            |
| Arquitectura               | Cliente/servidor con replicación orientada a datos                                              |
| Operaciones de repositorio | `commit`, `push`, `apply`, `lock`, `unlock`, `rm`, `add`, `purge_orphan_nodes`, vía `porcelain` |
| Permiso declarado          | Red, para sincronización de datos de escena                                                     |

La librería `replication` fue reescrita para sustituir ZMQ por TCP sobre la biblioteca estándar (`socket`, `selectors`, `queue`). Se distribuye como wheel propio. Sobre esa base se incorporó el motor Live Sync (`live_sync.py`, 63 202 bytes, 1 663 líneas).

### 1.2 Objetivo

Flujo de revisión de animación en tiempo real: el Host opera como editor con timeline libre; el Cliente opera como visor en bucle continuo; las ediciones interactivas de pose del Host se manifiestan en el Cliente como keyframe temporal en el frame de edición; el timeline del Cliente no se detiene, pausa ni altera.

### 1.3 Motivación

El animador trabaja una pose concreta mientras el revisor observa la animación completa en bucle, evaluando la edición en contexto con sus transiciones de entrada y salida.

### 1.4 Filosofía de diseño

**Principio rector**: la inserción de un keyframe constituye la intención de conservar una pose. La confirmación de un R/G/S no constituye por sí sola dicha intención. El addon es para animadores 3D: debe ayudar al animador, no ponerle trabas.

**Principio de operación**: fricción cero. Se prohíbe como requisito de diseño la exigencia de cuentas de usuario, la configuración manual de red y la transferencia del `.blend` por canales externos (Discord, Drive, etc.) como paso necesario del flujo.

**Principio de extensibilidad**: debe operar con rigs de complejidad arbitraria. El código debe permitir la incorporación de correcciones ante rigs no contemplados mediante ampliación declarativa (DD-25), no reescritura. El maintenedor declara que, incluso si R1 se considera completa, llegarán usuarios con rigs más complejos que el de referencia (cervitaura, Auto-Rig Pro); el diseño debe absorber esas correcciones como proceso, no como arqueología.

### 1.5 Sistema de permisos

Multi-User gobierna la modificación mediante propiedad (`owner`) y bloqueo por datablock. Solo el propietario o un datablock común (`RP_COMMON`) puede modificarse. No existe modo global de solo lectura. El Host conserva la propiedad; el Cliente es receptor pasivo. `can_publish` restringe la publicación al Host.

**Nota de coherencia con el SRS**: el SRS §1.4 afirma que la modificación «no altera este sistema de permisos», pero su propio §4.7 introduce un gate `can_publish(..., host_peer)` y este TDD extiende la restricción host-only a todas las rutas de publicación (DD-02, D-11). La restricción host-only **sí altera** el modelo de permisos vanilla (donde cualquier propietario podía publicar). Se registra como enmienda implícita de DD-02: el Cliente no publica por ninguna vía, preservando la soberanía del Host (INV-1). La afirmación del SRS §1.4 se entiende como «el Host sigue siendo propietario y el Cliente receptor pasivo», que es el comportamiento esperado y se preserva.

### 1.6 Glosario

| Término                       | Definición                                                                                                                                |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Host                          | Instancia editora. Timeline libre. Única con capacidad de publicación                                                                     |
| Cliente                       | Instancia visora. Timeline en bucle. Sin capacidad de publicación                                                                         |
| Overlay / keyframe temporal   | Keyframe escrito por el Cliente en su Action para representar la edición del Host, durante el stream activo. Feature core                 |
| Keyframe fantasma             | Keyframe temporal que persiste tras finalizar el stream sin push que lo reemplace. R1: bug a eliminar. PKU: feature                       |
| Keyframe real                 | Presente en el Dope Sheet del Host, transportado por porcelain                                                                            |
| Porcelain                     | Capa de abstracción de `replication`                                                                                                      |
| Canal live                    | TCP dedicado en `puerto de sesión + 1`, separado del canal de replicación                                                                 |
| Stream                        | Secuencia `start` → `update`\* → `end`                                                                                                    |
| Adopción                      | Emparejar un datablock local preexistente con su equivalente remoto y estampar el UUID del Host, en lugar de crear un duplicado           |
| Desdoblamiento de referencias | Distintos punteros de un mismo objeto resuelven a datablocks distintos que representan la misma entidad lógica                            |
| Modo A / B / C                | Copia del archivo / archivo recién abierto / archivo con contenido propio                                                                 |
| R1 / PKU                      | Release 1.0.0 / Phantom Keyframe Update                                                                                                   |
| Invención de curvas           | Creación por el overlay de las fcurves ausentes (location/rotación/scale) para previsualizar un hueso con keyframes parciales o sin ellos |

---

## 2. Escenario de trabajo y configuración

### 2.1 Roles y modos de conexión

| Rol / Modo       | Archivo de partida                                      | Timeline       |
| ---------------- | ------------------------------------------------------- | -------------- |
| Host (Editor)    | `.blend` original                                       | Libre          |
| Cliente — Modo A | Copia directa del `.blend` del Host                     | Bucle continuo |
| Cliente — Modo B | `.blend` recién abierto, objetos eliminados manualmente | Bucle continuo |
| Cliente — Modo C | `.blend` con contenido propio                           | Bucle continuo |

Los tres modos son requisito de R1 (enmienda E-01 al SRS §2.1, que solo describe el modo copia). Cada modo ejercita una ruta distinta de resolución de identidad.

**Modo B**: el borrado total de objetos (A+X) suprime los objetos pero conserva datablocks (`Material`, malla del cubo, luz, cámara). Blender crea `Material` por defecto al abrir un archivo nuevo, lo que produce colisión de nombre con el datablock homónimo del Host.

**Modo A**: la copia realizada antes de iniciar la sesión no contiene UUIDs o contiene UUIDs distintos. La resolución por UUID falla y el sistema opera por coincidencia de nombre. El Cliente estampa sus propios UUIDs antes de resolver, de modo que nada matchea por UUID (causa raíz de DEF-02).

**Modo C**: el archivo del Cliente contiene datablocks con nombres potencialmente coincidentes pero semánticamente no relacionados (el `Cube` del paisaje del Cliente vs el `Cube` del Host). La resolución por nombre sin verificación estructural produce fusión indebida de contenido. El guard estructural D-39d debe impedirlo.

### 2.2 Restricciones fundamentales (SRS §3 y §7)

| ID      | Restricción                                                                                                                                                                                                                                       |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SRS-3.1 | Prohibido pausar, detener o alterar el timeline del Cliente. Prohibido `is_animation_playing = False` y `frame_set()` en el flujo de Live Sync                                                                                                    |
| SRS-3.2 | Prohibido silenciar el hueso editado. Prohibido `fcurve.mute = True`. Prohibido congelar o desconectar el hueso editado de la animación del Cliente. El hueso debe continuar animándose con normalidad                                            |
| SRS-3.3 | Prohibido el intercambio de Actions. Prohibido crear Actions vacías temporales, sustituir `animation_data.action` o desconectar la Action original. Los keyframes temporales se escriben dentro de la Action existente                            |
| SRS-3.4 | Prohibida la creación masiva de keyframes. Un único keyframe temporal por hueso seleccionado durante la edición, en el frame del Host. Se actualiza, no se recrea (enmienda E-08: «por hueso seleccionado», no «por hueso editado»)               |
| SRS-3.5 | La modificación temporal de la animación del Cliente es esperada y reversible. Respaldos en memoria antes de toda sobrescritura. Restauración íntegra en cancelación. Reemplazo por commit/push en confirmación. Excepción: modo Phantom Keyframe |
| SRS-3.6 | Prohibido `bone.keyframe_insert()`. Obligatorio `fcurve.keyframe_points.insert(frame, value, options={'FAST'})`. El stream usa el canal TCP directo, sin `porcelain.commit()` ni `porcelain.push()`                                               |
| SRS-3.7 | Prohibido el acceso a `bpy.data` desde hilos. Obligatorio `bpy.app.timers` y handlers nativos                                                                                                                                                     |
| SRS-6.4 | El Cliente procesa primero la señal `end` (limpia el keyframe temporal) y después aplica los datos del repositorio. Vinculado a KI-01 y a la enmienda E-12 (§2.8)                                                                                 |
| SRS-7.8 | Prohibido commit/push por porcelain durante la transformación interactiva, antes de la confirmación                                                                                                                                               |
| SRS-7.9 | Prohibido dejar keyframes temporales huérfanos. Toda modificación temporal se limpia por confirmación o cancelación, salvo bajo Phantom Keyframe                                                                                                  |

### 2.3 Sincronización de memoria

El sistema sincroniza `bpy.data`, no el archivo en disco. Al conectar, el Cliente descarga los datablocks y los construye con `porcelain.construct()` y `porcelain.apply()`.

**Restricción operativa**: prohibido abrir el mismo archivo físico desde carpeta de red compartida. Blender bloquea los `.blend` abiertos y puede corromperlos.

### 2.4 Flujo de conexión

1. El Host abre su archivo original e inicia el servidor.
2. El Cliente abre su archivo, introduce la dirección del Host y conecta.
3. El Cliente descarga los datablocks, construye la escena local y **debe asignar los mismos UUIDs que el Host**.

El punto 3 es un requisito del SRS §2.2 que el sistema actual no satisface: el Cliente estampa UUIDs propios antes de resolver, los UUIDs locales difieren de los del dump en todos los objetos, y toda la sesión corre por fallback de nombre. Su incumplimiento es la causa raíz de DEF-02 (§6.2). Corrección: D-39b (el Cliente no estampa UUIDs propios antes de adoptar; adopta los del Host).

### 2.5 Visibilidad del overlay en el bucle del Cliente

El overlay escribe un único keyframe **en cada hueso seleccionado** en el frame del Host. El Cliente en reproducción manifiesta la edición al pasar el playhead por dicho frame. Es el comportamiento especificado y deseado (D-40, SRS §5.5); no se amplía a rangos de frames ni se retiene la pose en frames adyacentes.

### 2.6 Enmiendas al SRS

| ID   | Disposición del SRS                                                                                      | Enmienda                                                                                                                                                           | Justificación / respaldo                                                                                     |
| ---- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| E-01 | §2.1 describe solo el modo copia                                                                         | Se reconocen tres modos de conexión (§2.1)                                                                                                                         | Los tres modos están soportados y deben operar correctamente (D-45)                                          |
| E-03 | §5.10 y §10 sitúan el motor en `libs/replication/live_sync.py`                                           | Reside en la raíz: `live_sync.py`                                                                                                                                  | Registrado en `Discrepancias.md`; hecho verificado                                                           |
| E-04 | §9 casos 08/09/10: ante fallo, «crear otra copia y reconectarse»                                         | Se rechaza como flujo prescrito. La resolución de identidad opera sin transferencia de archivo                                                                     | Fricción incompatible con §1.4 (D-39e)                                                                       |
| E-05 | Denominación True Level: Reviewer                                                                        | True Level: Playblast                                                                                                                                              | Decisión D-18                                                                                                |
| E-06 | §5.7 especifica transporte de `matrix_basis`                                                             | La implementación **añade** canales crudos (`location`, `rotation_mode`, `rotation`, `scale`) y **conserva** `matrix_basis` por compatibilidad con Host anteriores | Preservación de rotaciones multi-vuelta; adición, no sustitución                                             |
| E-07 | Transporte original ZMQ                                                                                  | TCP sobre biblioteca estándar                                                                                                                                      | Registrado en `Discrepancias.md`; hecho verificado                                                           |
| E-08 | §5.3 y SRS-3.4 asumen «hueso editado» en singular                                                        | El stream se abre por **hueso seleccionado**; la tabla de fases de SRS §6 se corrige en consecuencia                                                               | El código abre stream sobre `selected_pose_bones` completo; comportamiento deliberado (E.4)                  |
| E-09 | §9 caso 05 ofrece «crear una Acción» para el Cliente sin animación                                       | Se declara inválida esa opción; rige el manejo directo del hueso (§4.5)                                                                                            | Crear una Acción contradice SRS-3.3 y §7.4; ya implementado (E.3(a))                                         |
| E-10 | §8.6 exige liberar los respaldos inmediatamente tras confirmar o cancelar                                | Se extiende la excepción de SRS-3.5 a SRS-8.6: los respaldos de fantasmas retenidos bajo el toggle de Phantom Keyframe pueden conservarse                          | Requiere PKU; acotado a fantasmas bajo el toggle (E.5)                                                       |
| E-11 | §5.3/§5.4/§6.2/§6.3 sitúan `start` en el inicio de la transformación y `end` en la confirmación          | El stream se abre por selección y `end` se emite por deselección, con `is_cancelled` calculado por comparación de pose contra snapshot                             | Divergencia deliberada documentada en el código; explica estructuralmente A-02 (§4.4)                        |
| E-12 | §5.4 (retener el temporal hasta que el push lo reemplace) vs §6.4 (limpiar en `end`, luego aplicar push) | R1 sigue el orden de §6.4: `end` → restauración → aplicación del push. La semántica de retención de §5.4 se difiere al grace-period                                | Resuelve la ambigüedad interna del SRS; el flicker resultante está aceptado (KI-01, D-05; aprobación msg #7) |

_La enmienda E-02 de versiones previas (estado `RESOLVING` «especificado y no implementado») queda **retirada**: el código define y usa `IDLE`, `INTERACTING` y `RESOLVING`; la máquina implementada es la del SRS §5.8. Hecho verificado en código, no divergencia._

### 2.7 Mapeo de los casos límite del SRS §9

| Caso SRS                                                                | Cobertura en este documento                                           |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 01 · Cancelación restaura; si no había keyframe, se elimina el temporal | §4.5 `_restore`; T-01                                                 |
| 02 · Confirmación reemplazada por commit/push                           | §4.5 `_on_end`; A-02 cuando el push no llega                          |
| 03 · Cambio de frame a mitad de edición                                 | §4.5 `_on_update` (rama de cambio de frame); T-20                     |
| 04 · Múltiples huesos simultáneos                                       | §4.4; E-08                                                            |
| 05 · Cliente sin animación                                              | §4.5 manejo directo; E-09                                             |
| 06 · Edición de frame sin keyframe previo                               | §4.5 `_backup` (`"created": True`)                                    |
| 07 · Edición de frame con keyframe previo                               | §4.5 `_backup` (respaldo de `co` y handles)                           |
| 08 · Desconexión del Cliente durante edición                            | `shutdown()` → `cleanup()`; E-04                                      |
| 09 · Desconexión del Host durante edición                               | `cleanup()` restaura; E-04                                            |
| 10 · Hueso inexistente en el Cliente                                    | §4.5 `_on_start` y `_revive_overlay`; E-04                            |
| 11 · Toggle no armado al confirmar → vacío visual                       | KI-08 (causa: `push_animation_dependencies` condicionada a `is_live`) |

### 2.8 Orden de procesamiento `end` → datos del repositorio (E-12)

El SRS contiene dos semánticas de limpieza del keyframe temporal en confirmación:

- **§5.4 Paso 3a**: el keyframe temporal «se mantiene momentáneamente»; el commit/push real llega y actualiza; el temporal se limpia y es reemplazado por los datos reales.
- **§6.4 punto 3**: el Cliente procesa primero la señal `end` (limpia el temporal) y luego aplica los datos del repositorio; si el orden se invierte, puede mostrar brevemente datos inconsistentes.

R1 adopta el orden de §6.4: al recibir `end` con `is_cancelled=False` y `PHANTOM_KEYFRAME_MODE=False`, el Cliente restaura el overlay (`_restore`) de inmediato; el push real, si llega, recarga el Action y sustituye el overlay por el keyframe real. La diferencia con §5.4 es que no se retiene el temporal hasta la llegada del push: de ahí el flicker breve cuando confirmar+keyear+deseleccionar ocurren en rápida sucesión (KI-01). Este trade-off está aceptado por el mantenedor («aceptar flicker menor pero guardar como known issue»). La retención-hasta-reemplazo de §5.4 se implementará como grace-period diferido (D-05, FUT). KI-01 no «contraviene el orden de §6.4» —R1 lo implementa—; se aparta de la retención de §5.4, que es la decisión aceptada.

---

## 3. Invariantes del sistema

Las invariantes son propiedades que el sistema debe satisfacer en todo estado alcanzable (DD-01). Los defectos críticos **A-02, A-03 y A-04** corresponden a la violación de una invariante. **A-01, A-05 y A-06** son defectos de mecanismo sin invariante asociada.

### INV-1 · Soberanía del Host

Ninguna acción de un Cliente (conexión, aplicación de datos, resolución de dependencias, interacción) puede mutar el `.blend` del Host.
_Violaciones constatadas_: DEF-03 (§6.3), casos 2A/2B del diario, VE-05 (textura de `JawsHorseCock` en magenta en el Host tras conectar el Cliente).
_Corolario_: la interacción del Cliente nunca genera publicaciones (INV-3.1). DD-02 cierra todas las rutas de publicación del Cliente.

### INV-2 · Identidad por UUID

El UUID es la única identidad válida entre peers. El nombre es cosmético y nunca determina identidad. Ningún datablock local se renombra para acomodar a uno remoto.
_Violaciones constatadas_: duplicación de colecciones y actions (1A, 2A), creación de `rig.001` (DEF-02), colisión con `Material` por defecto (2B).
_Excepción acotada_: el pase de adopción D-39b es una heurística única, explícita y visible de onboarding para el caso copia; después, UUID only. No es una regla de identidad.

### INV-3 · Integridad del contenido local

El contenido local del Cliente no se destruye ni sobrescribe sin consentimiento explícito. Protege **datos**, no **interacción**: el Cliente puede seleccionar armaduras y huesos, entrar en Modo Pose, navegar el viewport e inspeccionar el estado; dicha interacción es necesaria para observar la generación de keyframes en vivo (aprobación explícita del mantenedor, msg #14).

**Corolario INV-3.1**: la interacción del Cliente nunca genera publicaciones. La selección de huesos produce actualizaciones del depsgraph que el handler descarta vía `can_publish`.

**Implicación para `hide_select`**: la seleccionabilidad en el Cliente no puede gobernarse por propiedad/readonly de forma que impida seleccionar huesos para observar al Host trabajar. La premisa del temporizador `DynamicRightSelectTimer` («solo lectura ⇒ no seleccionable») está en tensión con INV-3 y debe reconciliarse (§6.4.2, DD-03, §10.9).

### INV-4 · Adopción sobre recreación

Cuando el Cliente ya posee el datablock correspondiente, se actualiza. No se descarta ni se crea una instancia nueva.
_Violaciones constatadas_: recreación de colecciones con `.001` y pérdida de sus flags (§6.4.3).

### INV-5 · Evaluación frente a simulación

El Cliente evalúa; la simulación local es una capa de evaluación, no una divergencia de datos. El Cliente puede activar/desactivar física (Wiggle Bones u otros) para evaluar junto con simulación. Su resultado nunca se commitea ni se pushea (INV-1), y no se confunde con actualización del Host ni edición local.

**Nota de implementación**: el estado efectivo se lee del ajuste de **escena** del addon de física, no de la propiedad por hueso. La coexistencia de `wiggle_mute = 0` por hueso con silenciamiento de escena es un estado válido y observado. El overlay live escribe keyframes → el depsgraph evalúa la armadura → la simulación corre encima; ese orden debe respetarse y la supresión de ecos del depsgraph debe ignorar los cambios inducidos por simulación. Es un requisito de compatibilidad de primera clase.

---

## 4. Arquitectura del sistema en la revisión auditada

### 4.1 Mapa de componentes

```
blender-sync/
├── __init__.py              2 355 B
├── .gitignore                  42 B
├── blender_manifest.toml      752 B   (§4.11 · identidad 1.0.0 en f00d655)
├── live_sync.py            63 202 B   motor Live Sync · 1 662 líneas
├── handlers.py             16 769 B   on_scene_update, can_publish, filtros
├── timers.py               20 021 B   temporizadores (§4.7)
├── operators.py            44 005 B
├── preferences.py          24 988 B
├── shared_data.py           1 736 B
├── ui.py                   31 620 B
├── presence.py             19 109 B
├── utils.py                 6 658 B
├── environment.py           1 005 B   detección de IP
├── icons/                   7 PNG + registro de preview collection
├── bl_types/                          serializadores por tipo
│   ├── bl_object.py        38 005 B
│   ├── dump_anything.py    26 712 B   Dumper/Loader genéricos
│   ├── bl_material.py      22 776 B
│   ├── bl_action.py        22 423 B
│   ├── bl_scene.py         20 224 B
│   ├── bl_gpencil*.py      36 985 B   (tres variantes: 13 874 + 11 717 + 11 394)
│   ├── bl_mesh.py           7 840 B
│   ├── bl_armature.py       7 536 B
│   ├── bl_curve.py          7 162 B
│   ├── bl_image.py          5 928 B
│   ├── bl_collection.py     4 772 B
│   ├── bl_datablock.py      1 920 B   resolución de identidad
│   └── (15 tipos adicionales + __init__.py)
├── replication/                       ⚠ vendorizado, uso exclusivo del IDE
│   ├── net.py              23 502 B   ⚠ código muerto, sombreado por net/
│   ├── net/__init__.py     23 192 B   transporte TCP efectivo
│   ├── server.py           30 023 B
│   ├── interface.py        16 315 B
│   ├── porcelain.py        15 681 B
│   ├── repository.py       10 182 B
│   ├── protocol.py          7 961 B
│   ├── objects.py           5 604 B
│   ├── ttl.py               2 259 B
│   ├── constants.py         1 725 B
│   ├── utils.py             1 396 B
│   ├── exception.py         1 294 B
│   └── __init__.py            757 B   __version__ = "0.10.2"
├── deepdiff/                          ⚠ vendorizado, uso exclusivo del IDE
├── orderly_set/                       ⚠ vendorizado, uso exclusivo del IDE
├── wheels/
│   ├── replication-0.10.3-py3-none-any.whl   55 262 B
│   ├── deepdiff-8.1.1-py3-none-any.whl       84 655 B
│   └── orderly_set-5.2.3-py3-none-any.whl    12 024 B
├── tests/test_live_sync.py 60 649 B   arnés · 52/52 checks en 4.5 y 5.1
├── LICENSE                 35 770 B   GPL-3.0 + aviso propio + atribución upstream (D-21)
├── SRS.md                  53 248 B   especificación de requisitos¹
├── README.MD               45 145 B   spec legada · sigue presente¹
├── README2.MD              45 221 B   duplicado legado
├── Arquitectura.md         11 309 B
├── DesicionesDeDiseno.md   22 618 B   diario de desarrollo
├── Discrepancias.md         7 948 B
├── test.MD                      9 B   marcador de posición
└── 4 × reportes de sesión .txt       ≈ 607 765 B   (196 017 + 136 165 + 138 129 + 137 454)
```

¹ **Estado verificado en `f00d655`**: el commit `a19c2ef` («Renombrado README.MD a SRS.md») se ejecutó como **copia, no como movimiento**. `README.MD` (45 145 B) y `SRS.md` (53 248 B) **coexisten** en el árbol. El renombrado de D-45 sigue pendiente de ejecución: falta retirar `README.MD`. El archivo entregado al mantenedor pesa 53 988 B, ligeramente distinto del `SRS.md` del repo (53 248 B), por el apéndice no normativo (§0.3, §17.H.4). Tamaños y líneas de este árbol medidos sobre `f00d655`; el código Python es idéntico al de `00b90a2` (§17.J).

### 4.2 Capa de replicación

**Transporte**: TCP sobre `socket`, `selectors`, `queue`.
**Formato de cable**: `4 bytes nº chunks (BE)` + por chunk `4 bytes longitud (BE)` + payload.
**Handshake de conexión**: `4 bytes longitud identidad (BE)` + identidad + `1 byte rol` (`ROLE_COMMAND` | `ROLE_DATA` | `ROLE_TTL`).
**Puertos**: puerto único para todos los roles, default 5555. El servidor autentica por contraseña de sesión.
**Dirección**: Host inicia servidor y conecta a `127.0.0.1`; Cliente conecta a la dirección del Host. `environment.get_ip()` usa socket UDP a `8.8.8.8` sin envío.
**`sys.path`**: `replication/server.py:30` y `ttl.py:28-29` manipulan `sys.path`, precedidos de `# REFACTOR`. Código heredado con riesgo de sombreado (M-07, FUT).

### 4.3 Canal Live Sync

**Topología**: Host = `LiveSyncBroadcaster`, servidor TCP bind `0.0.0.0`, puerto `remote.port + 1` (5556). Cliente = `LiveSyncReceiver`. **Unidireccional Host→Cliente**.
**Entramado**: frames con prefijo de longitud, `FrameDecoder`, `MAX_MESSAGE_SIZE = 1 MB`, payload JSON. Payload malformado → `ValueError` capturado y tratado como reconexión.

**Mensajes** (campos planos, verificados en código; §17.F):

| Tipo     | Campos                                                                            | Semántica                                   |
| -------- | --------------------------------------------------------------------------------- | ------------------------------------------- |
| `start`  | `channel`, `action`, `armature_uuid`, `bone_name`, `frame`                        | Apertura. **Sin canales de transformación** |
| `update` | los anteriores + `matrix_basis`, `location`, `rotation_mode`, `rotation`, `scale` | Actualización                               |
| `end`    | `channel`, `action`, `armature_uuid`, `bone_name`, `frame`, `is_cancelled`        | Cierre                                      |

La clave de overlay es `(armature_uuid, bone_name)`. El Cliente resuelve la armadura por UUID (fallback `get_datablock_from_uuid`) y el hueso **por nombre exacto** (DD-36).

**Canales crudos y `rotation_mode`**: la rotación viaja como valores crudos en el `rotation_mode` del hueso, no derivada de `matrix_basis` (una matriz no representa más de una vuelta completa). `_rotation_data_path` elige `rotation_quaternion`, `rotation_axis_angle` o `rotation_euler`; `_rotation_values_for_mode` convierte solo entre modos distintos, preservando el modo idéntico exactamente. `matrix_basis` se conserva por compatibilidad con Host anteriores (E-06). Este mecanismo confirma por código que el overlay contempla los modos de rotación (HI-05 descartada por lectura, no solo por VE-04).

**Constantes**:

| Constante                | Valor        | Función                                       |
| ------------------------ | ------------ | --------------------------------------------- |
| `UPDATE_INTERVAL`        | 1/60         | Periodo de emisión                            |
| `TIMER_INTERVAL`         | 1/60         | Tick del hilo principal                       |
| `HEARTBEAT_INTERVAL`     | 0.1          | Reenvío por hueso en reposo                   |
| `TRANSFORM_IDLE_TIMEOUT` | 1.0 (`:97`)  | Umbral de inactividad de movimiento           |
| `MAX_DRAIN_PER_TICK`     | 256          | Mensajes drenados por tick                    |
| `MAX_MESSAGE_SIZE`       | 1 MB         | Límite de frame                               |
| `SOCKET_TIMEOUT`         | 1.0          | Timeout de `sendall`                          |
| `RECONNECT_INTERVAL`     | 1.0          | Reintento del receiver                        |
| Timeout de `accept`      | 0.1 (`:270`) | Cadencia de vaciado de la cola (P-01: → 0.01) |
| `_MATRIX_EPSILON`        | 1e-4         | Comparación de matrices                       |
| `_VALUE_EPSILON`         | 1e-5         | **Muerta**, sin uso                           |

### 4.4 HostPoseStreamer

Estados `IDLE` e `INTERACTING`. **El disparador del stream es la selección, no el inicio de la transformación** (E-11): `_stream_wanted()` devuelve verdadero cuando una armadura rastreada en Modo Pose tiene hueso seleccionado (`selected_pose_bones` o `active_pose_bone`). El comentario del código lo declara deliberado: «Gating on a running transform meant the overlay only ever appeared mid-drag and never followed the frame».

| Método              | Línea | Responsabilidad                                                                                         |
| ------------------- | ----- | ------------------------------------------------------------------------------------------------------- |
| `_begin`            | —     | Snapshot de `matrix_basis`, apertura de streams para los huesos seleccionados, primer `_update` forzado |
| `tick`              | 492   | Orquestación; cálculo de `transform_active` (`:531-535`)                                                |
| `_motion_stalled`   | 544   | Tras el umbral de inactividad, devuelve `not _pose_differs_from_snapshot()`                             |
| `_update_selection` | 635   | Comparación de nombres seleccionados; `start`/`end` por hueso añadido/retirado                          |
| `_update`           | —     | Lectura por `evaluated_get`; detección de cambio de frame; emisión                                      |
| `_finish`           | —     | Cierre con `is_cancelled` por hueso                                                                     |

**Detección del operador**: `TRANSFORM_OPERATORS` lista las tres formas de nombre (`bl_idname`, identificador RNA, nombre de clase) para rotate, translate, resize, transform y **trackball** (la tecla R usa trackball según el método de rotación del keymap; una sesión real mostró ese identificador). `interactive_transform_active()` inspecciona solo `operators[-1]`. Ante un stack de operadores ilegible, la bandera global `_operator_stack_error_reported` garantiza que el warning se emita **una sola vez** en lugar de 60 veces por segundo.

**`is_cancelled`**: `matrices_equal(bone.matrix_basis, record["original"])` con `_MATRIX_EPSILON`. Una transformación menor que el épsilon se trata como cancelada.

**Reenvío por incorporación**: el cambio de `client_count` fuerza un `_update` inmediato para que quien conecta a mitad de selección no espere el heartbeat.

**Sutileza de `_motion_stalled`**: una pose confirmada no retorna al snapshot, por lo que `transform_active` permanece verdadero mientras el hueso siga seleccionado. Correcto para el stream, inadecuado como señal de fin de drag a efectos de publicación (fundamento del gate de dos piezas de PKU, §12.4).

**Consecuencia estructural de E-11 sobre A-02**: una transformación confirmada cuyo hueso sigue seleccionado no emite `end` hasta deseleccionar; al deseleccionar, la pose difiere del snapshot → `is_cancelled=False` → rama de retención → fantasma. El fantasma es consecuencia de la combinación «stream por selección» + «retención en confirmación», no solo de la segunda. R1 elimina la retención (D-02/03/04).

### 4.5 ClientPoseApplier

**Drenado**: `_drain_inbox` coalesce conservando el último `update` por hueso por lote, hasta `MAX_DRAIN_PER_TICK`. `start`/`end` preservan orden. `_ended` impide resurrección.

**Revivir overlay**: `_revive_overlay` reconstruye un overlay cuyo `start` no se recibió (Cliente que conecta a mitad de selección), usando el `update`, custodiado por `_ended`. El SRS asume que `start` se recibe; este mecanismo cubre el hueco.

**Escritura**: `_write_pose` localiza o crea las fcurves de location, rotación (según `rotation_mode`) y scale. `_set_keyframe` reutiliza el punto existente (`_find_point`, tolerancia `0.5`, `:905` — A-04) o inserta con `options={"FAST"}` y luego ejecuta `fcurve.update()` (P-04).

**Invención de curvas (X-01, feature core)**: cuando `fcurves.find(data_path, index)` devuelve `None`, `_write_pose` crea la curva con `_new_fcurve` y la registra en `edit.created_curves`. Tres casos, todos preexistentes al TDD y confirmados por el mantenedor (msg #50):

| Caso                                                                | Ruta                                                                   | Comportamiento                                                           |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| El hueso tiene solo location keyeado; el Host rota o escala         | `fcurves.find` → `None` para rotation/scale → `_new_fcurve` + registro | Inventa los canales ausentes                                             |
| El hueso no tiene ningún keyframe, pero la armadura sí tiene Action | Misma ruta, para los 9–10 canales                                      | Inventa todos los canales                                                |
| La armadura no tiene Action (o no hay slot gobernable)              | rama `fcurves is None` → `_set_bone_transform`                         | Maneja el hueso directamente, reversible vía `edit.direct_matrix` (E-09) |

`_backup` marca `"created": existing is None`; `_restore` elimina el punto y después la curva si queda vacía. **La invención de curvas no se toca ni se poda mientras el overlay esté vivo** (§4.8, X-01).

**Cambio de frame a mitad de edición** (SRS §9.03): si el `frame` del mensaje difiere de `edit.frame`, `_on_update` ejecuta `_restore(edit)`, reasigna `edit.frame`, limpia `backups` y `created_curves`, anula `direct_matrix`, y escribe en el nuevo frame. El keyframe del frame anterior queda restaurado exactamente.

**Sin Action o sin slot gobernable**: `_fcurve_container` devuelve `None` y `_write_pose` maneja el hueso directamente mediante `_set_bone_transform`, de forma reversible vía `edit.direct_matrix`. Comportamiento existente, registrado en §5.6.

**Hueco defensivo de `slots[0]` (KI-09)**: si hay Action con slots pero `action_slot` es `None`, el código toma `slots[0]` y le asegura un channelbag. Como ese slot puede no gobernar la armadura, el overlay se escribiría en un contenedor que nada evalúa, y no cae al fallback directo porque `fcurves` no es `None`. La auditoría identificó este hueco por lectura de código; **no se ha medido que se alcance en la práctica**. Conforme a la regla de no-cambio (§0.7) y a la retractación nocturna, R1 conserva el comportamiento existente. Se registra como limitación defensiva diferida (KI-09, FUT) con la recomendación preservada: sin slot gobernante, caer al manejo directo del hueso. La corrección de DEF-02 (D-39) elimina el trigger de Action duplicada que lo hacía alcanzable. DD-35 fija el objetivo: escribir en el Action que gobierna efectivamente la armadura vía el slot correcto.

**Cierre**: `_on_end` registra `_ended` antes que nada; cancelación → `_restore`; confirmación → en la revisión auditada retiene el keyframe y limpia respaldos (origen de A-02). R1 sustituye la rama de confirmación por restauración (§10.1).

**Restauración**: `_restore` revierte puntos y elimina curvas creadas vacías, usando el registro `created_curves` y los respaldos.

**Supresión de ecos**: `_suppress_depsgraph` añade el UUID de la armadura a `shared_data.session.applied_updates`.

**Invalidación**: `invalidate_overlays` descarta el registro **sin restaurar** (la recarga ya sustituyó los puntos y los respaldos quedan stale). _Nota_: su docstring afirma que `BlAction.load` «removes all of an Action's slots and rebuilds them»; esa premisa está desactualizada respecto al reuso de slots de §4.8. La conclusión sigue siendo válida porque `load_fcurve` reemplaza los puntos. Corrección de documentación (§13). **Requisito de X-01**: el registro de procedencia (`created_curves`) debe sobrevivir a la invalidación o capturarse antes de ella para acotar la poda (§10.8).

### 4.6 LiveSyncEngine

| Método                  | Línea | Responsabilidad                                               |
| ----------------------- | ----- | ------------------------------------------------------------- |
| `_endpoint_for_session` | —     | `(is_host, remote.address, remote.port + 1)`                  |
| `_sync_transport`       | 1467  | Creación/destrucción del transporte                           |
| `_transport_alive`      | 1504  | Comprobación de vitalidad (solo `_thread is not None`) — A-05 |
| `tick`                  | 1511  | Orquestación con captura de excepciones                       |
| `shutdown`              | 1603  | Cierre ordenado                                               |

Accessors de módulo: `is_host()` (`:121`), `interactive_transform_active()` (`:126`), `pose_stream_active()`, `is_streaming()`, `tick()` (`:1629`), `invalidate_overlays()` (`:1651`), `shutdown()` (`:1661`). Instancia única `engine`. `LiveSyncTimer` registrado para ambos roles; su `unregister` invoca `live_sync.shutdown()`.

**Dos predicados distintos**: `is_streaming()` = `streamer.state == INTERACTING` (verdadero con mera selección); `pose_stream_active()` = `streamer.transform_active` (verdadero solo con drag real en curso). El comentario de `pose_stream_active` advierte: «treating a mere selection as an active transform would block every armed toggle for as long as the user has a bone selected».

### 4.7 Integración con el addon base

**Handlers registrados** (`handlers.register` y `live_sync`):

| Handler               | Evento                   | Comportamiento                                                |
| --------------------- | ------------------------ | ------------------------------------------------------------- |
| `on_scene_update`     | `depsgraph_update_post`  | Publicación filtrada                                          |
| `resolve_deps_graph`  | `undo_post`, `redo_post` | `sanitize_deps_graph(remove_nodes=True)`                      |
| `load_pre_handler`    | `load_pre`               | **`session_quit()`** si la sesión está activa o sincronizando |
| `update_client_frame` | `frame_change_pre`       | Publica `frame_current` como metadato de presencia            |

`load_pre_handler` tiene consecuencia normativa: abrir un archivo con sesión activa **cierra la sesión**. No es una reconexión; es una salida. Interactúa con D-41a. `update_client_frame` publica en cada cambio de frame del Cliente en bucle; es metadato de presencia legítimo, no publicación de datos de escena.

**Gate de transformación** (`:170-174`):

```python
dragging = live_sync.interactive_transform_active() is not None
streaming = live_sync.is_streaming()
if dragging:
    return
```

**Filtro de streaming** (V-01 resuelta): usa `is_streaming()` (mera selección). Durante streaming, bloquea commits de Armature y Object, permite Action:

```python
if streaming:
    if is_armature or is_object:
        continue
```

El comentario declara el alcance: «Durante streaming activo: BLOQUEAR commits de Armaduras y Objects (rendimiento), PERMITIR commits de Actions». La divergencia es con el snippet de SRS §4.7, que prescribe `pose_stream_active()` y un `continue` sin filtro por tipo: predicado más amplio y filtro por tipo añadido. El alcance es deliberado (el filtro por tipo es lo que permite publicar keyframes); se registra como enmienda, no como defecto. _Matiz_: los dos chequeos `is_live and pose_stream_active()` son inalcanzables en régimen normal (`pose_stream_active()` verdadero implica operador activo y el gate superior ya habría retornado); solo son alcanzables en la ventana de ≤16 ms entre el fin del operador y el siguiente tick. El filtro no debe describirse como activo sin ese matiz.

**`can_publish`**: host-only (`if not host_peer: return False`). El comentario documenta que un visor llegó a commitear y pushear Collections, que el Host aplicó, desmontando y reconstruyendo objetos y rompiendo el vínculo armadura↔Action.

**`push_animation_dependencies`**: publica las Actions detrás de un datablock armado. El docstring explica que insertar un keyframe reporta el _Object_, nunca el Action, y que un Action en `UP` con llaves nuevas nunca llega al servidor. **Está condicionada a `is_live`**, por lo que con el toggle desarmado el Action puede no publicarse (KI-08, V-02 resuelta). Force-push solo ante datos stranded (evita destruir overlays).

**Rutas de publicación sin guard de rol** (relevantes para DD-02, todas nombradas):

| Ruta                                            | Comprobación                                                                                                                                                                                                                                                                                                  |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `update_external_dependencies()`                | Solo propiedad (`owner in [username, RP_COMMON]`); commit/push de `WindowsPath`/`PosixPath`/`Image`. Se ejecuta al final de `on_scene_update` **fuera de todo guard**, también en el Cliente. **Causa exacta de DEF-03**                                                                                      |
| Rama de Scene sin UUID en `on_scene_update`     | `porcelain.add` + `commit` + `push` sin `can_publish` ni `host_peer`. Estado propio del Modo B                                                                                                                                                                                                                |
| `AnnotationUpdates.execute`                     | Solo propiedad; `lock`/`commit`/`push` de grease pencil. **Acotación de versión verificada**: el timer solo se registra si `bpy.app.version < (5,0,0)` (`operators.py:292-293`), de modo que esta ruta **no existe en la plataforma de referencia 5.2.2**; sigue siendo válida para el mínimo declarado 4.5.0 |
| `SessionToggleLiveSync`                         | Commit y `push(force=True)` inmediato sin someterse a `can_publish` (M-05)                                                                                                                                                                                                                                    |
| `sanitize_deps_graph` (`undo_post`/`redo_post`) | `porcelain.rm`; captura `NonAuthorizedOperationError`, pero `can_publish` advierte que la biblioteca acepta `RP_COMMON` para cualquier peer. Ruta a auditar (no violación constatada)                                                                                                                         |

**Temporizadores** (inventario completo, `timers.py` define diez):

| Temporizador                | Periodo                                   | Observaciones                                                                                                                              |
| --------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `ApplyTimer`                | `depsgraph_update_rate` (default ctor 10) | Aplica con guard de identidad `_applied_data`                                                                                              |
| `SessionListenTimer`        | 0.001                                     | Escucha de sesión                                                                                                                          |
| `LiveSyncTimer`             | `TIMER_INTERVAL` (1/60)                   | Invoca `engine.tick`; blindado por captura interna; `unregister` → `shutdown()`                                                            |
| `DynamicRightSelectTimer`   | 0.1                                       | Fija `hide_select` desde `is_node_readonly`; accede a `WinMan.windows[0]`                                                                  |
| `SessionBackupTimer`        | `save_interval` (default 10)              | **Bajo demanda**: solo se instancia desde `SessionSaveBackupOperator` con `enable_autosave` (`operators.py:963-966`); si no, volcado único |
| `AnnotationUpdates`         | 1                                         | Anotaciones. **Solo se registra en Blender < 5.0.0** (`operators.py:292-293`)                                                              |
| `ClientUpdate`              | 0.1                                       | Metadatos de presencia                                                                                                                     |
| `SessionStatusUpdate`       | 1                                         | Refresco de sidebar                                                                                                                        |
| `SessionUserSync`           | 1                                         | Sincronización de usuarios                                                                                                                 |
| `ContinuousActionPushTimer` | 0.4                                       | **No registrado. Código muerto** (`:441`) — M-04                                                                                           |

`Timer.main` desconecta la sesión ante cualquier excepción no capturada (R-03).

**Recuento activo verificado** (`setup_timer`, `operators.py:272-293`): **8 temporizadores en Blender 5.x, 9 en 4.x** —la diferencia es `AnnotationUpdates`—, más `SessionBackupTimer` si el usuario activa el autoguardado. `ContinuousActionPushTimer` nunca se instancia. Los tres de sesión (`SessionListenTimer`, `SessionStatusUpdate`, `SessionUserSync`) se registran de inmediato; el resto queda en `deleyables` y se registra en `initialize_session`.

**Registro de handlers verificado**: `handlers.register()` (`handlers.py:362-367`) registra solo cuatro —`undo_post`, `redo_post`, `load_pre`, `frame_change_pre`—. **`on_scene_update` no se registra ahí**: se añade a `depsgraph_update_post` en `initialize_session` (`operators.py:197`) al establecerse la sesión, y se retira en `on_connection_end` (`operators.py:224-225`). Es el único handler de publicación y es dinámico; cualquier prueba que lo ejercite debe simular ese alta, no solo el registro del addon.

### 4.8 Gestión de Actions en Blender 5.x (`bl_types/bl_action.py`)

1. **Guard de recarga vacía**: si el dump trae cero curvas y el Action local tiene curvas, se conservan y se emite warning. Se cuentan curvas, no slots: un payload con slot de `fcurves` vacía supera una comprobación por presencia de slots y borraría sin registrar.
2. **Recuento de pérdida**: `keys_before = count_action_keyframes()` registra pérdidas silenciosas. El guard solo cubre cero curvas; un payload parcial sigue suprimiendo lo que no transporta.
3. **Reuso de slots**: emparejamiento por `(target_id_type, name_display)`. El identificador no sobrevive el round-trip porque `slots.new()` acuña uno nuevo. El reuso evita el desmontaje que dejaba la armadura sin animación evaluable y congelaba el hueso en cada apply, y mantiene vivo el channelbag para que los wrappers del overlay no se invaliden.
4. **Rama `>= (5,0,0)`**: `anim_utils.action_ensure_channelbag_for_slot`, `fcurves.find/new`, y poda de curvas ausentes del dump.

**Evidencia verificada en `f00d655`** — la poda actual es **incondicional** (`bl_action.py:476-483`, cita literal):

```python
                dumped_keys = {
                    (fc["data_path"], fc["dumped_array_index"])
                    for fc in slot["fcurves"]
                }
                for stale_curve in list(channelbag.fcurves):
                    if (stale_curve.data_path,
                            stale_curve.array_index) not in dumped_keys:
                        channelbag.fcurves.remove(stale_curve)
```

No consulta procedencia ni si hay un overlay vivo sobre ese Action. El propio comentario del código describe el síntoma que produce: la curva que el overlay tuvo que inventar sobrevive a la recarga, `invalidate_overlays` descarta la contabilidad sin restaurar, y el keyframe temporal queda huérfano «lo que se lee exactamente como un keyframe que el viewer había confirmado». Del otro lado, `_BoneEdit.created_curves` (`live_sync.py:957`) **sí** registra la procedencia, pero `invalidate_overlays` ejecuta `self._edits.clear()` en **`live_sync.py:1414`** y ese registro se destruye. La información necesaria para acotar la poda existe hoy; se elimina antes de poder usarse. Es la prueba en el fuente de la tensión X-01 y del requisito previo de §10.8.

**X-01 · Resolución de la tensión poda ↔ D-44** (cuestión bloqueante, definida y resuelta):

La poda existente elimina las curvas de un slot que el dump no menciona. D-44 establece que no se suprime contenido que el dump no describe. Las dos reglas ordenan cosas opuestas sobre la misma operación. La reconciliación, conforme a la auditoría y a la declaración del mantenedor (msg #52: «no hay que podar ninguna curva que haya inventado el keyframe temporal mientras el keyframe temporal siga ahí»), es:

- **La invención de curvas es feature core y se preserva** (§4.5). Ninguna curva inventada se poda **mientras el overlay esté vivo**.
- **En R1, el caso normal se resuelve solo**: al restaurar en el `end` confirmado (D-02), `_restore` elimina las curvas inventadas que quedan vacías. No se necesita poda adicional.
- **La poda se conserva únicamente para la recarga a mitad de stream**, antes del `end`: `invalidate_overlays` descarta el registro y las curvas inventadas quedan sin dueño. La poda elimina esas curvas huérfanas (prohibidas por SRS-7.9).
- **Acotación por procedencia**: la poda se limita a las curvas **creadas por el overlay** (`created_curves`) **y ausentes del dump**. Nunca poda curvas legítimas ausentes de un dump parcial (eso vulneraría D-44). Esto exige que el registro de procedencia sobreviva a la invalidación o se capture antes de ella (§10.8).
- **Implementación R1, opción (i)**: `bl_action.load` consulta el registro de `live_sync` con import diferido dentro de la función (evita el ciclo, pues `live_sync` ya importa de `bl_types`). Exacta, sin heurísticas; deuda de acoplamiento aceptada y registrada. **Refactor preferido FUT, opción (iii)**: marcar la procedencia en la propia FCurve (propiedad personalizada), sujeto a verificación de que la FCurve la admite y sobrevive al apply. La opción (ii) (sacar la poda al apply) se descarta porque pierde la información de qué describió el dump y requeriría una heurística con caso de fallo identificado.

_La poda así acotada queda excluida de D-44 (§9.3): es limpieza de basura propia del overlay tras su invalidación, no supresión de contenido legítimo por omisión._

_Nota_: los docstrings de `is_action_node` y `ApplyTimer.execute` afirman que la recarga «removes and rebuilds every slot»; esa premisa está desactualizada respecto al reuso de slots. Corrección de documentación (§13).

### 4.9 Contrato de threading

El acceso a `bpy` se realiza exclusivamente en el hilo principal. Los hilos de red (`LiveSyncBroadcaster._service_loop`, `LiveSyncReceiver._read_loop`) manipulan únicamente sockets y JSON. La aplicación a `bpy.data` ocurre en `engine.tick`, invocado por `LiveSyncTimer` desde `bpy.app.timers`. Las colas `deque` y `queue.Queue` son seguras para `append`/`popleft` y `put`/`get` concurrentes. El módulo lo declara explícitamente en su docstring. Cumple SRS-3.7.

### 4.10 Empaquetado y dependencias

**Resolución de imports**: instalado como extensión, los imports absolutos (`import replication`, `import deepdiff`, `import orderly_set`) resuelven al wheel en el `site-packages` de la extensión. Las carpetas vendorizadas en la raíz existen exclusivamente para la resolución de símbolos por el entorno de desarrollo y no participan en la ejecución (D-27).

**Equivalencia de contenido**: el contenido del wheel coincide con el vendorizado; la única diferencia es la codificación de saltos de línea.

**Duplicación de módulo**: coexisten `replication/net.py` y `replication/net/__init__.py`, en repositorio y wheel. El paquete sombrea al módulo; `net.py` es código muerto (M-01, D-26).

**Discrepancia de versión**: el wheel se denomina `replication-0.10.3` y su `__init__.py` declara `__version__ = "0.10.2"` (M-03, D-26).

### 4.11 Manifiesto en la revisión auditada

```toml
schema_version = "1.0.0"
id = "multi_user"
version = "0.8.2"
name = "Multiuser"
blender_version_min = "4.5.0"
tagline = "Enable real-time collaborative workflow inside blender"
maintainer = "Swann Martinez <swann.martinez@protonmail.com>"
license = ["SPDX:GPL-3.0-or-later"]
type = "add-on"
platforms = ["windows-x64", "macos-x64", "macos-arm64", "linux-x64"]
website = "https://slumber.gitlab.io/multi-user/index.html"
wheels = [
   "./wheels/orderly_set-5.2.3-py3-none-any.whl",
   "./wheels/deepdiff-8.1.1-py3-none-any.whl",
   "./wheels/replication-0.10.3-py3-none-any.whl",
]
[permissions]
network = "Need to sync scene data to server"
[build]
paths_exclude_pattern = ["__pycache__/", ".*", "*.zip", "cache/", "venv/", "tests/"]
```

_Nota_: este es el manifiesto de la revisión auditada `00b90a2` (identidad heredada `multi_user`, sin `LICENSE`). En `main` ya se sustituyó por la identidad propia, se escribió el `tagline` y se incorporó `LICENSE`; el estado objetivo y verificado se documenta en §10.7.

### 4.12 Preferencias y estado compartido

| Propiedad                            | Línea | Default | Semántica                                                                              |
| ------------------------------------ | ----- | ------- | -------------------------------------------------------------------------------------- |
| `SessionProps.pose_live_sync`        | 627   | `True`  | Habilita el stream de poses                                                            |
| `SessionProps.global_live_sync`      | —     | `False` | Convierte los botones de commit en toggles armados; su update limpia `live_sync_nodes` |
| `SessionProps.is_host`               | 684   | `False` | Rol de la instancia                                                                    |
| `SessionPrefs.depsgraph_update_rate` | 263   | —       | Periodo del `ApplyTimer`                                                               |

`shared_data.SessionData` mantiene `repository`, `remote`, `server`, `applied_updates` (lista) y `live_sync_nodes` (diccionario de UUIDs armados). La ausencia de una clave se interpreta como desarmado.

---

## 5. Auditoría técnica

### 5.1 Defectos críticos

**A-01 · Bloqueo de publicación tras confirmar transformación** — `handlers.py:170-174`.
`wm.operators` es historial acumulativo; el transform confirmado queda en el tope; `interactive_transform_active()` inspecciona solo `operators[-1]`, devuelve el operador finalizado y el handler retorna prematuramente, bloqueando commit/push. Enmascarado al pulsar I (añade `anim.keyframe_insert`). Roto con auto-key (inserta dentro del propio operador). El handler es event-driven: el último update llega al confirmar, cuando el movimiento aún es reciente, y tras el umbral no hay updates que lo reinvoken. El gate existe por una razón deliberada del mantenedor: dragear actualiza el depsgraph constantemente y enviar muchos commits congela la computadora; por eso R/G/S va en crudo por TCP. → PKU (D-28). KI-02. Defecto de mecanismo sin invariante asociada.

**A-02 · Persistencia de keyframes fantasma** — `live_sync.py:1133-1165`.
La rama de confirmación de `_on_end` retiene el keyframe, descarta respaldos y extrae el edit, delegando el reemplazo a un push que puede no llegar (el Host confirma sin insertar keyframe → el Action no cambia → no hay commit de Action; o el push está bloqueado por A-01; o durante streaming se saltan los commits de Object/Armature). **Es consecuencia estructural de E-11**: el stream se abre por selección y `end` se emite por deselección; una transformación confirmada cuyo hueso sigue seleccionado no emite `end` hasta deseleccionar, y al deseleccionar `is_cancelled=False` → retención. Repro del diario: transformar → seleccionar otro hueso → cambiar frame → volver. Desaparecen al insertar un keyframe real **en el mismo Action**. Sobreviven a `shutdown()`. Viola SRS-3.5/§7.9 con PKU desactivado. → R1 sin fantasmas (D-02/03/04). Violación de INV (integridad del contenido del Cliente).

**A-03 · Cambio de armadura activa no detectado** — `live_sync.py:635`.
`_update_selection` no verifica cambio de armadura; `self.armature_uuid` queda viejo mientras `_update()` lee la nueva. Los huesos retirados emiten `end` bajo el UUID viejo; los añadidos emiten `start` bajo el UUID viejo con snapshot de la armadura nueva. Con nombres de hueso coincidentes («Bone», «Bone.001») se transmite la pose de una armadura bajo el UUID de otra. Confirmado por VE-06. Enmascarado al salir/reentrar de Modo Pose. → R1 (D-06). Violación de INV-2.

**A-04 · Captura de keyframes fraccionarios** — `live_sync.py:905`. `_find_point` usa `abs(co.x - frame) < 0.5`; una llave en 10.3 se identifica como la del frame 10 y se sobrescribe; si la transformación se confirma (A-02), esa llave real queda modificada permanentemente. → R1, épsilon `1e-4` (D-07). Violación de INV-3.

**A-05 · Transporte muerto no reiniciado** — `:1504`. Comprueba `_thread is not None`, no `.is_alive()`. Si el hilo del broadcaster muere (p. ej. `OSError` en `accept`), el engine lo cree vivo y nunca reconstruye el socket: el Live Sync muere en silencio. → R1 (D-08). Defecto de mecanismo.

**A-06 · Sin backoff ante puerto ocupado** — `:1467`. Si `bind(port+1)` falla, reintento a 60 Hz con warning por tick → spam masivo de log y CPU desperdiciada. → R1, backoff 5 s (D-09). Defecto de mecanismo.

### 5.2 Hallazgos de rendimiento

| ID   | Hallazgo                                            | Evidencia                                                                                                                                                                                                                                                    | Resolución                                                                                                                                                                                                                   |
| ---- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P-01 | Vaciado a ~10 Hz en lugar de 60 Hz                  | `accept()` con timeout 0.1 s (`:270`); latencia hasta ~100 ms en ráfagas. Contraviene SRS §8.2                                                                                                                                                               | R1: timeout 0.01 (D-10)                                                                                                                                                                                                      |
| P-02 | Cliente lento bloquea a todos                       | `sendall` con timeout 1 s en hilo único                                                                                                                                                                                                                      | FUT. KI-04                                                                                                                                                                                                                   |
| P-03 | Log INFO por lote y por tick                        | Registros de ~600 KB                                                                                                                                                                                                                                         | ESTABLECIDO: se conserva (D-24)                                                                                                                                                                                              |
| P-04 | `fcurve.update()` incondicional tras inserción FAST | `_set_keyframe`: ambas ramas llaman `update()`, que rehace ordenamiento y handles; 9–10 llamadas por hueso por tick (location 3 + rotación 3/4 + scale 3). SRS-3.6 se cumple literalmente (el `insert` lleva `FAST`), pero el propósito de SRS §8.1 se anula | Hallazgo de rendimiento (X-07 resuelta). Verificación de corrección antes de tocar; en la rama de inserción `update()` ordena los puntos que FAST dejó sin ordenar; en la rama de punto existente es candidato a innecesario |

**Coste de conexión medido**: 13:31:06 conexión → 13:31:22 snapshot (16 s) → 13:31:24–13:31:44 carga de objetos (20 s) → 13:32:15 registro de temporizadores → 13:32:22 canal live. Total ≈ 76 s. El guard estructural D-39d añade milisegundos contra este baseline (ruido estadístico).

### 5.3 Hallazgos de seguridad

| ID   | Hallazgo                                     | Resolución           |
| ---- | -------------------------------------------- | -------------------- |
| S-01 | Canal live sin autenticación, bind `0.0.0.0` | R1: handshake (D-14) |
| S-02 | Transmisión en claro                         | FUT: cifrado. KI-05  |
| S-03 | Sin TLS el Cliente no verifica al Host       | FUT                  |
| S-04 | Sin traversal de NAT                         | FUT (R1 = LAN)       |

La direccionalidad Host→Cliente es correcta por diseño (broadcaster solo envía, receiver solo lee), condicionada al cierre de las rutas de DD-02. El riesgo de inyección requiere MITM o impersonación, no que el cliente mande datos.

### 5.4 Hallazgos de empaquetado y mantenimiento

| ID   | Hallazgo                                                                                                                                                                                                                                                                                                                                                  |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| M-01 | Coexistencia de `net.py` y `net/`; el paquete sombrea al módulo, en repo y wheel                                                                                                                                                                                                                                                                          |
| M-02 | Vendorizados duplican los wheels; el runtime resuelve al wheel                                                                                                                                                                                                                                                                                            |
| M-03 | Wheel `0.10.3` con `__version__ = "0.10.2"`                                                                                                                                                                                                                                                                                                               |
| M-04 | `ContinuousActionPushTimer` (`:441`): no registrado; si se activara, commit y force-push de todos los Actions cada 0.4 s sin verificación de propiedad ni de drag, destruyendo overlays, permitiendo publicación por clientes y transmitiendo el nodo completo; `_last_hashes` sin usar; `node.data.get()` fuera del `try` → AttributeError → desconexión |
| M-05 | `SessionToggleLiveSync` elude `can_publish`                                                                                                                                                                                                                                                                                                               |
| M-06 | Acceso literal `"WinMan"` en `live_sync.py:121`, `_runtime_settings`, operators y timers                                                                                                                                                                                                                                                                  |
| M-07 | Manipulación de `sys.path` en `server.py:30` y `ttl.py:28-29`                                                                                                                                                                                                                                                                                             |
| M-08 | Identidad del manifiesto heredada; `id = "multi_user"` colisiona con la extensión original; ausencia de `LICENSE` en la revisión auditada → **resuelto en `main`** (identidad propia D-18/19/20, `tagline` propio y `LICENSE` incorporado D-21)                                                                                                           |
| M-09 | Higiene del repositorio: reportes de sesión (~600 KB), documentación duplicada, marcador de posición                                                                                                                                                                                                                                                      |
| M-10 | Colisión de puerto con dos sesiones en la misma máquina (5555/5556 de la primera chocan con la segunda)                                                                                                                                                                                                                                                   |

### 5.5 Defectos menores

| ID   | Defecto                                                                                                                                  | Resolución |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| R-01 | Carrera en `LiveSyncReceiver.stop()`: hilo entre `connect()` y asignación de `self._sock` deja `recv()` bloqueante orphanado como daemon | FUT (D-33) |
| R-02 | `invalidate_overlays` limpia `_ended`; un update en vuelo tras la recarga puede resucitar un overlay cerrado                             | FUT (D-33) |
| R-03 | `Timer.main` desconecta la sesión ante cualquier excepción no capturada                                                                  | FUT (D-33) |

### 5.6 Comportamientos correctos a preservar

1. Gestión de slots y channelbags 5.x: reuso, guardas de recarga vacía, poda de curvas ausentes acotada por procedencia (X-01, §4.8).
2. Preservación de rotaciones multi-vuelta mediante canales crudos con `rotation_mode`.
3. Coalescido de mensajes entrantes.
4. Contrato de threading: `bpy` solo en hilo principal.
5. `push_animation_dependencies` fuerza push solo ante stranded.
6. Manejo directo del hueso cuando no hay Action gobernable, reversible vía `direct_matrix` (E-09).
7. `_revive_overlay` para Clientes que conectan a mitad de selección.
8. **Invención de curvas** para huesos con keyframes parciales o sin ellos (X-01, feature core, §4.5).
9. Serialización verificada como fiel de: geometría de curvas Bezier, radios, tilt, handles, resolución, `eval_time`, `path_duration`, `twist_mode`; ajustes de Spline IK; payload genérico y específico de constraints; drivers y sus variables; propiedades personalizadas de armadura y de hueso; `nodes.active` de árboles de shaders (DD-04 verificada, retirada como defecto); geometría base de malla; valores y geometría de shape keys; flags de herencia y parámetros de B-Bone; rest pose.
10. Registro de desviaciones en `Discrepancias.md`.

---

## 6. Análisis de causa raíz de los defectos funcionales visibles

### 6.1 DEF-01 · Escalado anómalo del Horse

**Síntoma**: al conectar el Cliente, una sección de la malla se escala desproporcionadamente. Afecta **al original y al low poly** (comparten armadura). Determinista, probado.

**Cadena causal**:

1. Los cinco Hook modifiers de `_MCH-Shaft_spline` pierden su array `vertex_indices`.
2. Un Hook sin índices afecta a la totalidad de los puntos de la curva.
3. Los cinco hooks actúan simultáneamente sobre los cinco puntos Bezier; la curva evaluada colapsa.
4. La Spline IK de `MCH-Shaft_07.ik` no puede posicionar la cadena.
5. `MCH-Shaft_01.ik`…`07.ik` quedan en reposo, con escala no uniforme.
6. `ORG-Shaft_*` (COPY*TRANSFORMS desde MCH) y `DEF-Shaft*\*` (desde ORG) heredan el estado.
7. La malla se deforma con huesos escalados.

**E-DEF01-1 · índices**:

| Hook          | Host         | Cliente A | Cliente B |
| ------------- | ------------ | --------- | --------- |
| Shaft_start   | `[0,1,2]`    | `[]`      | `[]`      |
| Shaft_start01 | `[3,4,5]`    | `[]`      | `[]`      |
| Shaft_mid01   | `[6,7,8]`    | `[]`      | `[]`      |
| Shaft_end01   | `[9,10,11]`  | `[]`      | `[]`      |
| Shaft_end     | `[12,13,14]` | `[]`      | `[]`      |

**E-DEF01-2 · escala de `DEF-Shaft`** (longitud de columnas de la submatriz 3×3 de `pose_bone.matrix`):

| Hueso           | Host (X,Y,Z)      | Cliente (X,Y,Z)     |
| --------------- | ----------------- | ------------------- |
| DEF-Shaft_01    | ≈0.98, 1.00, 0.98 | **5.5, 1.0, 5.5**   |
| DEF-Shaft_02…07 | ≈0.95, 1.00, 0.95 | **10.0, 1.0, 10.0** |

La traslación es idéntica en ambos lados. El error es exclusivamente de escala, en X y Z, con Y —eje del hueso— intacta.

**E-DEF01-3 · correspondencia con el radio de la curva**. Radios: `1.0` (p0), `10.0` (p1–p4), idénticos en ambos. Spline IK: `use_curve_radius = true`, `y_scale_mode = FIT_CURVE`, `xz_scale_mode = VOLUME_PRESERVE`, `chain_count = 7`, idénticos.

| Hueso           | Escala XZ Cliente | Correspondencia  |
| --------------- | ----------------- | ---------------- |
| DEF-Shaft_01    | 5.5               | (1.0 + 10.0) / 2 |
| DEF-Shaft_02…07 | 10.0              | radio de p1–p4   |

En el Host, la escala XZ es la compensación de volumen (0.98 → 0.20 decreciente), porque la curva tiene longitud real y `FIT_CURVE` estira la cadena en Y. En el Cliente, con curva degenerada, no hay estiramiento en Y y XZ recibe el radio sin compensación. El patrón «X e Z escaladas, Y intacta» es la firma de `xz_scale_mode`.

**E-DEF01-4 · posición de reposo**: en el Cliente, la traslación de cada `MCH-Shaft_NN.ik` coincide exactamente con su `head_local` (−0.21724, −0.29353, −0.36981, −0.44207, −0.51091, −0.57751, −0.65748). En el Host, posiciones animadas.

**E-DEF01-5 · segundo defecto en el mismo payload**. En el Cliente Modo A, `matrix_inverse` de los hooks contiene valores de espacio de pose:

| Hook        | Escala de `matrix_inverse` | Escala de pose del target | Recíproco |
| ----------- | -------------------------- | ------------------------- | --------- |
| Shaft_start | −1.0                       | 1.0                       | 1.0       |
| Shaft_mid01 | −1.08786                   | 0.91924                   | 1.08786   |
| Shaft_end01 | −15.82365                  | 0.0632                    | 15.82365  |
| Shaft_end   | −15.82366                  | 0.0632                    | 15.82366  |

En el Cliente Modo B, la inversa es de reposo e idéntica al Host. **Asimetría create/update**: la creación escribe el valor de reposo correcto; la actualización lo sobrescribe con un valor derivado de la pose vigente.

**Controles idénticos**: geometría base de la curva, `resolution_u = 12`, `use_cyclic_u = false`, `eval_time = 0.0`, `path_duration = 100`, `use_path = true`, `twist_mode = MINIMUM`, `matrix_world` identidad; todos los ajustes de Spline IK; `rest_hash_ordenado = ac04d181cc2b`; `orden_huesos` idéntico; escala y `matrix_world` de la armadura.

**Decisiones correctoras**: DD-19 (payload completo de modificadores), DD-20 (unificación create/update).

### 6.2 DEF-02 · Desplazamiento y deformación de las mallas de Lilia

**Síntoma**: al conectar el Cliente, cabeza, ojos, cabello y pestañas se presentan desplazados o deformados. Afecta **solo al original**; el low poly sigue correctamente al hueso.

**Cadena causal**:

1. La resolución por UUID falla para todos los objetos: el UUID local difiere del UUID del dump (el Cliente estampa UUIDs propios antes de resolver).
2. Se crea un objeto duplicado `rig.001` que comparte el datablock de armadura `rig`.
3. El duplicado no se vincula a ninguna escena ni colección y no recibe Action.
4. Los punteros de **parent** de las mallas originales resuelven al duplicado.
5. Los modificadores Armature siguen apuntando al objeto `rig` original.
6. La malla se deforma según un objeto y se posiciona según otro. El modificador Armature calcula la transformación relativa objeto↔armadura; con el objeto en −0.23125 y la armadura en +0.23125, la relativa se desvía 0.4625 y los huesos deforman los vértices como si la malla ocupara otra posición. El error varía por hueso.

**E-DEF02-1 · punteros de parent y matriz mundial**:

| Objeto            | Parent Host | Parent Cliente | `matrix_world` Y Host | Cliente      |
| ----------------- | ----------- | -------------- | --------------------- | ------------ |
| `Lilia`           | `rig`       | **`rig.001`**  | 0.0                   | **−0.23125** |
| `Hair`            | `rig`       | **`rig.001`**  | 0.0                   | **−0.23125** |
| `Eyebrows`        | `rig`       | **`rig.001`**  | 0.0                   | **−0.23125** |
| `Eyelashes`       | `rig`       | **`rig.001`**  | 0.0                   | **−0.23125** |
| `Teeth`           | `rig`       | **`rig.001`**  | 0.0                   | **−0.23125** |
| `this`            | `rig`       | **`rig.001`**  | −0.34951              | **−0.58077** |
| **Todas las Low** | `rig`       | `rig`          | 0.0                   | 0.0          |

La totalidad de los originales resuelve al duplicado; la totalidad de los low poly al original. Reproduce exactamente la asimetría del síntoma.

**E-DEF02-2 · aritmética**:

```
rig       matrix_world Y = +0.23125
rig.001   matrix_world Y =  0.0       (identidad)
pinv      Y = −0.23125                (idéntico en Host y Cliente)
Host:     +0.23125 × (−0.23125) × basis(0) =  0.0
Cliente:   0.0     × (−0.23125) × basis(0) = −0.23125
```

`matrix_parent_inverse` es correcto en ambos lados. El defecto reside en el puntero de parent, no en la matriz inversa (HI-03 descartada con datos).

**E-DEF02-3 · propiedades del duplicado**: existe solo en el Cliente; vinculación a escenas `[]` (orphanado); `data = 'rig'` (comparte el datablock); `animation_data.action = None`.

**E-DEF02-4 · modificadores**: en Host y Cliente, los Armature de `Lilia`, `Lilia Low`, `Hair`, `Eyelashes` apuntan a `rig`. `Eyelashes` añade Mirror y Subdivision; `Lilia` y `Hair`, Subdivision. Orden y targets idénticos (HI-04/HI-06 descartadas).

**E-DEF02-5 · Action duplicada**: `rigAction` (Host), `rigAction.001` (Cliente). Ambos con **262 fcurves y 3 276 keyframes**, y valores evaluados idénticos canal por canal en el frame 573.

| Canal                        | Valor evaluado | Nº de keys |
| ---------------------------- | -------------- | ---------- |
| `c_head.x.rotation_euler[0]` | −0.151098      | 43         |
| `c_head.x.rotation_euler[1]` | −1.056775      | 42         |
| `c_head.x.rotation_euler[2]` | 0.170141       | 42         |
| `c_head.x.location[0..2]`    | 0.0            | 3          |
| `c_neck.x.rotation_euler[0]` | 0.019861       | 10         |
| `c_neck.x.rotation_euler[1]` | 0.046715       | 10         |
| `c_neck.x.rotation_euler[2]` | −0.068485      | 10         |
| `c_neck.x.location[1]`       | −0.037512      | 2          |
| `c_neck.x.location[2]`       | 0.007678       | 2          |
| `c_eyelid_top.r.location[2]` | −0.009112      | 57         |
| `c_eyelid_top.l.location[2]` | −0.00695       | 54         |

No hay curvas inventadas ni fantasmas acumulados en esa sesión (HI-15 descartada).

**E-DEF02-6 · registro de conexión**:

```
BlObject.load on 'rig' (uuid '6adbd400-…', dump uuid '1ca82a4c-…')
Parent of 'cs_user_Stick' resolved by NAME fallback to 'cs_grp';
  uuid ad032a23-… matched no local object
…
BlObject.load on 'cs_grp' (dump uuid 'ad032a23-…')
```

UUID local ≠ UUID del dump en todos los objetos. Hijos cargados antes que padres (`rig` en 13:31:34, `char_grp` en 13:31:37). La resolución de parent opera por fallback de nombre y depende del orden de aplicación (DD-22).

**Por qué las mediciones de armadura salieron limpias**: `rig.001` comparte el datablock `rig`. Toda medición de rest, constraints, drivers, propiedades y pesos operaba sobre el mismo datablock en ambos lados. El defecto reside en la capa de objetos.

**Decisiones correctoras**: D-39, D-39b, D-39c, D-39d, D-43, DD-22, DD-33, DD-36.

### 6.3 DEF-03 · Pérdida de texturas

**Síntoma**: partes de los personajes se presentan negras en Solid con color Texture; Rendered muestra las texturas. En el Host, la textura de un objeto se pierde y se presenta en magenta tras la conexión del Cliente (VE-05).

**Causa exacta** (verificada en `handlers.py`, V-01/DEF-03): al final de `on_scene_update`, **fuera de todo guard**, se ejecuta `update_external_dependencies()`, que commitea y pushea nodos `WindowsPath`/`PosixPath`/`Image` comprobando **solo propiedad** (`owner in [username, RP_COMMON]`), no `can_publish` ni `host_peer`. Como se ejecuta en cada batch del depsgraph **también en el Cliente**, el Cliente publica dependencias de imagen que el Host aplica, mutando su `.blend`. El guard host-only existe y funciona en el bucle principal; la violación de INV-1 entra por esta ruta.

**Cadena causal del daño de imagen**:

1. Las imágenes empaquetadas se tratan como dependencias de fichero externo y se extraen al cache.
2. Se genera un nodo de dependencia sobre `WindowsPath`, que no admite estampado de UUID (`Can stamp uuid on WindowsPath`).
3. La identidad del nodo es inestable; cada reexposición crea un datablock con sufijo incremental.
4. El Cliente commitea y pushea esos nodos (vía `update_external_dependencies`).
5. El Host aplica el push y su datablock de imagen pierde los datos o la referencia.

**E-DEF03-1 · datos empaquetados truncados**:

```
BKE_packedfile_blend_read: nullptr packedfile data (source: '…cache\0000000056353F38.png')
libpng error: IDAT: Read error: hit end of file in png reader
IMB_load_image_from_memory: unknown file-format (<packed data>)
Failed to create GPU texture from Blender image
```

**E-DEF03-2 · identidad no estampable**: `Can stamp uuid on <class 'pathlib._local.WindowsPath'>`; `Reading Sin_nombre.png content: 250106 bytes`.

**E-DEF03-3 · duplicados sin datos en el Host**: `could not export image 'ColorAssets.png.001' … does not have any image data` (×3); `image 'hiikaiti.mp4.001' points to a missing file`.

**E-DEF03-4 · discrepancia de ruta**: la dependencia se registra como `WindowsPath('…/cache/Sin_nombre.png ')` con espacio final; el aviso de lectura indica `…cache\Sin_nombre.png doesn't exist, skipping` sin espacio.

**E-DEF03-5 · cache compartido**: Host y Cliente en la misma máquina escriben en `…/extensions/.user/user_default/multi_user/cache/`. Colisión y condición de carrera.

**E-DEF03-6 · eco de commits**: el mismo nodo Material commiteado tres veces con intervalos de 11 s.

**E-DEF03-7 · violación de INV-1 en vivo**: VE-05.

**Persistencia del daño**: los avisos de datos empaquetados nulos y PNG truncado se emiten durante la **lectura del archivo** (16:55), antes de la conexión (13:31). El daño está incorporado al `.blend` por sesiones anteriores. Su corrección es DD-13 (dejar de generar el daño) más DD-27 (reparar los archivos ya dañados); la escritura atómica de DD-06 evita que se reproduzca por la vía de la carrera, pero no reconstruye bytes ya perdidos.

**Hipótesis descartada**: `nodes.active` se preserva. El mapa de nodo activo por material es idéntico en las cinco mediciones, incluida `"Lillia.001": ["Normal Map", null]`, presente ya en el archivo original del Host (DD-04 retirada).

**Decisiones correctoras**: DD-02 (cierre de rutas), DD-13 (imágenes packed), DD-06 (cache), DD-05 (invalidación de shaders), DD-27 (reparación).

### 6.4 Defectos de fidelidad sin síntoma funcional atribuible

#### 6.4.1 Reordenamiento de pesos de deformación

El orden de las entradas de peso por vértice difiere en 7 de 8 mallas. El Cliente presenta orden ascendente por índice de grupo; el Host conserva el orden nativo.

| Objeto              | Hash de orden Host | Cliente        |
| ------------------- | ------------------ | -------------- |
| Horse_Cock_Jaws     | `6d3d3f6d50e0`     | `a861066025d0` |
| Horse_Cock_Jaws Low | `43ef435ebc95`     | `df820ad56fd2` |
| Lilia               | `6479c8a2e6c4`     | `089bc9defaf3` |
| Lilia Low           | `6c42c798bf85`     | `5ca8cca11402` |
| Hair                | `ad9a9d481ec6`     | `f0795553861a` |
| Eyelashes           | `105f9779921e`     | `546d83897769` |
| Teeth               | `8e3c228feec9`     | `795649a5e649` |
| Eyebrows            | `663cd2205e04`     | `663cd2205e04` |

Valores, grupos, índices y recuento total idénticos: `n_pesos_total` = 27 923 / 1 599 / 213 017 en las cinco mediciones. No hay pérdida de datos (HI-01 descartada).

El primer entry por vértice cambia de grupo dominante a grupo de peso despreciable:

| Vértice Horse_Cock_Jaws | Host (grupo, peso) | Cliente (grupo, peso) |
| ----------------------- | ------------------ | --------------------- |
| 34                      | 2, 0.99969         | 0, 0.00392            |
| 35                      | 2, 0.99785         | 0, 0.00540            |
| 32                      | 21, 0.002713       | 0, 7.5e-05            |
| 33                      | 21, 0.013840       | 0, 0.004747           |
| 38                      | 21, 0.000958       | 0, 5.4e-06            |

**Invención de canales**: el overlay crea las curvas ausentes en los tres casos de §4.5 (X-01), todos cubiertos por el código y preexistentes al TDD. **Ninguna de estas curvas se poda mientras el overlay esté vivo**; la poda acotada por procedencia solo opera tras la recarga que invalida el overlay (§4.8).

**Clasificación**: defecto de fidelidad. La hipótesis de que causa el síntoma de deformación queda descartada por VE-01 (Preserve Volume OFF no altera el síntoma). Ocurre en Modo A y Modo B por igual. El caso peligroso es un vértice con ≥2 pesos no nulos cuyo orden cambie con dual-quaternion; la muestra capturó pesos nulos reordenados (huella de reescritura, no necesariamente síntoma).
**Decisiones**: DD-14 (idempotencia por contenido), DD-28 (transporte del orden de pesos).

#### 6.4.2 Pérdida de flags de restricción y visibilidad

**Defecto confirmado por el mantenedor**: cuando un interruptor de restricción/visibilidad está desactivado en el Host (p. ej. `hide_viewport` activo para no mostrar un objeto en el viewport), al conectarse el Cliente **no se conserva el estado** y el objeto pasa a mostrarse/habilitarse. Lo mismo ocurre con los **huesos ocultos**, cuya visibilidad reside en la **DATA de la armadura** (no en la pose). Es un bug de transmisión de estados, no un comportamiento deseado. DD-03 lo corrige serializando y restaurando **todos** los interruptores siguientes:

| Interruptor / flag                                            | Capa                 | Modo A     | Modo B     | Diagnóstico                                                                                       |
| ------------------------------------------------------------- | -------------------- | ---------- | ---------- | ------------------------------------------------------------------------------------------------- |
| `Object.hide_render`                                          | Objeto               | conserva   | conserva   | Serializado                                                                                       |
| `Object.hide_viewport` (monitor / `RESTRICT_VIEW_OFF`)        | Objeto               | conserva   | **pierde** | No serializado; sobrevive en Modo A por inercia del valor preexistente; en Modo B nace en default |
| `Object.hide_select` (`RESTRICT_SELECT_OFF`)                  | Objeto               | **pierde** | **pierde** | No serializado **y gobernado localmente por permisos**                                            |
| `Collection.hide_viewport` (`RESTRICT_VIEW_OFF`)              | Colección            | pierde     | pierde     | La colección se recrea (§6.4.3)                                                                   |
| `Collection.hide_render` (`RESTRICT_RENDER_OFF`)              | Colección            | pierde     | pierde     | La colección se recrea                                                                            |
| `Collection.hide_select` (`RESTRICT_SELECT_OFF`)              | Colección            | pierde     | pierde     | La colección se recrea                                                                            |
| `ViewLayerCollection.exclude`                                 | View layer           | pierde     | pierde     | No aplicado                                                                                       |
| `Bone.hide` / `Bone.hide_viewport`                            | **DATA de armadura** | pierde     | pierde     | Visibilidad de hueso en la data, no serializada                                                   |
| Visibilidad de bone collections (`BoneCollection.is_visible`) | **DATA de armadura** | —          | pierde     | No serializada                                                                                    |

**Nota sobre la capa de los huesos**: la visibilidad de los huesos (`bone.hide`) y de las bone collections vive en `armature.data` (edit bones / bone collections), **no** en `armature.pose`. Por tanto debe transportarla el serializador de la **data de armadura** (`bl_armature.py`), no el de pose (`bl_object.load_pose`). Es un dato de organización de la vista del animador, independiente de la pose evaluada y del frame. Su pérdida hace que huesos que el Host tenía ocultos aparezcan visibles en el Cliente.

**Causa raíz verificada a nivel de filtro (`f00d655`)**. El defecto no es un fallo de transmisión: **las propiedades nunca se serializan**, así que no existe mensaje que las transporte ni nada que restaurar en el Cliente. Los `include_filter` reales son:

| Serializador                                     | `include_filter` verificado                                                                                                                                              | Consecuencia                                                                                                                                               |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `BlObject.dump` (`bl_object.py:803`)             | lista blanca con `name`, `rotation_mode`, `data`, `library`, empties, `color`, `instance_*`, `lock_*`, **`hide_render`** (`:823`), `display_*`, `show_*`…                | `hide_viewport` y `hide_select` **no están en la lista** → nunca salen del Host                                                                            |
| `BlCollection.dump` (`bl_collection.py:120-123`) | **`["name", "instance_offset"]`** + `objects` + `children`                                                                                                               | **ningún** flag de colección viaja; la familia `RESTRICT_*` está ausente por completo                                                                      |
| `BlArmature.dump` (`bl_armature.py:166-176`)     | **`['bones','tail_local','head_local','tail_radius','head_radius','use_connect','parent','name','layers']`**, `depth=4`, más `roll` por hueso y `user*`/`animation_data` | ni `Bone.hide` ni visibilidad de bone collections. **`layers` es el mecanismo legado de 3.x**, no la visibilidad de 4.x/5.x: su presencia no cubre el caso |
| `bl_scene.py`                                    | exclusiones condicionales de `eevee`/`cycles`/`view_settings`/`render`/`camera` (`:621-633`)                                                                             | no hay serialización de `ViewLayerCollection.exclude`                                                                                                      |

Esto confirma que DD-03 son **fixes distintos por capa** y fija el punto exacto de intervención (§10.9).

**Mecanismo de `hide_select`** (causa concreta identificada): dos causas se suman. (1) `hide_select` está ausente del `include_filter` de `BlObject.dump` → no viaja al Cliente. (2) `DynamicRightSelectTimer` (0.1 s) lo fija desde `is_node_readonly`: `obj.hide_select = not is_node_readonly(uuid)`; los nodos son `RP_COMMON`, así que `is_selectable` da verdadero → `hide_select = False` → el objeto aparece habilitado. El comentario del código documenta que el marcador `applied_updates` que se añadía antes provocaba que el handler clasificara la siguiente edición del usuario como dato remoto y omitiera el commit (un toggle armado dejaba de publicar en silencio); por eso el cambio es local-only y nunca se pushea.

**Decisión DD-03 (conforme a la aprobación fundacional, restituida)**: se serializan y restauran `hide_viewport`, **`hide_select`**, flags de colección, `exclude` de view layer y visibilidad de bone collections. Son tres fixes distintos, no uno. La fila de `hide_select` se mantiene como «pierde en ambos modos»; el diagnóstico «se resetea activamente» es correcto y ahora tiene causa concreta.

**Nota de implementación (reconciliación con INV-3)**: transmitir el flag no basta; el temporizador lo sobrescribe a los 0.1 s. La implementación debe reconciliar ambos: sincronizar el valor del Host como dato y evitar que el temporizador lo pise. Además, la premisa del temporizador («solo lectura ⇒ no seleccionable») choca con INV-3, que establece que el Cliente puede seleccionar armaduras y huesos para observar al Host trabajar. La seleccionabilidad en el Cliente no puede gobernarse por propiedad/readonly de forma que bloquee esa interacción. §10.9 prescribe la reconciliación: el temporizador deja de forzar `hide_select` desde `is_node_readonly`; la propiedad/readonly no gatea la selección (INV-3); el valor sincronizado del Host se preserva como dato de organización de la vista. Esta nota es de implementación, no una decisión nueva que divida DD-03 (la propuesta nocturna de división fue retractada).

Casos verificables: `_MCH-Shaft_spline` pasa de `[OcultoVista|NoRender|NoSeleccion]` a `[OcultoVista|NoRender]` en Modo A y a `[NoRender]` en Modo B. `char_grp` pasa de `[NoRender|NoSeleccion]` a `[NoRender]` en ambos modos (confirma que `hide_select` se escribe False, no es omisión). `Fondo`, `Fondo.001`, `Piso`, `Eyes`, `Hair`, `Teeth` pasan de `[NoSeleccion]` a `[Visible/Activo]` en ambos modos. Las tres cámaras de `Escena` y `Punto` pierden `OcultoVista` en Modo B.

#### 6.4.3 Duplicación de colecciones y actions en Modo A

Las colecciones aparecen con sufijo `.001` y las originales no figuran en el árbol. Acciones con `.001`/`.002`/`.003`. El fallback de adopción por nombre cubre Objetos y sus datos, no Colecciones ni Actions. La recreación de la colección es la causa de la pérdida de sus flags.

**Causa estructural verificada (`f00d655`)**: `BlArmature.load` (`bl_armature.py:82-118`) reconstruye el objeto usuario y su colección contenedora cuando no existen, y contiene un `# TODO: recursive parent collection loading` explícito en **`:96`**. Solo se resuelve **el primer nivel** (`data['user_collection'][0]`); la anidación profunda de colecciones no se reconstruye. Esto explica tanto la aparición de colecciones `.001` como la alteración de la jerarquía observada en el Caso 3A, y es el punto de intervención de D-40a/D-41 junto con la adopción por identidad de D-39c.
**Decisiones**: D-39c, INV-4, D-40a/D-41.

#### 6.4.4 Linkado incorrecto de colecciones en Modo B

`Lillia Low` (9 objetos) queda con solo `this.001 Low`; los ocho restantes se reparten: siete Low se vinculan a `Lillia`; `Horse_Cock_Jaws Low` se vincula a `Horse`. Balance 17+1+1 = 19 = 10+9. Todos presentes, mal linkados. Patrón de emparejamiento por prefijo de nombre.
**Decisión**: DD-08.

#### 6.4.5 Nombres con espacio final

El objeto de ojos se denomina `'Eyes '` con espacio final, presente en Host y Cliente. `bpy.data.objects.get("Eyes")` devuelve `None`. La adopción y toda resolución por nombre deben operar sobre nombres exactos, sin normalización implícita.
**Decisión**: DD-36.

---

## 7. Hipótesis de diagnóstico evaluadas y descartadas

| ID    | Hipótesis                                               | Evidencia de descarte                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ----- | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| HI-01 | Desalineamiento del array de pesos                      | `n_pesos_total` idéntico; los pares coinciden como conjunto                                                                                                                                                                                                                                                                                                                                                                                                       |
| HI-02 | Sensibilidad del dual-quaternion al orden de pesos      | VE-01: desactivar `use_deform_preserve_volume` no altera el síntoma                                                                                                                                                                                                                                                                                                                                                                                               |
| HI-03 | `matrix_parent_inverse` corrupto                        | `pinv` idéntico; el defecto reside en el puntero de parent                                                                                                                                                                                                                                                                                                                                                                                                        |
| HI-04 | Modificador Armature apuntando a armadura incorrecta    | Todos apuntan a `rig`                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| HI-05 | `rotation_mode` no contemplado                          | VE-04 + lectura de código: `_rotation_data_path` y `_rotation_values_for_mode` cubren quaternion, axis-angle y euler                                                                                                                                                                                                                                                                                                                                              |
| HI-06 | Orden o flags de modificadores alterados                | Idénticos                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| HI-07 | Transforms delta no preservadas                         | `delta_location` nulo, `delta_scale` unitario, `matrix_basis` identidad                                                                                                                                                                                                                                                                                                                                                                                           |
| HI-08 | Geometría de curva o ajustes de Spline IK no replicados | Todos idénticos                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| HI-09 | Simulación de física activa en el Cliente               | Silenciada a nivel de escena                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| HI-10 | Drivers o propiedades personalizadas divergentes        | `drivers_hash` y `constraints_hash` idénticos; 384 drivers; props por hueso idénticas                                                                                                                                                                                                                                                                                                                                                                             |
| HI-11 | `nodes.active` no serializado                           | Mapa idéntico en las cinco mediciones (DD-04 retirada)                                                                                                                                                                                                                                                                                                                                                                                                            |
| HI-12 | Rest pose, roll o B-Bone no replicados                  | Comparación numérica directa idéntica en ~70 huesos del rig y 40 del Horse. **La verificación numérica exhaustiva de los ~500 huesos del rig tras el apply no se ha ejecutado**; el respaldo de cobertura total proviene de la igualdad de hash M1↔M3. La divergencia de rest observada en una medición intermedia (~37 huesos) no se reprodujo en la medición numérica directa → escritura no determinista del apply, no carencia de serialización (DD-14/DD-20) |
| HI-13 | Payload específico de constraint no replicado           | `CHILD_OF.inverse_matrix`, `IK.iterations`, `TRANSFORM` mapeos, `LIMIT_ROTATION`, `DAMPED_TRACK.track_axis`, flags `COPY_*`: idénticos (`constraints_hash = 7adda1562646` en host-pre, host-post y cliente; DD-17 descartada)                                                                                                                                                                                                                                     |
| HI-14 | Corrupción UTF-8 en nombres                             | `repr()` correcto (`'\u00c1rea'` = `Área`); `Ãrea` es artefacto de consola (DD-21 descartada como corrupción; fix cosmético de logging)                                                                                                                                                                                                                                                                                                                           |
| HI-15 | Fantasmas o curvas inventadas en el Action del Cliente  | 262 fcurves / 3 276 keys idénticos                                                                                                                                                                                                                                                                                                                                                                                                                                |
| HI-16 | Shape keys o su driver como causa                       | 23 keys idénticas salvo `Parpado` Δ7e-06; driver `is_valid = true`; variables idénticas; VE-03                                                                                                                                                                                                                                                                                                                                                                    |
| HI-17 | Asignación posicional de datos de hueso                 | Rest fiel por comparación numérica; `orden_huesos` idéntico                                                                                                                                                                                                                                                                                                                                                                                                       |

**Sobre HI-16**: la expresión del driver es `(z+x)*-100`. La diferencia de entrada medida es ~7e-08; amplificada produce 7e-06, sin efecto geométrico observable. La discrepancia entre la aritmética de `pb.location` y el valor de la key se explica porque el driver lee `LOC_Z` evaluado post-constraint, mientras `pb.location` expone el valor crudo pre-constraint.

**Sobre HI-12 / no determinismo**: la contradicción entre mediciones (rest distinto en una, idéntico en otra) ES el hallazgo: el apply a veces deja el rest mal y otras veces bien según el orden/timing de lo aplicado. Unifica los síntomas intermitentes reportados («volví a testear y ahora sí se transmiten, sin cambiar nada»; «el problema desaparece en el .blend vacío»; «unas partes se deforman y otras no»). Apunta a DD-14 (apply no idempotente) + DD-20 (asimetría create/update), no a «falta el campo X».

**Decisiones de hipótesis retiradas** (registradas para mantenimiento futuro): DD-04 (`nodes.active`, ya preservado), DD-07 (`rotation_mode`, descartada por VE-04 + código), DD-09 (reformulada: orden/flags de modificadores y deltas descartados como causa, se conserva como verificación de cobertura), DD-15 (rest/roll, reclasificada a escritura no determinista, fusionada en DD-14/DD-20), DD-17 (payload de constraints, descartada), DD-21 (mojibake, artefacto de log).

---

## 8. Protocolo de verificación

### 8.1 Condiciones

| ID    | Condición                                                                     |
| ----- | ----------------------------------------------------------------------------- |
| PV-01 | Frame idéntico en Host y Cliente en toda comparación de pose                  |
| PV-02 | Reproducción pausada en ambos lados                                           |
| PV-03 | Host sin actividad de edición durante la medición                             |
| PV-04 | Prohibido guardar el `.blend` durante la medición                             |
| PV-05 | Comparación con tolerancia `1e-4` para matrices; exacta para valores de canal |
| PV-06 | Estado completamente evaluado antes de la lectura                             |
| PV-07 | Estabilización posterior a la conexión: `Live Sync: connected to …` + 5 s     |
| PV-08 | El apply bloquea el hilo principal; los huecos en el muestreo son esperados   |

### 8.2 Errores de medición conocidos

| ID    | Error                                                     | Consecuencia                                                                     | Norma                                                                                        |
| ----- | --------------------------------------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| PM-01 | Hash sobre `str()` de matrices redondeadas                | `round(-1e-06,5)` produce `-0.0`; `str(-0.0) != str(0.0)`                        | Comparar con tolerancia o cuantizar a entero (`int(round(x*1e5))` normaliza el cero firmado) |
| PM-02 | Precisión de redondeo distinta entre instrumentos         | Hashes no comparables                                                            | Fijar una única precisión por campaña                                                        |
| PM-03 | Filtro de nombres específico de un objeto aplicado a otro | Muestra vacía interpretada como dato                                             | Parametrizar el filtro                                                                       |
| PM-04 | Lectura de `spline.points` en curvas BEZIER               | Siempre vacío; los datos están en `bezier_points`                                | Seleccionar según `spline.type`                                                              |
| PM-05 | Conversión de matriz completa con función escalar         | `float(Vector)` falla y devuelve cadena truncada (`ufunc subtract` dtype `<U60`) | Conversión recursiva por filas                                                               |
| PM-06 | `Object.to_curve()` sin `depsgraph` en Blender 5.x        | `required parameter "depsgraph" not specified`                                   | Pasar `depsgraph` explícitamente                                                             |
| PM-07 | Comparación de pose sin igualar frame ni pausar           | Divergencia legítima atribuida a defecto                                         | Aplicar PV-01 y PV-02                                                                        |
| PM-08 | Descarte de hipótesis por juicio cualitativo sin medición | HI-03 descartada prematuramente                                                  | Toda hipótesis se descarta con evidencia medida                                              |
| PM-09 | Hash sensible al orden para verificar identidad           | Reordenamiento produce hash distinto con datos idénticos                         | Ordenar por clave antes de hashear; reportar el orden aparte                                 |
| PM-10 | Volcado de grupos de ID properties mediante `str()`       | Direcciones de memoria, no comparables                                           | Recorrer recursivamente hasta valores escalares                                              |

### 8.3 Puntos de medición

**Serie M — muestreo temporal** (rest, pose evaluada, nodo activo; 10 muestras cada 0.5 s, o 90 s en M4):

| Punto | Instante                                | Objeto                      |
| ----- | --------------------------------------- | --------------------------- |
| M1    | Host, antes de iniciar sesión           | Línea base                  |
| M2    | Host, tras iniciar sesión sin Cliente   | Efecto aislado del snapshot |
| M3    | Cliente Modo A, antes de conectar       | Fidelidad de la copia       |
| M4    | Cliente Modo A, durante y tras el apply | Efecto de la aplicación     |
| M5    | Host, tras la conexión del Cliente      | Verificación de INV-1       |

**Serie P — captura estática** (rest numérico, primer entry de pesos por vértice, orden de huesos, recuentos):

| Punto | Instante                           | Objeto                  |
| ----- | ---------------------------------- | ----------------------- |
| P1    | Host, antes de iniciar sesión      | Línea base numérica     |
| P2    | Cliente Modo A, antes de conectar  | Fidelidad de la copia   |
| P3    | Cliente Modo A, tras el apply      | Efecto de la aplicación |
| P4    | Host, tras la conexión del Cliente | Verificación de INV-1   |
| P5    | Cliente Modo B, tras el apply      | Contraste entre modos   |

_Nota de campaña_: en M4 la consola solo conservó las últimas 9000 líneas; el muestreo debe escribirse a `.txt`, no a prints de consola.

---

## 9. Registro de decisiones de diseño

Fuente normativa única para las decisiones `D-xx`/`DD-xx` (§0.5). Cada decisión indica estado y justificación. Las alternativas consideradas se registran cuando la auditoría las evaluó.

### 9.1 Filosofía y alcance

| ID    | Decisión                                                                                                                            | Justificación                                                                                                                             | Estado      |
| ----- | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| D-01  | La inserción de keyframe constituye la intención de conservar la pose. Confirmar R/G/S no compromete. Animator-first, fricción cero | Principio rector del mantenedor (§1.4)                                                                                                    | ESTABLECIDO |
| D-02  | R1 sin keyframes fantasma para el usuario; código de retención presente e inactivo                                                  | Defecto en R1, capacidad en PKU                                                                                                           | R1          |
| D-28  | Gate de publicación y auto-keying se difieren conjuntamente a PKU                                                                   | El fix atiende principalmente el auto-keying, que el mantenedor no usa; prioridad a defectos con síntoma visible. «Primero lo importante» | PKU · KI-02 |
| D-40  | El overlay de un único frame en el frame del Host es el comportamiento especificado y deseado                                       | Satisface SRS §5.5; confirmado por el mantenedor                                                                                          | ESTABLECIDO |
| D-41  | La opción Reemplazar afecta exclusivamente la escena activa del Cliente; «una sola escena» no es literal                            | La existencia de escenas adicionales no autoriza su modificación (corrección del mantenedor, msg #13 6.2)                                 | R1          |
| D-42  | En colisión de nombre cede el datablock remoto (`.001`); jamás se renombra lo local ni nada del Host                                | INV-2, INV-3                                                                                                                              | R1          |
| DD-34 | No se crea colección de sesión; la jerarquía no se altera. El contenido remoto entra con las mismas colecciones del Host            | Corrección explícita del mantenedor sobre la propuesta de anidar (msg #41.2)                                                              | R1          |

### 9.2 Identidad y adopción

| ID    | Decisión                                                                                                                                                                                                                                                                                                                                                                             | Justificación                                                                                                                                                                                                                            | Estado      |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| D-39  | El UUID es la única identidad entre peers. El nombre nunca decide identidad                                                                                                                                                                                                                                                                                                          | Los `.001` producen desdoblamiento de referencias (DEF-02)                                                                                                                                                                               | R1          |
| D-39b | Pase de adopción único al conectar, con informe («X adoptados, Y creados, Z renombrados»). Emparejamiento por (tipo, nombre) sobre datablocks no reclamados, estampado del UUID del Host, actualización en lugar de recreación. El Cliente no estampa UUIDs propios antes de la adopción. El Host puede emitir un manifiesto de identidad (nombre+tipo+huella → UUID) en el snapshot | Excepción acotada a D-39: heurística única y visible de onboarding, no regla de identidad. Evita emparejar con el `Material` por defecto. Elimina la transferencia de archivo. Sirve para LAN e internet futuro                          | R1          |
| D-39c | Colecciones y Actions se resuelven por la misma ruta de adopción que los Objetos                                                                                                                                                                                                                                                                                                     | Corrige casos 1A/2A y la pérdida de flags de colección                                                                                                                                                                                   | R1          |
| D-39d | Guard estructural O(1) sobre recuentos ya almacenados (nº vértices en mallas, nombres de huesos en armaduras), más huella de contenido sobre las primeras 32 coordenadas de vértice                                                                                                                                                                                                  | Dos mallas distintas pueden compartir recuento. Se ejecuta una vez por datablock al conectar; cero bytes extra en el canal de alta frecuencia; milisegundos vs ~76 s de conexión. Impide la fusión de homónimos no relacionados (Modo C) | R1          |
| D-39e | Se rechaza el flujo «hostear, guardar, copiar después» como prescripción                                                                                                                                                                                                                                                                                                             | La transferencia por canal externo es fricción, peor con internet                                                                                                                                                                        | ESTABLECIDO |
| D-43  | Estampado de UUID obligatorio tras la adopción. Dependencias de fichero identificadas por hash de contenido                                                                                                                                                                                                                                                                          | `WindowsPath` no admite estampado                                                                                                                                                                                                        | R1          |
| DD-22 | Orden topológico de aplicación: padres antes que hijos                                                                                                                                                                                                                                                                                                                               | El registro evidencia carga de hijos antes que padres; funciona de milagro por el fallback de nombre                                                                                                                                     | R1          |
| DD-33 | Prohibida la creación de objetos orphanados. Todo datablock creado queda vinculado y con su animación, o no se crea                                                                                                                                                                                                                                                                  | Un orphanado sin Action usado como parent produce desdoblamiento silencioso (`rig.001`)                                                                                                                                                  | R1          |
| DD-36 | Resolución por nombre exacto, sin normalización de espacios                                                                                                                                                                                                                                                                                                                          | Existencia de `'Eyes '`                                                                                                                                                                                                                  | R1          |

### 9.3 Aplicación de datos

| ID    | Decisión                                                                                                                                                                                                                                                                                                                                                                                                        | Justificación                                                                                                                                                                                                                                                                           | Estado |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| D-44  | Aplicación por identidad, nunca por posición. No se suprime contenido que el dump no describe. R1 propaga borrados exclusivamente mediante señal explícita (mensaje de borrado con UUID), nunca por omisión. **D-44 no aplica a la poda de curvas de Action acotada por procedencia de §4.8 (X-01), que se conserva según §5.6**                                                                                | Inferir borrado por omisión produce corrupción; un borrado explícito es seguro. Sin esto el reviewer vería geometría que el animador ya eliminó. La poda acotada es limpieza de basura propia del overlay tras su invalidación, no supresión de contenido legítimo                      | R1     |
| X-01  | **Resolución de la tensión poda ↔ D-44** (cuestión bloqueante definida y resuelta): invención de curvas preservada como feature core; nunca poda con overlay vivo; poda post-invalidación acotada por procedencia (solo curvas creadas por el overlay y ausentes del dump, nunca legítimas); el registro `created_curves` sobrevive a la invalidación; implementación opción (i) en R1, (iii) como refactor FUT | Compatible con la declaración del mantenedor (msg #52) y con D-44. Detalle en §4.8 y §10.8. Alternativas (ii) descartada (pierde información del dump, requiere heurística con fallo identificado)                                                                                      | R1     |
| DD-14 | Aplicación idempotente: **comparación por contenido dentro de los loaders**, complementaria al guard por identidad de nodo que ya existe en `ApplyTimer` (`_applied_data`)                                                                                                                                                                                                                                      | `ApplyTimer` ya evita reaplicar datos idénticos por identidad de objeto; lo que no existe es la comparación por contenido ante un objeto de datos nuevo con contenido idéntico, que es lo que reescribe los pesos. No reimplementar el guard de nodo. Mata el churn y el eco de commits | R1     |
| DD-28 | El dump transporta el orden de los pesos por vértice y el loader lo reproduce                                                                                                                                                                                                                                                                                                                                   | El reordenamiento ocurre en Modo A y Modo B; la idempotencia sola no corrige el Modo B                                                                                                                                                                                                  | R1     |
| DD-20 | Unificación de las rutas de creación y actualización: ambas aplican el dump en lugar de derivar valores del estado vigente                                                                                                                                                                                                                                                                                      | La creación escribe `matrix_inverse` de reposo; la actualización la sobrescribe con la pose (E-DEF01-5). Explica el no determinismo del rest                                                                                                                                            | R1     |

### 9.4 Payloads y fidelidad

| ID    | Decisión                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Justificación                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Estado      |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- |
| DD-19 | Serialización del payload completo de todos los modificadores: `vertex_indices`, `matrix_inverse` en espacio de reposo, `center`, `strength`, `falloff_*`, `vertex_group`; Mirror (`mirror_object`, ejes, merge, bisect); Displace; Shrinkwrap; Lattice y equivalentes                                                                                                                                                                                                                                                                                                                                                    | Causa raíz probada de DEF-01. Alcance ampliado tras identificar el Mirror sin payload verificado en `Eyelashes`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | R1          |
| DD-13 | Las imágenes empaquetadas se transportan como datos empaquetados, identificadas por el UUID del datablock `Image`. No se extraen al cache ni generan nodo de fichero                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Causa raíz de DEF-03. Elimina el `WindowsPath` sin UUID, la carrera de cache, el espacio final y el eco de commits                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | R1          |
| DD-06 | Cache: **se conserva la base ya existente**, `bpy.utils.extension_path_user(__package__, path='cache', create=True)` (`preferences.py:50-54`, con respaldo `""` ante `ValueError`), expuesta como preferencia `cache_directory`; subárbol `<hash_endpoint>/<username>/` sin `pid`; nombre de fichero por hash de contenido (`<sha256>.<ext>`); escritura atómica (temporal + `os.replace`); purga al cierre limpio **de los ficheros escritos por la propia instancia** (registro de escrituras propias); subárboles abandonados reportados vía DD-27, nunca borrados automáticamente                                     | **Corrección respecto a la propuesta nocturna** (§17.J.3): mover la base a `__file__/cache` sería un cambio de comportamiento sobre una ruta no declarada rota, y metería el cache **dentro** del directorio de la extensión, justo lo contrario de lo deseable. La justificación «`extension_path_user` sin verificar» es factualmente incorrecta: es el default ya embarcado y operativo en la plataforma de referencia. El `pid` proliferaría directorios abandonados; la colisión por contenido es benigna con hash + escritura atómica. La purga acotada a escrituras propias protege INV-3 ante un peer homónimo (mismo username+endpoint) en la misma máquina. **Efecto del renombrado D-18**: al cambiar `__package__`, la raíz de cache cambia y el subárbol de `multi_user` queda abandonado; es benigno (cache regenerable) y se reporta vía DD-27, nunca se borra. Solo relevante para imágenes externas reales (DD-13 excluye las packed) | R1          |
| DD-03 | Serialización y restauración de **todos** los interruptores de restricción/visibilidad: `Object.hide_viewport`/`hide_select`/`hide_render`; familia de colección `RESTRICT_VIEW_OFF`/`RESTRICT_RENDER_OFF`/`RESTRICT_SELECT_OFF` (`Collection.hide_viewport`/`hide_render`/`hide_select`); `ViewLayerCollection.exclude`; y visibilidad en la **DATA de la armadura** (`Bone.hide`, visibilidad de bone collections). **Nota de implementación**: reconciliar `hide_select` con `DynamicRightSelectTimer` (no debe pisar el valor sincronizado ni gatear la selección por propiedad/readonly, que choca con INV-3); §10.9 | Bug confirmado por el mantenedor: los estados desactivados en el Host no se conservan en el Cliente y los objetos/huesos pasan a mostrarse (§6.4.2). Aprobación fundacional restituida tras la retractación nocturna de la propuesta de excluir `hide_select`. Fixes distintos por capa (objeto, colección, view layer, data de armadura). La división de DD-03 propuesta en la noche fue retractada; no es decisión nueva                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | R1          |
| DD-05 | Tras el apply, `update_tag` y redraw para invalidar la caché de shaders                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | El material preview permanece negro hasta crear un espacio nuevo                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | R1          |
| DD-08 | Corrección del linkado de colecciones en Modo B                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | §6.4.4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | R1          |
| DD-09 | Verificación de cobertura de orden y flags de modificadores y de transformaciones delta                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Descartada como causa; se conserva como verificación                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | ESTABLECIDO |
| DD-07 | Corrección de `rotation_mode` en el overlay                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Descartada por VE-04 y lectura de código                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | RETIRADA    |
| DD-04 | Serializar/restaurar `nodes.active` del árbol de shaders                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Verificado como ya preservado (mapa idéntico en M1–M5); no requiere acción                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | RETIRADA    |

### 9.5 Incorporación de escena y experiencia de uso

| ID    | Decisión                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Justificación                                                                                                                                       | Estado |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| D-40a | Diálogo al conectar: **Combinar** (por defecto), **Reemplazar**, **Nueva escena**, más casilla «Recordar mi elección». Combinar conserva todo lo local y entra con la jerarquía del Host (DD-34); Reemplazar purga la escena activa y datos huérfanos con confirmación (D-41); Nueva escena crea una escena dedicada leaving la local intacta                                                                                                                                                  | Operación sin configuración con capacidad de elección; plug-and-play real tras recordar la elección                                                 | R1     |
| D-41a | Reconexión automática (caída de red, misma sesión e instancia, escena ya poblada): no se reejecuta incorporación, adopción ni purga; solo resincronización. Reconexión manual (salió de la sesión o cerró el archivo): se detecta contenido ya adoptado por UUID y se reanuda, no se re-adopta. La purga requiere confirmación explícita en todo caso. El diálogo aparece solo en el primer connect del par (archivo, sesión); la opción recordada jamás dispara acción destructiva automática | La purga automática en reconexión sería destructiva. `load_pre_handler` cierra la sesión al abrir otro archivo: eso es salida, no reconexión (§4.7) | R1     |
| D-41b | En Nueva escena, la adopción excluye objetos ya utilizados por escenas locales                                                                                                                                                                                                                                                                                                                                                                                                                 | La adopción busca en `bpy.data` (global al archivo); sin esta exclusión podría robar un objeto de la escena del usuario para la remota              | R1     |
| D-45  | Documentación de los tres modos. `SRS.md` es especificación interna; el README actual es spec legada; el manual de usuario real se emite cuando la release esté completa. **Estado verificado**: el commit `a19c2ef` ejecutó el renombrado como copia; `README.MD` y `SRS.md` coexisten en `f00d655`, por lo que falta retirar `README.MD` (§4.1 nota 1, §17.J.4)                                                                                                                              | Delimitación de audiencias                                                                                                                          | R1     |

### 9.6 Seguridad y transporte

| ID    | Decisión                                                                                                                                                      | Justificación                                                                                                            | Estado      |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ----------- |
| D-14  | Handshake en LAN: challenge/response HMAC-SHA256 con secreto derivado de la contraseña de sesión (`sha256("live-sync:" + password)`, domain-separated)        | Fricción nula, sin cuentas, reutiliza la auth existente                                                                  | R1          |
| D-16  | Sesión sin contraseña: se permite, se omite el handshake y se documenta la ausencia de autenticación                                                          | No introducir fricción en LAN                                                                                            | R1 · KI-03  |
| D-17  | Handshake con timeout corto y expulsión de handshakes lentos                                                                                                  | El `recv` va en el bucle de accept; no debe reintroducir el bloqueo de D-10                                              | R1          |
| DD-10 | R1 opera en LAN y misma máquina. Internet se difiere a una actualización posterior a PKU                                                                      | Sin infraestructura de retransmisión no hay operación remota sin configuración; Tailscale añade cuenta = fricción        | R1 / FUT    |
| DD-11 | Multiplexado del canal live en la conexión de replicación                                                                                                     | Habilitador de la retransmisión; elimina el segundo puerto y M-10                                                        | FUT         |
| D-15  | Operación remota mediante servicio de retransmisión propio (relay/rendezvous con código de sesión), sin cuentas ni configuración, cuando haya infraestructura | §1.4; única arquitectura plug-and-play real a través de internet (ambos peers conectan hacia afuera; nadie abre puertos) | FUT         |
| D-38  | Se descarta la VPN de terceros (Tailscale/ZeroTier) como requisito                                                                                            | Fricción de cuenta                                                                                                       | ESTABLECIDO |

### 9.7 Robustez y empaquetado

| ID   | Decisión                                                                                                                                                                                             | Estado      |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| D-03 | Gating por constante `PHANTOM_KEYFRAME_MODE = False`; imposible activar por accidente en R1; promoción a preferencia/toggle en PKU                                                                   | R1          |
| D-04 | Registro `_retained` implementado en R1 como puente a PKU: `invalidate_overlays` lo limpia sin restaurar; `cleanup` lo restaura                                                                      | R1          |
| D-05 | Flicker menor aceptado (restore-on-end simple); grace-period diferido                                                                                                                                | R1 · KI-01  |
| D-06 | Guard de armadura: cierre y reapertura al cambiar el UUID activo; filtrado de huesos por objeto dueño (`id_data`)                                                                                    | R1          |
| D-07 | Tolerancia `1e-4` en `_find_point`                                                                                                                                                                   | R1          |
| D-08 | `.is_alive()` en `_transport_alive` (ambos chequeos, broadcaster y receiver)                                                                                                                         | R1          |
| D-09 | Backoff de 5 s con warning por episodio                                                                                                                                                              | R1          |
| D-10 | Vaciado a ~100 Hz mediante `settimeout(0.01)`; `select()` como optimización futura                                                                                                                   | R1          |
| D-11 | Publicación restringida al Host en `SessionToggleLiveSync`. Los clientes no publican por ninguna vía                                                                                                 | R1          |
| D-12 | Eliminación de `ContinuousActionPushTimer` y `_last_hashes`                                                                                                                                          | R1          |
| D-13 | Sustitución de `bpy.data.window_managers["WinMan"]` por `bpy.context.window_manager`, con prueba de regresión                                                                                        | R1          |
| D-18 | Denominación True Level: Playblast, id `true_level_playblast`                                                                                                                                        | R1          |
| D-19 | Versión 1.0.0                                                                                                                                                                                        | R1          |
| D-20 | Maintainer Soilent                                                                                                                                                                                   | R1          |
| D-21 | Archivo `LICENSE` incorporado en `main` (GPL-3.0 completo + aviso propio `Copyright (C) 2026 Soilent` + aviso upstream `Multi-User / Swann Martinez (Slumber)`); requisito de publicación satisfecho | ESTABLECIDO |
| D-22 | Reportes de sesión permanecen en el repositorio y se excluyen de la release                                                                                                                          | ESTABLECIDO |
| D-23 | `README2.MD` y `test.MD` se conservan sin valor normativo y se excluyen de la release                                                                                                                | ESTABLECIDO |
| D-24 | Se conserva el nivel de log INFO                                                                                                                                                                     | ESTABLECIDO |
| D-25 | Exclusión de build de vendorizados, documentación y reportes                                                                                                                                         | R1          |
| D-26 | Eliminación de `net.py` de la fuente del wheel y corrección de `__version__` a `0.10.3`                                                                                                              | R1          |
| D-27 | Vendorizados con uso exclusivo del entorno de desarrollo                                                                                                                                             | ESTABLECIDO |
| D-33 | Carreras de receiver, `_ended` y endurecimiento de `Timer.main` diferidos                                                                                                                            | FUT         |
| D-34 | Extensión del arnés de tests para los fixes de R1 (pruebas de regresión T-xx, §15.3)                                                                                                                 | R1          |
| D-35 | Rango de versiones de Blender sin modificación (min 4.5.0); validación en 5.2.2; anteriores se decidirán tras probar                                                                                 | ESTABLECIDO |
| D-36 | Commits separados por área de responsabilidad                                                                                                                                                        | PROCESO     |
| D-37 | El TDD incorpora código ilustrativo y funcional                                                                                                                                                      | PROCESO     |

### 9.8 Mantenibilidad y diagnóstico

| ID    | Decisión                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Justificación                                                                                                                                                                                                                    | Estado      |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| DD-01 | Adopción de INV-1…INV-5 como propiedades normativas                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Los defectos críticos corresponden a su violación                                                                                                                                                                                | ESTABLECIDO |
| DD-02 | INV-1 con prioridad absoluta. Auditoría y cierre de **todas** las rutas de publicación del Cliente, nombradas explícitamente: `update_external_dependencies` (`handlers.py:333`, fuera de todo guard), rama de Scene sin UUID (`:319-323`), `AnnotationUpdates` (`timers.py:265-267`, **solo activa en Blender < 5.0.0**), `SessionToggleLiveSync` y `SessionCommit` (`operators.py:845-847` / `:812-814`), y auditoría de `sanitize_deps_graph` (`rm`). **Hallazgo adicional verificado**: en `handlers.py:42-44` se asigna `node.instance` antes del test `if node is None`, por lo que ese test es inalcanzable y el bucle no protege frente a un `None` del grafo | Un visor que daña el trabajo del animador es el fallo de mayor gravedad; es silencioso, se propaga y contamina el diagnóstico. La acotación de versión de `AnnotationUpdates` no exime de cerrarla: 4.5.0 es el mínimo declarado | R1          |
| DD-24 | Operador de diagnóstico integrado con comparación Host↔Cliente y emisión de JSON por categorías (rest, pose, constraints genérico+específico, drivers, props por hueso, pesos, modificadores, flags, colecciones, imágenes)                                                                                                                                                                                                                                                                                                                                                                                                                                           | Convierte la atención de rigs no contemplados en un procedimiento de un paso (un botón y un archivo, no diez viajes de ida y vuelta)                                                                                             | R1          |
| DD-25 | Registro de cobertura del serializador: listas explícitas por tipo, lista de propiedades no cubiertas, y prueba que falle ante una propiedad relevante ni cubierta ni excluida                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | La corrección futura se reduce a ampliar una tabla. §1.4                                                                                                                                                                         | R1          |
| DD-27 | Detección e informe de archivos dañados por versiones anteriores; reparación mediante reobtención desde el Host. Prohibida la eliminación automática de datablocks                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Los bytes perdidos no son reconstruibles localmente; la eliminación automática vulnera INV-3                                                                                                                                     | R1          |
| DD-35 | El overlay escribe en el Action que gobierna efectivamente la armadura, a través del slot correcto. El fallback a `slots[0]` y el manejo directo sin Action se documentan como comportamiento existente, sin cambio en R1; el hueco defensivo de `slots[0]` se difiere (KI-09)                                                                                                                                                                                                                                                                                                                                                                                        | Con Action duplicada, escribir en un Action no gobernante anularía la previsualización. La corrección de DEF-02 elimina el trigger                                                                                               | R1          |
| D-46  | Matriz de validación {Modo A, B, C} × {escena ligera, pesada, real}, más prueba de determinismo por N conexiones y caso de homónimos no relacionados                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | El mismo defecto se manifiesta de forma distinta por modo; el arnés actual no los distingue                                                                                                                                      | R1          |
| DD-37 | La tabla de peso por datablock de SRS §8.3 se registra como **orientativa, no restrictiva**: `mesh`, `image` y `volume` no conviene armarlos por defecto, pero el toggle permite al usuario elegirlos                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | El propio SRS §8.3 lo califica de elección del usuario                                                                                                                                                                           | ESTABLECIDO |

### 9.9 Decisiones preservadas para la Phantom Keyframe Update

| ID   | Decisión                                                                                                                                                                                               | Estado |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| D-29 | Persistencia de fantasmas del Host mediante **ID Properties** serializadas con el `.blend`; carrier **fuera del payload replicado del Action** (identificador estable a nivel de armadura o de escena) | PKU    |
| D-30 | Cliente efímero: los fantasmas desaparecen al desconectar; reaparecen al reconectar **solo** con el toggle del Host activo                                                                             | PKU    |
| D-31 | Replay al conectar mediante mensaje de handshake gated por el toggle del Host; el registro del Cliente es la mitad receptora                                                                           | PKU    |
| D-32 | Preguntas de diseño de ID Properties guardadas para PKU (§12.5)                                                                                                                                        | PKU    |

### 9.10 Decisiones y enmiendas retiradas

| ID    | Disposición original                         | Motivo de retirada                                                                        |
| ----- | -------------------------------------------- | ----------------------------------------------------------------------------------------- |
| E-02  | `RESOLVING` especificado y no implementado   | El código define y usa `IDLE`/`INTERACTING`/`RESOLVING`; hecho verificado, no divergencia |
| DD-04 | Serializar `nodes.active`                    | Ya preservado (HI-11); sin acción                                                         |
| DD-07 | Corregir `rotation_mode` en el overlay       | Descartada por VE-04 + lectura de código (HI-05)                                          |
| DD-15 | Rest pose / roll / B-Bone como campo perdido | Reclasificada a escritura no determinista; fusionada en DD-14/DD-20 (HI-12)               |
| DD-17 | Payload type-specific de constraints         | Todo idéntico (`constraints_hash`); descartada (HI-13)                                    |
| DD-21 | Mojibake UTF-8 en nombres                    | Artefacto de consola; `repr()` correcto (HI-14); fix cosmético de logging, FUT            |

_Verificaciones de código resueltas durante la auditoría (no decisiones): C-02 (`start` sin canales), X-07 (FAST + `fcurve.update()` incondicional → P-04), T-03 (D-46 en §9), V-01 (filtro usa `is_streaming()`), V-02 (rama `state==UP` existe; `push_animation_dependencies` condicionada a `is_live` → KI-08). Detalle en §17.I._

---

## 10. Especificación de la release 1.0.0

### 10.1 Eliminación de keyframes fantasma (D-02/03/04)

```python
# live_sync.py — puerta de capacidad
PHANTOM_KEYFRAME_MODE = False   # R1: False. PKU: preferencia gobernada por el toggle del Host
```

```python
# ClientPoseApplier.__init__
self._retained = {}   # clave -> edit. Inactivo en R1
```

```python
# _on_end — sustituye la rama de confirmación
if is_cancelled:
    self._restore(edit)           # cancelación → siempre restaura
elif PHANTOM_KEYFRAME_MODE:
    self._retained[key] = edit    # PKU: retención + store del Host
else:
    self._restore(edit)           # R1: sin fantasmas
# se conserva el _edits.pop(key) existente
```

```python
# invalidate_overlays(reason) — adición, sin restaurar
self._retained.clear()
```

```python
# cleanup() — restauración de retenidos (adaptar al helper existente)
for edit in list(self._retained.values()):
    self._restore(edit)
self._retained.clear()
```

**Efecto normativo**: la finalización de un stream confirmado sin keyframe real restaura el overlay. La recarga del Action sustituye el overlay por el keyframe real y descarta el registro sin restaurar. El cierre de sesión restaura los retenidos, impidiendo su incorporación al archivo en disco. El flicker del caso confirmar+keyear+deseleccionar rápido se acepta (KI-01, E-12).

### 10.2 Guard de armadura activa (D-06)

```python
# _update_selection — inicio
active = self._active_armature()
if active is not None and active.uuid != self.armature_uuid:
    self._finish(broadcaster)
    self._begin(broadcaster)
    return
```

```python
# reunión de huesos en _begin y _update_selection
bones = [b for b in selected_pose_bones if b.id_data == active]
```

### 10.3 Correcciones puntuales (D-07/08/09/10)

```python
# _find_point (:905)
if abs(point.co.x - frame) < 1e-4:
```

```python
# _transport_alive (:1504) — ambos chequeos de hilo
... and t._thread is not None and t._thread.is_alive()
```

```python
# LiveSyncBroadcaster.start (:270)
self._listener.settimeout(0.01)
```

```python
# LiveSyncEngine.__init__
self._next_retry = 0.0

# _sync_transport — inicio
if time.monotonic() < self._next_retry:
    return False

# _sync_transport — except OSError, junto al shutdown existente
self._next_retry = time.monotonic() + 5.0
logging.warning("Live Sync: fallo al bindear el puerto; reintento en 5 s")
```

### 10.4 Acceso al WindowManager (D-13)

Sustitución en `live_sync.py`, `operators.py` y `timers.py`:

```python
bpy.data.window_managers["WinMan"]  →  bpy.context.window_manager
```

Las escrituras de estado ya usan `context.window_manager.session` (`operators.py:220`, `:451`, `:496`); las lecturas usan el nombre literal (`live_sync.py:121`, `_runtime_settings`). En el caso nominal ambos identifican el mismo objeto y el cambio es conductualmente nulo; en el caso de WindowManager renombrado corrige la divergencia. Estrictamente más seguro (alinea lecturas con escrituras). Prueba de regresión T-08.

### 10.5 Publicación restringida al Host (DD-02, D-11, D-12)

`SessionToggleLiveSync.execute` somete el commit y push inmediato a la verificación de rol del Host. **Cierre de las rutas nombradas en DD-02**:

- `update_external_dependencies` se somete a `can_publish` o se restringe al Host (causa exacta de DEF-03).
- La rama de Scene sin UUID y `AnnotationUpdates` se someten a la misma verificación.
- Se audita `sanitize_deps_graph` (`rm`): verificar si la biblioteca acepta `RP_COMMON` para cualquier peer; si lo acepta, restringir al Host.

Eliminación de `ContinuousActionPushTimer` y `_last_hashes` (D-12). Los clientes no publican por ninguna vía (AC-07, T-07).

### 10.6 Handshake de LAN _(ilustrativo)_ (D-14/16/17)

```python
live_key = hashlib.sha256(b"live-sync:" + session_password.encode()).digest()

# Host, tras accept — con timeout corto (D-17)
nonce = os.urandom(16)
conn.sendall(nonce)
resp = recv_exact(conn, 32)
if hmac.compare_digest(resp, hmac.new(live_key, nonce, hashlib.sha256).digest()):
    ...   # autenticado: incorporar a _clients
else:
    conn.close()

# Cliente, tras connect
nonce = recv_exact(sock, 16)
sock.sendall(hmac.new(live_key, nonce, hashlib.sha256).digest())
```

Sesión sin contraseña: el handshake se omite (D-16, KI-03). El receiver obtiene la contraseña de la configuración de sesión existente. La derivación es domain-separated. El timeout corto evita reintroducir el stall de latencia de D-10 (el `recv` va en el bucle de accept).

### 10.7 Manifiesto objetivo (D-18/19/20/25/26)

```toml
schema_version = "1.0.0"
id = "true_level_playblast"
name = "True Level: Playblast"
version = "1.0.0"
tagline = "Zero-render live playblast. Animate while it plays."
maintainer = "Soilent"
blender_version_min = "4.5.0"
license = ["SPDX:GPL-3.0-or-later"]
type = "add-on"
platforms = ["windows-x64", "macos-x64", "macos-arm64", "linux-x64"]
# website omitido intencionadamente: decisión diferida por el mantenedor (no bloqueante)
wheels = [
   "./wheels/orderly_set-5.2.3-py3-none-any.whl",
   "./wheels/deepdiff-8.1.1-py3-none-any.whl",
   "./wheels/replication-0.10.3-py3-none-any.whl",
]
[permissions]
network = "Need to sync scene data to server"
[build]
paths_exclude_pattern = [
  "__pycache__/", ".*", "*.zip", "cache/", "venv/", "tests/",
  "replication/", "deepdiff/", "orderly_set/",
  "*.md", "*.MD", "*.txt",
]
```

**Reconstrucción del wheel de `replication`**: eliminar `net.py` de la fuente y corregir `__version__` a `0.10.3`.

**Estado de `tagline`, `website` y `LICENSE` (verificado en `main`).** El mantenedor ya escribió el `tagline` en el manifiesto publicado: `"Zero-render live playblast. Animate while it plays."`. El campo `website` se **omitió intencionadamente** para decidirlo más adelante; no es un requisito bloqueante de R1 y queda como decisión diferida no crítica (§13). El archivo `LICENSE` ya está incorporado en `main` (35 770 B): texto completo GPL-3.0, aviso propio `Copyright (C) 2026 Soilent (github.com/Devangel-gonzalez)` y preservación del aviso upstream `Based on "Multi-User" for Blender — Copyright (C) 2019-2026 Swann Martinez (Slumber)`. Con ello quedan satisfechos D-18/19/20 (identidad), D-21 (LICENSE) y el `tagline`; la única decisión de contenido abierta es el valor futuro de `website`, explícitamente postergada por el mantenedor.

### 10.8 Poda acotada por procedencia (X-01, D-44)

**Objetivo**: conservar la poda como limpieza de curvas inventadas por el overlay que quedan huérfanas tras una recarga a mitad de stream, sin podar nunca curvas legítimas ni curvas con overlay vivo.

**Requisito previo**: el registro de procedencia debe sobrevivir a la invalidación. Hoy `invalidate_overlays` hace `self._edits.clear()` en `live_sync.py:1414`, que destruye `created_curves` (registro definido en `:957`). Se captura el registro antes de limpiarlo:

```python
# ClientPoseApplier.invalidate_overlays(reason) — ilustrativo
# Capturar la procedencia ANTES de descartar el registro
provenance = {}
for key, edit in self._edits.items():
    if edit.created_curves:
        provenance.setdefault(edit.action_uuid, set()).update(edit.created_curves)
self._pending_provenance = provenance   # consultable por bl_action.load
self._retained.clear()
self._edits.clear()
# ... resto de la invalidación existente
```

**Opción (i) R1**: `bl_action.load` consulta el registro con import diferido (evita el ciclo `live_sync` → `bl_types` → `live_sync`):

```python
# bl_types/bl_action.py — load(), rama >= (5,0,0), tras cargar las curvas del dump
# ilustrativo: adaptar accessors al código real
def _prune_overlay_orphans(action, dump_curves):
    from .. import live_sync   # import diferido dentro de la función
    created = live_sync.overlay_created_curves(action.uuid)   # set (data_path, index)
    if not created:
        return
    channelbag = ...   # el slot reciamente cargado
    for (data_path, index) in created:
        if (data_path, index) in dump_curves:
            continue          # el Host keyeó ese canal: es legítima, NO podar
        fcurve = channelbag.fcurves.find(data_path, index=index)
        if fcurve is not None:
            channelbag.fcurves.remove(fcurve)   # inventada y ausente del dump → huérfana
```

**Invariantes de la poda**:

- Nunca se ejecuta con un overlay vivo (la invalidación ya lo descartó).
- Solo poda curvas en `created_curves` (procedencia overlay) y ausentes del dump.
- Una curva inventada que recibió datos reales del Host (el Host keyeó después ese canal) está en el dump → se conserva.
- Nunca poda curvas legítimas ausentes de un dump parcial (D-44).

**Refactor FUT, opción (iii)**: marcar la procedencia en la propia FCurve mediante propiedad personalizada al crearla; `bl_action.load` poda las marcadas y ausentes del dump sin consultar a `live_sync`. Sujeto a verificación de que la FCurve admite propiedades personalizadas y el marcador sobrevive al apply (con reuso de slots la curva sobrevive, plausible pero no comprobado). Elimina la deuda de acoplamiento de (i).

### 10.9 Flags de restricción/visibilidad y reconciliación de `hide_select` (DD-03)

**Alcance exhaustivo** (bug confirmado por el mantenedor: los estados desactivados en el Host no se conservan en el Cliente). Serializar y restaurar, por capa:

| Capa                 | Propiedades a transportar                                                                                               | Serializador                                |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| Objeto               | `hide_viewport`, `hide_select`, `hide_render`                                                                           | `bl_object.py` (añadir al `include_filter`) |
| Colección            | `hide_viewport`, `hide_render`, `hide_select` (familia `RESTRICT_VIEW_OFF`/`RESTRICT_RENDER_OFF`/`RESTRICT_SELECT_OFF`) | `bl_collection.py`                          |
| View layer           | `ViewLayerCollection.exclude`                                                                                           | `bl_scene.py` / view layer                  |
| **DATA de armadura** | `Bone.hide` (y `hide_viewport` si aplica), visibilidad de bone collections (`BoneCollection.is_visible`)                | `bl_armature.py` (data, **no** pose)        |

**Nota de capa**: la visibilidad de huesos y bone collections reside en `armature.data`, no en `armature.pose`. Debe transportarla el loader de la data de armadura; es independiente de la pose evaluada y del frame. En Modo B estos valores nacen en default y deben recibir el valor del Host; en Modo A el objeto preexistente puede conservarlos por inercia, pero el apply debe restaurarlos explícitamente para no depender de la inercia.

**Punto exacto de intervención (verificado en `f00d655`)**. Los filtros actuales y el comportamiento del lado de carga determinan cuánto trabajo requiere cada capa:

| Capa             | Filtro actual del dump                                                                                     | Lado de carga                                                                                                                                                             | Trabajo requerido                                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Objeto           | `include_filter` con `hide_render` (`bl_object.py:823`); faltan `hide_viewport` y `hide_select`            | `BlObject.load` usa `Loader()` **sin `include_filter`**, solo `exclude_filter=["modifiers","constraints"]` (`:692`)                                                       | **Basta ampliar el `include_filter` del dump**: el loader no tiene lista blanca, escribe lo que llegue en el payload |
| Colección        | `include_filter = ["name","instance_offset"]` (`bl_collection.py:120-123`)                                 | `BlCollection.load` usa `Loader()` **sin ningún filtro** (`:103-104`)                                                                                                     | **Basta ampliar el `include_filter`** con los tres flags                                                             |
| View layer       | no existe serialización de `exclude`                                                                       | no existe ruta                                                                                                                                                            | **Bloque nuevo** de dump y load; es la capa con más trabajo                                                          |
| DATA de armadura | `include_filter` con `layers` (legado 3.x), sin `Bone.hide` ni bone collections (`bl_armature.py:166-176`) | `BlArmature.load` entra en **modo EDIT** (`:129`) y escribe sobre `edit_bones`, con asignaciones explícitas por hueso más `loader.load(new_bone, bone_data)` (`:131-150`) | **No basta ampliar el filtro.** Ver nota siguiente                                                                   |

**Nota crítica sobre la armadura**: existe una asimetría estructural verificada. El dump lee `armature.bones` (tipo `Bone`), mientras el load escribe sobre `datablock.edit_bones` (tipo `EditBone`) **dentro de un bucle en modo EDIT**. La visibilidad de hueso y la visibilidad de las bone collections son propiedades de nivel de `Bone` y de la armadura, no del hueso de edición. Por tanto deben transportarse y aplicarse en un **bloque propio, fuera del bucle de `edit_bones` y tras volver a modo OBJECT**, sobre `datablock.bones[...]` y sobre las colecciones de la armadura; y las bone collections requieren su propia entrada de dump porque no cuelgan de un hueso individual. Ampliar el `include_filter` y dejar que `loader.load(new_bone, ...)` lo escriba en modo EDIT no es una implementación válida. La comprobación de qué propiedades de visibilidad expone cada tipo en 4.5 y 5.2 queda registrada como verificación previa (§15.4).

**Reconciliación con `DynamicRightSelectTimer`** _(ilustrativo)_: el temporizador deja de forzar `hide_select` desde `is_node_readonly`. La propiedad/readonly no gatea la seleccionabilidad (INV-3). El valor sincronizado del Host se preserva como dato de organización de la vista:

```python
# timers.py — DynamicRightSelectTimer.execute (ilustrativo)
# ANTES: obj.hide_select = not session.repository.is_node_readonly(uuid)
# R1: no forzar hide_select por permisos. La seleccionabilidad del Cliente
# no se gobierna por propiedad/readonly (INV-3: el Cliente puede seleccionar
# cualquier hueso para observar al Host trabajar). El hide_select sincronizado
# del Host (DD-03) se preserva como dato. Si se necesita un indicador de
# permisos, se usa un canal visual distinto que no bloquee la selección.
```

**Nota**: la premisa «solo lectura ⇒ no seleccionable» queda retirada por tensión con INV-3. El indicador de permisos, si se conserva, no debe impedir la selección de huesos. La transmisión del flag por sí sola no basta mientras el temporizador lo sobrescriba; de ahí la reconciliación. Pruebas de regresión T-16 (flags preservados en los tres modos, incluida la visibilidad de huesos en la data de armadura) y T-21 (seleccionabilidad del Cliente no gateada por propiedad).

### 10.10 Orden de ejecución

| Fase | Contenido                                                                                                      | Defecto que resuelve              |
| ---- | -------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 1    | DD-02 · cierre de todas las rutas de publicación del Cliente                                                   | INV-1, DEF-03                     |
| 2    | D-39, D-39b, D-39c, D-39d, D-43, DD-22, DD-33, DD-36 · identidad, adopción, orden, orphanados, nombres exactos | DEF-02                            |
| 3    | DD-19, DD-20 · payload de modificadores y unificación create/update                                            | DEF-01                            |
| 4    | DD-13, DD-06 · imágenes empaquetadas y cache                                                                   | DEF-03                            |
| 5    | DD-14, DD-28 · idempotencia por contenido y orden de pesos                                                     | §6.4.1, no determinismo           |
| 6    | D-44 · propagación explícita de borrados                                                                       | Divergencia por acumulación       |
| 7    | DD-03, DD-08, DD-05 · flags/visibilidad (incl. hide_select y huesos en data), linkado, invalidación de shaders | §6.4.2, §6.4.4                    |
| 8    | D-02, D-03, D-04, X-01 · eliminación de fantasmas y poda acotada                                               | A-02                              |
| 9    | D-06…D-13 · correcciones puntuales                                                                             | A-03…A-06, P-01, M-04, M-05, M-06 |
| 10   | D-40a, D-41, D-41a, D-41b, DD-34 · incorporación de escena                                                     | Modos B y C                       |
| 11   | D-14, D-16, D-17 · handshake                                                                                   | S-01                              |
| 12   | D-18…D-26 · empaquetado                                                                                        | M-01, M-03, M-08                  |
| 13   | DD-24, DD-25, DD-27, D-46, D-34 · diagnóstico, cobertura, reparación, pruebas                                  | Mantenibilidad                    |

### 10.11 Separación de commits (D-36)

`identidad-adopcion` · `host-only` · `payload-modificadores` · `imagenes-packed` · `apply-idempotente` · `borrados-explicitos` · `flags-linkado` · `sin-fantasmas` · `correcciones-puntuales` · `onboarding` · `handshake` · `packaging` · `diagnostico-pruebas`

### 10.12 Revisión previa de mitigaciones existentes

Los commits de 19 oct 2026 «Keep modifier object references when reloading an Object» y «Extend the apply identity guard to forced parent and dependency reloads» indican mitigaciones previas sobre las mismas familias de defectos. **Requisito**: revisar dichos diffs antes de implementar las fases 2, 3 y 5, para no revertir mitigaciones vigentes. El guard de identidad por nodo de `ApplyTimer` (`_applied_data`) ya implementa idempotencia a nivel de nodo y no debe reimplementarse (DD-14 es complementaria, por contenido, dentro de los loaders).

---

## 11. Defectos conocidos aceptados en la release 1.0.0

| ID    | Síntoma                                                                                                                                                            | Mecanismo                                                                                                                                                                | Mitigación operativa                                            | Corrección planificada         |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- | ------------------------------ |
| KI-01 | Flicker breve al confirmar, keyear y deseleccionar en rápida sucesión                                                                                              | R1 sigue el orden de SRS §6.4 (E-12): `end` restaura el overlay antes de la llegada del push real; se aparta de la retención-hasta-reemplazo de §5.4, trade-off aceptado | Ninguna requerida                                               | Grace-period · FUT · D-05      |
| KI-02 | Con auto-keying o ediciones exclusivas de panel, la publicación se demora hasta el siguiente operador                                                              | A-01                                                                                                                                                                     | Insertar keyframe manualmente o ejecutar otro operador          | PKU · D-28                     |
| KI-03 | Sesión sin contraseña: canal live sin autenticación                                                                                                                | D-16                                                                                                                                                                     | Establecer contraseña de sesión                                 | Documentado                    |
| KI-04 | Un cliente lento puede bloquear la emisión hasta 1 s                                                                                                               | P-02                                                                                                                                                                     | Número reducido de clientes en LAN                              | FUT                            |
| KI-05 | Transmisión en claro en LAN                                                                                                                                        | S-02                                                                                                                                                                     | —                                                               | FUT                            |
| KI-06 | Carreras de cierre de receiver, resurrección de overlay y desconexión por excepción                                                                                | R-01, R-02, R-03                                                                                                                                                         | —                                                               | FUT · D-33                     |
| KI-07 | Operación por internet no soportada ni validada                                                                                                                    | DD-10                                                                                                                                                                    | Operación en LAN                                                | Posterior a PKU                |
| KI-08 | Con el toggle desarmado, el Action puede no publicarse al confirmar                                                                                                | `push_animation_dependencies` condicionada a `is_live`; insertar un keyframe reporta el Object, no el Action (V-02)                                                      | Armar el toggle (recomendación SRS §6.5)                        | SRS §9.11 registrado con causa |
| KI-09 | Action con slots pero `action_slot=None`: el overlay puede escribirse en `slots[0]` no gobernante, sin efecto visible ni fallback directo (`live_sync.py:929-930`) | Hueco defensivo leído en código, no medido en práctica; alcanzable tras Action duplicada (DEF-02)                                                                        | La corrección de DEF-02 (D-39) elimina el trigger; aviso en log | FUT · fallback directo (DD-35) |

---

## 12. Phantom Keyframe Update — diseño preservado

### 12.1 Capacidad especificada

- Generación de keyframes fantasma sin la secuencia de conmutación de hueso y retorno (el repro del bug actual: transformar → seleccionar otro hueso → cambiar frame → volver).
- Toggle **Phantom Keyframe Mode**: activado, permite la inserción; desactivado, los fantasmas permanecen almacenados en el Host; reactivado, se rematerializan en el Cliente.
- Operación de borrado de fantasmas.
- Operación de confirmación, con incorporación al Dope Sheet del Host.

### 12.2 Semántica de persistencia (D-29, D-30)

**Host**: los fantasmas deben sobrevivir al cierre y reapertura del archivo y al reinicio de la sesión. No pueden residir en memoria de proceso. Se emplean **ID Properties**, serializadas con el `.blend`.

**Restricción de aislamiento**: el store de fantasmas del Host debe residir **fuera del payload replicado del Action**. Su inclusión en los datos que porcelain dumpea y recarga provocaría su supresión en cada recarga o su replicación a los peers como keyframes reales. El carrier debe ser un identificador estable a nivel de armadura o de escena, independiente del dump del Action.

**Cliente**: los fantasmas son efímeros. Se restauran al desconectar y se rematerializan al reconectar **únicamente si el toggle del Host está activo**.

### 12.3 Puente construido en la release 1.0.0 (D-03, D-04)

Puerta de capacidad `PHANTOM_KEYFRAME_MODE`, registro `_retained`, su limpieza en `invalidate_overlays` y su restauración en `cleanup`. La PKU requiere exclusivamente: promoción de la puerta a preferencia y toggle, store de ID Properties en el Host, replay al conectar e interfaz de operación. No se requiere reescritura.

### 12.4 Corrección del gate de publicación y auto-keying (D-28, A-01)

**Diseño de dos componentes**, requerido por la naturaleza event-driven del handler:

```python
# live_sync.py
def transform_in_progress() -> bool:
    if interactive_transform_active() is None:
        return False
    s = engine.streamer
    if s.state == INTERACTING:
        return (time.monotonic() - s._last_motion) <= TRANSFORM_IDLE_TIMEOUT
    return True   # pose_live_sync desactivado: comportamiento heredado
```

```python
# handlers.py:170
dragging = live_sync.transform_in_progress()
```

```python
# LiveSyncEngine.__init__
self._last_gate_closed = False

# engine.tick, rama Host, tras streamer.tick()
closed = transform_in_progress()
if self._last_gate_closed and not closed:
    from . import handlers
    handlers.on_scene_update(bpy.context.evaluated_depsgraph_get())
self._last_gate_closed = closed
```

**Fundamento del primer componente**: `_motion_stalled()` exige el retorno de la pose al snapshot, condición que una pose confirmada no satisface; ninguna señal existente abre el gate tras la confirmación. El gate se basa en recencia de movimiento.

**Fundamento del segundo componente**: la última actualización del depsgraph se produce en la confirmación, con movimiento aún reciente. Transcurrido el umbral de inactividad no se generan nuevas actualizaciones, por lo que se requiere una reinvocación única del handler.

**Preservación del comportamiento anti-bloqueo**: durante una transformación real el gate permanece cerrado y no se ejecuta ningún commit; el transporte es exclusivamente el canal TCP crudo (preserva el rationale anti-freeze del mantenedor: dragear actualiza el depsgraph constantemente y muchos commits congelan la computadora). Si el movimiento se detiene más allá del umbral a mitad de transformación, el gate se abre pero el Action no ha cambiado, por lo que no existe nada que publicar.

### 12.5 Cuestiones de diseño diferidas (D-32)

- Carrier exacto de las ID Properties: armadura o escena.
- Esquema por fantasma: UUID de armadura, nombre de hueso, frame, canales, valores, handles, autor, marca temporal.
- Indexación y acumulación de múltiples fantasmas por hueso en distintos frames.
- Reglas de limpieza: si la inserción de un keyframe real en el frame suprime su fantasma; semántica de confirmación y de borrado.
- Versionado del esquema para migración entre actualizaciones.

### 12.6 Replay al conectar (D-31)

Mensaje de handshake que retransmite los fantasmas retenidos del Host, condicionado al estado del toggle. El registro del Cliente constituye la mitad receptora y ya está implementado en R1.

---

## 13. Trabajo diferido

| Área           | Contenido                                                                                                                                     | Referencia         |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| Rendimiento    | Aislamiento de cliente lento mediante emisión no bloqueante por cliente                                                                       | P-02               |
| Rendimiento    | Evaluación de la eliminación de `fcurve.update()` en la rama de punto existente de `_set_keyframe`, con verificación de corrección            | P-04               |
| Presentación   | Grace-period para eliminación del flicker (retención-hasta-reemplazo de SRS §5.4)                                                             | D-05 · E-12        |
| Seguridad      | TLS nativo con certificado autofirmado fijado por el Cliente                                                                                  | S-02, S-03         |
| Robustez       | Carreras de receiver y `_ended`; endurecimiento de `Timer.main`                                                                               | D-33               |
| Rendimiento    | Vaciado basado en `select()`                                                                                                                  | D-10               |
| Herencia       | Eliminación de la manipulación de `sys.path` en `replication`                                                                                 | M-07               |
| Multi-objeto   | UUID por hueso (`bone.id_data.uuid`) en el protocolo                                                                                          | A-03               |
| Transporte     | Operación por internet, servicio de retransmisión, multiplexado del canal live                                                                | DD-10, DD-11, D-15 |
| Compatibilidad | Validación y declaración de versiones de Blender anteriores                                                                                   | D-35               |
| Publicación    | ~~Archivo `LICENSE`~~ — incorporado en `main` (D-21, ESTABLECIDO)                                                                             | D-21               |
| Documentación  | Corrección de los docstrings desactualizados sobre desmontaje de slots (`is_action_node`, `ApplyTimer.execute`, `invalidate_overlays`)        | §4.5, §4.8         |
| Documentación  | Fix cosmético de logging mojibake UTF-8 (artefacto de consola, no corrupción de datos)                                                        | HI-14 · DD-21      |
| Arquitectura   | Refactor de la poda a marcador de procedencia en la FCurve (opción iii)                                                                       | X-01 · §10.8       |
| Arquitectura   | ~~Ubicación del directorio de cache fuera del directorio de la extensión~~ — **ya satisfecho**: el código usa `extension_path_user` (§17.J.3) | DD-06              |
| Empaquetado    | Valor futuro de `website` del manifiesto (omitido intencionadamente por el mantenedor; no bloqueante). `tagline` ya escrito                   | §10.7              |
| Overlay        | Fallback al manejo directo del hueso cuando no hay slot gobernante (eliminar `slots[0]`)                                                      | KI-09 · DD-35      |

---

## 14. Modelo de amenazas y seguridad

### 14.1 Modelo de comunicación

Direccionalidad Host→Cliente. El broadcaster emite exclusivamente; el receiver lee exclusivamente. El Cliente no transmite datos que se apliquen. La propiedad «solo el Host inyecta» se satisface por diseño, **condicionada al cierre de las rutas de DD-02**.

### 14.2 Amenazas

| Amenaza                      | Cobertura en R1               |
| ---------------------------- | ----------------------------- |
| Interceptación               | No cubierta · KI-05           |
| Interposición / suplantación | No cubierta                   |
| Conexiones no autorizadas    | Handshake D-14                |
| Traversal de NAT             | No aplicable: R1 opera en LAN |

### 14.3 Nota sobre exposición directa de puertos

La operación remota sin servicio de retransmisión requeriría abrir en el router del Host el puerto de sesión **y el incrementado en uno**, con conexión del Cliente a la dirección pública. Expone ambos puertos y exige autenticación y cifrado. No constituye un flujo soportado en R1. El plug-and-play real por internet requiere relay propio (D-15): ambos peers conectan hacia afuera a un servidor público con un código de sesión; nadie abre puertos.

---

## 15. Estrategia de verificación y validación

### 15.1 Arnés existente

`tests/test_live_sync.py` se ejecuta dentro de Blender mediante `--background --python`. El bootstrap copia el addon a un directorio temporal como paquete e incorpora los wheels al `sys.path` por importación desde zip. Ejercita registro del addon, formato de cable, round-trip de matrices, transporte TCP y overlay reversible sobre datos reales de `bpy`. Emite `RESULT: PASS` o `RESULT: FAIL`. Estado declarado: **52/52 verificaciones en Blender 4.5 y 5.1**. D-34 extiende el arnés con las pruebas de §15.3.

### 15.2 Matriz de validación (D-46)

|               | Modo A    | Modo B    | Modo C    |
| ------------- | --------- | --------- | --------- |
| Escena ligera | requerido | requerido | requerido |
| Escena pesada | requerido | requerido | requerido |
| Escena real   | requerido | requerido | requerido |

**Escena ligera**: una armadura con centenares de huesos controlando un cubo. _Contexto del diario de desarrollo_: este caso existía porque la escena ligera llegó a congelar el sistema con centenares de huesos, fallo ya corregido; el caso de prueba preserva esa regresión.

**Escena pesada**: dos armaduras de seis huesos, rig duplicado con subdivisión aplicada múltiples veces, 1 966 137 caras / 1 966 112 vértices / 3 932 226 triángulos, textura de imagen empaquetada.

**Escena real**: personaje cervitaura con centenares de keyframes, rig que controla múltiples objetos, versiones original y low poly gobernadas por la misma armadura, centenares de imágenes, formas de hueso personalizadas en colección propia.

**Casos adicionales obligatorios**: determinismo por N conexiones consecutivas (resultado idéntico); homónimos no relacionados (el `Cube` del paisaje del Cliente vs el `Cube` del Host: el guard estructural D-39d y la huella deben impedir la fusión).

### 15.3 Pruebas de regresión específicas

| ID   | Verificación                                                                                                                                                                                                       |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| T-01 | Overlay confirmado sin keyframe real se restaura al `end`; no queda keyframe retenido                                                                                                                              |
| T-02 | Con la puerta activa, el overlay se retiene en `_retained`; `cleanup` restaura; `invalidate_overlays` limpia sin restaurar                                                                                         |
| T-03 | El cambio de armadura activa reinicia el stream con el UUID correcto; los huesos de otras armaduras no se transmiten bajo UUID ajeno                                                                               |
| T-04 | Un keyframe real en frame fraccionario no se captura al editar el frame entero adyacente                                                                                                                           |
| T-05 | La finalización del hilo de transporte se detecta y el transporte se reconstruye                                                                                                                                   |
| T-06 | El fallo de bind produce backoff sin saturación de log                                                                                                                                                             |
| T-07 | El Cliente no puede publicar por ninguna vía, incluidas `update_external_dependencies`, la rama de Scene sin UUID y `AnnotationUpdates`                                                                            |
| T-08 | El renombrado del WindowManager no afecta a la lectura del estado de sesión                                                                                                                                        |
| T-09 | Handshake: contraseña correcta acepta, incorrecta rechaza, ausencia omite el handshake                                                                                                                             |
| T-10 | No se crea `rig.001` ni ningún duplicado de armadura; los punteros de parent resuelven al original                                                                                                                 |
| T-11 | Los Hook modifiers conservan `vertex_indices` y `matrix_inverse` de reposo en Modo A y Modo B                                                                                                                      |
| T-12 | Las imágenes empaquetadas no generan nodo de fichero ni entrada de cache                                                                                                                                           |
| T-13 | El overlay escribe en el Action gobernante a través del slot correcto                                                                                                                                              |
| T-14 | Los nombres con espacio final se resuelven correctamente                                                                                                                                                           |
| T-15 | El borrado explícito en el Host se propaga; la omisión en el dump no suprime contenido                                                                                                                             |
| T-16 | Los flags `hide_viewport`/`hide_select`/`hide_render` de objeto, la familia `RESTRICT_*` de colección, `exclude` y la visibilidad de huesos/bone collections en la DATA de armadura se preservan en los tres modos |
| T-17 | El linkado de colecciones en Modo B reproduce la pertenencia del Host                                                                                                                                              |
| T-18 | El orden de pesos por vértice se reproduce en los tres modos                                                                                                                                                       |
| T-19 | La rotación se transmite correctamente en quaternion, axis-angle y los modos euler, con preservación de vueltas múltiples                                                                                          |
| T-20 | El cambio de frame a mitad de edición mueve el overlay y restaura el frame anterior (SRS §9.03)                                                                                                                    |
| T-21 | La seleccionabilidad del Cliente no está gateada por propiedad/readonly (INV-3); `DynamicRightSelectTimer` no pisa el `hide_select` sincronizado                                                                   |
| T-22 | La poda solo elimina curvas creadas por el overlay y ausentes del dump, tras invalidación; nunca con overlay vivo; nunca curvas legítimas (X-01, D-44)                                                             |
| T-23 | La invención de curvas funciona para huesos con solo location keyeado y para huesos sin ningún keyframe (X-01, feature core)                                                                                       |

### 15.4 Validación pendiente de ejecución

LAN con segunda máquina: handshake, latencia, multi-armadura, los tres modos. Versiones de Blender anteriores a 5.2.2. Verificación numérica exhaustiva de los ~500 huesos del rig tras el apply (HI-12). Verificación de que una FCurve admite propiedades personalizadas para el refactor de la poda (X-01 opción iii).

**Verificaciones previas añadidas tras el mapeo de `f00d655`** (§17.J), bloqueantes para la implementación de DD-03 en la capa de armadura:

| ID    | Verificación                                                                                                                                                      | Por qué es previa                                                                                                |
| ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| VP-01 | Qué propiedades de visibilidad expone `bpy.types.Bone` frente a `bpy.types.EditBone` en 4.5 y 5.2, y cómo se lee/escribe la visibilidad de las bone collections   | El dump lee `Bone` y el load escribe `EditBone` en modo EDIT (§10.9). Determina dónde debe ir el bloque de carga |
| VP-02 | Que el bloque de visibilidad de armadura aplicado fuera de modo EDIT persiste tras el apply y no lo revierte la recarga de huesos                                 | El load entra y sale de modo EDIT; hay que confirmar que el dato sobrevive al cambio de modo                     |
| VP-03 | Disponibilidad y comportamiento de `bpy.utils.extension_path_user` en el mínimo declarado 4.5.0, incluido el camino de `ValueError`                               | DD-06 conserva esa base; hoy solo está probada en la plataforma de referencia 5.2.2                              |
| VP-04 | Que ampliar el `include_filter` de `BlObject.dump` y `BlCollection.dump` basta para que el valor llegue y se escriba, dado que sus loaders no tienen lista blanca | Es la premisa del trabajo mínimo descrito en §10.9; debe confirmarse con T-16                                    |

---

## 16. Criterios de aceptación y lista de verificación de publicación

### 16.1 Criterios de aceptación funcional

| ID    | Criterio                                                                                                                                                                       |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| AC-01 | Ninguna secuencia de edición, deselección o cambio de frame produce keyframes retenidos en el Cliente                                                                          |
| AC-02 | El Host no experimenta modificación de datablocks como consecuencia de la conexión o actividad de un Cliente                                                                   |
| AC-03 | No se crean duplicados de armadura, Action ni colección en Modo A; ningún puntero resuelve a un duplicado                                                                      |
| AC-04 | La escala de los huesos de deformación del Cliente coincide con la del Host dentro de tolerancia `1e-4`                                                                        |
| AC-05 | Las mallas del Cliente presentan las texturas del Host en Solid con color Texture y en Rendered                                                                                |
| AC-06 | Los flags de restricción/visibilidad de objeto, colección, view layer y DATA de armadura (incl. `hide_select` y visibilidad de huesos) coinciden con el Host en los tres modos |
| AC-07 | El Cliente no publica por ninguna vía                                                                                                                                          |
| AC-08 | La publicación definitiva se produce en el flujo de inserción manual de keyframe sin regresión                                                                                 |
| AC-09 | El transporte se reconstruye tras la finalización de su hilo; el fallo de bind no produce saturación de log                                                                    |
| AC-10 | La latencia de emisión del canal live es compatible con la tasa de 60 Hz                                                                                                       |
| AC-11 | El handshake acepta clientes autenticados y rechaza los no autenticados; la sesión sin contraseña se documenta                                                                 |
| AC-12 | La incorporación de escena respeta la opción elegida, no altera la jerarquía de colecciones y no ejecuta acciones destructivas en reconexión                                   |
| AC-13 | El arnés emite `RESULT: PASS` con la totalidad de las pruebas de §15.3                                                                                                         |
| AC-14 | La invención de curvas y la previsualización de huesos con keyframes parciales o sin ellos opera sin regresión; la poda no destruye el overlay vivo                            |

### 16.2 Lista de verificación de publicación

- [ ] Implementación de §10 completada en commits por área (§10.11)
- [ ] Revisión previa de los diffs de mitigación de 19 oct 2026 (§10.12)
- [ ] Arnés ampliado y en estado PASS
- [x] Manifiesto con identidad, versión, maintainer y exclusiones objetivo (verificado en `main`: `true_level_playblast` / `True Level: Playblast` / `1.0.0` / `Soilent`; exclusiones de vendorizados, docs y reportes)
- [ ] Wheel de `replication` reconstruido sin `net.py` y con `__version__` corregida
- [x] **Archivo `LICENSE` incorporado** en `main`: texto completo GPL-3.0, aviso propio `Copyright (C) 2026 Soilent` y preservación del aviso upstream `Multi-User / Swann Martinez (Slumber)`. Requisito de distribución pública satisfecho (D-21)
- [ ] Defectos conocidos documentados (§11)
- [ ] Validación en LAN con segunda máquina completada
- [ ] Manual de usuario emitido
- [ ] Operador de diagnóstico DD-24 operativo
- [ ] Detector de archivos dañados DD-27 operativo
- [x] `tagline` del manifiesto escrito por el mantenedor (`"Zero-render live playblast. Animate while it plays."`)
- [ ] Valor futuro de `website` (omitido intencionadamente; decisión diferida no bloqueante, §10.7/§13)
- [ ] SRS limpiado y versionado como línea base estable (recomendación, §17.H.4)

---

## 17. Anexos

### 17.A Referencias de código

| Localización                                 | Contenido                                                         |
| -------------------------------------------- | ----------------------------------------------------------------- |
| `handlers.py:170-174`                        | Gate de transformación                                            |
| `handlers.py` filtro de streaming            | Usa `is_streaming()`; sub-filtro por tipo durante streaming       |
| `handlers.py` `update_external_dependencies` | Publicación de `WindowsPath`/`PosixPath`/`Image` sin guard de rol |
| `handlers.py` rama de Scene sin UUID         | `add`+`commit`+`push` sin `can_publish`                           |
| `handlers.py` `push_animation_dependencies`  | Condicionada a `is_live`                                          |
| `handlers.py` `load_pre_handler`             | `session_quit()` al abrir archivo con sesión activa               |
| `handlers.py` `update_client_frame`          | Publica `frame_current` como metadato                             |
| `live_sync.py:97`                            | `TRANSFORM_IDLE_TIMEOUT = 1.0`                                    |
| `live_sync.py:121`                           | `is_host()` con acceso literal a WinMan                           |
| `live_sync.py:270`                           | Timeout de `accept`                                               |
| `live_sync.py:492`                           | `streamer.tick`                                                   |
| `live_sync.py:531-535`                       | Cálculo de `transform_active`                                     |
| `live_sync.py:544`                           | `_motion_stalled()`                                               |
| `live_sync.py:635`                           | `_update_selection`                                               |
| `live_sync.py:897/905`                       | `_find_point` / tolerancia                                        |
| `live_sync.py:1133/1148`                     | `_on_end` / `is_cancelled`                                        |
| `live_sync.py:1388/1395`                     | `cleanup` / `invalidate_overlays`                                 |
| `live_sync.py:1467/1504/1511/1603`           | `_sync_transport` / `_transport_alive` / `tick` / `shutdown`      |
| `live_sync.py:1629/1651/1661`                | Accessors de módulo                                               |
| `timers.py:441/447`                          | `ContinuousActionPushTimer` / `_last_hashes`                      |
| `timers.py` `ApplyTimer._applied_data`       | Guard de idempotencia por identidad de nodo                       |
| `timers.py` `DynamicRightSelectTimer`        | Fija `hide_select` desde `is_node_readonly`                       |
| `operators.py:220/451/496`                   | Escrituras de `is_host` vía contexto                              |
| `preferences.py:263/627/684`                 | `depsgraph_update_rate` / `pose_live_sync` / `is_host`            |
| `bl_object.py:748-755 / 889-912`             | `load_pose` / dump de pose                                        |
| `bl_armature.py`                             | Serializador de la DATA de armadura (visor de huesos, DD-03)      |
| `replication/server.py:30`, `ttl.py:28-29`   | Manipulación de `sys.path`                                        |

**Referencias añadidas tras el mapeo de `f00d655`** (§17.J):

| Localización                                     | Contenido                                                                                                       |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| `operators.py:197` / `:224-225`                  | Alta y baja dinámicas de `on_scene_update` en `depsgraph_update_post`                                           |
| `operators.py:272-293`                           | `setup_timer`: qué temporizadores se registran y el gate de versión de `AnnotationUpdates`                      |
| `operators.py:346` / `:412` / `:490` / `:1034`   | Llamadas a `utils.clean_scene()` (Join sin contraseña de admin, Host con `init_method='EMPTY'`, Init, LoadSave) |
| `operators.py:963-966`                           | Instanciación bajo demanda de `SessionBackupTimer`                                                              |
| `preferences.py:50-54`                           | `DEFAULT_CACHE_DIR = bpy.utils.extension_path_user(__package__, path='cache', create=True)`                     |
| `preferences.py:239-246`                         | `init_method` enum `{EMPTY, BLEND}`, default `BLEND`                                                            |
| `handlers.py:42-44`                              | `node.instance` asignado antes del test `node is None` (test inalcanzable)                                      |
| `bl_object.py:803` / `:823`                      | `include_filter` de `BlObject.dump`; `hide_render` presente, `hide_viewport`/`hide_select` ausentes             |
| `bl_object.py:692`                               | `BlObject.load`: `Loader()` sin `include_filter`, solo `exclude_filter=["modifiers","constraints"]`             |
| `bl_collection.py:120-123` / `:103-104`          | `include_filter` mínimo de colección; `Loader()` sin filtros                                                    |
| `bl_armature.py:166-176`                         | `include_filter` de la DATA de armadura, con `layers` legado y sin visibilidad                                  |
| `bl_armature.py:129` / `:131-150`                | Entrada en modo EDIT y bucle de carga sobre `edit_bones`                                                        |
| `bl_armature.py:96`                              | `# TODO: recursive parent collection loading` — jerarquía de un solo nivel                                      |
| `bl_action.py:476-483`                           | Poda incondicional de curvas ausentes del dump (X-01)                                                           |
| `bl_action.py:408-415` / `:434-442` / `:513-518` | Guard de payload vacío, reuso de slots, aviso de pérdida de keys                                                |
| `live_sync.py:957` / `:1414`                     | `created_curves` definido / destruido por `_edits.clear()`                                                      |
| `live_sync.py:929-930`                           | Fallback a `action.slots[0]` (KI-09)                                                                            |
| `live_sync.py:531-535`                           | Cálculo de `transform_active` (referencia corregida)                                                            |
| `bl_types/__init__.py:20-54`                     | `__all__`: el orden define el orden de ejecución; `bl_pointcloud` excluido por limitación de la API             |
| `__init__.py:49-51`                              | `ID.uuid` con `HIDDEN` + `SKIP_SAVE` (identidad efímera)                                                        |

### 17.B Inventario de la escena real de referencia

**Colecciones**: `Scene Collection`; `Lillia Low` (excluida de vista, oculta, sin render, 9 objetos); `Lillia` (10); `character1_cs` (sin render, ~130 mallas `cs_*`); `Horse` (4, incluida `_MCH-Shaft_spline`); `WGTS_JawsHorseCock_Armature` (oculta, sin render, 29 mallas `WGT-*`); `Luces` (6); `Escena` (12).

**Armaduras**: `rig` — Auto-Rig Pro, `arp_updated 3.61.18`, ~500 huesos, 384 drivers, 451 constraints. `JawsHorseCock_Armature` — 63 constraints, 35 drivers, cadena Spline IK de 7 huesos sobre curva Bezier de 5 puntos.

**Mallas medidas**: `Lilia` 9 760 vértices / 170 grupos / 213 017 pesos / 23 shape keys / Armature + Subdivisión. `Horse_Cock_Jaws` 3 669 vértices / 24 grupos / 27 923 pesos / Armature. `Horse_Cock_Jaws Low` 216 vértices con peso / 1 599 pesos.

**Nombres con espacio final**: `'Eyes '`.

**Shape keys de `Lilia`**: `Base`, `FixV`, `Anus`, `Push HARD` (max 1.5), `In`, `Push Out`, `Push`, `Annie Edit` (1.0), `Annie Press` (1.0), `Annie In` (1.0), `Annie InX` (0.999169), `Annie InX 269` (0.122984), `Annie Push Out` (−0.999992, min −2.4, max 5.0), `Annie Prolapse`, `Frame 321`, `Frame 323`, `Frame 392` (0.261518), `Frame 421` (max 2.0), `Frame 804`, `Respiro` (0.610766), `Face` (1.0), `O` (1.0), `Parpado` (0.980368, gobernada por driver).

**Driver de shape key**: `key_blocks["Parpado"].value`, expresión `(z+x)*-100`, variables TRANSFORMS `x` → `rig`/`c_eyelid_top.r`/`LOC_Z`/`LOCAL_SPACE`/`AUTO`, `z` → `rig`/`c_eyelid_top.l`/idem.

### 17.C Matriz de casos del diario de desarrollo

|               | Cliente Modo A        | Cliente Modo B        |
| ------------- | --------------------- | --------------------- |
| Escena ligera | Caso 1A · prioridad 0 | Caso 1B · prioridad 2 |
| Escena pesada | Caso 2A · prioridad 5 | Caso 2B · prioridad 2 |
| Escena real   | Caso 3A · prioridad 4 | Caso 3B · prioridad 4 |

**Caso 1A**: `Collection` → `.001`; `Esqueleto_acción` → `.001`; `Camera`, `Esqueleto`, `Cube`, `Hueso.005` conservan nombre.
**Caso 1B**: la totalidad de los nombres se conserva.
**Caso 2A prueba 1**: `Collection` → `.001`; `Esqueleto_acción` → `.003`; `Esqueleto_acción.001` → `.002`; `Esqueleto.001` y `Cube.001` conservan nombre.
**Caso 2A prueba 2**: `Sin_nombre` → `Sin_nombre.001` en el Host, con bloqueo de varios segundos; `Material.001` y `Sin_nombre.001` orphanados en el Cliente.
**Caso 2B**: el borrado total de objetos conserva los datablocks; el bloqueo persiste; la retirada y reinserción de la imagen incrementa el sufijo a `.001`, `.002`, `.003`. La colisión se debe al datablock `Material` creado por defecto al abrir el archivo.
**Caso 3A**: alteración de flags de restricción; deformación anómala de partes riggeadas; la pintura de pesos con valor 1 produce escalado desproporcionado; partes del personaje se desplazan siguiendo al hueso.
**Caso 3B**: la deformación anómala no se manifiesta; objetos y huesos ocultos se revelan; transmisión parcial de transformaciones; huesos que se mueven sin deformar la malla; texturas negras en Solid con color Texture, correctas en Rendered; la selección manual del nodo de textura corrige la presentación; el material preview permanece negro hasta crear un espacio nuevo.

**Commits de referencia**: `e9121508dbdb3e77158043c36424d7cdd8466e28`, `731f3a4f`, `0d71c8a5ad6a06756fd69a34b75718eefc67fd7e`.

### 17.D Verificaciones empíricas ejecutadas

| ID    | Procedimiento                                                                        | Resultado                                                                                         | Implicación                                                            |
| ----- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| VE-01 | Desactivación de `use_deform_preserve_volume` en el modificador Armature del Cliente | El síntoma persiste                                                                               | Descarta HI-02                                                         |
| VE-02 | Alt+G sobre el objeto afectado                                                       | Sin efecto                                                                                        | `matrix_basis` ya es identidad; el error reside en la cadena de parent |
| VE-03 | Eliminación de las shape keys del objeto afectado                                    | Sin efecto                                                                                        | Descarta HI-16 como causa                                              |
| VE-04 | Repetición de la prueba de transmisión de huesos                                     | Transmisión y deformación correctas en quaternion, Euler XYZ y Euler XZY, sin modificación alguna | Descarta HI-05; establece la intermitencia del fenómeno                |
| VE-05 | Observación del Host tras la conexión del Cliente                                    | La textura de `JawsHorseCock` se pierde y se presenta en magenta                                  | Confirma la violación de INV-1 en vivo                                 |
| VE-06 | Selección simultánea de huesos de varias armaduras                                   | Se produce conflicto de transmisión                                                               | Confirma A-03                                                          |
| VE-07 | Eliminación de los grupos de vértice afectados                                       | El síntoma desaparece; los huesos dejan de controlar la región                                    | El efecto se propaga íntegramente por la deformación de armadura       |
| VE-08 | Recuento de modificadores de `_MCH-Shaft_spline` en Modo B                           | 5 modificadores presentes, con `vertex_indices` vacío                                             | El defecto de payload existe en ambos modos                            |

### 17.E Historial de commits

| Fecha       | Área                                                              | Contenido                                                                                                                                                                                                                                                                                                                                                                                |
| ----------- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 16 oct 2026 | Inicial                                                           | Multi-User v0.8.2 con las dos capacidades del SRS: toggle global de commit/push armado con push inicial, y Live Sync de Modo Pose por TCP en puerto+1 con keyframe temporal reversible, lectura por `evaluated_get`, restauración en cancelación, hilos de trabajador restringidos a sockets y acceso a `bpy.data` en hilo principal vía `LiveSyncTimer`. Arnés en 52/52 sobre 4.5 y 5.1 |
| 18 oct 2026 | `operators.py`                                                    | Fix live pose streaming, armed pushing and rotation transport                                                                                                                                                                                                                                                                                                                            |
| 19 oct 2026 | `tests`                                                           | Keep modifier object references when reloading an Object                                                                                                                                                                                                                                                                                                                                 |
| 19 oct 2026 | `timers.py`                                                       | Extend the apply identity guard to forced parent and dependency reloads                                                                                                                                                                                                                                                                                                                  |
| 03 nov 2026 | `handlers`, `live_sync`, `replication`, `deepdiff`, `orderly_set` | Bug fixes                                                                                                                                                                                                                                                                                                                                                                                |
| 08 nov 2026 | `bl_types`, `Arquitectura.md`                                     | Add architecture documentation and design decisions                                                                                                                                                                                                                                                                                                                                      |
| 12 nov 2026 | Documentación y reportes                                          | Reportes de sesión, especificación, `Discrepancias.md`, `test.MD`                                                                                                                                                                                                                                                                                                                        |
| 16 nov 2026 | `a19c2ef` · documentación                                         | «Renombrado README.MD a SRS.md». **Verificado**: se ejecutó como copia; `README.MD` sigue en el árbol (§17.J.4)                                                                                                                                                                                                                                                                          |
| 16 nov 2026 | `660883e` · empaquetado                                           | GPL-3.0 con atribución upstream (D-21). `LICENSE` de 35 770 B: texto completo + `Copyright (C) 2026 Soilent` + `Based on "Multi-User" for Blender — Copyright (C) 2019-2026 Swann Martinez (Slumber)`                                                                                                                                                                                    |
| 16 nov 2026 | `f00d655` · empaquetado                                           | Identidad 1.0.0 del manifiesto (D-18, D-19, D-20, D-25): `true_level_playblast`, `True Level: Playblast`, `1.0.0`, `Soilent`, `tagline` propio, `website` omitido, exclusiones de build ampliadas                                                                                                                                                                                        |

**Los tres commits del 16 nov no tocan ningún `.py`.** El diff `00b90a2..f00d655` es exclusivamente `LICENSE` (+695), `SRS.md` (+736) y `blender_manifest.toml` (24 líneas cambiadas). Ver §17.J.1.

### 17.F Esquema de mensajes del canal live (verificado en código)

`start`:

```json
{
  "channel": "live_sync",
  "action": "start",
  "armature_uuid": "<uuid>",
  "bone_name": "<nombre exacto>",
  "frame": 573
}
```

`update`:

```json
{ "channel": "live_sync", "action": "update",
  "armature_uuid": "<uuid>", "bone_name": "<nombre exacto>", "frame": 573,
  "matrix_basis": [[…]], "location": [x,y,z],
  "rotation_mode": "<QUATERNION|AXIS_ANGLE|XYZ|…>",
  "rotation": [ … ], "scale": [x,y,z] }
```

`end`:

```json
{
  "channel": "live_sync",
  "action": "end",
  "armature_uuid": "<uuid>",
  "bone_name": "<nombre exacto>",
  "frame": 573,
  "is_cancelled": false
}
```

`start` no incluye canales de transformación; la primera pose llega en un `update` forzado inmediato. El canal de rotación depende de `rotation_mode`. `matrix_basis` se conserva por compatibilidad con Host anteriores. Los campos son planos, no un objeto `channels` anidado. `end` transporta `frame`.

### 17.G Nota de contraste final con el SRS

Se contrastó el SRS sección por sección contra el código y este documento:

- **§1, §2, §3, §7**: el flujo nominal de F1/F2 cumple el comportamiento observable esperado (overlay en bucle, sin pausa/mute/swap, reversible, un keyframe por hueso seleccionado, FAST, TCP directo, threading). Se registran los incumplimientos conductuales que R1 corrige (§0.3): A-02 (§3.5/§7.9), KI-01 (§5.4/§6.4, E-12), P-01 (§8.2), KI-02/A-01 (§4.4/§6.2), DEF-03 (§1.4), DEF-01/02 (§2.2). El incumplimiento literal de §2.2 punto 3 (UUIDs idénticos) es causa raíz de DEF-02; corrección D-39b.
- **§4**: cubierto por §4.12 (toggle global, `live_sync_nodes`) y §4.7 (handler, armado, `push_animation_dependencies`).
- **§5 y §6**: enmendados (E-06 canales crudos, E-08 multi-hueso, E-11 disparador por selección, E-12 orden end→push).
- **§8**: registrado como orientativo (DD-37 para §8.3), con hallazgos P-01 (§8.2) y P-04 (§8.1), y E-10 para la excepción de §8.6.
- **§9**: enmendado (E-04 rechazo de re-copia, E-09 opción b inválida) y mapeado completo en §2.7; caso 11 registrado como KI-08 con causa precisa.
- **§10 y §11**: referencia de estructura y dependencias; E-03 corrige la ubicación del motor.

**No se afirma un cumplimiento global del SRS.** Las dos funcionalidades están implementadas y operan en el flujo nominal; R1 corrige los incumplimientos registrados e incorpora capacidades no contempladas. Esta redacción sustituye la sobreafirmación «satisface el comportamiento observable» sin matices de versiones previas, que contradecía el catálogo de defectos (§0.3, §17.H.1).

### 17.H Contraste de las aprobaciones nocturnas contra las fundacionales

El mantenedor declaró haber aprobado precipitadamente el último 40% de la sesión por fatiga. Este anexo registra el contraste solicitado: primer 60% = verdad absoluta; último 40% contrastado.

#### 17.H.1 Incoherencias corregidas en v4.0

| #   | Tema                  | Verdad fundacional (60%)                                                  | Aprobación nocturna (40%)                                                                                                                                    | Estado en v3.0                                                         | Corrección v4.0                                                                                                                                                                                                                                                                                       |
| --- | --------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | DD-03 / `hide_select` | DD-03 incluye `hide_select` (aprobado msg #14)                            | #54 propone excluirlo; #56 el usuario rechaza; #56/#58 retractan y confirman inclusión                                                                       | **Excluido** (incoherente)                                             | **Incluido**, restituido; §6.4.2/§9.4/§10.9. Confirmado además por el mantenedor: es bug que todos los interruptores (`hide_viewport`, `hide_select`, familia `RESTRICT_*`, visibilidad de huesos en la DATA) no se conserven en el Cliente                                                           |
| 2   | X-01 poda ↔ D-44      | D-44 no borra lo no descrito; invención de curvas es core (msg #50, #52)  | #49 reconcilia (poda acotada por procedencia); #56 retracta como cambio sobre ruta funcionando                                                               | Header «X-01 resuelto» sin definir; poda excluida de D-44 sin precisar | X-01 definido y resuelto: poda post-invalidación acotada por procedencia, nunca con overlay vivo, nunca legítimas; opción (i) R1, (iii) FUT (§4.8/§9.3/§10.8)                                                                                                                                         |
| 3   | §0.3 cumplimiento     | msg #41: «el comportamiento actual ya cubre el SRS» (flujo nominal)       | E.1(a): «satisface el comportamiento observable» + un incumplimiento literal                                                                                 | Sobreafirmación global                                                 | Acotado al flujo nominal de F1/F2 + registro honesto de incumplimientos (§0.3/§17.G)                                                                                                                                                                                                                  |
| 4   | KI-01 / §5.4↔§6.4     | msg #7: aceptar flicker menor como known issue                            | KI-01 «contraviene el orden de §6.4»                                                                                                                         | Redacción invertida                                                    | E-12: R1 sigue §6.4; se aparta de la retención de §5.4 (trade-off aceptado); KI-01 corregido                                                                                                                                                                                                          |
| 5   | DD-35 / `slots[0]`    | Ruta declarada funcionando; regla de no-cambio sin fallo medido           | #53/#54 proponen eliminar `slots[0]`; #55 usuario rechaza; #56 retracta                                                                                      | «sin cambio»                                                           | Conservado en R1; hueco defensivo registrado como KI-09 (FUT) con recomendación preservada                                                                                                                                                                                                            |
| 6   | DD-06 purga y base    | msg #13: cache no compartido, subcarpeta propia (nada sobre la ruta base) | Sin pid, hash de contenido, escritura atómica, purga de lo propio; base movida a `__file__/cache` con la justificación «`extension_path_user` sin verificar» | Purga del subárbol propio; base `__file__/cache`                       | Purga acotada a ficheros escritos por la propia instancia (INV-3 ante peer homónimo). **Base corregida**: el código ya usa `extension_path_user` (`preferences.py:50-54`), la justificación era falsa y moverla metería el cache dentro de la extensión; se conserva el mecanismo embarcado (§17.J.3) |

#### 17.H.2 Elementos nocturnos validados (se conservan)

E.2(a) enmiendas normativas por sí mismas; E.3(a) manejo directo sin Action (E-09); E.4 multi-hueso (E-08); E.5 excepción de respaldos PKU (E-10); E.6(b) estado borrador; E-11 disparador por selección; E-02 retirada (RESOLVING existe); E-06 reformulada como adición; V-01 filtro `is_streaming()`; V-02 `push_animation_dependencies` condicionada a `is_live` (KI-08); causa exacta de DEF-03 (`update_external_dependencies`); DD-02 cuatro rutas + auditoría de `sanitize_deps_graph`; DD-14 reformulada (idempotencia por contenido complementaria al guard de nodo); inventario completo de handlers/temporizadores; `load_pre_handler` → `session_quit`; esquema de mensajes plano + `rotation_mode` + `end.frame`; P-04; `_VALUE_EPSILON` muerta; tolerancia por dominio; DD-37 (§8.3 orientativa); D-46 en §9; C-02/X-07/T-03 resueltos; D-40a/D-41/D-41a/D-41b/DD-34 (onboarding sin colección de sesión); DD-10/DD-11/D-15/D-38 (LAN R1, internet post-PKU, relay futuro, Tailscale descartado); PKU D-29…D-32.

#### 17.H.3 Referencias colgantes y colisiones resueltas

`X-01` definido y resuelto. `C-02`/`X-07`/`T-03`/`V-01`/`V-02` registrados como verificaciones de código resueltas (§17.I). `S-xx` reservado a seguridad; los huecos de cobertura del SRS usados como `S-01..S-06` en la noche se resolvieron como enmiendas (E-08…E-12, DD-37) y mapeo (§2.7). `D-29…D-32` y `D-34` incorporados a §9. Decisiones retiradas registradas (§9.10).

#### 17.H.4 Nota sobre la línea base del SRS

El `SRS.md` entregado (53 988 B) difiere del auditado (45 145 B): incluye apéndice de chat, roadmap y texto ajeno (War Thunder), título «Documento de Diseño Técnico y Arquitectura», y contradicciones internas (§1.4 vs §4.7; §5.4 vs §6.4; §5.10 vs §10). El TDD toma como normativo el comportamiento esperado (msg #64), no el pseudocódigo ni el apéndice. Recomendación: limpiar y versionar el SRS (SRS v2) como línea base estable; retirar el apéndice no normativo, corregir el título, resolver §5.4↔§6.4 (E-12) y §1.4↔§4.7 (DD-02), actualizar §5.10 (E-03) y §2.1 (E-01). El TDD no depende del apéndice.

### 17.I Verificaciones de código resueltas durante la auditoría

| ID     | Cuestión                                                         | Resolución                                                                                          | Efecto                                                                                        |
| ------ | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| C-02   | ¿`start` transporta canales iniciales?                           | No. `_begin` emite `start` sin canales; la primera pose llega en un `update` forzado                | §4.3 corregida; §17.F concuerda                                                               |
| X-07   | ¿La inserción usa `FAST` y se ejecuta `fcurve.update()` después? | Sí, `insert` con `options={"FAST"}`; ambas ramas llaman `fcurve.update()`                           | SRS-3.6 literal cumplido; propósito de §8.1 anulado → P-04 (rendimiento, sin fix a ciegas)    |
| T-03   | ¿`D-46` figura en §9?                                            | Registrada en §9.8                                                                                  | Trazabilidad cerrada                                                                          |
| V-01   | ¿Qué predicado usa el filtro de streaming?                       | `is_streaming()` (mera selección), con filtro por tipo (bloquea Armature/Object, permite Action)    | §4.7 correcta; divergencia con snippet SRS §4.7 registrada como enmienda, no defecto          |
| V-02   | ¿Existe la rama `node.state == UP`?                              | Sí, pero `push_animation_dependencies` está condicionada a `is_live`                                | SRS §9.11 aplicable → KI-08 con causa precisa                                                 |
| X-01   | Poda ↔ D-44                                                      | Resuelta: poda acotada por procedencia post-invalidación (§4.8, §10.8)                              | Cuestión bloqueante cerrada; documento puede pasar de borrador a aprobado tras implementación |
| DEF-03 | Ruta exacta de violación de INV-1                                | `update_external_dependencies()` al final de `on_scene_update`, fuera de todo guard, solo propiedad | DD-02 debe cubrirla explícitamente                                                            |

---

### 17.J Verificación del código vigente (`f00d655`)

Tras la consolidación v4.0 se mapeó el código actual de `main` (documento hermano `MAPA-ARQUITECTURA_v1.0.md`). Este anexo registra lo que ese mapeo aportó y cómo se integró en el TDD.

#### 17.J.1 Delta entre la revisión auditada y la vigente

| Revisión  | Fecha       | Commits | Contenido                      |
| --------- | ----------- | ------- | ------------------------------ |
| `00b90a2` | 12 nov 2026 | 28      | revisión auditada por este TDD |
| `f00d655` | 16 nov 2026 | 31      | `HEAD` de `main` al mapear     |

Diff verificado: `LICENSE` (+695), `SRS.md` (+736), `blender_manifest.toml` (24 líneas). **Ningún fichero `.py` cambió.** Consecuencias:

1. Todas las referencias de línea de este TDD siguen siendo válidas para el código vigente.
2. **Todos los defectos, huecos y limitaciones registrados siguen presentes en `main`.** Los tres commits posteriores son de empaquetado y documentación.
3. El TDD y el mapa de arquitectura describen el mismo fuente y no pueden contradecirse en lo relativo al código.

#### 17.J.2 Hallazgos nuevos incorporados

| #   | Hallazgo verificado                                                                                                                                                                                                                                                                                               | Evidencia                                                                                           | Integración                                                     |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| 1   | DD-03 no es un fallo de transmisión: **las propiedades nunca se serializan**. Los `include_filter` reales omiten `hide_viewport`/`hide_select` (objeto), todos los flags de colección, `exclude` de view layer y la visibilidad de huesos/bone collections; `layers` en la armadura es el mecanismo legado de 3.x | `bl_object.py:803/823`, `bl_collection.py:120-123`, `bl_armature.py:166-176`, `bl_scene.py:621-633` | §6.4.2 (tabla de filtros), §10.9 (punto exacto de intervención) |
| 2   | **Asimetría `Bone`/`EditBone`**: el dump de armadura lee `armature.bones` y el load escribe `edit_bones` en modo EDIT. Ampliar el filtro no basta para la visibilidad                                                                                                                                             | `bl_armature.py:129`, `:131-150`                                                                    | §10.9 (nota crítica), VP-01/VP-02 en §15.4                      |
| 3   | Los loaders de objeto y colección **no tienen lista blanca**, así que ampliar el `include_filter` del dump basta en esas dos capas                                                                                                                                                                                | `bl_object.py:692`, `bl_collection.py:103-104`                                                      | §10.9 (tabla de trabajo requerido), VP-04                       |
| 4   | La ruta `AnnotationUpdates` **solo se registra en Blender < 5.0.0**: no existe en la plataforma de referencia 5.2.2, sigue válida para el mínimo 4.5.0                                                                                                                                                            | `operators.py:292-293`                                                                              | §4.7 (rutas y timers), DD-02                                    |
| 5   | `on_scene_update` **no se registra en `handlers.register()`**: alta dinámica al establecerse la sesión                                                                                                                                                                                                            | `operators.py:197`, baja en `:224-225`                                                              | §4.7 (nota de registro)                                         |
| 6   | Recuento activo: **8 timers en 5.x, 9 en 4.x**; `SessionBackupTimer` es bajo demanda; `ContinuousActionPushTimer` nunca se instancia                                                                                                                                                                              | `operators.py:272-293`, `:963-966`                                                                  | §4.7                                                            |
| 7   | La poda de curvas es **incondicional** y `created_curves` se destruye antes de poder usarse                                                                                                                                                                                                                       | `bl_action.py:476-483`, `live_sync.py:957` / `:1414`                                                | §4.8 (cita literal), §10.8                                      |
| 8   | La jerarquía de colecciones solo se reconstruye **un nivel**, con `TODO` explícito                                                                                                                                                                                                                                | `bl_armature.py:96`                                                                                 | §6.4.3                                                          |
| 9   | En `sanitize_deps_graph` el test `node is None` es **inalcanzable** porque `node.instance` se asigna antes                                                                                                                                                                                                        | `handlers.py:42-44`                                                                                 | DD-02                                                           |
| 10  | El renombrado a `SRS.md` se ejecutó **como copia**; `README.MD` sigue presente                                                                                                                                                                                                                                    | árbol de `f00d655`                                                                                  | §4.1 nota 1, D-45, §17.J.4                                      |
| 11  | `clean_scene()` se invoca en cuatro puntos y `init_method` tiene default `BLEND`                                                                                                                                                                                                                                  | `operators.py:346/412/490/1034`, `preferences.py:239-246`                                           | §17.A                                                           |

#### 17.J.3 Corrección de DD-06 — base del cache

La propuesta nocturna fijaba la base del cache en `pathlib.Path(__file__).resolve().parent / "cache"`, justificándolo con que `extension_path_user` estaba «sin verificar en 4.5/5.2». **El código auditado ya usa esa API**:

```python
# preferences.py:49-57 — verificado en f00d655
try:
    DEFAULT_CACHE_DIR = bpy.utils.extension_path_user(
        __package__,
        path='cache',
        create=True
    )
except ValueError as e:
    logging.error("Failed to create default cache directory ({e})")
    DEFAULT_CACHE_DIR = ""
```

Por tanto la propuesta era incorrecta en tres sentidos: (a) la justificación factual es falsa —es el default embarcado y operativo en la plataforma de referencia—; (b) mover la base a `__file__/cache` introduciría el cache **dentro** del directorio de la extensión, lo contrario de lo deseable y contradictorio con el propio trabajo diferido que pedía sacarlo de ahí; (c) constituía un cambio de comportamiento sobre una ruta no declarada rota, sin evidencia medida de fallo, en contra de la regla fijada en la sesión (§0.7).

**Resolución**: DD-06 **conserva `extension_path_user`** y mantiene el resto de la decisión (subárbol por peer sin `pid`, nombre por hash de contenido, escritura atómica, purga acotada a escrituras propias). Se registra el efecto del renombrado D-18: al cambiar `__package__` cambia la raíz, el subárbol de `multi_user` queda abandonado, es benigno por ser cache regenerable y se reporta vía DD-27 sin borrado automático. La entrada de §13 que pedía mover el cache fuera de la extensión queda **satisfecha de origen**. Disponibilidad en 4.5.0 cubierta por VP-03.

#### 17.J.4 Estado del renombrado documental

El commit `a19c2ef` se titula «Renombrado README.MD a SRS.md» pero el árbol de `f00d655` contiene **ambos**: `README.MD` (45 145 B) y `SRS.md` (53 248 B). Fue una copia. D-45 sigue pendiente en su parte de ejecución: retirar `README.MD`. El `SRS.md` del repo (53 248 B) difiere ligeramente del entregado al mantenedor (53 988 B); ambos incorporan el apéndice no normativo criticado en §17.H.4.

#### 17.J.5 Correcciones de precisión

| Elemento                        | Antes                  | Verificado                                      |
| ------------------------------- | ---------------------- | ----------------------------------------------- |
| `live_sync.py` líneas           | 1 663                  | **1 662**                                       |
| `blender_manifest.toml`         | 802 B                  | **752 B** (identidad 1.0.0)                     |
| `SRS.md` en el árbol            | 45 145 B               | **53 248 B**; `README.MD` conserva los 45 145 B |
| `LICENSE` en el árbol           | ausente                | **35 770 B**, presente                          |
| Cálculo de `transform_active`   | `live_sync.py:533-537` | **`:531-535`**                                  |
| Fallback `slots[0]`             | sin línea              | **`live_sync.py:929-930`**                      |
| Destrucción de `created_curves` | sin línea              | **`live_sync.py:1414`**                         |

El resto de referencias de línea del TDD se comprobaron una por una contra el fuente y son correctas.

#### 17.J.6 Método

Clon de `main` en `f00d655`, `git fetch --deepen` para recuperar la historia, diff contra `00b90a2`, parsing AST de los módulos propios para el inventario de clases y métodos, y lectura dirigida de las rutas críticas. Volumen medido: **18 163 líneas propias en 54 ficheros `.py`**, más 8 951 vendorizadas y 1 627 del arnés. El detalle completo, el índice de símbolos y los comandos de reproducción están en `MAPA-ARQUITECTURA_v1.0.md` y su versión visual `mapa-arquitectura.html`.

---

_Fin del documento TDD-PLAYBLAST-001 v4.0._

---

Esta versión 4.0 consolida la totalidad de la auditoría, contrasta las aprobaciones nocturnas contra las fundacionales (§17.H), restituye DD-03 con `hide_select` y la visibilidad de huesos en la DATA de la armadura, define y resuelve X-01, acota la afirmación de cumplimiento del SRS, corrige KI-01/E-12, registra KI-09, refina la purga de DD-06 e incorpora D-29…D-32/D-34 a §9. Verificado en `main`: identidad y `tagline` propios escritos, `LICENSE` GPL-3.0 incorporado con avisos propio y upstream (D-21 satisfecho).

**Verificación posterior contra el código vigente `f00d655`** (§17.J): el código Python es idéntico al auditado `00b90a2`, por lo que todos los hallazgos siguen presentes. Se aportó la causa raíz de DD-03 a nivel de `include_filter` con el punto exacto de intervención y la asimetría `Bone`/`EditBone`; se acotó `AnnotationUpdates` a Blender < 5.0.0; se precisaron el registro dinámico de `on_scene_update` y el recuento de timers; se documentó la poda incondicional vigente; **se corrigió la base del cache de DD-06**, que vuelve al `extension_path_user` ya embarcado; y se registró que el renombrado a `SRS.md` se ejecutó como copia. La única decisión de contenido abierta es el valor futuro de `website`, omitido intencionadamente por el mantenedor y no bloqueante para R1.
