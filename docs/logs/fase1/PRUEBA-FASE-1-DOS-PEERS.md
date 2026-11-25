# PRUEBA DE FASE 1 · DOS PEERS — protocolo y hoja de resultados

**Documento no normativo.** Ejecuta el mantenedor. Deriva de `ORDEN-FASE-1.md` §3.2/§3.3 y del TDD v4.7 (AC-07 / T-07).
**Código bajo prueba:** HEAD `0a0e511` (P0–P8 + P11 + tests AC-F1).
**Punto de retorno:** `fase-01-inicio` → `5e05db9`. Línea base: `r1-base` → `f00d655`.

---

## 0. Estado automático ya medido

| Ejecución | Commit    | Resultado        | Fallos        |
| --------- | --------- | ---------------- | ------------- |
| Base      | `f00d655` | `129/131` · FAIL | 2 del overlay |
| HEAD      | `0a0e511` | `164/166` · FAIL | los mismos 2  |

Comparación literal de etiquetas: ninguna check de la base desaparece; las 35 nuevas son todas `AC-F1-…`.
**Sin regresión automática.** Los 2 fallos son deuda preexistente del overlay al confirmar (`live_sync.py`), fuera de fase 1.

---

## 1. Preparación

1. Usa un **`.blend` desechable**, nunca animación real.
2. Dos instancias de Blender en la misma máquina: una **Host**, otra **Cliente**.
3. En **ambas**: `Pose Live Sync` **ON**, `Live Sync` (master) **OFF**.
4. En el Cliente, ten a mano un objeto replicado, un Action del que dependa otro nodo, y el panel con los botones commit / ✖ / 🔓.
5. Abre la consola de sistema en ambos (`Window > Toggle System Console`) para ver el tráfico.
6. Anota la versión de Blender y que usas la extensión/copia del repo, no otra instalación distinta.

**Regla de registro:** si una fila «antes» no reproduce el defecto esperado, **se anota tal cual**. No se fuerza la observación.

---

1. .blend desechable con trabajo real, personaje, animacion, etc.
2. La instancia del cliente se conecta desde un .blend fresco
3. OK
4. OK
5. OK
6. 5.2.0

## 2. Bloque A · El Cliente no publica (AC-F1-7…11)

Se ejecuta **en el Cliente**, mirando lo que recibe el Host.

| ID           | Acción en el Cliente                           | Después del parche (obligatorio)                                     | Resultado | Notas                                                                                                                                                               |
| ------------ | ---------------------------------------------- | -------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AC-F1-7**  | Pulsar el botón de **commit** de un datablock  | Aviso al Cliente; el Host **no recibe nada**                         | ☐ PASS    |                                                                                                                                                                     |
| **AC-F1-8**  | Pulsar **✖** sobre un objeto                   | Aviso; el objeto **sigue replicándose**; no llega `Delete`           | ☐ PASS    |                                                                                                                                                                     |
| **AC-F1-9**  | Pulsar **🔓**                                  | Aviso inmediato; **el diálogo no se abre**; el propietario no cambia | ☐ PASS    |                                                                                                                                                                     |
| **AC-F1-10** | **Crear una escena nueva**                     | El Host **no recibe** la escena                                      | ☐ PASS    |                                                                                                                                                                     |
| **AC-F1-11** | Editar un **Action** del que otro nodo depende | **No se publica** por `update_external_dependencies`                 | ☐ PASS    | Si pasó el test al mover otro keyframe, pero al mover el keyframe temporal en el hueso que el host estaba trabajando, blender crasheó, log en las notas adicionales |

**Texto de aviso esperado:**

```
Only the Host publishes changes; this session is a viewer.
```

**Notas adicionales:**  
Mover un objeto en el cliente genera esta salida en la consola:  
Objeto `rig` de tipo `Armadura`  
03:59:54 CLIENT INFO Live Sync: depsgraph batch Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low

Objetos `Piso` y `this.001 Low` de tipo `Mesh`  
04:02:43 CLIENT INFO Live Sync: depsgraph batch Object:Piso, Object:this.001 Low

**AC-F1-11**
04:15:54 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:54 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:54 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: depsgraph batch Armature:rig, Object:rig, Scene:Scene
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:55 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:55 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:56 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:56 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:57 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:57 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:58 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:58 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:15:59 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:15:59 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:00 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:00 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:01 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:01 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: depsgraph batch Action:rigAction, Key:Key.004, Mesh:mesh_27.004, Object:Camara con modificaciones, Object:Eyebrows, Object:Eyebrows Low, Object:Eyelashes, Object:Eyelashes Low, Object:Eyes , Object:Eyes Low, Object:Hair, Object:Hair Low, Object:Lilia, Object:Lilia Low, Object:POV, Object:Teeth, Object:Teeth Low, Object:rig, Object:this, Object:this Low, Object:this.001, Object:this.001 Low
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:02 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:02 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: viewer applied 1/1 message(s) [update=1]
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
04:16:03 CLIENT INFO Live Sync: Action 1da26c2c-d8e2-4199-a3e1-312af7cb0ebe rejected - owner is 'COMMON', not 'user_Ijj4BE' and not common
Error : EXCEPTION_ACCESS_VIOLATION
Address : 0x00007FFA03AD5BB6
Module : ntdll.dll
Thread : 00004304

**Cómo confirmar «el Host no recibe nada»:** ausencia de commit/push entrante en el log del Host y ausencia de cambio en el datablock correspondiente. Si dudas, repite la acción y observa el log en ventana de 5 s.

---

