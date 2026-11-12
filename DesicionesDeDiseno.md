Arreglaste el problema de las imágenes, te confirmo que si, las imágenes estaban dentro del .blend.
A partir de ahora no escribas código, necesitamos tomar algunas decisiones de diseño y necesito que me ayudes con ello. Surgieron comportamientos que no estaban contemplados, y antes de decidir si son bugs o features necesitamos tomar dichas decisiones de diseño.

Diario de desarrollo 07/11/2026

Surgieron 2 escenarios sobre los qué tomar una decisión:

#### Diccionario

- Keyframe temporal (overlay): el que el cliente escribe en su Action mientras el host arrastra/selecciona.
- Keyframe fantasma: el keyframe temporal que sobrevive después de cambiar de hueso.
- Keyframe real: el que existe en el Dope Sheet del host y viaja por porcelain.

### `Escenario 1:` d

Este es un "bug" que me encontré pero lo quiero convertir en feature porque si sabes que existe y entiendes lo que haces, es algo muy poderoso y util en animacion:
El host y el cliente están conectados correctamente.
El host está posicionado en cualquier frame_current (frame 30 por ejemplo)
El host selecciona un hueso, se genera el keyframe temporal en el cliente, el host transforma el hueso, la transformación se transmite al cliente (hasta aquí todo correcto y esperado), el comportamiento no contemplado y que no se si tratarlo como bug o feature surge a partir de aqui:  
Con la transformación transmitida, el keyframe temporal correctamente colocado en el cliente, existen múltiples casos de uso que el usuario puede tomar a partir de ahí;

1. `Flujo 1.1 (Flujo original / Esperado)`: El usuario sigue en el mismo hueso (hueso.001, importante tener en cuenta que el usuario puede seleccionar multiples huesos a la vez [hueso.001, hueso.004, hueso.032, etc] pero para efectos practicos solo usare el hueso.001 en esta explicacion del caso 1) decide que ahora quiere pasar del frame 30 a cualquier otro frame, en este caso, el addon funciona como lo esperado, el keyframe temporal en el cliente sigue al frame_current del host así que el keyframe en el frame 30 del cliente se limpia o vuelve a su estado original según si existía un keyframe o no (esperado), hasta aquí todo correcto, el usuario puede volver a transformar ese hueso (hueso.001) en otro frame y volver a repetir, cambiar frame_current y transformar el mismo hueso.001 e insertar o confirmar transformaciones. Este es el flujo que se tenía en mente originalmente al idear este addon.

2. `Flujo 1.2 (Caso de uso no contemplado, convertir bug en feature)`: El cliente tiene el keyframe temporal correctamente de la transformación, ahora, el host selecciona otro hueso (hueso.002 o varios huesos) (y aquí es donde surge el comportamiento no contemplado originalmente y que podría ser tanto un bug como una feature, si no sabes que ese comportamiento existe pareciera un bug, pero alguien que sabe lo que hace y conoce este comportamiento anómalo, lo podría ver como algo muy útil, como me pasó a mi, primero lo ví como bug pero luego de ver cómo funciona lo ví útil), al seleccionar el otro hueso (hueso.002), en el primer hueso (hueso.001), en el cliente, el keyframe temporal del hueso.001 no desaparece, queda como keyframe fantasma (este es el bug/feature), pero a veces al seleccionar otro hueso si desaparece ese keyframe fantasma, para que el bug sea consistente y que el keyframe temporal no desaparezca, el usuario, al cambiar de hueso, hueso.001 a hueso.002, tiene que primero, que transformar el hueso.001, los keyframes fantasmas se quedan si la transformacion es confirmada, si cancelas la transformacion (ESC o click derecho) los keyframes fantasmas no aparecen, entonces el flujo es:  
   `transformar -> confirmar transformacion -> seleccionar otro hueso -> volver al hueso -> ver la aparicion del keyframe fantasma`.
   Esto se puede concatenar multiples veces acumulando multiples frames fantasma.
   Los keyframes fantasma desaparecen si el host simplemente inserta un keyframe en donde sea.

