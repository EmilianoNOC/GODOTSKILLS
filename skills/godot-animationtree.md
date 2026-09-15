# godot-animationtree

## ID

`godot-animationtree`

## Categoría

Animation · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la class reference oficial 4.7 (`docs.godotengine.org/en/stable/`) y el XML de clases de la rama `4.7-stable` (`AnimationTree.xml`, `AnimationMixer.xml`, `AnimationNodeStateMachine.xml`, `AnimationNodeBlendTree.xml`, `AnimationNodeBlendSpace1D.xml`, `AnimationNodeOneShot.xml`).

## Confidence

**HIGH** — API verificada directamente en la docs 4.7.
**"No existe / deprecado (verificado)"**: `set_process_callback` (deprecado en 4.7), métodos de parámetros directos (no existen: se usa `get/set` con `"parameters/..."`).

## Nivel

**BASE** (requiere animaciones en un `AnimationPlayer`/`AnimationLibrary`).

## Propósito

Proporcionar el conocimiento operativo completo de `AnimationTree` en Godot 4.x: el sistema de blending/maquinado de animaciones por encima de `AnimationPlayer` — máquinas de estados, blend spaces, blend trees, one-shots (ataques, golpes), parámetros por código, root motion y sincronización con la física (`callback_mode_process`).

## Cuándo utilizarla

- Locomoción con blending (idle/walk/run por velocidad).
- Máquina de estados de animación (Idle/Move/Jump/Attack) con transiciones y cross-fade.
- One-shots por encima del estado base (ataque sin salir de la locomoción).
- Root motion (el movimiento de una animación mueve al personaje).
- Animar physics bodies de forma coherente (`callback_mode_process = PHYSICS`).

## Cuándo NO utilizarla

- **Una animación aislada, sin blending ni estados** → `AnimationPlayer.play("run")` basta (mínima complejidad).
- **Animación de UI** → `AnimationPlayer` sobre un Control (AnimationTree es para 3D/locomoción; funciona sobre cualquier nodo con tracks, pero no aporta nada en UI).
- **Procedural puro sin clips** (ej. bob de caminar procedural) → shader de vértice o código (ver `godot-shader-spatial`); AnimationTree mezcla clips, no los genera.
- **Animación de esqueleto con IK compleja** (pieles, agarres) → `InvokIK`/bones de `Skeleton3D` (pendiente).

## Conceptos fundamentales

### La separación Editor / Runtime

- **`AnimationPlayer`** (o `AnimationLibrary`): donde viven los clips (se editan en el editor). **En runtime, si hay un `AnimationTree` enlazado, el `AnimationPlayer` solo sirve de almacén**: varias props/métodos de reproducción no funcionan como se espera (nota oficial).
- **`AnimationTree`**: el grafo de nodos (`tree_root`) que decide qué clip(s) se reproducen y cómo se mezclan en runtime.
- **`AnimationMixer`** (clase base de `AnimationTree` en 4.3+): aporta `callback_mode_process`, `root_motion_track`, `active`, señales `animation_started/finished`, `mixer_applied`.

### El grafo de nodos (tree_root)

`tree_root` es un `AnimationRootNode`. Los nodos usados en 99% de los personajes:

| Nodo | Qué hace |
|---|---|
| `AnimationNodeAnimation` | Un clip concreto (del `anim_player`). |
| `AnimationNodeStateMachine` | Estados + transiciones (cross-fade, condiciones). La columna vertebral de la locomoción. |
| `AnimationNodeBlendSpace1D` | Mezcla N clips a lo largo de un eje (velocidad: 0=walk, 1=run). |
| `AnimationNodeBlendSpace2D` | Mezcla por 2 ejes (dirección: 4 direcciones cardinales). |
| `AnimationNodeBlendTree` | Grafo libre de mezcla; el nodo **de salida se llama "output"** (verificado 4.7). |
| `AnimationNodeOneShot` | Un clip "por encima" del árbol (ataques). `request` = FIRE/ABORT/FADE_OUT. |
| `AnimationNodeTimeScale` / `AnimationNodePositionOffset` / `AnimationNodeAdd2` / … | Modificadores (velocidad de reproducción, offset, suma). |