## 3. Bloque B · Regresión — lo que NO puede cambiar (RG-01…06)

**Este es el bloque más importante de la fase.** Cualquier FAIL aquí → revertir y registrar.

| ID        | Comportamiento declarado bueno                                              | Obligatorio tras el parche | Resultado | Notas                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------- | --------------------------------------------------------------------------- | -------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RG-01** | Host transforma un hueso → la pose llega al Cliente                         | Igual                      | ☐ PASS    | Funciona igual que antes de los parches, a veces no llegan al cliente, pero cuando selecciono todos los huesos con A, si se transfieren las transformaciones, es un bug de la version de antes de la fase 1. Segun lo que estoy probando, en el .blend real, si existen keyframes insertados, no se tranfieren al cliente, pero en huesos sin keyframes, si se transfieren, y si a esos huesos les inserto keyframes y los transformo, ahora si se transfieren por el Live. Aunque tengo otro .blend mas ligero en el que tengo un rig basico de humano y si funciona bien aunque tenga keyframes insertados, aunque a veces algunos huesos no respondan y tenga que seleccionar todos para ver respuesta. Pero es un bug de antes de tus parches. |
| **RG-02** | Seleccionar un hueso **sin curvas** crea keyframe temporal + curvas         | Igual                      | ☐ PASS    |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **RG-03** | Deseleccionar limpia el temporal y las curvas                               | Igual                      | ☐ PASS    | Si, pero si transformar y seleccionas otro hueso, no se limpia hasta que insertas un keyframe. Pero de nuevo es un bug de antes de los parches.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **RG-04** | Insertar/borrar keyframes llega al Cliente por porcelain **sin toggles F1** | Igual                      | ☐ PASS    |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **RG-05** | El Cliente puede seleccionar                                                | Igual                      | ☐ PASS    |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **RG-06** | La cadencia del canal no empeora                                            | Igual o mejor              | ☐ PASS    | Es igual, hay un bug que no identifico y es que en el .blend real en el Host, al mover los huesos del personaje, se ralentiza blender, pero en otro .blend de pruebas donde tengo 2 armaduras con 2000 huesos, no tengo ese problema. Y solo pasa al hostear en el .blend pesado, lo probe antes de hostear y blender no se laguea.                                                                                                                                                                                                                                                                                                                                                                                                                |

**Nota sobre RG-02/RG-03:** son el mecanismo de `_set_keyframe`. La fase 1 no lo toca; si fallan, el sospechoso es una guarda de rol que corta una publicación legítima del Host, no el overlay.

---

## 4. Bloque C · Undo/redo en el Cliente (por P11)

| ID        | Acción                                                     | Obligatorio                                                     | Resultado | Notas |
| --------- | ---------------------------------------------------------- | --------------------------------------------------------------- | --------- | ----- |
| **P11-a** | En el Cliente, hacer una transformación local y **Ctrl+Z** | **No** se publica `Delete`; la sincronización **no se rompe**   | ☐ PASS    |       |
| **P11-b** | Tras el undo, el Host mueve un hueso                       | El Cliente **sigue recibiendo** la pose (punteros re-resueltos) | ☐ PASS    |       |
| **P11-c** | Repetir undo/redo varias veces                             | Sin errores de nodos huérfanos que rompan la sesión             | ☐ PASS    |       |

**Residuo aceptado:** el Cliente puede acumular nodos huérfanos hasta la siguiente actualización del Host o reconexión. Es esperado; **no** es un FAIL mientras la sincronización siga viva.

---

## 5. Bloque D · El Host sigue pudiendo (control positivo)

| ID      | Acción en el **Host**        | Obligatorio                                                | Resultado | Notas |
| ------- | ---------------------------- | ---------------------------------------------------------- | --------- | ----- |
| **H-a** | Pulsar commit                | Publica con normalidad                                     | ☐ PASS    |       |
| **H-b** | Pulsar 🔓                    | **El diálogo se abre** y el cambio de propietario funciona | ☐ PASS    |       |
| **H-c** | Armar un toggle de Live Sync | Arma y publica                                             | ☐ PASS    |       |

Sin este bloque, un FAIL del bloque A podría deberse a una sesión mal establecida y no a las guardas.

---

## 6. Qué devolver

1. Esta hoja con las casillas marcadas y notas.
2. Logs relevantes del **Host** y del **Cliente** para cualquier FAIL.
3. Versión de Blender y confirmación de que ambos peers usan el mismo código.

---

## 7. Si algo falla

**No edites el código para «arreglarlo» sobre la marcha.** Anota el fallo y devuélvelo.

Reversión selectiva, según el bloque que falle:

```
# operadores (P0–P5)
git restore --source=fase-01-inicio -- operators.py

# handlers/timers (P6–P8 + P11)
git restore --source=fase-01-inicio -- handlers.py timers.py
```

Si el fallo ya está commiteado, se deshace con `git revert <sha>`, **nunca** con `git reset --hard`: el intento fallido queda en la historia.

---

## 8. Cierre de fase

La fase 1 se declara cerrada **solo** cuando:

- Bloque A completo en PASS.
- Bloque B completo en PASS.
- Bloque C sin rotura de sincronización.
- Bloque D en PASS.
- Se registra la entrada de cierre en `DIARIO-DESARROLLO.md` con la evidencia literal.
- `python3 verificar_docs.py repo` sigue en `TODO CORRECTO`.

Los dos fallos del overlay **no bloquean el cierre de la fase 1**: son deuda previa, medida en la línea base, y quedan como defecto abierto con fase propia.