Como dije antes, tiene potencial de ser una feature valiosa, ya que el cliente evalua los keyframes fantasma sin que el host tenga keyframes insertados ni tenga sucio el Dope Sheet, asi el animador puede ver como seria la animacion sin necesidad de ensuciar el Dope Sheet, mi vision es que puedas activar el modo `Phantom keyframe mode` y que en el host puedas insertar los keyframes fantasmas al cliente (para evitar que el usuario tenga que estar cambiando de hueso cada vez que quiera insertar uno), y que tenga un boton para confirmar los keyframes fantasma, para ello el host tiene que guardar los keyframes fantasma en memoria y no en el Dope Sheet, para que el usuario pueda activar y desactivar el `Phantom keyframe mode` cuando quiera, y tambien para evitar problemas futuros. Como animador lo veo muy valioso.

### `Escenario 2:`

Aqui me encontré con un fallo importante al testear en diferentes .blends, y hay que tomar una desicion de diseño y de arquitectura importante. Al trabajar con .blends tipo tests, pequeños, con pocos huesos, con 2 armaduras cada una controlando su propio objeto, el resultado de las pruebas del addon es el esperado y no salen a la luz bugs tan rapido como si lo hacen en .blends mas pesados con personajes riggeados, imagenes, o por ejemplo una armadura que controla 2 objetos (el modelo original y el modelo low poli que ya existia en el .blend original), te voy a explicar el comportamiento actual para que podamos tomar una desicion.

#### Diccionario de casos

|                     | Cliente: copia.blend | Cliente: .blend vacio |
| ------------------- | -------------------- | --------------------- |
| .blends test ligero | Caso 1A              | Caso 1B               |
| .blends test pesado | Caso 2A              | Caso 2B               |
| .blends reales      | Caso 3A              | Caso 3B               |

#### Set up

Host y cliente corren en la misma máquina y es posible que compartan el mismo `cache_directory`, desconozco si es asi o no.
El addon original Multi-User dice que si puede funcionar a traves de internet, esta modificacion debería tener en cuenta que se deberia poder usar en la misma maquina y tambien a traves de distintas redes y maquinas.

`Cliente: copia.blend:` El cliente usa la copia directa del .blend original, desde ese .blend, el cliente se conecta.  
`Cliente: .blend vacio:` El cliente abre Blender y borra manualmente todos los objetos, luego se conecta.

`.blends test ligero:` El .blend tiene una armadura con cientos de huesos y controla un cubo, se considera ligero porque, a pesar de que anteriormente al tener cientos de huesos, existia un bug que alentaba todo blender al hostear, ahora el addon puede manejar cientos de huesos en las armaduras.  
`.blends test pesado:` Se considera pesado porque, a pesar de que las armaduras tienen solo 6 huesos cada uno y son solo 2 armaduras controlando su objeto, al riggear el objeto, se duplicó el rig, y al objeto duplicado se le aplico el modificador subdivision multiples veces, llegando a tener `1966137 caras`, `1966112 vertices` y `3932226` triangulos. Ademas se les aplicó una textura de imagen ya que anteriormente representó problemas, aunque dependiendo de los casos a continuacion, siguen habiendo problemas.  
`.blends reales:` Este es el motivo que me obligó a escribir este documento, en este .blend existe un personaje que, antes de escribir este addon, ya estaba animando, tiene cientos de keyframes, un personaje riggeado, otros objetos riggeados, el rig del personaje no controla un solo modelo, controla multiples objetos como normalmente se hace al riggear, los ojos son un objeto, el cabello es otro objeto, las cejas otro, el cuerpo otro, etc. Ademas de que obviamente existe presencia de cientos de imagenes para texturas, normales, especularidad, etc. Y es importante mencionar que, el personaje tiene 2 versiones, uno low poly y el original, ambos controlados por el mismo rig (aqui se generan ciertos bugs).

#### Casos