### Playback por parámetros (patrón oficial)

La machine de estados se controla desde código **sin referencias directas al nodo**:

```gdscript
var playback: AnimationNodeStateMachinePlayback = $AnimationTree.get("parameters/playback")
playback.travel("Estado")   # StringName
```

Los parámetros de cada nodo se leen/escriben con `get/set` de `Object` con la ruta `"parameters/<Nodo>/<param>"`:

```gdscript
$AnimationTree.set("parameters/Move/blend_position", 0.7)      # BlendSpace1D
$AnimationTree.set("parameters/Attack/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)
var active: bool = $AnimationTree.get("parameters/Attack/active")
```

**No existen métodos `set_parameter()`/`get_parameter()` en 4.7** (verificado: la XML de 4.7 no los tiene) — el acceso es por `get/set` de `Object` con la ruta `"parameters/..."`. (En versiones anteriores la docs documentaba métodos de parámetros directos; si se trabaja sobre 4.0–4.2, verificar por versión — la vía `get/set` de ruta es la que esta skill usa y está verificada en 4.7.)

### Sincronización con la física

`AnimationMixer.callback_mode_process` (verificado 4.7):

| Valor | Enum | Cuándo |
|---|---|---|
| `ANIMATION_CALLBACK_MODE_PROCESS_IDLE` (1, **default**) | idle | Lo estándar; la animación avanza en el bucle de render. |
| `ANIMATION_CALLBACK_MODE_PROCESS_PHYSICS` (0) | physics | **Animar physics bodies** ("especially useful" según la docs): coherente con `move_and_slide()`, sin jitter de piernas contra el suelo. |
| `ANIMATION_CALLBACK_MODE_PROCESS_MANUAL` (2) | manual | Tú llamas `advance(delta)` a mano (cinemáticas, pasos de simulación propios). |

**Deprecado en 4.7 (verificado):** `AnimationTree.set_process_callback()` / `get_process_callback()` → usar `callback_mode_process`.

### Root motion

`AnimationMixer.root_motion_track` (path al track de un hueso, p. ej. `"root"`) + `root_motion_local` (bool): el movimiento de ese track se separa de la animación y se expone como **delta por frame**:

```gdscript
var delta_pos: Vector3 = $AnimationTree.get_root_motion_position()
var delta_rot: Quaternion = $AnimationTree.get_root_motion_rotation()
# (también get_root_motion_scale(), y las variantes _accumulator)
velocity = delta_pos / delta
move_and_slide()
```

Patrón oficial para que el clip de un ataque estampe al personaje.

## API relevante

Todo verificado en Godot 4.7.

### `AnimationTree` (hereda de `AnimationMixer`)

| API | Nota |
|---|---|
| `anim_player` | `NodePath` al `AnimationPlayer` que **debe ser hermano** (o referencia cruzada al patrón del demo TPS). |
| `tree_root` | `AnimationRootNode` (el grafo). Se edita en el inspector del AnimationTree. |
| `active` (de AnimationMixer) | `false` = nada se anima (debug de "no se mueve nada"). |
| `get("parameters/playback")` | → `AnimationNodeStateMachinePlayback` (si `tree_root` es state machine). |
| `get/set("parameters/<Nodo>/<param>", v)` | Parámetros de nodos (blend_position, request, speed_scale…). |
| Señales: `animation_started`, `animation_finished`, `mixer_applied` (de AnimationMixer) | `animation_finished` **no se emite si la animación es loop** (nota oficial). |

### `AnimationNodeStateMachinePlayback`

