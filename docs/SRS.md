# SRS · True Level: Playblast

**Especificación de requisitos de software**

| Campo                        | Valor                                                                                                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Documento                    | SRS-PLAYBLAST-001                                                                                                                                                        |
| Versión                      | **2.1** — línea base estable                                                                                                                                             |
| Fecha                        | 20 de noviembre de 2026                                                                                                                                                  |
| Estado                       | **Revisado y cerrado por el mantenedor.** 21 decisiones aplicadas y aprobadas, más 5 aclaraciones de su revisión final (changelog 36–40). Sustituye íntegramente a la v1 |
| Producto                     | True Level: Playblast · versión 1.0.0                                                                                                                                    |
| Mantenedor                   | Soilent                                                                                                                                                                  |
| Release especificada         | **R1 = 1.0.0**                                                                                                                                                           |
| Documento de diseño asociado | `TDD-PLAYBLAST-001.md`                                                                                                                                                   |
| Registro de decisiones       | `Decisiones_SRS-v2.md` (DEC-01…DEC-21) · `Contradicciones_decisiones.md` (CT-01…CT-10)                                                                                   |
| Código de referencia         | `f00d655`                                                                                                                                                                |

---

### 0. NATURALEZA, ALCANCE Y CONVENCIONES

#### 0.1 Qué es normativo y qué es ilustrativo

Este documento especifica **comportamiento observable esperado**. Esa es su parte vinculante.

- **Normativo**: todo enunciado de comportamiento, toda restricción de §3 y §7, todo criterio de aceptación.
- **Ilustrativo**: el pseudocódigo, los nombres de función, los nombres de propiedad y las rutas de fichero. Se incluyen como ayuda de lectura y **pueden quedar desactualizados por un refactor sin que el requisito cambie**. Se han corregido los que se sabía erróneos (§13.3), pero no se garantiza su sincronía con el código.

Esta separación es deliberada y es la corrección estructural más importante de la v2: la v1 mezclaba ambas cosas, y de ahí que sobreviviera durante meses una referencia a ZeroMQ ya sustituida por TCP. **Si el pseudocódigo y el comportamiento descrito difieren, manda el comportamiento descrito.** El diseño técnico vinculante vive en el TDD, no aquí.

#### 0.2 Alcance de la Release 1.0.0

| Área                                                              | Disposición en R1                                                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Funcionalidad 2** — Live Sync de Modo Pose (§5)                 | **En alcance.** Es el núcleo de la release                                                                                                                                                                                                                                                    |
| **Funcionalidad 1** — Toggle global de commit/push constante (§4) | **Implementada, pero FUERA DE SOPORTE.** Se conserva intacta, sigue siendo utilizable y no se retira del producto. No es un requisito de R1: no se garantiza, no se mantiene y no se corrigen sus defectos. Su documentación se conserva completa por si se recupera en una versión posterior |
| **Porcelain** (commit/push/apply del repositorio)                 | **En alcance y fundamental.** No debe confundirse con F1: F1 es el armado continuo por datablock; porcelain es el transporte de datos confirmados y es imprescindible                                                                                                                         |
| **Phantom Keyframe Update (PKU)**                                 | **Posterior a R1.** En R1 no sobrevive ningún keyframe temporal. Las cláusulas condicionales sobre `Phantom Keyframe Mode` que aparecen en §3.5, §6.2 y §7.9 están **reservadas para esa versión** y no alteran el comportamiento de R1, que es incondicional                                 |
| **Auto-keying**                                                   | **Fuera de alcance de R1**, exclusión deliberada. Se corrige como bugfix dentro de la PKU                                                                                                                                                                                                     |
| **Red**                                                           | **LAN y misma máquina únicamente**, con un Host y N Clientes. Internet descartado para R1; las VPN de terceros no se usarán tampoco después (§2.4)                                                                                                                                            |
| **Multi-hueso**                                                   | **En alcance**, caso normal de trabajo (§5.3, §9.4)                                                                                                                                                                                                                                           |
| **Multi-armadura simultánea**                                     | **En alcance en R1** (§5.11). Promovida desde trabajo diferido por decisión del mantenedor                                                                                                                                                                                                    |
| **Rigs complejos**                                                | El sistema debe admitir rigs de mayor complejidad por **ampliación declarativa**, no por reescritura                                                                                                                                                                                          |

#### 0.3 Documentos relacionados

| Documento                       | Contenido                                                                             |
| ------------------------------- | ------------------------------------------------------------------------------------- |
| `TDD-PLAYBLAST-001.md`          | Diseño técnico, invariantes, defectos, decisiones D-xx/DD-xx, criterios de aceptación |
| `ROADMAP.md`                    | Fases posteriores a R1 y rationale de naming                                          |
| `MAPA-ARQUITECTURA_v1.0.md`     | Mapa del código real, con rutas y líneas                                              |
| `Decisiones_SRS-v2.md`          | Las 21 decisiones que produjeron esta v2                                              |
| `Contradicciones_decisiones.md` | Revisión de contradicciones del mantenedor y sus resoluciones                         |

#### 0.4 Convenciones

«Host» = instancia editora. «Cliente» = instancia visora. «Overlay» o «keyframe temporal» = el keyframe que el Cliente inserta o sobreescribe en el frame donde el Host trabaja. «R1» = la release 1.0.0. **F1** y **F2** = Funcionalidad 1 y Funcionalidad 2.

### 1. INFORMACIÓN GENERAL DEL PROYECTO

#### 1.1 Identidad del Producto

Datos del manifiesto (`blender_manifest.toml`), verificados en `main`:

| Campo                            | Valor                                                 |
| -------------------------------- | ----------------------------------------------------- |
| Nombre                           | **True Level: Playblast**                             |
| Identificador de extensión       | `true_level_playblast`                                |
| Versión                          | 1.0.0                                                 |
| Tagline                          | «Zero-render live playblast. Animate while it plays.» |
| Mantenedor                       | Soilent                                               |
| Repositorio                      | https://github.com/Devangel-gonzalez/blender-sync     |
| Licencia                         | GPL-3.0-or-later                                      |
| Blender mínimo declarado         | 4.5.0                                                 |
| Plataforma de referencia probada | Blender 5.2.2                                         |
| Permiso requerido                | Red (Network)                                         |

El campo `website` del manifiesto se deja **vacío de forma deliberada**; no es un olvido.

**Arquitectura.** Cliente/Servidor con replicación orientada a datos. Usa TCP (sockets estándar de Python). La sincronización se apoya en un repositorio local con operaciones tipo Git (`commit`, `push`, `apply`, `lock`, `unlock`, `rm`) a través de una capa de abstracción llamada **porcelain**. Sobre esa base, el producto añade un **canal TCP directo separado** para la pose en curso (§5.2).

**Naturaleza del producto.** Es un derivado de Multi-User orientado a un único flujo de trabajo: revisión de animación sin render, con un Host que anima y un Cliente que reproduce en bucle. La atribución del addon base se conserva en §1.5 y en el `LICENSE`.

#### 1.2 Objetivo de la Modificación

Modificar el addon Multi-User para habilitar un flujo de trabajo de animación en tiempo real donde:

- El Host actúa como editor/animador principal.
- El Cliente actúa como un visor/reviewer que reproduce la animación en bucle.
- El Host puede editar poses interactivamente (R, G, S en Modo Pose) y el Cliente ve esos cambios en vivo, integrados dentro de su reproducción en bucle como un keyframe temporal en el frame específico de edición.
- El timeline del Cliente nunca se detiene, pausa ni altera.
- **Host y Cliente pueden ser la misma persona**: dos instancias de Blender en el mismo equipo, típicamente en dos monitores. «Cliente» nombra un rol de la sesión, no a una segunda persona (§1.3).

#### 1.3 Motivación

El valor principal es la **autorrevisión en vivo del propio animador**. Una segunda instancia de Blender —en el mismo equipo, por `127.0.0.1`, y normalmente en un segundo monitor— reproduce la animación en bucle mientras el animador edita en la primera. Desaparece el ciclo detener la reproducción → editar → volver a reproducir → hacer scrub → editar: la edición se ve en contexto, con sus transiciones de entrada y salida, **sin interrumpir el flujo de trabajo**. No hace falta otra persona, ni otro equipo, ni transferir el `.blend`.

Que **otras personas** puedan conectarse como revisores es una característica adicional, no el propósito del producto. El mismo mecanismo sirve para que un segundo animador, un director o un cliente observe la animación en bucle desde otro equipo de la LAN y vea la edición del Host en el frame trabajado, evaluando las transiciones de entrada y salida dentro del flujo completo. Es un caso de uso real y soportado —**un Host y N Clientes**, §2.6—, pero **secundario**: el addon se justifica con una sola persona y un solo equipo.

#### 1.4 Sistema de Permisos: herencia y añadidura

**Lo que se hereda.** El sistema de permisos procede del addon base Multi-User y funciona mediante propiedad y bloqueo de data blocks. Cada data block tiene un propietario (`owner`); la biblioteca de replicación acepta la modificación por parte del propietario o de cualquiera si el nodo es común (`RP_COMMON`). No existe un modo global de «solo lectura» para clientes. **Ese sistema se conserva.**

**Lo que este producto añade encima.** La propiedad por sí sola **no basta** para cumplir el requisito de que el Cliente nunca publica: la biblioteca acepta `RP_COMMON` para cualquier peer, y varios tipos pasan la comprobación de común para cualquiera. Por eso el producto añade una **puerta por rol**: solo el Host puede hacer `commit`/`push`. Nuestra implementación de esa puerta es `can_publish()` — el mecanismo de permisos es heredado; `can_publish()` es código propio.

> **Requisito normativo (INV-1).** El Cliente no puede ni debe publicar nada por ninguna vía. El Host es el único que inyecta datos. Esta regla tiene prioridad absoluta sobre cualquier otra consideración de este documento.

**La puerta se aplica en todos los puntos de entrada, no solo en el automático.** Hoy `can_publish()` gobierna el handler del depsgraph, pero el botón manual de commit llama a `porcelain.push` directamente y no pasa por ella. Eso se corrige en R1 (**D-49**, TDD §17.L.6) reutilizando el mismo predicado con los mismos argumentos. La prohibición ya estaba escrita en §7.12; lo que faltaba era implementarla en esa ruta. La misma guarda se extiende a los botones ✖ (dejar de seguir un nodo) y 🔓 (cambiar el propietario), que se saltan la puerta por el mismo motivo (**D-49b**, TDD §17.L.8). El de mayor gravedad es ✖: con un solo clic envía un borrado del nodo **y de todas sus dependencias** que el servidor reemite a todos los peers, de modo que ese nodo deja de sincronizarse en silencio también para el Host. **Ninguno de esos dos botones pertenece a F1.**

La v1 afirmaba que la modificación «no altera este sistema de permisos». Era inexacto: **sí añade una regla**, la puerta por rol, y es imprescindible. Corregido por DEC-06.

#### 1.5 Atribución del Addon Base

Este producto deriva de **Multi-User** para Blender. Se conserva el crédito íntegro:

- Nombre: Multi-User
- Autor original: Swann Martinez (Slumber)
- Repositorio: https://gitlab.com/slumber/multi-user
- Extensión Blender: https://extensions.blender.org/add-ons/multi-user/
- Documentación: https://slumber.gitlab.io/multi-user/index.html · https://multi-user.readthedocs.io/
- Copyright upstream: (C) 2019-2026 Swann Martinez (Slumber)
- Copyright del producto: (C) 2026 Soilent

Ambos avisos figuran en el `LICENSE` del repositorio (GPL-3.0). El «estado del addon» que la v1 citaba aquí —«Under development, use it at your own risks. Currently tested on Windows platform»— es el estado del **upstream**, no el de este producto, y por eso se ha retirado de la descripción del producto.

De Multi-User se heredan, sin cambio de titularidad conceptual: el repositorio y porcelain, el protocolo de replicación, el sistema de propiedad y bloqueo, la presencia de usuarios y el panel de sincronización. Son código de terceros que este producto reutiliza y extiende.

### 2. ESCENARIO DE TRABAJO Y CONFIGURACIÓN

#### 2.1 Configuración de Archivos

| Rol             | Archivo                                                                                                             | Comportamiento del Timeline                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Host (Editor)   | Abre el .blend original                                                                                             | Libre. Puede pausar, hacer scrub, ir a cualquier frame, editar donde quiera. No está en bucle.                              |
| Cliente (Visor) | Abre el archivo que prefiera: copia del .blend, escena propia o archivo recién abierto (§2.3). La copia es opcional | Se usará como reproductor, reproduciendo en bucle constantemente. No se pausa, no se detiene, no es controlado por el Host. |

- Son 2 instancias separadas de Blender, cada una con su propio archivo y su propio timeline independiente.
- El Host tiene control libre de su timeline.
- El Cliente SÍ está en bucle (controlado por el usuario dentro de Blender, el addon no lo controla). El Cliente es el que reproduce la animación continuamente.
- El addon sincroniza la memoria (`bpy.data`), no el archivo en disco. Cuando el Cliente se conecta, el addon descarga los data blocks del servidor y los construye en su escena local mediante porcelain.construct() y porcelain.apply().
- No se debe abrir el mismo archivo físico desde una carpeta compartida de red. Blender bloquea archivos .blend abiertos y puede causar corrupción.
- La copia del .blend es **una** de las tres situaciones de partida admitidas (§2.3, eje 1), no un requisito. Conectarse desde una instancia recién abierta está soportado: el addon descarga y construye la escena del Host. Ningún flujo exige transferirse el archivo.
- **El cache no se comparte entre Host y Cliente**, tampoco en LAN. Cada peer escribe en su propia subcarpeta. Un visor nunca debe leer ni escribir el cache del Host.

#### 2.2 Flujo de Conexión