En esta seccion describiré detalladamente el comportamiento segun cada caso, y señalaré el nivel de prioridad de 0 a 5, donde 5 es máxima prioridad y atención.

`Caso 1A` `Nivel de prioridad 0`  
El cliente se abre en la copia del .blend original [Host huesos.blend, Cliente huesos - copia.blend] cuando se conecta el cliente, al ser copias exactas, provoca ciertos fallos minimos, no interfieren con el funcionamiento correcto del addon (por lo menos a simple vista). Por ejemplo:
Mientras que Host tiene una coleccion llamada `Collection`, al conectarse el cliente, la coleccion del cliente en vez de heredar el mismo nombre `Collection` hereda el nombre `Collection.001`, solo pasa con ciertos objetos y datos dentro de los objetos, la mejor manera de ilustrarlo es a través de una tabla.

| Host             | Cliente              | Detalles |
| ---------------- | -------------------- | -------- |
| Collection       | Collection.001       |          |
| Camera           | Camera               |          |
| Esqueleto        | Esqueleto            |          |
| Cube             | Cube                 |          |
| Esqueleto_acción | Esqueleto_acción.001 |          |
| Hueso.005        | Hueso.005            |          |

El nivel de prioridad es `0`, porque no representa un fallo crítico y se menciona con fines de documentacion.

`Caso 1B` `Nivel de prioridad 2`  
El cliente se conecta desde un .blend limpio, con todos los objetos eliminados, al conectarse recibe los objetos del host, hasta aqui todo correcto, incluso no hay fricción con los nombres.

| Host             | Cliente          | Detalles |
| ---------------- | ---------------- | -------- |
| Collection       | Collection       |          |
| Camera           | Camera           |          |
| Esqueleto        | Esqueleto        |          |
| Cube             | Cube             |          |
| Esqueleto_acción | Esqueleto_acción |          |
| Hueso.005        | Hueso.005        |          |

El nivel de prioridad es `2`, porque a pesar de no haber ningun fallo aparente, se le debe poner atención tiene que ver con el `Caso XX` y se requieren tomar decisiones de diseño importantes.

`Caso 2A` `Nivel de prioridad 5`  
El host abre la sesion y se tarda un poco, es normal al tener un objeto con `3000000` de vertices, desconozco si en un futuro, el que tarde tanto o tener tantos datos que cargar al inicio pueda generar errores, pero de momento no me he topado con un problema de ese tipo. El comportamiento es similar al `Caso 1A`, solo que con la diferencia de que existen 2 armaduras y 2 objetos.

Tabla del `Caso 2A` test 1

| Host                 | Cliente              | Detalles                                       |
| -------------------- | -------------------- | ---------------------------------------------- |
| Collection           | Collection.001       |                                                |
| Camera               | Camera               |                                                |
| Esqueleto            | Esqueleto            |                                                |
| Cube                 | Cube                 |                                                |
| Esqueleto_acción     | Esqueleto_acción.003 | Si, por alguna razon pasa a llamarse asi       |
| Hueso.005            | Hueso.005            |                                                |
| Esqueleto.001        | Esqueleto.001        | Este es el rig duplicado                       |
| Cube.001             | Cube.001             | Cubo duplicado pero con `3000000`de triángulos |
| Esqueleto_acción.001 | Esqueleto_acción.002 |                                                |
| Hueso.005            | Hueso.005            |                                                |

El nivel de prioridad es `5`, los cambios de nombre no generan problemas con el funcionamiento y objetos pesados no generan friccion con la funcionalidad del addon.  
Pero, para el `Caso 2A` hice 2 tests, el primero sin texturas, y el segundo test con texturas para el cubo y empaquetadas en el .blend.  
En el segundo test surge un comportamiento anómalo, en el .blend original, la textura del cubo se llama Sin_nombre (hasta aqui todo correcto), ahora, el cliente tiene la copia del .blend y todavia no se ha conectado, asi que la informacion es la misma. Cuando el cliente se conecta, unos segundos despues el Host se congela por unos segundos y la textura del Host pasa de llamarse `Sin_nombre` a `Sin_nombre.001`, ademas, en el host si se queda un solo material llamado `Material`, pero en el cliente `Material` se queda asignado a `Cube.001 ` con la textura `Sin_nombre`, mientras que a Cube se le asigna un nuevo material creado solamente en el cliente `Material.001` con otra textura `Sin_nombre.001`.