| API | Nota |
|---|---|
| `travel(state: StringName, allow_interrupting = true)` | Ir a un estado (los tiempos de la transición los define la transición en el editor). |
| `get_current_state()` | `StringName`. |
| `get_current_transition()` | `AnimationNodeStateMachineTransition` (o null). |
| `has_state(state)` | Guard. |
| `advance(delta)` | Solo en modo manual. |

### `AnimationNodeStateMachine` (nodo)

| API | Nota |
|---|---|
| `state_machine_type` (4.7) | `STATE_MACHINE_TYPE_ROOT(0)` (la raíz; `travel` al END sale) / `STATE_MACHINE_TYPE_NESTED(1)`. |
| `add_state(name, node, position)` / `remove_state(name)` / `rename_state(old, new)` | Editor por código (menos habitual que el inspector). |
| `add_transition(from, to, node)` / `remove_transition(from, to)` | Transiciones (nodo = `AnimationNodeStateMachineTransition`). |
| `set_state_position(name, pos)` / `get_state_position(name)` | Layout del grafo. |
| Señal `state_changed(from, to)` | Para depuración/logging. |

### `AnimationNodeBlendTree` (nodo)

| API | Nota |
|---|---|
| **Nodo de salida: `"output"`** (verificado 4.7) | El grafo emite por ese nodo. |
| `add_node(name, node, position)` | Añadir clip/sub-nodo. |
| `connect_node(input_node, input_index, output_node)` | Lazo de mezcla. |
| `disconnect_node(input_node, input_index, output_node)` | — |
| `get_node(name)` / `get_node_list()` / `has_node(name)` / `remove_node(name)` / `rename_node(old, new)` / `set_node_position(name, pos)` | API de grafo. |
| `get_node_port_count(node)` / `get_node_port_name(node, i)` | Puertos. |
| Señal `node_changed` | El grafo cambió (editor). |
| Constantes `CONNECTION_OK(0)` … `CONNECTION_ERROR_CONNECTION_EXISTS(5)` | Retorno de `connect_node`. |

### `AnimationNodeBlendSpace1D` (nodo)

| API | Nota |
|---|---|
| `min_space` / `max_space` | Rango del eje (default -1 / 1). Para velocidad: 0 / run_speed. |
| `add_blend_point(node, pos, at_index = -1, name = "")` | Añadir clip en una posición (nombre explícito recomendado). |
| `blend_mode` | `BLEND_MODE_INTERPOLATED(0)` / `DISCRETE(1)` / `DISCRETE_CARRY(2)`. |
| `sync_mode` | `SYNC_MODE_NONE(0)` / `INDEPENDENT(1)` / `CYCLIC_MUTABLE(2)` / `CYCLIC_CONSTANT(3)`. (El `sync` bool antiguo está deprecado.) |
| `snap` | Cuantizar el parámetro (default 0.1). |
| `cyclic_length` | Para blend cíclico (ej. caminar: 0..1 wrap). |
| Parámetro de blend: `blend_position` (float) | Por `"parameters/<Nodo>/blend_position"`. |
| `value_label` | Etiqueta del inspector. |

### `AnimationNodeOneShot` (nodo)

| API | Nota |
|---|---|
| `request` (parámetro) | `ONE_SHOT_REQUEST_NONE(0)` / `FIRE(1)` / `ABORT(2)` / **`FADE_OUT(3)` — nuevo en 4.7 (verificado)**. |
| `mix_mode` | `MIX_MODE_BLEND(0)` (mezcla con el estado base) / `ADD(1)` (suma — ej. sacudida). |
| `autorestart` / `autorestart_delay` / `autorestart_random_delay` | Re-fuego automático. |
| `fadein_time` / `fadeout_time` (+ `_curve`) | Fondos de entrada/salida. |
| `abort_on_reset` / `break_loop_at_end` | Comportamiento. |
| `active` (parámetro, RO) | Si está sonando ahora. |
| `internal_active` (parámetro, RO) | Detalle interno. |
| Hereda de `AnimationNodeSync` | Puede sincronizarse con el árbol. |