1. Host: Abre su .blend original. Inicia el servidor Multi-User desde el panel del addon.
2. Cliente: Abre su archivo de partida —copia del .blend, escena propia o archivo recién abierto; **la copia no es obligatoria** (§2.3)—. En el panel Multi-User, introduce la IP del Host y se conecta; en el mismo equipo, la dirección local `127.0.0.1`.
3. Al conectarse: El addon en el Cliente descarga todos los data blocks del servidor, construye la escena local, y asigna **los mismos UUIDs que en el Host** para mantener la coherencia de rastreo.

> **Requisito firme, no aspiracional.** La identidad de los datablocks debe ser la misma en ambos peers y debe **sobrevivir a guardar y reabrir el .blend**. Hoy no se cumple: el UUID se registra sin persistencia, de modo que al reabrir el archivo la correlación cae en emparejado por nombre. Ese es el origen de los duplicados `.001` y de la armadura que pierde el enlace con su Action. **Se corrige el código; no se relaja el requisito** (DEC-07). Relajarlo a «identidad por nombre o UUID» legitimaría el defecto y dejaría la jerarquía de colecciones sin criterio de aceptación.

#### 2.3 Modos de Incorporación de la Escena del Cliente

Hay **dos ejes distintos** que no deben confundirse: desde qué archivo se conecta el Cliente, y qué elige hacer con su escena.

**Eje 1 — situación de partida.** No es una opción del addon; es el archivo que el usuario abrió:

| Punto de partida                                               | Qué encuentra el sistema                                                       |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Copia del .blend del Host**                                  | Los datablocks ya existen localmente; se adoptan por identidad y se actualizan |
| **Escena poblada** (un paisaje, otros personajes con sus rigs) | Contenido local que el usuario puede querer conservar                          |
| **Archivo recién abierto**                                     | El cubo, la luz y la cámara por defecto                                        |

**Eje 2 — el diálogo de incorporación.** Es la opción real del usuario, y es la misma en los tres casos anteriores:

| Opción                       | Qué hace                                                                                                                                                                   |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Combinar** _(por defecto)_ | Conserva **todo** lo local y añade el contenido del Host con su jerarquía de colecciones                                                                                   |
| **Reemplazar**               | Purga la escena **activa** y los datos huérfanos, **con confirmación explícita**                                                                                           |
| **Nueva escena**             | Crea una escena dedicada y deja la local intacta. La adopción **excluye los objetos ya usados por las escenas locales**, para no robarle un objeto a la escena del usuario |

Junto a las tres opciones, una casilla **«Recordar mi elección»**.

**Cuándo aparece el diálogo, y cuándo no.**

- Aparece **solo en el primer connect del par (archivo, sesión)**.
- **Reconexión automática** (caída de red, misma sesión e instancia, escena ya poblada): **no** se reejecuta incorporación, adopción ni purga; solo resincronización.
- **Reconexión manual** (el usuario salió de la sesión o cerró el archivo): se detecta el contenido ya adoptado por UUID y se **reanuda**, no se re-adopta.
- **La opción recordada jamás dispara una acción destructiva.** La purga requiere confirmación explícita en todo caso.

**Restricciones normativas de la incorporación.**

1. **No se crea una «colección de sesión».** El contenido replicado no se agrupa en un contenedor inventado por el addon.
2. **No se altera la jerarquía de colecciones.** La estructura de colecciones del Host se reproduce tal cual, con sus anidamientos.
3. Cuando el usuario elige **«Reemplazar»**, el alcance es **exclusivamente la escena activa del Cliente**. Si el Cliente tiene varias escenas, las demás no se tocan. «Una sola escena» no debe leerse literalmente.
4. **El sistema no purga contenido por su cuenta, por ningún motivo.** Ninguna ruta de conexión puede vaciar la escena del Cliente sin que el usuario lo haya elegido y confirmado. Esto incluye rutas heredadas del addon base que hoy purgan según una condición ajena a esta elección (§13.4).

**Política de identidad y adopción al conectar.** Aplica a los tres modos y es lo que permite conectarse desde un archivo propio sin necesidad de transferirse el `.blend`:

5. **La identidad entre peers es el UUID, nunca el nombre.** El nombre no decide identidad en régimen normal.
6. **Pase de adopción único al conectar**, con **informe visible para el usuario** del tipo «X adoptados, Y creados, Z renombrados». Empareja por `(tipo, nombre)` **solo sobre datablocks que aún no han sido reclamados** por otro UUID, estampa el UUID del Host y **actualiza el datablock existente en lugar de recrearlo**. El Cliente no estampa UUIDs propios antes de la adopción. Cubre **Objetos, sus datos, Colecciones y Actions** por la misma ruta. Es una heurística **única y visible de onboarding**, no una regla de identidad permanente.
7. **Guard estructural previo al emparejamiento**, para no fusionar homónimos no relacionados: se comparan recuentos ya disponibles (nº de vértices en mallas, nombres de huesos en armaduras) y una huella del contenido. Un «Cube» local no se adopta como el «Cube» del Host si son mallas distintas. Se ejecuta una vez por datablock al conectar y no añade carga al canal de alta frecuencia.
8. **Resolución por nombre exacto, sin normalizar espacios.** Existen nombres legítimos con espacio final (`'Eyes '`); normalizarlos rompería la resolución.
9. **Tras la adopción, el estampado de UUID es obligatorio** y la identidad pasa a ser solo UUID. Las dependencias de fichero se identifican por **hash de contenido**, porque las rutas no admiten estampado.
10. **Aplicación en orden topológico** (padres antes que hijos) y **prohibición de crear datablocks orphanados**: todo lo que se crea queda vinculado y con su animación, o no se crea.
11. **No se prescribe el flujo «hostear, guardar, copiar el archivo después».** La transferencia por un canal externo es fricción y queda rechazada como recomendación de uso.

#### 2.4 Alcance de Red de R1

- **R1 funciona en LAN y entre instancias de la misma máquina.** En ambos casos la topología es **un Host y N Clientes** (§2.6): en un solo equipo pueden convivir el Host y **uno o varios** Clientes, no solo dos instancias. Nada más.
- **Internet queda descartado para R1.** No es un requisito de esta release.
- **Las VPN de terceros no se usarán tampoco después.** Tailscale y ZeroTier quedan descartados **como requisito del producto**, no solo para R1: obligan a crear una cuenta y a instalar y configurar un cliente aparte, que es exactamente la fricción que la filosofía de abajo prohíbe (TDD **D-38** y **DD-10**). La dirección ya registrada para esa actualización es un **relay propio sin cuentas**, condicionado a que haya infraestructura (DD-11, D-15, `ROADMAP.md` §4.1); nombrar Tailscale en este documento no lo adopta como solución.
- El transporte por internet se retoma en una actualización **posterior a la PKU** (§0.2 y `ROADMAP.md`).
- **Filosofía de producto, normativa**: el addon es para animadores 3D y **no debe añadir fricción**. Sin cuentas de usuario, sin configurar el router, sin pasar el .blend por Discord o Drive. Hostear → conectarse → editar.
- **Consecuencia operativa que hay que declarar**: al usar un canal separado para la pose (§5.2), la operación remota requiere **dos puertos abiertos**, el base y el base+1, no uno solo.

#### 2.5 Opciones de sincronización que el usuario controla

Heredadas del addon base y operativas en R1. Se documentan porque afectan a lo que el revisor ve:

| Opción                          | Default                                                              | Efecto                                                                                                                                                                                                           |
| ------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Synchronize active camera**   | **Desactivado** _(cambiado en R1; el default heredado era activado)_ | Replicaba el puntero de cámara activa del Host. Con el default de R1 el revisor encuadra libremente. **No fuerza la vista del viewport**: solo cambia qué cámara es la activa                                    |
| **Synchronize render settings** | Desactivado                                                          | Sincroniza ajustes de Eevee y Cycles                                                                                                                                                                             |
| **Edit mode updates**           | Desactivado                                                          | Actualiza objetos en Modo Edición. El propio addon advierte del impacto en rendimiento                                                                                                                           |
| **Pose Live Sync**              | **Activado**                                                         | El canal de pose de F2. Es el núcleo de R1 (§5)                                                                                                                                                                  |
| **Live Sync** (F1)              | Desactivado                                                          | Armado continuo por datablock. **Fuera de soporte** (§4)                                                                                                                                                         |
| **Apply delay**                 | **0 s** _(cambiado en R1; el default heredado era 1 s)_              | Periodo del bucle que **aplica los datos entrantes** en el receptor. Con el default heredado, una instalación limpia actualizaba una vez por segundo. El valor de R1 es el que el mantenedor verificó como bueno |

**El rango de fotogramas llega al conectarse; después es del Cliente.** `frame_start`, `frame_end` y `frame_step` viajan en el volcado de escena (`bl_scene.py:509-511`), así que la sincronización inicial deja al Cliente con el rango del Host. **A partir de ahí el Cliente puede cambiarlo con Blender vanilla y su cambio se conserva.** No es una prohibición: es un valor inicial heredado.

**Por qué no se le reescribe encima.** La rama de escena solo publica cuando el depsgraph informa de un cambio en la escena (`handlers.py:319-323`), y `porcelain.commit` únicamente produce commit si el volcado nuevo difiere del anterior (`replication/porcelain.py:182-203`). Como `frame_current` **no** está en el volcado, **scrubear y animar no reenvían la escena**. El rango del Cliente solo vuelve a sobrescribirse si el Host modifica un campo que sí viaja: su propio rango, el mundo, o la cámara activa teniendo ese toggle activado. Con `Synchronize active camera` desactivado en R1 (§2.5) desaparece uno de esos disparadores.

Lo que **no** se replica en ningún caso es la posición del playhead: el Cliente controla su propio timeline (§3.1).

#### 2.6 Opciones heredadas con efectos que el usuario debe conocer

El addon base trae controles que no fueron diseñados para este producto. Se documentan porque **dos de ellos pueden romper la promesa central** o destruir trabajo:

| Control                                              | Qué hace realmente                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Disposición en R1                                                                                                                                                                                                              |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Contraseña de administrador**                      | Los campos de contraseña viven **solo en el panel `Hosting`** (`ui.py:293-302`, propiedades `host_*`). **No existe campo de contraseña de administrador en el flujo de conexión.** Su única función es autorizar a **expulsar usuarios** (`porcelain.kick` comprueba `is_admin()`); no concede derechos de replicación. **Efecto colateral grave y ajeno a su propósito**: la purga del Cliente se decide con un campo del _preset_ de conexión que ninguna UI expone y que vale `False` por defecto ⇒ **al conectar por LAN la escena se purga siempre**; solo se libra el preset `localhost` embarcado, que trae `use_admin_password = True` | **Se retiran `operators.py:345-346`.** La purga queda gobernada únicamente por el diálogo confirmado (§2.3 punto 4). **Sin cambios de UI**: la separación Host/Conectar ya es correcta. Análisis completo en TDD §17.K.9 DD-38 |
| **«snap to user»** (`wm.session_view_snap`)          | Operador modal a 10 Hz que **sobrescribe la matriz de vista del viewport** con la del usuario elegido. Solo se detiene con `Esc`, clic derecho o desactivándolo a mano. **Por eso, si el revisor orbita, la vista vuelve al Host**: es fricción real                                                                                                                                                                                                                                                                                                                                                                                           | **Debe desactivarse automáticamente en cuanto el revisor navega** (orbitar, encuadrar o hacer zoom). Requisito de R1                                                                                                           |
| **«snap to user time»** (`wm.session_timeline_snap`) | Operador modal a 20 Hz que ejecuta `context.scene.frame_current = <frame del otro usuario>`. Mientras está activo, el Cliente **sigue el tiempo del Host** en lugar de reproducir su propio bucle                                                                                                                                                                                                                                                                                                                                                                                                                                              | **Se conserva tal cual.** Es una elección explícita y reversible del usuario, no el addon alterando el timeline por su cuenta (§3.1). Al desactivarlo, el Cliente recupera su bucle independiente                              |

**Nota sobre «snap to user».** Este control es la causa real de que la vista del revisor vuelva al Host al orbitar. **No es `Synchronize active camera`**, que solo replica el puntero de cámara activa. Son dos mecanismos distintos y conviene no confundirlos al diagnosticar.

**Varios revisores.** R1 admite **un Host y N Clientes simultáneos**. Es capacidad heredada del addon base y **sigue funcionando**; se documenta y se protege, no se rediseña.

### 3. RESTRICCIONES FUNDAMENTALES

#### 3.1 NO PAUSAR EL CLIENTE

- Bajo ninguna circunstancia el sistema debe pausar, detener, ni alterar el timeline del Cliente.
- El Cliente reproduce su bucle de animación de forma continua e ininterrumpida.
- Toda la lógica de sincronización debe funcionar alrededor del bucle del Cliente, no en contra de él.
- No se debe ejecutar bpy.context.screen.is_animation_playing = False en el Cliente como parte del flujo de Live Sync.
- No se debe ejecutar bpy.context.scene.frame_set() en el Cliente para forzar un frame específico como parte del flujo de Live Sync.
- **La prohibición cubre cualquier mecanismo equivalente**, no solo la llamada literal: asignar `scene.frame_current` desde un temporizador u operador modal tiene el mismo efecto que `frame_set()`.
- **Alcance exacto de la prohibición: el comportamiento automático del addon.** Lo que nunca puede ocurrir es que el sistema altere el timeline del Cliente **por su cuenta**, como parte del flujo de sincronización. **No** se prohíbe ofrecer al usuario un control **explícito, voluntario y reversible** para seguir el tiempo del Host: eso es el usuario manejando su propio timeline, no el addon imponiéndoselo. El control «snap to user time» (§2.6) es de esa clase y **se conserva**.

#### 3.2 NO SILENCIAR EL HUESO