| .blend original | Material | textura    |
| --------------- | -------- | ---------- |
| Cube            | Material | Sin_nombre |
| Cube.001        | Material | Sin_nombre |

Segundos despues de que el cliente se conecte

| .blend Host | Material | textura        |
| ----------- | -------- | -------------- |
| Cube        | Material | Sin_nombre.001 |
| Cube.001    | Material | Sin_nombre.001 |

`Sin_nombre.001` reemplaza a `Sin_nombre` en el Host.

| .blend Cliente | Material | textura    |
| -------------- | -------- | ---------- |
| Cube           | Material | Sin_nombre |
| Cube.001       | Material | Sin_nombre |

`Material.001` junto con `Sin_nombre.001` en el cliente quedan huerfanos y no deberian haberse generado desde un principio, en versiones anteriores surgio este mismo problema con las armaduras, y otros objetos y se corrigió como se demuestran en las tablas.

Tabla del `Caso 2A` test 2

| Host                 | Cliente              | Detalles                                       |
| -------------------- | -------------------- | ---------------------------------------------- |
| Collection           | Collection.001       |                                                |
| Camera               | Camera               |                                                |
| Esqueleto            | Esqueleto            |                                                |
| Cube                 | Cube                 |                                                |
| Esqueleto_acción     | Esqueleto_acción.003 | Si, por alguna razon pasa a llamarse asi       |
| Hueso.005            | Hueso.005            |                                                |
| Esqueleto.001        | Esqueleto.001        | Este es el rig duplicado                       |
| Cube.001             | Cube.001             | Cubo duplicado pero con `3000000`de triángulos |
| Esqueleto_acción.001 | Esqueleto_acción.002 |                                                |
| Hueso.005            | Hueso.005            |                                                |

Logs relacionados con el freeze del Host y la creacion de `Sin_nombre.001` y el reemplazo de `Sin_nombre` en el Host.

`Log Host`  
05:27:35 CLIENT DEBUG Nothing to commit on node 5b397789-8f4e-4463-8333-aa73af273770  
05:27:35 CLIENT DEBUG Nothing to push  
05:27:35 CLIENT DEBUG Evaluate Cube.001  
05:27:35 CLIENT DEBUG Clearing 2 dependencies.  
05:27:35 CLIENT DEBUG found dependencies: [bpy.data.meshes['Cube.001'], bpy.data.objects['Esqueleto.001']]  
05:27:41 CLIENT DEBUG Nothing to commit on node bdc7633c-07c2-4aa3-b26a-f173995719bc  
05:27:41 CLIENT DEBUG Nothing to push  
05:27:41 CLIENT DEBUG Evaluate Material  
05:27:41 CLIENT DEBUG Clearing 1 dependencies.  
05:27:41 CLIENT DEBUG found dependencies: [bpy.data.images['Sin_nombre.001']]  
05:27:41 CLIENT DEBUG Committed 14a15252-e0c0-4856-abe0-d6d38aaf38ee  
05:27:41 CLIENT INFO Live Sync: committed Material 14a15252-e0c0-4856-abe0-d6d38aaf38ee (state 4 -> 1)  
05:27:41 CLIENT DEBUG Push 14a15252-e0c0-4856-abe0-d6d38aaf38ee  
05:27:41 CLIENT DEBUG Evaluate Cube  
05:27:41 CLIENT DEBUG Clearing 1 dependencies.  
05:27:41 CLIENT DEBUG found dependencies: [bpy.data.materials['Material']]  
05:27:41 SERVER DEBUG Committed 14a15252-e0c0-4856-abe0-d6d38aaf38ee  
05:27:41 CLIENT DEBUG Nothing to commit on node 014510e7-f540-4e08-82a4-7f2112f86e3b  
05:27:41 CLIENT DEBUG Nothing to push  
05:27:41 CLIENT DEBUG Evaluate Cube  
05:27:41 CLIENT DEBUG Clearing 2 dependencies.  
05:27:41 CLIENT DEBUG found dependencies: [bpy.data.meshes['Cube'], bpy.data.objects['Esqueleto']]  
05:27:41 CLIENT DEBUG Nothing to commit on node 716a9cf3-0100-4c27-8332-ad93ff79bc24  
05:27:41 CLIENT DEBUG Nothing to push  
05:27:41 CLIENT DEBUG found dependencies: []  
05:27:41 CLIENT DEBUG Nothing to push  
05:27:41 CLIENT DEBUG Clearing 1 dependencies.  
05:27:41 CLIENT DEBUG found dependencies: [WindowsPath('C:/Users/israe/AppData/Roaming/Blender Foundation/Blender/5.2/extensions/.user/user_default/multi_user/cache/Sin_nombre.png')]  
05:27:41 CLIENT DEBUG Nothing to commit on node dd57c2a7-070b-4a01-b618-91910bdef7b7  
05:27:41 CLIENT DEBUG Nothing to push