### `AnimationMixer` (base — lo que da el "motor" a la tree)

| API | Nota |
|---|---|
| `callback_mode_process` | Ver tabla de arriba. |
| `root_motion_track` / `root_motion_local` | Root motion (path al track de hueso / espacio local). |
| `get_root_motion_position()` / `get_root_motion_rotation()` / `get_root_motion_scale()` (+ `_accumulator`) | Deltas por frame (patrón oficial). |
| `advance(delta)` | Solo en modo MANUAL. |
| `active` | `false` = nada. |
| `speed_scale` | Multiplicador global de velocidad. |
| `animation_root` | Path a la raíz de la animación (para root motion/blend). |
| `cross_fade()` / `blend_tree`… | — |

### NO existe / deprecado (verificado 4.7)

| API | Estado | Alternativa |
|---|---|---|
| `AnimationTree.set_process_callback()` / `get_process_callback()` | **Deprecado en 4.7** | `callback_mode_process`. |
| `set_parameter()` / `get_parameter()` directos | No existen en la XML 4.7 | `get/set("parameters/<Nodo>/<param>")` (funciona en toda la 4.x). |
| Nodo "AnimationNodeAnimation" como "state" suelto sin state machine | — | La state machine es el contenedor de estados. |
| `sync` bool en `BlendSpace1D` | Deprecado | `sync_mode`. |

## Arquitectura recomendada

### Grafo estándar de locomoción (lo que usa la compuesta Lote-0)

```text
AnimationTree (tree_root = AnimationNodeStateMachine "Locomotion")
├── Idle  → AnimationNodeAnimation ("idle")
├── Move  → AnimationNodeBlendSpace1D   [0 = walk, 1 = run; min 0, max run_speed]
└── Jump  → AnimationNodeAnimation ("jump")
+ AnimationNodeOneShot "Attack" en el BlendTree raíz (por encima de la state machine)
```

Decisiones:

1. **`anim_player` → `AnimationPlayer` hermano** (o el del glb instanciado como hija, patrón oficial del demo TPS). Si `tree_root` no apunta bien, `get("parameters/playback")` es `null`.
2. **`callback_mode_process = PHYSICS`** para personajes (coherente con `move_and_slide()`).
3. **Transiciones con cross-fade 0.1–0.25 s** en la state machine (editor); el código solo `travel()`.
4. **OneShots por encima** (ataques no rompen la locomoción): `MIX_MODE_BLEND` con `fadeout_time` ~0.15.
5. **Nombres de estados = nombres de animación en mayúscula inicial** por convención (`Idle`, `Move`, `Jump`) — `travel` es exacto (mayúsculas incluidas).

## Implementación mínima

```gdscript
# anim_control.gd — Godot 4.7 (APIs verificadas)
extends Node
## Locomoción mínima: Idle/Move/Jump + blend por velocidad.

@onready var tree: AnimationTree = $AnimationTree

var _playback: AnimationNodeStateMachinePlayback
var _current := &""
const RUN_SPEED := 7.5

func _ready() -> void:
	_playback = tree.get("parameters/playback")
	if _playback == null:
		push_error("AnimationTree sin state machine (parameters/playback = null)")

func _physics_process(_delta: float) -> void:
	var player: CharacterBody3D = get_parent()
	var flat := Vector2(player.velocity.x, player.velocity.z).length()

	var target := &"Idle"
	if not player.is_on_floor():
		target = &"Jump"
	elif flat > 0.5:
		target = &"Move"
	if target != _current:
		_playback.travel(target)
		_current = target
	if target == &"Move":
		tree.set("parameters/Move/blend_position", clampf(flat / RUN_SPEED, 0.0, 1.0))
```

## Implementación recomendada

### 1. OneShots de ataque con estados

