Arreglaste el problema de las imágenes, te confirmo que si, las imágenes estaban dentro del .blend.
A partir de ahora no escribas código, necesitamos tomar algunas decisiones de diseño y necesito que me ayudes con ello. Surgieron comportamientos que no estaban contemplados, y antes de decidir si son bugs o features necesitamos tomar dichas decisiones de diseño.

Surgieron 2 escenarios sobre los qué tomar una decisión:
Escenario 1, este es un "bug" que me encontré pero lo quiero convertir en feature porque si sabes que existe y entiendes lo que haces, es algo muy poderoso y util en animacion:
El host y el cliente están conectados correctamente.
El host está posicionado en cualquier frame_current (frame 30 por ejemplo)
El host selecciona un hueso, se genera el keyframe temporal en el cliente, el host transforma el hueso, la transformación se transmite al cliente (hasta aquí todo correcto y esperado), el comportamiento no contemplado y que no se si tratarlo como bug o feature surge apartir de aqui:  
Con la transformación transmitida, el keyframe temporal correctamente colocado en el cliente, existen múltiples casos de uso que el usuario puede tomar a partir de ahí;

1. `Caso 1 (Flujo original / Esperado)`: El usuario sigue en el mismo hueso y decide que ahora quiere pasar del frame 30 a cualquier otro frame, en este caso, el addon funciona como lo esperado, el keyframe temporal en el cliente sigue al frame_curre del host así que el keyframe en el frame 30 del cliente se limpia o vuelve a su estado original según si existía un keyframe o no (esperado), hasta aquí todo correcto, el usuario puede volver a transformar ese hueso en otro frame y volver a repetir, cambiar frame_current y transformar el mismo hueso e insertar o confirmar transformaciones. Este es el flujo que se tenía en mente originalmente al idear este addon.

2. `Caso 2 (Caso de uso no contemplado, convertir bug en feature)`: El cliente tiene el keyframe temporal correctamente de la transformación, ahora, el cliente selecciona otro hueso (y aquí es donde surge el comportamiento no contemplado originalmente y que podría ser tanto un bug o una feature, si no sabes que ese comportamiento existe pareciera un bug, pero alguien que sabe lo que hace y conoce este comportamiento anómalo lo podría ver como algo muy útil, como me pasó a mi, primero lo ví como bug pero luego de ver cómo funciona lo ví útil), al seleccionar otro hueso, en el primer hueso, llamemosle hueso.1, en el cliente se guarda el keyframe temporal del hueso uno

En el cliente, el hueso.1 guarda el keyframe temporal, y el surge un nuevo keyframe temporal en el hueso.2 (importante aclarar que en el hueso.1 el keyframe temporal se queda como si hubieses insertado keyframes, pero el keyframe temporal en vivo si deja de transmitirse al hueso.1), así que si cambiaste de hueso en el frame 30, en el hueso.1 se quedó un keyframe insertado, si