`Log cliente`  
05:27:30 CLIENT DEBUG Nothing to push  
05:27:34 CLIENT INFO Live Sync: depsgraph batch Material:Material.001, Mesh:Cube, Object:Camera, Object:Cube, Object:Cube.001, Object:Esqueleto, Object:Esqueleto.001, Object:Light, Scene:Scene, ShaderNodeTree:Shader Nodetree
05:27:34 CLIENT DEBUG Clearing 1 dependencies.  
05:27:34 CLIENT DEBUG found dependencies: [WindowsPath('C:/Users/israe/AppData/Roaming/Blender Foundation/Blender/5.2/extensions/.user/user_default/multi_user/cache/Sin_nombre.png')]  
05:27:34 CLIENT DEBUG Nothing to commit on node dd57c2a7-070b-4a01-b618-91910bdef7b7  
05:27:34 CLIENT DEBUG Nothing to push  
05:27:34 CLIENT DEBUG found dependencies: []  
05:27:34 CLIENT DEBUG Nothing to push  
05:27:41 CLIENT DEBUG Committed 14a15252-e0c0-4856-abe0-d6d38aaf38ee  
05:27:41 CLIENT DEBUG Applying 14a15252-e0c0-4856-abe0-d6d38aaf38ee  
05:27:41 CLIENT DEBUG Applying 744d4b4d-5165-4865-b8a2-539bd0c63be3  
05:27:41 CLIENT WARNING C:\Users\israe\AppData\Roaming\Blender Foundation\Blender\5.2\extensions\.user\user_default\multi_user\cache\Sin_nombre.png doesn't exist, skipping  
05:27:41 CLIENT DEBUG Applying dd57c2a7-070b-4a01-b618-91910bdef7b7  
05:27:41 CLIENT INFO Live Sync: depsgraph batch Image:Sin_nombre.001, Material:Material.001, Mesh:Cube, Object:Camera, Object:Cube, Object:Cube.001, Object:Esqueleto, Object:Esqueleto.001, Object:Light, Scene:Scene, ShaderNodeTree:Shader Nodetree
05:27:41 CLIENT DEBUG Ignoring distant update of ['14a15252-e0c0-4856-abe0-d6d38aaf38ee']  
05:27:41 CLIENT DEBUG Clearing 1 dependencies.  
05:27:41 CLIENT DEBUG found dependencies: [WindowsPath('C:/Users/israe/AppData/Roaming/Blender Foundation/Blender/5.2/extensions/.user/user_default/multi_user/cache/Sin_nombre.png')]  
05:27:41 CLIENT DEBUG Nothing to commit on node dd57c2a7-070b-4a01-b618-91910bdef7b7  
05:27:41 CLIENT DEBUG Nothing to push  
05:27:41 CLIENT DEBUG found dependencies: []  
05:27:41 CLIENT DEBUG Nothing to push