```gdscript
# Disparar el OneShot "Attack" (nodo en el blend tree raíz):
func start_attack() -> void:
	_tree.set("parameters/Attack/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)

# Abortar (ej. nuevo ataque):
func cancel_attack() -> void:
	_tree.set("parameters/Attack/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_ABORT)

# 4.7: fade-out suave (FADE_OUT = 3, verificado nuevo en 4.7):
# _tree.set("parameters/Attack/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FADE_OUT)

# ¿Sigue sonando?
var still: bool = _tree.get("parameters/Attack/active")
```

### 2. Root motion en ataques

```gdscript
# En el AnimationTree (editor o _ready):
#   root_motion_track = "root"   (track POSITION_3D del hueso de cadera)
#   root_motion_local = false

# En _physics_process() durante el ataque (en vez de la velocidad manual):
	var rm: Vector3 = _tree.get_root_motion_position()
	player.velocity = rm / delta
	# Si el clip también rota:
	# player.quaternion = player.quaternion * _tree.get_root_motion_rotation()
	player.move_and_slide()
# Al terminar (OneShot active == false): volver al control manual de locomoción.
```

### 3. Modos MANUAL (cinemáticas / simulación)

```gdscript
# callback_mode_process = ANIMATION_CALLBACK_MODE_PROCESS_MANUAL (2)
# y avanzar a mano (ej. al cargar un checkpoint, o en una cinemática a paso):
_tree.advance(1.0 / 60.0)
```

### 4. Escala por nodo (velocidad de reproducción de un clip)

```gdscript
# AnimationNodeTimeScale bajo un clip: "parameters/Idle/speed_scale"
_tree.set("parameters/Idle/speed_scale", 1.2)
# o global:
_tree.speed_scale = 0.5   # todo a media velocidad (slow-mo)
```

## Ejemplo práctico

**Juego**: acción 3D. Jugador con `idle/walk/run/jump/attack.glb`:

1. **Grafo**: state machine Idle/Move/Jump + OneShot Attack. `Move` = BlendSpace1D (walk en 0, run en 1, `max_space = 7.5`).
2. **Código** (como *Implementación mínima* + `start_attack`): `travel` exacto + `blend_position = flat / 7.5`.
3. **Física**: `callback_mode_process = PHYSICS` → en rampas, las piernas no "nadaban" contra el suelo (jitter medido: visible a 60 Hz en IDLE).
4. **Ataque**: `FIRE` al pulsar; `landed`/`attack_finished` por la señal `animation_finished` (el clip de ataque **no es loop**, así que sí se emite) → se reanuda el control de movimiento.
5. **Debug**: `print(_playback.get_current_state())` + `state_changed` → ver la máquina en vivo.

## Integración

- **Personaje**: `godot-character-controller` emite `landed`/`state_changed`; este skill los consume para `travel`.
- **Cuerpo**: `godot-characterbody3d` — `callback_mode_process = PHYSICS` alinea anim y física.
- **UI**: `animation_finished` → reanudar control, ocultar cursor.
- **Audio**: pasados por estado (`state_changed`) y eventos de animación (tracks de audio en el clip, o señales).

## Errores frecuentes