- El hueso (o los huesos) que el Host está transformando NO debe ser silenciado. No se debe usar fcurve.mute = True sobre las curvas F del hueso editado.
- El hueso que el Host está editando NO debe ser congelado ni desconectado de la animación.
- El hueso debe continuar animándose normalmente dentro del bucle del Cliente.
- La edición del Host se manifiesta como un keyframe temporal insertado o sobrepuesto en el frame específico donde el Host está trabajando.

#### 3.3 NO USAR ACTION SWAPPING

- NO se debe crear una Acción vacía temporal ni intercambiar Acciones (animation_data.action).
- NO se debe reemplazar la Acción original del Cliente por otra Acción.
- NO se debe desconectar la Acción original del Armature durante la edición interactiva.
- Los keyframes temporales se insertan o sobreescriben dentro de la Acción existente del Cliente.

#### 3.4 NO CREAR KEYFRAMES MASIVOS

- NO se deben crear 60 keyframes por segundo ni keyframes en cada frame del bucle.
- Solo se manipula UN keyframe temporal por hueso editado, ubicado en el frame específico del Host.
- Ese keyframe se actualiza (no se recrea) cada vez que el Host mueve el hueso durante la misma transformación interactiva.

#### 3.5 MODIFICACIÓN TEMPORAL DE LA ANIMACIÓN DEL CLIENTE

- La animación del Cliente es modificada temporalmente durante la edición interactiva del Host. Esto es necesario, esperado y aceptable.
- Se insertan o sobreescriben keyframes en el frame específico donde el Host está trabajando.
- Los valores originales de los keyframes afectados se guardan en memoria antes de cualquier sobreescritura.
- Si el Host cancela la edición, los valores originales se restauran completamente, devolviendo la animación a su estado previo a la edición interactiva.
- Si el Host confirma la edición, el commit/push real del repositorio actualiza los datos de la animación. Los datos temporales son reemplazados por los datos confirmados.
- Ninguna modificación temporal debe sobrevivir a la cancelación de la edición ni convertirse en permanente fuera del flujo de commit/push del repositorio. **En R1 esto es incondicional: no sobrevive ningún keyframe temporal.**
- _[Reservado para la PKU, no aplica a R1]_: el modo `Phantom Keyframe Mode` podrá autorizar la persistencia deliberada de keyframes fantasma. Mientras esa versión no exista, la cláusula anterior no tiene excepciones.

#### 3.6 RENDIMIENTO

- NO usar bone.keyframe_insert() estándar de Blender. Es demasiado lento para operaciones en tiempo real porque recalcula tangentes, actualiza el depsgraph y dispara handlers internos.
- SÍ usar fcurve.keyframe_points.insert(frame, value, options={'FAST'}) para inserción rápida de keyframes.
- El stream de datos de transformación en vivo debe usar un canal TCP directo, evitando pasar por porcelain.commit() y porcelain.push() que son operaciones pesadas de repositorio y generan historial de commits.

#### 3.7 THREADING

- NO usar hilos de Python (threading) para leer/escribir datos de `bpy.data`. Blender no es thread-safe para datos de escena y el acceso desde hilos secundarios causa crashes (Segmentation Fault).
- SÍ usar bpy.app.timers o handlers nativos de Blender (depsgraph_update_post, frame_change_post, etc.) para tareas periódicas o reactivas.

#### 3.8 EL CLIENTE PUEDE SELECCIONAR (restricción afirmativa)

**El Cliente sí puede seleccionar armaduras y huesos, incluidos los que el Host está editando**, para observar en vivo cómo se generan los keyframes. Es un derecho del usuario del visor, no un efecto colateral tolerado.

- La protección del Cliente es sobre sus **datos**, no sobre su **interacción**. INV-3 protege la integridad de la animación; no restringe la selección.
- **La seleccionabilidad del Cliente no se gobierna por propiedad ni por solo-lectura.** Un datablock que el Cliente no puede publicar sigue siendo seleccionable.
- Por tanto, **queda prohibido** cualquier mecanismo que fuerce `hide_select` sobre los objetos del Cliente en función de quién los posee o de si son de solo lectura. Un temporizador que reescriba `obj.hide_select` periódicamente según la propiedad viola este requisito.
- El Cliente puede además activar y desactivar físicas de terceros (por ejemplo Wiggle Bones) para evaluar su resultado. **El resultado de esa evaluación nunca se commitea ni se pushea.**

**Físicas y keyframe temporal: no hay conflicto, por diseño.** Las físicas actúan **sobre** los keyframes ya insertados, no en su lugar. El keyframe temporal es un keyframe real dentro del Action existente (§3.3), así que la simulación lo toma como objetivo exactamente igual que tomaría un keyframe confirmado. El principio es el mismo que en Blender vanilla: con un hueso padre sin físicas y un hijo con físicas **y con keyframes**, al mover el padre el hijo combina correctamente lo que dicen los keyframes con lo que dice la física. **El overlay hereda ese comportamiento sin necesidad de ningún mecanismo especial.**

**Interacción vanilla conocida, no defecto de este producto.** Si se activan físicas en el **mismo hueso que se está transformando**, hay flickering: la física interfiere con la posición a la que el usuario quiere llevar el hueso. Ocurre igual **sin** el addon instalado. No se corrige aquí y no se considera un fallo de Playblast.

#### 3.9 PRESERVACIÓN DEL ESTADO DE VISTA Y RESTRICCIÓN DEL HOST

**Requisito normativo.** Todo lo que el Host tenga oculto, excluido o no seleccionable debe llegar al Cliente **en el mismo estado** y restaurarse al conectarse. Que un interruptor esté desactivado en el Host y aparezca activado en el Cliente es un **defecto**, no una diferencia aceptable.

El alcance cubre **todas las capas**, no solo los objetos. Enumeración a nivel de propiedad RNA:

| Capa                 | Propiedad                               | Efecto                                       |
| -------------------- | --------------------------------------- | -------------------------------------------- |
| `Object`             | `hide_select`                           | No seleccionable                             |
| `Object`             | `hide_viewport`                         | Oculto global en viewport (icono de monitor) |
| `Object`             | `hide_render`                           | Excluido del render                          |
| `Collection`         | `hide_select`                           | Colección no seleccionable                   |
| `Collection`         | `hide_viewport`                         | Colección oculta global en viewport          |
| `Collection`         | `hide_render`                           | Colección excluida del render                |
| `LayerCollection`    | `exclude`                               | Colección excluida del View Layer            |
| `LayerCollection`    | `hide_viewport`                         | Oculta en este View Layer (icono de ojo)     |
| `LayerCollection`    | `holdout`                               | Holdout                                      |
| `LayerCollection`    | `indirect_only`                         | Solo contribución indirecta                  |
| **DATA de armadura** | `Bone.hide`                             | Visibilidad del hueso                        |
| **DATA de armadura** | Visibilidad de las **bone collections** | Visibilidad de grupos de huesos              |

**Notas normativas de esta enumeración.**

1. **La visibilidad de los huesos vive en la DATA de la armadura, no en la pose.** Serializar la pose no la transporta. Es la causa de que este requisito se pasara por alto.
2. `Collection.hide_viewport` y `LayerCollection.hide_viewport` **son dos interruptores distintos** (global frente a por-View-Layer) y ambos deben preservarse. No son intercambiables.
3. El tipo RNA correcto es **`bpy.types.LayerCollection`**. Los identificadores `RESTRICT_VIEW_OFF` / `RESTRICT_RENDER_OFF` / `RESTRICT_SELECT_OFF` son iconos del outliner, **no propiedades**, y no deben usarse para especificar este requisito.
4. **Candidata pendiente de verificación**: el ojo del outliner (tecla `H`) es por View Layer y se consulta con `Object.hide_get()` / `hide_set()`, no con `Object.hide_viewport`. Si se confirma, se añade a la tabla. Registrada como VP-05 en el TDD; no se afirma como hecho hasta comprobarlo en 5.2.2.
5. La enumeración es **ampliable de forma declarativa**. Si aparece otra capa de estado de vista, se añade a la tabla sin rediseñar el mecanismo.
6. **Todas** estas propiedades deben serializarse en el volcado del datablock y restaurarse en la carga del Cliente. Hoy la mayoría **no se serializa en absoluto**, que es la causa raíz del defecto.
7. **Serializar no basta: el cambio también tiene que publicarse.** Mientras haya un hueso seleccionado en Modo Pose, el sistema bloquea por rendimiento los commits de `Object` y `Armature`. Un interruptor de visibilidad cambiado en ese estado no sale del Host hasta que se deselecciona el hueso. **El requisito de esta sección incluye que el cambio llegue**, no solo que sepa serializarse; la implementación debe resolver la interacción con ese filtro sin retirar la protección de rendimiento.

### 4. FUNCIONALIDAD 1: TOGGLE GLOBAL DE COMMIT/PUSH CONSTANTE

> ### ⚠ FUERA DE SOPORTE EN R1
>
> **Esta funcionalidad está implementada y sigue siendo utilizable, pero no tiene soporte en la release 1.0.0.** No es un requisito de R1: no se garantiza su comportamiento, no se mantiene y no se corrigen sus defectos.
>
> **No se retira del producto.** El código y la interfaz se conservan **intactos**: retirarlos podría romper el comportamiento actual y esperado del addon. La decisión es dejar de darle soporte, no eliminarla.
>
> **Por qué.** El objetivo del producto es la **transmisión inteligente de datos**, no la transmisión masiva de commits. F1 es precisamente el mecanismo de commit continuo masivo por datablock. El canal rápido TCP de F2 existe porque los commits constantes de transformación provocaban un lag enorme.
>
> **Nota sobre el nombre.** La etiqueta `Live Sync` para F1 está mal elegida: el «live sync» real es el canal rápido TCP de F2 (`Pose Live Sync`). Se conserva el nombre por no alterar la interfaz, y se documenta la ambigüedad aquí.
>
> **No confundir F1 con porcelain.** El commit/push del repositorio **no** queda fuera de soporte: es fundamental para el producto y es la vía por la que llegan al Cliente los datos confirmados, incluidos los keyframes insertados. Lo que queda sin soporte es el **armado continuo por datablock**.
>
> **Recuperación.** Podría consolidarse en una versión posterior, posiblemente en la `Phantom Keyframe Update`. Por eso esta sección se conserva completa en lugar de borrarse.
>
> **Defaults verificados al conectar**, en ambos peers: `Pose Live Sync` **activado**, `Live Sync` (F1) **desactivado**. El flujo de trabajo esperado y probado es con F1 desarmado.

#### 4.1 Comportamiento Actual del Addon (Vanilla)

En el panel Repository del addon (pestaña Multiuser en la barra lateral N de la Vista 3D), cada data block (objeto, material, malla, colección, escena, etc.) tiene un botón individual de commit/push (wm.session_datablock_commit).

Al presionar ese botón, se ejecuta:

```python
porcelain.commit(session.repository, target_uuid)
porcelain.push(session.repository, 'origin', target_uuid, force=True)
```

Esto envía manualmente ese data block específico al servidor. Es una operación de una sola vez.

#### 4.2 Cómo funciona el depsgraph y el handler actual

Blender tiene un sistema llamado Dependency Graph (Depsgraph). Es un registro interno que rastrea todas las relaciones entre los datos de la escena. Cada vez que algo cambia en la escena (mueves un objeto, cambias un material, editas una malla, modificas un hueso, etc.), el depsgraph registra ese cambio y dispara una señal llamada `depsgraph_update_post`.

El addon Multi-User ya tiene un handler que escucha esta señal. Está en el archivo `handlers.py` y se llama `on_scene_update`.

> **El bloque siguiente es una instantánea del addon VANILLA**, no del estado actual del código. Se conserva por su valor didáctico. El handler real añade la puerta por rol (`can_publish`), el gate de arrastre interactivo y filtros por tipo de datablock durante el stream de pose. Ilustrativo (§0.1).

```python
@persistent
def on_scene_update(scene):
    """Forward blender depsgraph update to replication"""
    if session and session.state == STATE_ACTIVE:
        blender_depsgraph = bpy.context.view_layer.depsgraph
        dependency_updates = [u for u in blender_depsgraph.updates]

        for update in reversed(dependency_updates):
            update_uuid = getattr(update.id.original, 'uuid', None)
            if update_uuid:
                node = session.repository.graph.get(update_uuid)
                check_common = session.repository.rdp.get_implementation(update.id).bl_check_common
                if node and (node.owner == session.repository.username or check_common):
                    if node.state == UP:
                        porcelain.commit(session.repository, node.uuid)
                        porcelain.push(session.repository, 'origin', node.uuid)
```

Cada vez que cambia algo en la escena, esta función se ejecuta y recibe una lista de todos los data blocks que fueron modificados. Luego verifica condiciones de ownership y estado antes de hacer commit/push.

#### 4.3 Qué detecta y qué NO detecta el depsgraph

Esto es fundamental para entender el alcance de la Funcionalidad 1:

| Acción del usuario                                                | ¿Se dispara depsgraph_update_post? | ¿La Funcionalidad 1 lo detecta? |
| ----------------------------------------------------------------- | ---------------------------------- | ------------------------------- |
| Mover un objeto en el viewport y soltar (confirmar)               | ✅ Sí                              | ✅ Sí                           |
| Cambiar una propiedad en el panel de propiedades                  | ✅ Sí                              | ✅ Sí                           |
| Cambiar un material (color, rugosidad, etc.)                      | ✅ Sí                              | ✅ Sí                           |
| Añadir o eliminar un keyframe                                     | ✅ Sí                              | ✅ Sí                           |
| Presionar R/G/S y mientras estás arrastrando (antes de confirmar) | ❌ No                              | ❌ No                           |
| Editar una malla en Edit Mode y confirmar                         | ✅ Sí                              | ✅ Sí                           |
| Presionar ESC para cancelar una transformación                    | ❌ No (no hubo cambio)             | ❌ No                           |