`Caso 2B` `Nivel de prioridad 2`  
Para este test al entrar al .blend del cliente presione A y X para eliminar los objetos del viewport, pero hay que tener en cuenta que pueden quedar datos en el .blend, como el `Material` que crea blender automaticamente al abrir blender, al igual que el `Cubo`, `luz`, `camara` etc.

El Host inicia la sesion, incluye las armaduras y la textura imagen de los cubos, y al parecer todo funciona perfectamente, pero el problema de que se freezea el Host y su `textura` `Sin_nombre` es reemplazada por `Sin_nombre.001` persiste, aparecio cuando quite la imagen de la textura y la volvi a poner, este es el log, incluso en el log se puede ver que se freezeo

03:08:36 CLIENT INFO Live Sync: depsgraph batch Material:Material, Mesh:Cube, Mesh:Cube.001, Object:Camera, Object:Cube, Object:Cube.001, Object:Esqueleto, Object:Esqueleto.001, Object:Light, Scene:Scene, ShaderNodeTree:Shader Nodetree  
03:08:47 CLIENT INFO Live Sync: committed Material 38253d2e-639d-4549-bfd8-ad0d99f8feb4 (state 4 -> 1)  
03:09:11 CLIENT WARNING Can stamp uuid on <class 'pathlib.\_local.WindowsPath'>  
03:09:11 CLIENT INFO Extracting file metadata  
03:09:11 CLIENT INFO Reading Sin_nombre.png content: 250106 bytes  
03:09:11 CLIENT INFO Live Sync: committed Material 38253d2e-639d-4549-bfd8-ad0d99f8feb4 (state 4 -> 1)  
03:09:11 CLIENT INFO Live Sync: depsgraph batch Image:Sin_nombre.002, Material:Material, Mesh:Cube, Mesh:Cube.001, Object:Cube, Object:Cube.001, ShaderNodeTree:Shader Nodetree  
03:09:22 CLIENT INFO Live Sync: committed Material 38253d2e-639d-4549-bfd8-ad0d99f8feb4 (state 4 -> 1)

Si lo quito y lo vuelvo a poner, se incrementa el indice Sin_nombre.001 Sin_nombre.002 Sin_nombre.003

Descubrimiento: La causa de que en el cliente se creaba `Material.001` con los datos de Host mientras que `Material` tambien existe, es porque `Material` es el que crea el .blend cuando se abre. Entonces si en un .blend real no existe un material que se llame `Material`, no deberia haber fallos de nombres, ademas, aunque entre el Host y Cliente los nombres son distintos, siguen compartiendo los mismos datos, asi que no hay problemas.

`Caso 3A` `Nivel de prioridad 4`  
Utilice un script para extraer los datos del `Archivo de Blender` en un .txt.  
LA0206 - Buttfuck - 5.2.0_reporte_archivo_original.txt
LA0206 - Buttfuck - 5.2.0_reporte_host_instante.txt
LA0206 - Buttfuck - 5.2.0 - copia_reporte_cliente_conectado.txt
Primero reporte el archivo original, sin intervencion del addon . Luego reporte en otro .txt el archivo instantes despues de darle al boton de hostear para ver que cambiaba, el siguiente link es el commit que delata los cambios:  
https://github.com/Devangel-gonzalez/blender-sync/commit/e9121508dbdb3e77158043c36424d7cdd8466e28  
LA0206 - Buttfuck - 5.2.0_reporte_archivo_original.txt
LA0206 - Buttfuck - 5.2.0_reporte_host_instante.txt

Despues en otra instancia de Blender y en una copia del archivo, el cliente se conectó, este es el reporte del cliente despues de conectarse:

LA0206 - Buttfuck - 5.2.0 - copia_reporte_cliente_conectado.txt

Y por ultimo este es el reporte del Host despues de que el cliente se conectase:

`Caso 3B` `Nivel de prioridad 4`  
El cliente

### Desiciones de diseño