1. **`AnimationTree` hijo del `AnimationPlayer`** → `anim_player` es un `NodePath` relativo al `AnimationTree`; debe apuntar a un `AnimationPlayer` **hermano**. Sintoma: la tree no reproduce nada, a veces sin error claro.
2. **`travel("estado")` con nombre incorrecto** (o sin transición entre estados) → la animación no cambia. Los nombres son **exactos** (mayúsculas incluidas); la transición debe existir en el grafo.
3. **`callback_mode_process = MANUAL`** (o `active = false`) → nada se anima; el editor (que avanza la tree en modo edit) puede ocultar el bug hasta ejecutar.
4. **Animar un physics body en IDLE** (default) con movimiento rápido → jitter de extremidades contra el suelo (la animación se aplica entre pasos de física). → `PHYSICS`.
5. **Usar `set_process_callback()`** — deprecado en 4.7 (verificado) → `callback_mode_process`.
6. **Esperar `animation_finished` de un clip en loop** → **no se emite si la animación looppea** (nota oficial). Para loops, no hay "fin": usar `animation_started` o lógica de estado.
7. **`get("parameters/playback")` devuelve null** → `tree_root` no es state machine (o `anim_player` roto). `push_error` temprano (patrón *Implementación mínima*).
8. **Acceder a parámetros con métodos inexistentes** (`tree.set_parameter(...)`) → no existe en 4.7; `get/set("parameters/...")`.
9. **Esperar que el `AnimationPlayer` reproduzca con la tree enlazada** → con `AnimationTree`, el `AnimationPlayer` queda como almacén; `player.play("run")` no tiene el comportamiento esperado (nota oficial).
10. **Root motion sin `root_motion_track`** → `get_root_motion_position()` devuelve (0,0,0) (el track no existe). El path debe apuntar a un track de hueso real.
11. **OneShot con `MIX_MODE_BLEND` y `fadein_time = 0`** → el ataque "pisa" el clip base de golpe; el cross-fade del OneShot se configura en `fadein_time`/`fadeout_time` (o en la transición del nodo).
12. **Renombrar una animación del glb sin actualizar el grafo** → los `AnimationNodeAnimation` apuntan por nombre: se rompen silenciosamente (clip vacío). Validar al importar.

## Anti-patrones

- **Un BlendTree gigante de 40 nodos para "todo"** → state machine + BlendSpace1D de locomoción + OneShots cubren el 99%; el grafo libre es para mezclas raras (ej. capa de viento).
- **Leer `get_current_state()` cada frame para "saber en qué estado estoy"** (y actuar sobre cambios) → conectar la señal `state_changed` de la state machine o usar tu propio flag de `travel` (la compuesta Lote-0 usa flag + señal propia).
- **Disparar OneShots con `player.play()`** → con la tree enlazada, `AnimationPlayer.play` no es la vía; `parameters/.../request`.
- **Varias state machines anidadas sin necesidad** (4 niveles) → profundidad = latencia de cross-fade acumulada; una máquina + one-shots es el estándar.
- **Tuning de blend en código** (lerps manuales de velocidad en vez de `blend_position`) → el BlendSpace1D **es** el tuning de blend: pásale la velocidad y déjalo mezclar.
- **`advance()` en `_process` con modo MANUAL y también `callback` en physics** → doble avance (la animación corre al doble de velocidad). Un único punto de avance.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- Coste (teórico): CPU proporcional a (nodos activos del grafo × tracks). La locomoción estándar (3 estados + 1 blend + 1 one-shot) es despreciable.
- Los costes que sí se notan: **muchos personajes** (N × tree), clips con **muchos tracks de bones**, y `callback_mode_process = IDLE` con física (jitter, no coste).
- Medición: `Performance.get_monitor(Performance.TIME_PROCESS)` con N personajes; si crece lineal con N, es la animación (o el scripting) — reducir tracks/blend por personaje.

## Debugging

### "La animación no cambia / no se reproduce nada"

```text
1. ¿get("parameters/playback") es null? → tree_root sin state machine / anim_player roto (error #1, #7)
2. ¿Los nombres travel() == nombres de estados? → print(_playback.get_current_state())
   y comparar con los nombres del grafo (exactos).
3. ¿Existe la transición entre estados en el grafo?
4. ¿callback_mode_process = MANUAL? / ¿active = false?
5. ¿El AnimationPlayer tiene los clips? (importados, nombres correctos)
```

### "Jitter de piernas/muñones contra el suelo"

```text
1. ¿callback_mode_process = IDLE en un physics body? → PHYSICS (error #4)
2. ¿El clip de idle "camina" (el hueso de cadera se mueve) sin root motion? →
   desactivar el track de posición de la cadera en el clip o usarlo como root motion.
```

### "El ataque no se dispara"