Conclusión crítica: La Funcionalidad 1, basada en el depsgraph, es completamente ciega a las transformaciones interactivas (R, G, S) antes de que el usuario confirme la operación. Esto no es un error ni una limitación de nuestra modificación; es el comportamiento nativo de Blender. Las transformaciones interactivas son cubiertas por la Funcionalidad 2 (ver Sección 5).

#### 4.4 Comportamiento Deseado con la Modificación

Se añade un Toggle Global en la interfaz del addon.

**Cuando el Toggle Global está DESACTIVADO:**

- Todo funciona exactamente igual que el addon vanilla.
- Los botones individuales de commit/push siguen siendo botones normales de una sola acción.
- Al presionar un botón, se hace un commit/push manual de una sola vez para ese data block.

**Cuando el Toggle Global está ACTIVADO:**

- Los botones individuales de commit/push se convierten en toggles (botones presionables con estado visual depress=True/False).
- Cada toggle individual controla si ese data block específico hace commit/push constantemente. "Constantemente" significa: cada vez que el depsgraph detecta un cambio en ese data block, se ejecuta commit/push automáticamente. Si no hay cambios, no se envía nada.
- **Matiz importante**: la proporción «un cambio → un commit/push» **no se sostiene durante el arrastre interactivo ni con un stream de pose activo**. El gate de arrastre silencia porcelain mientras el usuario arrastra, y durante el stream se bloquean los commits de Object y Armature para no congelar la máquina. Ese gate no es un defecto de F1: es lo que evita que los commits masivos bloqueen el equipo.
- Al activar un toggle individual:
  - Se hace un commit/push inmediato de ese data block.
  - Se registra ese data block en un diccionario de seguimiento (shared_data.session.live_sync_nodes) para commit/push automático en cada cambio posterior detectado por el depsgraph.
- Al desactivar un toggle individual:
  - Se detiene el commit/push automático para ese data block.
  - El data block se elimina del diccionario de seguimiento.
  - Vuelve a requerir commit/push manual.
- Al desactivar el Toggle Global:
  - Todos los toggles individuales se desactivan.
  - Se limpia el diccionario de seguimiento.
  - Todos los data blocks dejan de hacer commit/push constante.
  - Los botones vuelven a ser botones normales de commit/push manual de una sola vez.

#### 4.5 Archivos a Modificar para esta Funcionalidad

| Archivo          | Modificación                                                                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ui.py`          | `draw_commit_control()` dibuja el toggle o el botón según el estado del toggle global. `draw_property()` dibuja la fila del datablock, no el toggle.                               |
| `operators.py`   | Operador `SessionToggleLiveSync` (`wm.session_toggle_live_sync`).                                                                                                                  |
| `handlers.py`    | `on_scene_update()` ejecuta commit/push automático en los datablocks registrados como «live sync».                                                                                 |
| `preferences.py` | La propiedad booleana vive en `SessionProps` como `WindowManager.session.global_live_sync`. **No existe `properties.py`** y la propiedad no se llama `multiuser_global_live_sync`. |

#### 4.6 Estructura de Datos para el Estado de Live Sync

El estado se inicializa como un diccionario vacío en SessionData. Contiene únicamente los UUID de los nodos activados; las claves desactivadas se eliminan. Si un UUID no está presente, su estado se considera desactivado.

```python
# Ejemplo de estado con dos nodos activos
shared_data.session.live_sync_nodes = {
    "uuid-del-nodo-1": True,
    "uuid-del-nodo-2": True,
}

# Comprobación: un nodo ausente se considera desactivado
is_live = shared_data.session.live_sync_nodes.get("uuid-del-nodo-3", False)
```

#### 4.7 Lógica del Handler Modificado (on_scene_update)

Dentro del handler on_scene_update en handlers.py, se debe añadir una condición adicional:

```python
if node and can_publish(node, session.repository.username, check_common, host_peer):
    is_live = shared_data.session.live_sync_nodes.get(node.uuid, False)

    # Rechazar commits de objeto/armadura durante pose-stream activo
    if live_sync.pose_stream_active() and is_live:
        continue

    # El “live sync” armado no reemplaza el control de publicación,
    # pero sí hace que el nodo se considere de alta prioridad:
    if is_live or node.state == UP:
        try:
            porcelain.commit(session.repository, node.uuid)
            porcelain.push(session.repository, 'origin', node.uuid)
        except Exception as e:
            logging.error(f"Live Sync Push Error: {e}")
```

#### 4.8 Lógica de la Interfaz (`draw_commit_control` en ui.py)

```
SI toggle_global_activo:
    dibujar operador "wm.session_toggle_live_sync" como toggle (depress=True/False)
    el estado visual del toggle refleja si ese nodo está en live_sync_nodes
SI NO:
    dibujar operador "wm.session_datablock_commit" como botón normal (comportamiento vanilla)
```

#### 4.9 Limitación Explícita de la Funcionalidad 1

La Funcionalidad 1 solo detecta cambios confirmados por el usuario, es decir, cambios que ya fueron escritos en `bpy.data`. Esto incluye:

- Mover un objeto y soltar.
- Cambiar una propiedad.
- Confirmar una transformación con click izquierdo o Enter.
- Añadir o eliminar keyframes.

La Funcionalidad 1 NO detecta cambios en curso que aún no han sido confirmados. Esto incluye:

- Estar rotando un hueso con R (sin haber hecho click aún).
- Estar moviendo un objeto con G (sin haber hecho click aún).
- Estar escalando con S (sin haber hecho click aún).

Estos cambios en curso son responsabilidad exclusiva de la Funcionalidad 2 (Live Sync de Modo Pose), descrita en la Sección 5.

### 5. FUNCIONALIDAD 2: LIVE SYNC DE MODO POSE (Transformaciones Interactivas)

#### 5.1 El Problema

En Blender vanilla, cuando estás en Modo Pose y presionas R (rotar), G (mover) o S (escalar) sobre un hueso:

- El hueso muestra la transformación visualmente en el viewport.
- Blender actualiza la representación visual interna (el depsgraph evaluado).
- Pero Blender NO escribe el cambio en `bpy.data` hasta que confirmas la operación (click izquierdo o Enter).
- El handler depsgraph_update_post NO se dispara durante la transformación interactiva.
- Por lo tanto, el addon Multi-User no detecta el cambio y no hace commit/push.
- El Cliente no ve la transformación hasta que el Host confirma.

#### 5.2 La Solución: Stream TCP Directo + Keyframe Temporal en el Frame del Host

Se implementa un canal de comunicación paralelo al sistema de repositorio (porcelain) del addon. Este canal envía datos de transformación en tiempo real directamente por TCP, sin pasar por el sistema de commits.

**El canal es un socket independiente, no un canal multiplexado dentro del socket de la réplica.** Escucha en el **puerto base + 1**, con su propio entramado de mensajes y sus propios hilos de red. Consecuencia operativa: la operación remota necesita **dos puertos abiertos** (§2.4). El multiplexado en un único socket queda como evolución futura, no como requisito de R1.

En el Cliente, la transformación recibida se manifiesta como un keyframe temporal insertado o sobrepuesto en el frame específico donde el Host está trabajando. El resto de la animación del Cliente sigue reproduciéndose normalmente. El hueso editado sigue animándose en todos los demás frames; solo en el frame específico del Host se muestra la pose editada.

#### 5.3 Flujo del Host (Editor)

> **Disparador: la SELECCIÓN, no el arrastre.** El stream se abre cuando hay huesos seleccionados en Modo Pose, sin exigir que el usuario pulse R, G o S. El arrastre es un caso **dentro** del stream, no su condición de apertura. Esto es lo que permite que el overlay siga al hueso seleccionado y al frame del Host mientras hace scrub, y que el visor vea la pose seleccionada aunque el Host no arrastre nada.
>
> El gate de arrastre existe **aparte** y su única función es silenciar porcelain mientras el usuario arrastra, para que los commits masivos no congelen la máquina. No gobierna el stream de pose.

**Paso 1: Apertura del stream por selección**

1. El Host entra en Modo Pose y selecciona **uno o varios huesos**, de **una o varias armaduras** (§5.11).
2. Se detecta la selección. Se envía por TCP una señal `"start"` **por cada hueso seleccionado**, con:
   - UUID de la armadura **dueña de ese hueso**
   - Nombre del hueso
   - Frame actual del Host (`current_frame`)
3. Se envía de inmediato la pose visual actual, para que el visor no quede esperando un intervalo completo.
4. Si la selección cambia en caliente, los huesos añadidos reciben `"start"` y los retirados reciben `"end"`. El stream no se cierra y se reabre: se ajusta.

**Paso 2: Transformación en curso**

5. Mientras el Host mueve, rota o escala el hueso (sin confirmar aún):

- Se lee la pose visual usando `evaluated_get(depsgraph)`, que es la única forma de obtener la transformación tal como se ve en pantalla antes de confirmar (ver §5.9).
- Se envía por el **canal TCP directo** una señal `"update"` con:
  - UUID de la armadura dueña del hueso
  - Nombre del hueso
  - Frame actual del Host
  - **Canales crudos**: `location`, `rotation`, `scale`
  - **`rotation_mode`** del hueso
  - `matrix_basis`, por compatibilidad con visores de compilación anterior
- **Los canales crudos son normativos, no un detalle de implementación.** Una matriz no puede representar más de una vuelta completa: un euler de 270° del Host se descompondría en −90° en el visor. Misma pose visible, valor almacenado distinto, y la animación del visor reproduciría mal. Enviar los canales crudos junto al modo de rotación preserva la fidelidad.
- Frecuencia: **60 Hz**. Se envía también un latido periódico aunque no haya movimiento, para que un visor que se conecte a mitad de la selección reconstruya el overlay.
- Si el Host hace scrub durante la edición, el overlay del visor **se mueve al nuevo frame** y el frame anterior se restaura (§9.3).

**Paso 3a: Confirmación (click izquierdo o Enter)**

6. El Host confirma la transformación.

- Blender escribe el cambio en `bpy.data` y el depsgraph se dispara.
- **La confirmación NO cierra el stream.** El hueso sigue seleccionado, así que el overlay del visor sigue vivo y siguiendo al Host.
- Los datos confirmados viajan al Cliente por **porcelain**, no por este canal (§6).

**Paso 3b: Cancelación (click derecho o ESC)**

7. El Host cancela la transformación.

- Blender NO escribe nada en `bpy.data`; el hueso vuelve a su pose previa.
- El stream sigue abierto mientras el hueso siga seleccionado. Al deseleccionar, la comparación de pose del paso 4 dará «cancelada» y el visor restaurará.

**Paso 4: Cierre del stream por deselección**

8. El Host deselecciona el hueso (o cambia de armadura activa, o sale de Modo Pose).

- Se envía `"end"` **por cada hueso retirado**, con `frame` y con `is_cancelled`.
- **`is_cancelled` se calcula comparando la pose actual del hueso contra la instantánea tomada al abrir el stream**, con una tolerancia. El criterio es **«¿cambió la pose?»**, no «¿se insertó un keyframe?»:

| Situación al deseleccionar                        | `is_cancelled` | Qué debe hacer el Cliente  |
| ------------------------------------------------- | -------------- | -------------------------- |
| Se seleccionó y no se movió el hueso              | `true`         | Restaurar; no queda rastro |
| Se canceló la transformación (Esc / clic derecho) | `true`         | Restaurar; no queda rastro |
| Se confirmó R/G/S, con o sin keyframe insertado   | `false`        | Restaurar en R1 (§5.4)     |

- **En R1 el Cliente restaura el overlay en ambos casos.** La diferencia entre `true` y `false` queda registrada para la PKU, no para R1.

#### 5.4 Flujo del Cliente (Visor)

**Paso 1: Recepción de la señal "start"**

1. El Cliente está reproduciendo su animación en bucle normalmente.
2. Recibe la señal "start".
3. Identifica el hueso y el frame del Host.
4. Guarda en memoria los valores originales del keyframe existente en ese frame (si existe) y anota qué curvas tuvo que crear. Esto es necesario para poder restaurar la animación al cerrarse el stream.
5. Prepara el estado para recibir actualizaciones.

**Paso 2: Recepción de señales "update"**

6. Recibe señales "update" continuamente mientras el Host edita.
7. Inserta o actualiza el keyframe temporal en el frame del Host con los canales recibidos, interpretados según su `rotation_mode` (§5.7). Si la curva F del canal no existe, la crea (§5.6, Caso C).

- Usa fcurve.keyframe_points.insert(frame, value, options={'FAST'}) para rendimiento.
- NO crea un keyframe nuevo cada vez. Si ya existe el keyframe temporal en ese frame, actualiza su valor.

8. La animación del Cliente sigue reproduciéndose. El keyframe temporal afecta únicamente el frame específico del Host.

**Paso 3: Cuánto vive el keyframe temporal**

9. **Mientras el hueso siga seleccionado en el Host, el keyframe temporal persiste indefinidamente.** No hay cuenta atrás ni caducidad. El usuario puede tomarse todo el tiempo que quiera para decidir si inserta un keyframe, moverse por el Dope Sheet, cambiar de frame o seguir ajustando la pose: el overlay sigue ahí y sigue al Host.

> **La ventana de decisión es la selección misma.** Ésa es la respuesta a «el keyframe temporal debe permanecer hasta que el usuario decida»: permanece mientras el Host mantenga el hueso seleccionado. No necesita un temporizador de gracia.

**Paso 4a: Recepción de `"end"` con cancelación (`is_cancelled = true`)**

10. El Cliente restaura los valores originales respaldados y elimina los keyframes que él insertó. No queda rastro de la edición.
11. Se liberan los respaldos.

**Paso 4b: Recepción de `"end"` con confirmación (`is_cancelled = false`)**

12. **En R1 el Cliente restaura igualmente el overlay, de inmediato.** El orden es: procesar `"end"` → restaurar → aplicar después los datos del repositorio si llegan (§6.4).
13. Se liberan los respaldos.
14. Si el Host insertó un keyframe real, éste llega por porcelain y el Cliente lo aplica como dato definitivo. Si el Host **no** insertó keyframe, no llega nada, y el Cliente simplemente vuelve a la animación original: **eso es correcto**, porque confirmar R/G/S no expresa intención de conservar la pose. Insertar un keyframe sí la expresa.

> **Por qué R1 restaura al deseleccionar.** Si se retiene el temporal esperando un push que puede no llegar —el caso exacto de confirmar sin insertar keyframe, donde el Action no cambia y por tanto no hay commit—, el keyframe queda en el Action del Cliente **sin plazo definido**. No es permanente: lo limpia el siguiente apply de ese Action, porque la carga borra todos los puntos de las curvas del dump y poda las curvas que el dump no trae. **Sobreviven las dos cosas: el keyframe y la curva que lo contiene.** El mecanismo es exacto: `_restore` primero quita los puntos que él insertó (`live_sync.py:1364-1366`) y después elimina las curvas inventadas **solo si se han quedado sin puntos** (`:1375-1381`). Al confirmar se vacían `backups` y `created_curves` (`:1157-1158`), así que no se ejecuta ni la retirada del punto ni la de la curva — y el punto que sobrevive es precisamente lo que impide que la curva se considere vacía. Pero mientras el Host no vuelva a tocar ese Action, el visor muestra **un keyframe real y visible en su Dope Sheet** que nadie insertó, en un hueso que el Host nunca keyeó. **Eso es el keyframe fantasma**, y §3.5 y §7.9 piden que en R1 no exista ninguno, no que desaparezca más tarde. La retención-hasta-reemplazo se implementará como período de gracia acotado en una versión posterior.
>
> **Coste aceptado.** Cuando confirmar, insertar keyframe y deseleccionar ocurren en rápida sucesión, el visor puede mostrar un **parpadeo breve** entre la restauración y la llegada del dato real. Se acepta y se registra como known issue (KI-01). Es preferible un parpadeo a un fantasma permanente.

#### 5.5 Resultado Visual en el Cliente

- El Cliente ve la animación corriendo en bucle normalmente.
- Todos los huesos se animan según la animación original.
- El hueso que el Host está editando también se anima normalmente en todos los frames.
- El hueso que el Host está editando también tiene el mismo comportamiento que vanilla.
- PERO, cuando el playhead del Cliente pasa por el frame específico donde el Host está trabajando, ese hueso muestra la pose editada por el Host en lugar de la pose original de ese frame.
- El reviewer puede ver cómo se ve la edición en contexto con las transiciones de entrada y salida de esa pose.

```
Cliente reproduce bucle:
Frame 1 → 2 → ... → 29 → [Frame 30: mano muestra la edición del Host] → 31 → ... → 100 → loop
                           ↑
                    Keyframe temporal insertado/sobrepuesto
                    con la pose que el Host está editando en vivo