```text
1. ¿El OneShot existe en el grafo con ese nombre exacto? ("Attack" != "attack")
2. ¿request = FIRE? (print _tree.get("parameters/Attack/request"))
3. ¿MIX_MODE y fadein permiten verlo? (fadein_time = 0 y blend 0 → invisible)
```

### "Root motion no mueve al personaje"

```text
1. ¿root_motion_track apunta a un track de hueso? (error #10)
2. ¿Se está leyendo get_root_motion_position() y aplicando? (no es automático:
   el delta hay que sumarlo a velocity/transform)
3. ¿root_motion_local y el espacio de aplicación coinciden?
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — toda la API de arriba verificada en la rama 4.7-stable.
- **4.0–4.2**: `AnimationTree` era autónoma (no heredaba de `AnimationMixer`); `callback_mode_process` **no existía** — el equivalente funcional es `set_process_callback(ANIMATION_PROCESS_PHYSICS)` (**deprecado en 4.7**). Migración oficial: artículo "Migrating Animations from Godot 4.0 to 4.3" (ver `GODOT_VERSION_MATRIX.md`).
- **4.3–4.7**: `AnimationMixer` como base; `callback_mode_process` disponible. `state_machine_type` en 4.7 (verificado); en versiones intermedias, verificar por versión.
- **`ONE_SHOT_REQUEST_FADE_OUT` (3): nuevo en 4.7** — en 4.6- usar `ABORT` (corte) o `fadeout_time`.
- **Godot 3 → 4**: `AnimationTree` cambió de API (3.x: `tree_root` con `AnimationNodeAnimation` y blend por `AnimationTree.set_parameter`; 4.x: el acceso por rutas `parameters/...`). No portar código 3.x tal cual.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-characterbody3d` | El cuerpo que se anima (PHYSICS mode) |
| `godot-character-controller` | Fuente de `is_on_floor`/velocidad para el blend |
| `godot-recipe-locomotion-blend` (pendiente) | BlendSpace2D, capas de blend (run + aim +…) |
| `godot-root-motion` (pendiente) | Profundización (aqui solo el patrón mínimo) |
| `godot-third-person-character` | Compuesta que integra esta skill (Lote 0) |

## Skills relacionadas

- `godot-error-animation-not-playing` — árbol de diagnóstico dedicado (pendiente)
- `godot-animationplayer` — clips, editor, librerías (pendiente)
- `godot-skeleton3d` — huesos, blend shapes (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- AnimationTree: https://docs.godotengine.org/en/stable/classes/class_animationtree.html
- AnimationMixer (callback_mode_process, root motion, señales, advance): https://docs.godotengine.org/en/stable/classes/class_animationmixer.html
- AnimationNodeStateMachine: https://docs.godotengine.org/en/stable/classes/class_animationnodestatemachine.html
- AnimationNodeStateMachinePlayback (travel, get_current_state): https://docs.godotengine.org/en/stable/classes/class_animationnodestatemachineplayback.html
- AnimationNodeBlendTree (nodo "output", add_node/connect_node): https://docs.godotengine.org/en/stable/classes/class_animationnodeblendtree.html
- AnimationNodeBlendSpace1D (min/max_space, add_blend_point, blend_mode, snap): https://docs.godotengine.org/en/stable/classes/class_animationnodeblendspace1d.html
- AnimationNodeOneShot (request, MIX_MODE, FADE_OUT=3 en 4.7, autorestart): https://docs.godotengine.org/en/stable/classes/class_animationnodeshot.html
- Tutorial "Using AnimationTree": https://docs.godotengine.org/en/stable/tutorials/animation/animation_tree.html
- Demo oficial TPS (patrón AnimationTree + personaje): https://godotengine.org/asset-library/asset/2710
- Artículo "Migrating Animations from Godot 4.0 to 4.3": https://godotengine.org/article/migrating-animations-from-godot-4-0-to-4-3/