```

#### 5.6 Detalle sobre la Modificación Temporal de la Animación del Cliente

La animación del Cliente sí se modifica temporalmente durante la edición interactiva. Esto es necesario y esperado. La modificación ocurre de la siguiente manera:

**Caso A: Ya existe un keyframe en el frame del Host (ej. frame 30)**

1. Se lee el valor actual del keyframe en el frame 30.
2. Se guarda ese valor en memoria como respaldo.
3. Se sobreescribe el keyframe en el frame 30 con el valor de la edición del Host.
4. La animación del Cliente queda temporalmente alterada en ese frame.

**Caso B: NO existe un keyframe en el frame del Host (ej. frame 30)**

1. Se inserta un keyframe nuevo en el frame 30 con el valor de la edición del Host.
2. La animación del Cliente queda temporalmente alterada (se añadió un dato que no existía).

**Caso C: NO existe la curva F del canal (requisito explícito de R1)**

Éste es el caso que la v1 no contemplaba y es **característica núcleo del producto**, no un caso límite.

1. El hueso puede tener keyframes en algunos canales y en otros no (por ejemplo solo `location` keyeado), o no tener ninguno.
2. Cuando la curva F del canal no existe, **el sistema la crea**. La creación es **explícita**: se busca la curva por ruta de datos e índice, y si no existe se crea y se inserta el punto. No se delega a Blender: la inserción rápida sobre una curva exige que la curva exista previamente.
3. **Se registra la procedencia** de cada curva creada, para poder distinguirla de las curvas legítimas del animador.
4. Una vez creada, Blender la evalúa por sí solo. No hace falta asignar un slot por curva.

> **Regla de protección, normativa e irrenunciable.** **No se poda ninguna curva inventada por el keyframe temporal mientras ese keyframe temporal siga vivo.** La poda de curvas solo puede ocurrir _después_ de que una recarga del repositorio haya invalidado el overlay, y **acotada por procedencia**: únicamente curvas que el overlay creó y que no figuran en los datos recibidos. **Nunca curvas legítimas del animador.**
>
> Esta característica ya funcionaba antes de que existiera el TDD. Se escribe como requisito precisamente para que una futura «limpieza» no la elimine creyendo que mejora algo.

**Restauración:**

- Si el Host cancela: en el Caso A se restaura el valor respaldado; en el Caso B se elimina el punto insertado; en el Caso C se elimina además la curva creada **si queda vacía**.
- Si el Host confirma y deselecciona: en R1 se restaura igualmente (§5.4 paso 4b). Si llega un commit/push real, sus datos se aplican como definitivos.
- Los respaldos guardan **solo datos simples** (valores, fotogramas, handles), nunca referencias vivas a curvas o keyframes: esas referencias quedan inválidas en cuanto el repositorio reconstruye el Action.

#### 5.7 Formato del Mensaje TCP (Canal Live Sync)

Esquema plano, un mensaje por hueso. **Los nombres de campo son ilustrativos (§0.1); la obligatoriedad y el contenido son normativos.**

**Campos obligatorios en los tres tipos de mensaje**: `channel`, `action`, `armature_uuid`, `bone_name`, `frame`. Un mensaje al que falte `armature_uuid` o `bone_name` **se descarta**: sin ellos no hay forma de saber a qué hueso de qué armadura pertenece.

`"start"` — apertura por selección:

```json
{
  "channel": "live_sync",
  "action": "start",
  "armature_uuid": "uuid-de-la-armadura-duena",
  "bone_name": "DEF-Shaft_01",
  "frame": 30
}
```

`"update"` — pose en curso:

```json
{
  "channel": "live_sync",
  "action": "update",
  "armature_uuid": "uuid-de-la-armadura-duena",
  "bone_name": "DEF-Shaft_01",
  "frame": 30,
  "location": [0.0, 0.0, 0.0],
  "rotation_mode": "QUATERNION",
  "rotation": [1.0, 0.0, 0.0, 0.0],
  "scale": [1.0, 1.0, 1.0],
  "matrix_basis": [
    [1, 0, 0, 0],
    [0, 1, 0, 0],
    [0, 0, 1, 0],
    [0, 0, 0, 1]
  ]
}
```

`"end"` — cierre por deselección:

```json
{
  "channel": "live_sync",
  "action": "end",
  "armature_uuid": "uuid-de-la-armadura-duena",
  "bone_name": "DEF-Shaft_01",
  "frame": 30,
  "is_cancelled": false
}
```

**Notas normativas del contrato.**

- `rotation` se interpreta según `rotation_mode`. **`rotation_mode` es obligatorio en `"update"`**: sin él, un cuaternión y un euler son indistinguibles y el visor escribiría en la ruta de datos equivocada.
- `matrix_basis` se conserva **por compatibilidad** con visores de compilación anterior. Un visor actual usa los canales crudos. Un mensaje que traiga solo matriz sigue siendo aceptable.
- **`end` incluye siempre `frame`.** La v1 decía que «puede» incluirlo; es obligatorio.
- `armature_uuid` identifica a la **armadura dueña del hueso**, no a la armadura activa del Host. Es lo que hace posible el multi-armadura (§5.11).
- Un `"update"` que llegue sin `"start"` previo —visor que se conectó a mitad de la selección— **reconstruye el overlay**: trae toda la información que un `"start"` aporta. Pero si ese visor ya vio el `"end"` de ese hueso, el `"update"` es un resto en vuelo y **no debe resucitar el overlay**.

#### 5.8 Máquina de Estados del Live Sync

```
IDLE
  ↓ (Host SELECCIONA uno o varios huesos en Modo Pose → señal "start" por hueso)
INTERACTING
  ↓ (Host DESELECCIONA el hueso → señal "end" con is_cancelled por comparación de pose)
RESOLVING
  ↓ (el Cliente restaura el overlay y libera los respaldos)
IDLE
```

- **IDLE**: no hay huesos seleccionados en Modo Pose. El sistema está en reposo.
- **INTERACTING**: hay huesos seleccionados. El Cliente recibe `"update"` y actualiza su keyframe temporal. **Este estado abarca el arrastre, la confirmación, la cancelación y el scrub**: confirmar o cancelar no lo abandona.
- **RESOLVING**: el Cliente está restaurando el overlay de un hueso cuyo stream se cerró.
- La transición de salida de INTERACTING es la **deselección**, no la confirmación. Es la diferencia estructural respecto a la v1.

#### 5.9 evaluated_get() Explicado

Durante una transformación interactiva (modal operator), `bpy.data` no se actualiza hasta confirmar. Para leer la transformación visual real, se debe usar el depsgraph evaluado:

```python
depsgraph = bpy.context.evaluated_depsgraph_get()
evaluated_bone = pose_bone.evaluated_get(depsgraph)
matrix = evaluated_bone.matrix_basis
```

- `evaluated_get(depsgraph)` devuelve una copia temporal y evaluada del objeto/hueso, tal como se ve en pantalla en ese momento.
- Esto incluye transformaciones interactivas no confirmadas, modificadores aplicados visualmente, y cualquier otro cambio que sea visible en el viewport pero que aún no se haya escrito en `bpy.data`.
- **Importante**: `porcelain.commit()` lee de `bpy.data` (los datos originales), no del depsgraph evaluado. Por eso las transformaciones interactivas deben ir por el canal TCP directo y no por `porcelain`.

#### 5.10 Archivos a Crear/Modificar para esta Funcionalidad

| Archivo                         | Modificación                                                                                                                                                            |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `live_sync.py` (raíz del addon) | Motor del Live Sync: socket propio en puerto base+1, stream de pose, overlay de keyframes temporales, respaldos, máquina de estados. **No está en `libs/replication/`** |
| `timers.py`                     | Temporizador que hace avanzar el stream y el applying. La detección es por selección, no por operador modal                                                             |
| `handlers.py`                   | Gate de arrastre que silencia porcelain; filtros por tipo durante el stream; puerta por rol                                                                             |
| `preferences.py`                | Propiedades `pose_live_sync` y `global_live_sync` en `SessionProps`                                                                                                     |

**Corrección respecto a la v1**: no existe `libs/replication/`, no existe `replication/session.py` y el canal **no** se inyecta en el socket TCP existente. Son un socket y unos hilos propios.

#### 5.11 MULTI-ARMADURA SIMULTÁNEA (requisito de R1)

**El usuario puede seleccionar huesos de varias armaduras a la vez y todas deben transmitirse.** No solo la armadura activa.

**Comportamiento requerido.**

1. Cada hueso seleccionado se transmite con el UUID de **su propia** armadura, no con el de la armadura activa del Host.
2. La pose de cada hueso se lee del depsgraph evaluado de **su propia** armadura.
3. Los `"start"`, `"update"` y `"end"` se emiten por hueso, agrupados por armadura.
4. El Cliente mantiene un overlay independiente por par **(armadura, hueso)**. Dos huesos con el mismo nombre en armaduras distintas **no colisionan**.
5. Cambiar de armadura activa no cancela los streams de las demás.

**Estado actual y alcance del arreglo.** El defecto está **solo en el emisor**: el Host toma únicamente el objeto activo y etiqueta todos los huesos seleccionados con ese único UUID, de modo que los huesos de las demás armaduras se descartan en silencio. El **receptor ya es multi-armadura**: indexa por `(armature_uuid, bone_name)` y resuelve cada armadura por su UUID.

**Consecuencia importante**: el campo `armature_uuid` **ya existe y ya es por mensaje y por hueso**. Por tanto **no hace falta cambiar el esquema del protocolo, ni subir su versión, ni romper la compatibilidad** entre un Host nuevo y un Cliente anterior. El arreglo consiste en rellenar correctamente el campo que ya existe y leer cada hueso de su propia armadura.

_Verificaciones previas registradas en el TDD_: que la propiedad de identidad del hueso apunte al **objeto** dueño de la pose (VP-06), y que la selección de huesos del contexto abarque varias armaduras en Modo Pose multi-objeto (VP-07). Si la segunda no se cumpliera, el agrupamiento se haría recorriendo los objetos en Modo Pose.

### 6. INTERACCIÓN ENTRE FUNCIONALIDAD 1 Y FUNCIONALIDAD 2

#### 6.1 Principio General

Las dos vías operan de forma complementaria sobre fases distintas de la edición: el **canal de pose** (F2) cubre la transformación en curso, antes de confirmar; **porcelain** cubre los datos ya confirmados y escritos en `bpy.data`.

**Reformulación respecto a la v1.** La v1 afirmaba que ambas «nunca actúan simultáneamente». Eso es inexacto con el disparador por selección: F2 puede estar abierta mientras porcelain publica otra cosa —por ejemplo el Host cambia una propiedad con un hueso seleccionado—. **Lo que no coexiste es publicar la misma transformación por las dos vías.** Durante el arrastre, porcelain está silenciado por el gate; durante el stream se bloquean los commits de Object y Armature. Los commits de Action sí se permiten, porque son los keyframes.

**Nota de alcance, y distinción que importa.** En R1, F1 está fuera de soporte (§4) y sus toggles están desarmados por defecto. **La publicación de datos confirmados la hace porcelain por la vía normal, no F1: son dos rutas de código distintas.** Con el toggle desarmado, el nodo se publica porque está en estado «modificado», que es el comportamiento heredado del addon base; con el toggle armado se publica además por el armado continuo. Esta sección describe la **vía normal**. Las menciones al «toggle activo» se conservan solo por fidelidad histórica y porque F1 sigue siendo utilizable.

#### 6.2 Fases de una Transformación Interactiva

**Fase 1: Selección del hueso**

- El Host selecciona uno o varios huesos en Modo Pose (§5.3).
- `depsgraph_update_post` NO se dispara por la selección. Blender no ha escrito nada en `bpy.data`.
- **Canal de pose (F2)**: se abre. Envía `"start"` por hueso y la pose visual inmediata.
- **Porcelain**: inactivo para esta selección. No hay cambio de datos que publicar.

**Fase 2: Transformación en curso (antes de confirmar)**

- El Host está rotando/moviendo/escalando el hueso. El cambio es solo visual.
- `depsgraph_update_post` NO se dispara.
- **Funcionalidad 1**: Inactiva. No hay cambios en `bpy.data` que detectar. Los toggles de Live Sync no tienen efecto en esta fase.
- **Funcionalidad 2**: Activa. Lee `evaluated_get(depsgraph)` y envía señales `"update"` con la matriz visual del hueso por el canal TCP directo. El Cliente actualiza el keyframe temporal.

**Fase 3a: Confirmación (click izquierdo o Enter)**

- El Host confirma la transformación. Blender escribe el cambio en `bpy.data`.
- `depsgraph_update_post` SE DISPARA.
- **Canal de pose (F2)**: **no envía `"end"`**. El hueso sigue seleccionado, el stream sigue abierto y el overlay del Cliente sigue vivo.
- **Porcelain**: el depsgraph detecta el cambio escrito y publica el nodo por la vía normal. **No hace falta que ningún toggle esté armado.** Si el usuario insertó un keyframe, el dato viaja por aquí.
- _[Reservado para la PKU, no aplica a R1]_: `Phantom Keyframe Mode` podrá modificar qué se conserva en esta fase.
- **Coordinación**: si el Host además deselecciona en rápida sucesión, el `"end"` y el commit/push pueden coincidir. El orden obligatorio es el de §6.4.

**Fase 3b: Cancelación (click derecho o ESC)**

- El Host cancela la transformación. Blender NO escribe nada en `bpy.data`.
- `depsgraph_update_post` NO se dispara.
- **Funcionalidad 1**: Inactiva. No hay cambios en `bpy.data` que detectar. No se ejecuta commit/push.
- **Canal de pose (F2)**: el stream sigue abierto mientras el hueso siga seleccionado. Al deseleccionar, la pose coincide con la instantánea inicial, `is_cancelled` resulta `true` y el Cliente restaura los valores originales y elimina el keyframe temporal.

#### 6.3 Tabla Resumen de Actividad por Fase

| Fase                                         | ¿depsgraph se dispara? | Porcelain (vía normal)                    | Canal de pose (F2)                                                       |
| -------------------------------------------- | ---------------------- | ----------------------------------------- | ------------------------------------------------------------------------ |
| **Selección del hueso**                      | ❌ No                  | Inactivo                                  | **Abre el stream**: `"start"` por hueso                                  |
| Transformación en curso (arrastrando)        | ❌ No                  | **Silenciado por el gate de arrastre**    | Activo: `"update"` a 60 Hz                                               |
| Confirmación (clic izquierdo / Enter)        | ✅ Sí                  | **Publica por la vía normal, sin toggle** | **Sigue abierto**; no envía `"end"`                                      |
| Cancelación (Esc / clic derecho)             | ❌ No                  | Inactivo                                  | Sigue abierto                                                            |
| Scrub del timeline con el hueso seleccionado | ✅ Sí                  | Publica lo que cambie                     | Activo: mueve el overlay al nuevo frame                                  |
| **Deselección del hueso**                    | ❌ No                  | Inactivo                                  | **Cierra el stream**: `"end"` con `is_cancelled` por comparación de pose |

#### 6.4 Coordinación en la Fase de Confirmación

En la fase de confirmación (Fase 3a), ambas funcionalidades actúan brevemente al mismo tiempo. Es importante que no haya conflicto entre ellas:

**Orden normativo de R1: `end` → restaurar → aplicar.**

1. El canal de pose envía `"end"` cuando el Host **deselecciona** el hueso.
2. Porcelain publica los datos confirmados por el canal del repositorio, si los hay.
3. **El Cliente procesa primero `"end"` y restaura el overlay; después aplica los datos del repositorio.** Este orden es obligatorio en R1 y prevalece sobre la redacción de retención que la v1 tenía en §5.4, ya corregida.
4. **Caso de invalidación**: si los datos del repositorio llegan y reconstruyen el Action _mientras_ el overlay está vivo, los respaldos del overlay quedan obsoletos. En ese caso el Cliente **descarta el overlay sin restaurarlo** —restaurar escribiría valores obsoletos sobre curvas recién aplicadas, que es como desaparecía un keyframe que el Host acababa de confirmar—. No se pierde nada: el Host reenvía la pose en un latido y el overlay se reconstruye. La contabilidad de procedencia de las curvas creadas (§5.6 Caso C) **debe sobrevivir a ese descarte**, para que la poda posterior pueda acotarse.

#### 6.5 Qué llega al Cliente con los toggles desarmados

**Éste es el flujo normal y esperado de R1**, no un caso degradado. Con `Pose Live Sync` activado y `Live Sync` (F1) desactivado en ambos peers —los valores por defecto—:

- La transformación en curso llega al Cliente por el canal de pose. **Verificado en prueba del mantenedor.**
- Al seleccionar un hueso que aún no tiene curvas keyeadas, el Cliente crea el keyframe temporal y las curvas que falten (§5.6 Caso C). **Verificado.**
- Al deseleccionar sin haber movido el hueso, el Cliente lo limpia. **Verificado.**
- **Insertar y eliminar keyframes en el Host se transmite al Cliente por porcelain, sin necesidad de ningún toggle de F1.** Ése es el comportamiento esperado e ideal.

**Matiz técnico que hay que declarar, no ocultar.** Insertar un keyframe hace que el depsgraph reporte el **Object**, no el **Action**. Un Action que se queda en estado «modificado» con keyframes nuevos puede no publicarse por sí solo. Existe una función que publica explícitamente esos Actions, y hoy **solo se invoca cuando el toggle está armado**. Con F1 fuera de soporte y desarmado por defecto, ese camino no se ejecuta.

- **No se modifica ese comportamiento en R1**: la ruta está declarada funcionando y no hay fallo medido que justifique tocarla. Cambiarlo sin evidencia violaría la regla de no alterar rutas que funcionan.
- Se registra como **known issue KI-08**, con una medición asociada (VP-08): con ambos toggles desarmados, insertar un keyframe en un hueso cuyo Action ya esté modificado y comprobar si llega al Cliente. Si llega siempre, KI-08 se cierra como no reproducible. Si falla, se corrige con la evidencia en la mano.

**Vacío visual breve.** Si el Host confirma, inserta keyframe y deselecciona en rápida sucesión, el Cliente puede mostrar un parpadeo entre la restauración y la llegada del dato real. Se acepta y se registra como **KI-01** (§5.4).

#### 6.6 La publicación de lo confirmado no puede depender de otro operador

**Requisito normativo de R1.** Cuando el Host confirma una edición, el dato debe publicarse **sin esperar a que el usuario ejecute otro operador**. El gate que silencia porcelain durante el arrastre debe distinguir un arrastre **en curso** de uno **ya finalizado**: hoy no lo hace, porque inspecciona el último operador del historial sin comprobar si sigue ejecutándose, y el historial es acumulativo.

**Por qué es requisito y no un detalle interno.** Al eliminar los fantasmas en R1 (§5.4), desaparece el efecto que **enmascaraba** este fallo: antes el Cliente conservaba el overlay y seguía mostrando la pose editada aunque el push no llegara. Con la restauración al deseleccionar, un push bloqueado deja al revisor mirando la **pose antigua** durante un tiempo indefinido. El parpadeo aceptado en KI-01 presupone que el dato llega pronto; este requisito es lo que lo garantiza.

**Preservación obligatoria.** El gate **debe seguir cerrando durante el arrastre real**: arrastrar actualiza el depsgraph continuamente y publicar en ese momento congela la máquina. Ese rationale no se toca. Lo único que cambia es que un arrastre terminado deja de bloquear.

_Auto-keying sigue fuera del alcance de R1 (§0.2); este requisito no lo incorpora como capacidad soportada._

### 7. LO QUE ESTRICTAMENTE NO SE DEBE IMPLEMENTAR

1. NO pausar el timeline del Cliente. El Cliente siempre controla la reproducción de su propio timeline, normalmente en bucle. No ejecutar `is_animation_playing = False` ni `frame_set()` en el Cliente como parte del flujo de Live Sync.
2. NO silenciar (`mute`) las Curvas F del hueso editado. No usar `fcurve.mute = True`. El hueso debe seguir animándose.
3. NO congelar ni desconectar el hueso editado de la animación del Cliente.
4. NO usar Action Swapping. No crear Acciones vacías temporales. No intercambiar `animation_data.action`. No reemplazar la Acción original del Cliente.
5. NO crear keyframes en cada frame del bucle del Cliente. Solo se manipula UN keyframe temporal por hueso editado, ubicado en el frame específico del Host.
6. NO usar `bone.keyframe_insert()` estándar. Usar `fcurve.keyframe_points.insert()` con `options={'FAST'}`.
7. NO usar hilos de Python (`threading`) para acceder a datos de `bpy.data`. Usar `bpy.app.timers` o handlers nativos de Blender.
8. NO hacer commit/push por `porcelain` durante la transformación interactiva (antes de confirmar). Usar el canal TCP directo para las actualizaciones en vivo.
9. NO dejar keyframes temporales huérfanos. Toda modificación temporal debe limpiarse al cerrarse el stream, tanto si la transformación se canceló como si se confirmó. **En R1 esto es incondicional.** _[Reservado para la PKU]_: `Phantom Keyframe Mode` podrá autorizar excepciones deliberadas; no existe en R1.
10. **NO podar ninguna curva F inventada por el overlay mientras el keyframe temporal siga vivo** (§5.6, Caso C). La poda solo se admite tras la invalidación del overlay y acotada por procedencia. Nunca sobre curvas legítimas del animador.
11. **NO forzar `hide_select` sobre los objetos del Cliente** en función de la propiedad o de la condición de solo-lectura (§3.8). El Cliente puede seleccionar siempre.
12. **El Cliente NO publica nada, por ninguna vía** (§1.4, INV-1). Esto incluye las rutas auxiliares: dependencias externas de imagen y rutas de fichero, la rama de escena, los temporizadores de anotación y los botones manuales de commit/push. Un visor que daña el trabajo del animador es el fallo de mayor gravedad del producto.
13. **NO compartir el cache entre Host y Cliente**, tampoco en LAN (§2.1). Cada peer tiene su propia subcarpeta y purga únicamente lo que él escribió.
14. **NO crear una colección de sesión ni alterar la jerarquía de colecciones** (§2.3).

### 8. CONSIDERACIONES DE RENDIMIENTO

1. **Inserción de keyframes**: Usar siempre `options={'FAST'}` para evitar recálculo de tangentes y actualizaciones pesadas del depsgraph. Esto hace la inserción casi instantánea.
2. **Frecuencia de actualización**: el stream de pose emite a **60 Hz** (intervalo de 1/60 s), con un latido periódico adicional cuando no hay movimiento. No saturar la red ni el socket. La v1 decía «30-60 FPS»; el valor real y normativo es 60 Hz.
3. **Recomendación de uso sobre qué datablocks armar** _(ORIENTATIVA — no normativa)_.

> **Estatus de esta tabla: recomendación de uso, no restricción.** El addon **no bloquea ningún tipo de datablock**: el usuario arma lo que quiera, incluidos `mesh`, `image` y `volume`. No existe ni debe existir un filtro por peso que le quite esa elección. La tabla orienta; no obliga. Se marca así porque la v1 la redactaba como prohibición («NO enviar…») y se contradecía a media frase al admitir que «el toggle es el que le permite al usuario elegir». Resuelto por DEC-13.
>
> **Nota de alcance**: F1 está fuera de soporte en R1 (§4), de modo que esta tabla es orientación para quien decida usarla, no un requisito de la release.

Cuando se hace commit/push constante, conviene priorizar datablocks ligeros y relevantes para animación:

| Data Block                       | Peso         | ¿Conviene Live Sync? |
| -------------------------------- | ------------ | -------------------- |
| `object` (transformación)        | Ligero       | ✅ Sí                |
| `scene` (frame actual, settings) | Ligero       | ✅ Sí                |
| `camera`                         | Ligero       | ✅ Sí                |
| `armature` (huesos)              | Ligero-Medio | ✅ Sí                |
| `action` (animación)             | Medio        | ✅ Sí                |
| `material`                       | Ligero       | ✅ Sí                |
| `mesh` (vértices, caras)         | Muy pesado   | ❌ No                |
| `image` (texturas)               | Muy pesado   | ❌ No                |
| `volume`                         | Muy pesado   | ❌ No                |

4. **Canal TCP directo**: Las transformaciones interactivas deben ir por un canal separado que no pase por el sistema de repositorio (`porcelain`). Esto evita generar commits basura en el historial del repositorio y no bloquea la interfaz del Host.
5. **Un solo keyframe temporal por cada hueso transformado**: En el Cliente, solo se mantiene UN keyframe temporal por hueso en edición. Se actualiza su valor, no se recrea el keyframe en cada update.
6. **Limpieza de memoria**: Los valores originales guardados para restauración deben liberarse inmediatamente después de que la edición se confirma o se cancela. No acumular datos en memoria.

### 9. EDGE CASES Y CASOS LÍMITE A CONSIDERAR

1. **Host cancela la transformación (click derecho / ESC)**: El Cliente debe restaurar los valores originales del keyframe en el frame del Host. Si no existía keyframe previo, el keyframe temporal se elimina completamente. No debe quedar rastro de la edición cancelada.
2. **Host confirma la transformación (clic izquierdo / Enter)**: la confirmación **no cierra el stream**; el overlay sigue vivo mientras el hueso siga seleccionado. Al deseleccionar, el Cliente restaura el overlay (§5.4). Si el Host insertó un keyframe, el commit/push real por porcelain trae el dato definitivo y el Cliente lo aplica.
3. **Host cambia de frame mientras edita**: Si el Host se mueve a otro frame durante la edición interactiva, el keyframe temporal en el Cliente debe moverse al nuevo frame. Se debe eliminar el keyframe del frame anterior y crear/actualizar uno en el nuevo frame. Los valores originales del frame anterior deben restaurarse.
4. **Múltiples huesos seleccionados simultáneamente**: **caso normal de trabajo, requisito normativo de R1**, no robustez hipotética. Se mantiene un overlay independiente por hueso, con su propio respaldo y su propio ciclo de `"start"`/`"update"`/`"end"`. La selección puede cambiar en caliente sin cerrar el stream. Que Blender solo permita una transformación interactiva a la vez no reduce el número de huesos que pueden estar transmitiéndose.
5. **Cliente sin Acción utilizable**: si la armadura del Cliente no tiene Acción asignada, o no hay un contenedor de curvas donde keyear, el sistema **conduce el hueso directamente**, guardando su matriz previa para poder restaurarla de forma reversible. **No crea ninguna Acción.** La opción de crear una Acción que la v1 admitía queda **descartada para R1**: añadiría datos al `.blend` del Cliente y tensionaría §3.3.

   **Precisión importante para el diagnóstico.** El overlay **no crea datablocks Action en ningún caso**. Lo que sí crea son **curvas F dentro de un Action que ya existe** (§5.6, Caso C). Si en el Cliente «aparece un Action» al seleccionar el Host un hueso, ese Action llegó por **replicación** desde el Host, no lo fabricó el overlay. Y si la armadura no tiene ningún Action utilizable, **no aparece keyframe temporal alguno**: se conduce el hueso directamente y debe quedar registrado en el log, porque es lo primero que hay que comprobar cuando un visor reporta «la pose nunca aparece».

6. **Host edita un frame donde no existe keyframe previo, o un canal sin curva F**: comportamiento esperado y normal. Ver §5.6, Casos B y C. El Caso C —la curva no existe— es requisito explícito de R1 y está protegido contra la poda.
7. **Host edita un frame donde SÍ existe keyframe previo**: El Cliente debe guardar el valor original del keyframe existente antes de sobrescribirlo con el valor temporal. Al cancelar, restaura el valor original. Al confirmar, el commit/push real reemplaza el dato.
8. **Desconexión del Cliente durante una edición**: no debe haber errores en el Host. Al reconectarse, el Cliente recibe el estado completo del repositorio y **no debe tener keyframes temporales huérfanos**. Un visor que se reconecta a mitad de una selección debe poder reconstruir el overlay a partir de los `"update"` (§5.7). _Recrear una copia del .blend y reconectarse es un último recurso del usuario, no el comportamiento esperado del sistema._
9. **Desconexión del Host durante una edición**: el Cliente detecta la pérdida de conexión, limpia cualquier keyframe temporal pendiente y restaura los valores originales. _Recrear una copia del .blend es último recurso, no comportamiento esperado._
10. **Host edita un hueso que no existe en el Cliente**: el sistema **ignora las señales sin fallar** y sin crashear. Debe quedar constancia en el log. Con el requisito de identidad firme de §2.2 cumplido, esta situación no debería darse; _volver a copiar el .blend es último recurso, no el remedio especificado_.
11. **Toggles de F1 desarmados al confirmar**: es el caso **normal** de R1, no una excepción. Los datos confirmados llegan por porcelain sin necesidad de toggles. Puede haber un parpadeo breve (KI-01) y queda registrado el matiz de publicación del Action (KI-08). Ver §6.5.

12. **Huesos de varias armaduras seleccionados a la vez**: requisito de R1, cada uno con el UUID de su propia armadura. Dos huesos homónimos en armaduras distintas no colisionan. Ver §5.11.

13. **El Host tiene objetos, colecciones o huesos ocultos al conectarse el Cliente**: el Cliente debe reproducir ese estado exactamente, incluida la visibilidad que vive en la DATA de la armadura. Ver §3.9.

14. **Flicker menor al confirmar, insertar y deseleccionar en rápida sucesión**: aceptado y registrado como KI-01. No es un defecto a corregir en R1; es el coste deliberado de no dejar fantasmas.

15. **Físicas activadas en el Cliente sobre un hueso que el Host edita**: la simulación se compone con el keyframe temporal, no lo sustituye (§3.8). El único flickering esperado es el de vanilla cuando las físicas están en el mismo hueso que se transforma, con o sin addon.

### 10. ESTRUCTURA DE ARCHIVOS DEL ADDON (ilustrativa)

Árbol real verificado en `f00d655`. **Ilustrativo (§0.1)**: la ubicación de los ficheros puede cambiar en un refactor sin que ningún requisito de este documento cambie. Se corrige respecto a la v1, que describía un `libs/replication/` inexistente y omitía la mitad de los módulos propios.

```
true_level_playblast/
├── __init__.py            # registro del addon; propiedades de ID (uuid)
├── blender_manifest.toml  # identidad, versión, permisos, build
├── preferences.py         # SessionProps: pose_live_sync, global_live_sync, cache
├── shared_data.py         # SessionData: estado de sesión, live_sync_nodes
├── ui.py                  # paneles; draw_commit_control() dibuja el toggle,
│                          #   draw_property() la fila del datablock
├── operators.py           # SessionCommit, SessionToggleLiveSync, SessionApply,
│                          #   conexión/hosteo, clean_scene()
├── handlers.py            # on_scene_update (gate de arrastre, can_publish,
│                          #   filtros por tipo), update_external_dependencies,
│                          #   sanitize/resolve_deps_graph, load_pre_handler
├── timers.py              # temporizadores: stream de pose, apply, presencia,
│                          #   selección dinámica, sincronización de usuarios
├── live_sync.py           # MOTOR de F2: socket propio en puerto base+1,
│                          #   broadcaster del Host y applier del Cliente,
│                          #   overlay de keyframes temporales y respaldos
├── presence.py            # presencia de usuarios
├── environment.py         # entorno y rutas
├── utils.py               # utilidades
├── bl_types/              # un módulo por tipo de datablock replicado
│   ├── bl_datablock.py    #   base común, resolución por UUID
│   ├── bl_object.py       #   volcado/carga de Object
│   ├── bl_armature.py     #   volcado/carga de Armature
│   ├── bl_action.py       #   volcado/carga de Action, slots y channelbags
│   ├── bl_collection.py   #   volcado/carga de Collection
│   ├── bl_image.py, bl_material.py, bl_mesh.py, bl_scene.py, …
│   └── dump_anything.py   #   serializador genérico
├── replication/           # biblioteca de replicación (heredada de Multi-User)
│   ├── interface.py       #   interfaz de sesión
│   ├── porcelain.py       #   commit, push, apply, add, rm, lock, unlock,
│   │                      #   purge_orphan_nodes
│   ├── protocol.py        #   protocolo de replicación
│   ├── repository.py      #   grafo de nodos y estados
│   ├── server.py          #   servidor
│   ├── net/               #   transporte de red
│   ├── objects.py, constants.py, exception.py, ttl.py, utils.py
├── deepdiff/, orderly_set/  # dependencias vendorizadas
├── wheels/                # ruedas de dependencias
├── icons/                 # iconos
└── tests/                 # pruebas, incluido test_live_sync.py
```

**Correcciones respecto a la v1**: no existe `libs/replication/`; no existen `replication/session.py` ni `replication/rdp.py` (son `interface.py` y `protocol.py`); `live_sync.py` está en la **raíz**; faltaban `timers.py`, `shared_data.py`, `preferences.py`, `utils.py`, `environment.py`, `bl_types/`, `wheels/` y `tests/`.

**Fuera de la release**: los informes de escena (`*_reporte_*.txt`), `README2.MD`, `test.MD` y los documentos de trabajo se conservan en el repositorio pero se excluyen del paquete distribuible. El manual de usuario real se escribirá cuando la release esté lista.

### 11. DEPENDENCIAS Y HERRAMIENTAS

- **TCP (sockets estándar)**: protocolo de toda la comunicación entre Host y Cliente. **Dos sockets**: el de replicación en el puerto base y el de pose en el puerto base+1 (§5.2). El addon requiere permiso de Red.
- **Cache local por peer**: cada instancia escribe en su propio subdirectorio de cache, resuelto por la vía de extensiones de usuario de Blender. **No se comparte entre Host y Cliente**, tampoco en LAN. Los ficheros se nombran por hash de contenido, se escriben de forma atómica y cada peer purga solo lo que él escribió.
- **Porcelain**: Capa de abstracción sobre el sistema de replicación. Provee operaciones tipo Git: `commit`, `push`, `apply`, `lock`, `unlock`, `rm`, `add`, `purge_orphan_nodes`.
- **Depsgraph**: Blender Dependency Graph. Usado para detectar cambios en la escena mediante el handler `depsgraph_update_post`.
- **Blender Python API (bpy)**: Acceso a datos de la escena, operadores, handlers, keyframes, F-Curves, depsgraph evaluado, etc.
- **Timers de Blender**: `bpy.app.timers.register()` para tareas periódicas no bloqueantes.
- **Handlers de Blender**: `bpy.app.handlers.depsgraph_update_post`, `frame_change_pre`, `undo_post`, `redo_post`, `load_pre`.
- **Depsgraph evaluado**: `evaluated_get(depsgraph)` para leer la pose visible antes de confirmar (§5.9). Es la única vía posible: porcelain lee de `bpy.data`, no del depsgraph evaluado.

### 12. RESUMEN EJECUTIVO

**True Level: Playblast** es un derivado de Multi-User para Blender orientado a un único flujo de trabajo: **revisión de animación sin render**. El Host anima; el Cliente reproduce en bucle y ve la edición del Host en contexto, en el frame exacto en que se está trabajando.

**El núcleo de R1 es la Funcionalidad 2 — Live Sync de Modo Pose.**

- El stream se abre por **selección** de huesos en Modo Pose, no por arrastre. Admite **varios huesos de varias armaduras** a la vez.
- La pose en curso se lee con `evaluated_get(depsgraph)` y viaja por un **canal TCP propio** en el puerto base+1, a 60 Hz, como **canales crudos más el modo de rotación** — no como matriz, para no perder vueltas completas.
- En el Cliente se manifiesta como **un keyframe temporal por hueso**, en el frame del Host, insertado o sobreescrito **dentro del Action existente**. Si la curva F del canal no existe, **se crea** y se registra su procedencia.
- El overlay **persiste mientras el hueso siga seleccionado**: la ventana de decisión del usuario es la selección misma. Al deseleccionar, R1 **restaura siempre**. No quedan fantasmas.
- El hueso **no** se silencia, **no** se congela, **no** se desconecta de su Action, y el timeline del Cliente **no** se pausa.

**Porcelain es la vía de los datos confirmados y es fundamental.** Insertar un keyframe en el Host lo transmite al Cliente **sin necesidad de ningún toggle**. F1 —el armado continuo por datablock— está **implementada pero fuera de soporte** en R1: se conserva intacta, no se retira, y no es un requisito de la release. El objetivo del producto es la transmisión inteligente de datos, no la transmisión masiva de commits.

**Las dos reglas de oro.**

1. **El Cliente es un visor que nunca se detiene y nunca publica.** Puede seleccionar cuanto quiera —la protección es sobre sus datos, no sobre su interacción— pero no inyecta nada por ninguna vía.
2. **Insertar un keyframe expresa intención de conservar la pose; confirmar R/G/S no.** De ahí que R1 restaure al deseleccionar y no retenga el temporal esperando un push que puede no llegar.

**Lo que R1 añade como requisito nuevo y verificable**: la preservación del estado de vista y restricción del Host en todas sus capas (§3.9), el derecho de selección del Cliente (§3.8), la invención de curvas protegida contra la poda (§5.6) y el multi-armadura (§5.11).

**Alcance**: LAN y misma máquina. Sin cuentas, sin configurar el router, sin pasar el .blend por terceros. Internet llega después de la `Phantom Keyframe Update`.

---

### 13. TRAZABILIDAD Y CONTROL DE CAMBIOS

#### 13.1 Changelog v1 → v2.1

Las filas **1–35** corresponden a la v2.0 (16 nov 2026). Las filas **36–40** son la v2.1: aclaraciones de la revisión final del mantenedor, sin cambio de comportamiento ni decisiones nuevas.

| #   | Cambio                                                                                                                                                                                                                                                                                    | Decisión                                     |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| 1   | Cabecera: un solo título de SRS, versión, fecha y estado. Retirado «Documento de Diseño Técnico y Arquitectura» — el diseño es el TDD                                                                                                                                                     | DEC-19                                       |
| 2   | Nueva §0: naturaleza normativa frente a ilustrativa, alcance de R1, documentos relacionados                                                                                                                                                                                               | DEC-20, DEC-16                               |
| 3   | §1.1 pasa a ser la **identidad del producto**; la atribución de Multi-User se traslada a §1.5 sin perder crédito                                                                                                                                                                          | **DEC-21c**                                  |
| 4   | §1.4 reformulado: el sistema de permisos se hereda de Multi-User y **se le añade** la puerta por rol; `can_publish()` es implementación propia. INV-1 explícita                                                                                                                           | DEC-06                                       |
| 5   | §2.2: identidad de UUID declarada **requisito firme** que el código debe cumplir, no aspiración                                                                                                                                                                                           | DEC-07                                       |
| 6   | Nueva §2.3: los **tres modos** de incorporación de escena y la elección del usuario; sin colección de sesión, sin alterar la jerarquía; «Reemplazar» afecta solo a la escena activa                                                                                                       | DEC-08                                       |
| 7   | Nueva §2.4: alcance de red **LAN + misma máquina**; dos puertos; filosofía sin fricción                                                                                                                                                                                                   | DEC-16, DEC-05                               |
| 8   | Nueva §3.8: **el Cliente puede seleccionar** (INV-3); prohibido forzar `hide_select` por propiedad                                                                                                                                                                                        | DEC-11                                       |
| 9   | Nueva §3.9: **preservación del estado de vista** en objeto, colección, View Layer y **DATA de armadura**, con enumeración a nivel de propiedad RNA                                                                                                                                        | DEC-12                                       |
| 10  | §4 marcada **FUERA DE SOPORTE**, conservada íntegra; aclarado que porcelain no queda fuera de soporte y que el nombre `Live Sync` está mal elegido                                                                                                                                        | Decisión del mantenedor (CT-02)              |
| 11  | §5.3: disparador por **selección**; multi-hueso y multi-armadura; canales crudos + `rotation_mode`; `end` por deselección con `is_cancelled` por comparación de pose                                                                                                                      | DEC-01, DEC-03, DEC-04                       |
| 12  | §5.4 reescrito: el overlay vive mientras el hueso esté seleccionado; al deseleccionar R1 restaura siempre; orden `end` → restaurar → aplicar; KI-01 declarado                                                                                                                             | DEC-02 + CT-01                               |
| 13  | §5.6: nuevo **Caso C** (la curva F no existe) como característica núcleo, con regla de protección contra la poda                                                                                                                                                                          | DEC-10                                       |
| 14  | §5.7: esquema real con campos obligatorios, `rotation_mode`, `end.frame` obligatorio, matriz solo por compatibilidad, resurrección de overlay y guard de `"end"` ya visto                                                                                                                 | DEC-04                                       |
| 15  | §5.8: máquina de estados con salida por deselección                                                                                                                                                                                                                                       | DEC-01                                       |
| 16  | §5.2 y §5.10: **socket propio en puerto base+1**, no multiplexado; rutas corregidas                                                                                                                                                                                                       | DEC-05                                       |
| 17  | Nueva §5.11: **multi-armadura en R1**, arreglo solo del emisor, sin cambio de protocolo                                                                                                                                                                                                   | CT-03                                        |
| 18  | §6.1: reformulado «nunca actúan simultáneamente»; §6.2 y §6.3 por fases de selección; §6.4 orden normativo y caso de invalidación; §6.5 reescrito como el flujo normal sin toggles, con KI-08 y su medición                                                                               | DEC-01, DEC-02                               |
| 19  | §7: cláusula PKU marcada como reservada; nuevas prohibiciones 10-14 (no podar con overlay vivo, no forzar `hide_select`, el Cliente no publica por ninguna vía, cache no compartido, sin colección de sesión)                                                                             | DEC-14, DEC-10, DEC-11                       |
| 20  | §8.2: **60 Hz**; §8.3 marcada **orientativa** y corregido el paréntesis contradictorio                                                                                                                                                                                                    | DEC-13                                       |
| 21  | §9.4 multi-hueso normativo; §9.5 **solo conducción directa**, sin crear Action, con la precisión de que el overlay nunca crea datablocks Action; §9.6 enlaza el Caso C; §9.8-9.10 el «volver a copiar el .blend» pasa a último recurso; §9.11 reescrito; nuevos §9.12-9.14                | DEC-03, DEC-09, DEC-10, DEC-07               |
| 22  | §10 y §11: árbol y dependencias reales                                                                                                                                                                                                                                                    | DEC-20                                       |
| 23  | §12 reescrito sobre el alcance real de R1                                                                                                                                                                                                                                                 | DEC-19                                       |
| 24  | **Apéndice no normativo retirado** (≈15% del documento): roadmap duplicado, chat en crudo. El roadmap y el rationale de naming se trasladan a `ROADMAP.md`. La confirmación de INV-3 se rescata antes de borrar y queda como §3.8                                                         | DEC-18 + EM-01                               |
| 25  | Notas de alcance: PKU posterior a R1, auto-keying excluido deliberadamente y previsto como bugfix de la PKU                                                                                                                                                                               | DEC-14, DEC-15                               |
| 26  | Versionado, changelog y esta tabla de trazabilidad                                                                                                                                                                                                                                        | DEC-17                                       |
| 27  | §2.3: **política de identidad y adopción** propagada desde el TDD §9.2 — pase único con informe visible, guard estructural contra homónimos no relacionados, nombre exacto, UUID obligatorio tras la adopción, orden topológico, sin orphanados, sin prescribir transferencia del archivo | D-39, D-39b/c/d/e, D-43, DD-22, DD-33, DD-36 |
| 28  | Nueva **§6.6**: la publicación de lo confirmado no puede depender de que el usuario ejecute otro operador. Requisito de R1, con preservación explícita del gate durante el arrastre real                                                                                                  | **D-28a** (promovida desde la PKU)           |
| 29  | §3.8 y §9.15: **físicas de terceros** — se componen con el keyframe temporal, no lo sustituyen; el flickering con físicas en el mismo hueso que se transforma es comportamiento de vanilla, no defecto del producto                                                                       | Aclaración del mantenedor                    |
| 30  | §2.5: nueva fila **Apply delay** con valor de R1 `0 s`. El default heredado de `1 s` hacía que una instalación limpia aplicara los datos entrantes una vez por segundo                                                                                                                    | **D-48**                                     |
| 31  | §1.4: la puerta por rol debe aplicarse en **todos** los puntos de publicación, incluido el botón manual de commit. Requisito ya presente en §7.12, ahora con punto de intervención                                                                                                        | **D-49**                                     |
| 32  | §2.6: corregido el mecanismo de la purga al conectar. Las contraseñas del preset **sí** son editables, en los diálogos `+` y ✎ (`operators.py:1151-1154`, `:1216-1220`), no en el sidebar. La disposición no cambia                                                                       | Corrección factual                           |
| 33  | §1.4: la puerta por rol se extiende a ✖ y 🔓 con aviso al usuario y **sin cambios de UI**. Verificado que ninguno de los dos depende de `Live Sync` y que ✖ corta la replicación del nodo y su subárbol para todos los peers                                                              | **D-49b**                                    |
| 34  | §2.5: el rango de fotogramas **se hereda al conectar**, no se impone de forma continua. Después el Cliente lo cambia con Blender vanilla y su cambio persiste; solo se sobrescribe si el Host modifica un campo que viaja en el volcado de escena                                         | Corrección del mantenedor, verificada        |
| 35  | §5.4: del keyframe fantasma **sobreviven el keyframe y la curva que lo contiene**, no solo la curva. El punto superviviente es lo que impide que la curva se considere vacía. Es una llave visible en el Dope Sheet del Cliente                                                           | Corrección del mantenedor, verificada        |
| 36  | §1.3 reescrita: el valor principal es la **autorrevisión en vivo del propio animador**; los revisores externos pasan a característica secundaria                                                                                                                                          | Feedback del mantenedor                      |
| 37  | §1.2: Host y Cliente pueden ser la misma persona en dos instancias del mismo equipo; «Cliente» es un rol, no una segunda persona                                                                                                                                                          | Feedback del mantenedor                      |
| 38  | §2.1 y §2.2: la copia del .blend es **opcional**, no un requisito. Alineado con §2.3 eje 1 (copia, escena propia, archivo recién abierto)                                                                                                                                                 | Feedback del mantenedor                      |
| 39  | §2.4: en la misma máquina caben el Host y **uno o varios** Clientes (1 Host + N Clientes), no solo dos instancias                                                                                                                                                                         | Feedback del mantenedor                      |
| 40  | §2.4 y §0.2: las VPN de terceros se descartan **como requisito del producto**, no solo para R1; el transporte por internet queda sin decidir                                                                                                                                              | D-38, DD-10                                  |

#### 13.2 Correcciones factuales aplicadas

Erratas y datos desactualizados verificados contra el código. No requirieron decisión.

| Ubicación         | Corregido                                                                                                                             |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Todo el documento | «True Level: **Playback**» → «True Level: **Playblast**» en los cuatro sitios donde el SRS del repositorio tenía el nombre equivocado |
| §5.3              | «canal **TPC**» → TCP                                                                                                                 |
| §2.1              | «por el **usario**» → usuario; «Se **usara**» → usará                                                                                 |
| §5.7              | «Formato **de** ilustrativo» → formato ilustrativo                                                                                    |
| §8.3              | «el **quele** permite» → el que le permite                                                                                            |
| §9.10             | «El usuario **tendra**» → tendrá                                                                                                      |
| §4.5              | `**init**.py o properties.py` → `preferences.py`; `multiuser_global_live_sync` → `WindowManager.session.global_live_sync`             |
| §4.5, §4.8, §10   | `draw_property()` → `draw_commit_control()` para el toggle                                                                            |
| §5.10, §10        | `libs/replication/live_sync.py` → `live_sync.py` en la raíz; `replication/session.py` y `rdp.py` → `interface.py` y `protocol.py`     |
| §5.7              | `end` «puede incluir frame» → **obligatorio**                                                                                         |
| §8.2              | «30-60 FPS» → **60 Hz**                                                                                                               |
| §4.4              | «10 cambios → 10 commit/push» matizado por el gate de arrastre y los filtros del stream                                               |
| §4.2              | pseudocódigo marcado como **instantánea del addon vanilla**                                                                           |
| §3.9              | `ViewLayerCollection` → **`bpy.types.LayerCollection`**; los `RESTRICT_*` son iconos del outliner, no propiedades                     |
| Todo el fichero   | finales de línea normalizados a **LF**                                                                                                |

**Verificación previa**: no queda ningún residuo de ZMQ/ZeroMQ en el documento. El precedente que motivó esta limpieza —una decisión ya tomada que el SRS no reflejaba— queda cubierto por §0.1 y por esta tabla.

#### 13.3 Restricciones protegidas sin cambio

Se verificaron en el código y se conservan **intactas**. La limpieza no las ha debilitado:

| Sección | Restricción                                                                           |
| ------- | ------------------------------------------------------------------------------------- |
| §3.1    | No pausar el timeline del Cliente; ni `is_animation_playing = False` ni `frame_set()` |
| §3.2    | No `fcurve.mute`; no congelar ni desconectar el hueso                                 |
| §3.3    | No Action swapping; no crear Acción vacía; no desconectar el Action original          |
| §3.4    | Un keyframe temporal por hueso; se actualiza, no se recrea                            |
| §3.6    | `insert(..., options={'FAST'})`; canal directo sin porcelain                          |
| §3.7    | Nada de `threading` sobre `bpy.data`; timers y handlers nativos                       |
| §5.9    | Lectura por `evaluated_get(depsgraph)`                                                |
| §7.8    | No commit/push por porcelain durante el arrastre                                      |
| §9.3    | Cambio de frame a mitad de edición: mover el overlay y restaurar el frame anterior    |
| §6.4    | Orden `end` → limpiar → aplicar datos del repositorio                                 |

#### 13.4 Pendientes que este SRS no cierra

| Punto                                                                             | Dónde se resuelve                                                                                                                             |
| --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `README.MD` es un archivo de legado y se retira del repositorio                   | Lo ejecuta el mantenedor (DEC-17)                                                                                                             |
| VP-05: el ojo del outliner (`H`) es por View Layer                                | Verificación en Blender 5.2.2, TDD §15.4                                                                                                      |
| VP-06 / VP-07: identidad del hueso y selección multi-objeto                       | Verificación previa al arreglo de §5.11                                                                                                       |
| VP-08: llegada del Action con los toggles desarmados                              | Medición que decide si KI-08 se corrige o se cierra                                                                                           |
| **D-28a** · gate de publicación en R1                                             | TDD §12.4, promovido desde la PKU. Requisito en §6.6                                                                                          |
| **Rutas de purga heredadas que contradicen §2.3**                                 | **Decidido**: se retiran `operators.py:345-346` al implantar D-40a. Sin cambios de UI. TDD §17.K.8 H-01 y §17.K.9 DD-38                       |
| **VP-11** · asimetría de purga `localhost` frente a LAN                           | Verificación en Blender 5.2.2, TDD §15.4                                                                                                      |
| **VP-09** · físicas del Host y canales transmitidos                               | Verificación en TDD §15.4                                                                                                                     |
| **VP-10** · coste real de publicar un `Object` durante el stream                  | Medición pedida por el mantenedor antes de resolver H-04 (TDD §17.K.8)                                                                        |
| **Timer de anotación**: `commit`/`push` del lápiz de anotación sin puerta por rol | Violación de INV-1 acotada a las anotaciones. Fuera de D-49/D-49b. TDD §17.L.8. Decidir si las anotaciones entran en el flujo soportado de R1 |
| **Migración de la ruta de caché**                                                 | Quien ya tenía el addon conserva la preferencia apuntando a `...\multi_user\cache`. TDD §17.L.7. Sin cambio en R1                             |
| **Precedencia DD-03 ↔ selecionabilidad por propiedad**                            | `timers.py:312-318` reescribe `hide_select` cada 0,1 s. Regla obligatoria al implantar DD-03. TDD §17.L.4                                     |
| Manual de usuario                                                                 | Cuando la release esté lista                                                                                                                  |

---

_Fin del SRS v2.1. Documento aprobado por el mantenedor tras las 21 decisiones de `Decisiones_SRS-v2.md` y las 10 contradicciones resueltas en `Contradicciones_decisiones.md`. El apéndice no normativo de la v1 se ha trasladado a `ROADMAP.md`._
