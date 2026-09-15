# godot-third-person-character

> **SKILL COMPUESTA** (spec §51) — integra 8 skills especializadas en un sistema completo y mantenible:
> personaje 3D en tercera persona para Godot 4.x (movimiento + cámara + animación + input + shaders + LOD + rendimiento).

## ID

`godot-third-person-character`

## Categoría

3D · Characters (skill compuesta / receta de integración)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra:

- Class reference oficial: `docs.godotengine.org/en/stable/` (rama 4.7)
- Tutorials oficiales (spring arm, AnimationTree, first 3D game)
- XML de clases del motor (rama `4.7-stable` de `godotengine/godot`)

## Confidence

**HIGH** — toda la API usada está verificada directamente en la documentación oficial 4.7.
Excepciones, marcadas explícitamente:

- **MEDIUM**: valores de tuning de gameplay (velocidades, gravedad, sensibilidades, tiempos de coyote/buffer). Son recomendaciones prácticas, no especificación oficial.
- **"No existe (verificado)"**: esta skill documenta también APIs que **NO existen** en 4.7, para que el agente no las invente (ver *Errores frecuentes* y *GODOT_VERSION_MATRIX.md*).

## Nivel

**INTERMEDIATE** (compuesta: requiere física + cámara + animación + input + rendering).

## Propósito

Permite al agente implementar, modificar, depurar y optimizar un **personaje jugable en tercera persona** completo en Godot 4.x, integrando correctamente:

1. Cuerpo físico y movimiento (`godot-characterbody3d`)
2. Controlador de personaje con tuning (`godot-character-controller`)
3. Input abstracto (teclado/ratón/gamepad) (`godot-input`)
4. Locomoción animada con máquina de estados (`godot-animationtree`)
5. Cámara orbital con colisión y anti-clipping (`godot-camera3d`)
6. Aspecto visual del personaje (materiales y shaders espaciales) (`godot-shader-spatial`)
7. Medición y optimización del coste del sistema (`godot-rendering-performance`)
8. LOD del personaje y del mundo alrededor (`godot-lod`)

## Cuándo utilizarla

- TPS (third-person shooter), acción-aventura, RPG 3D, exploración.
- "Quiero que el personaje camine/corra/salte y lo vea desde atrás".
- Integrar personaje con cámara orbital, animación de locomoción y tuning ajustable.
- Preparar el personaje para escalar (multiplayer: otros jugadores, enemigos, LOD).

## Cuándo NO utilizarla

- **Juegos 2D** → `CharacterBody2D` (skills 2D pendientes).
- **FPS con mira** → el rig de cámara es distinto (pivot en la cabeza, sin spring arm largo): ver futura `godot-recipe-first-person-controller`.
- **Espacio / vuelo** → `CharacterBody3D.motion_mode = MOTION_MODE_FLOATING` y control de 6DOF; este skill asume gravedad y suelo.
- **Cámara fija / cinemática** → `godot-cinematic-camera` (pendiente).
- **Vehículos** → `VehicleBody3D` (skill pendiente).

## Conceptos fundamentales

### Los tres bucles, separados

| Bucle | Función | API |
|---|---|---|
| `_physics_process(delta)` | Gravedad, salto, aceleración, `move_and_slide()` | delta del paso de física (fijo, normalmente 60 Hz) |
| `_process(delta)` | Suavizado de cámara, sticks de gamepad | delta de render |
| `_unhandled_input(event)` | Ratón/teclado tras la UI | eventos de input |

**Regla de oro**: `move_and_slide()` debe ejecutarse **siempre** en `_physics_process` (la docs oficial es explícita: usa internamente el `delta` del paso de física; en otro sitio la simulación corre a velocidad incorrecta).

### Espacios de coordenadas

- El **movimiento** es relativo a la cámara: el vector de input se rota por el **yaw de la cámara** (solo rotación Y; el tilt de la cámara no debe entrar en el movimiento).
- El **modelado visual** (rotar el mesh hacia la dirección de movimiento) se hace en un nodo hijo pivot (`Model`), **nunca** rotando el `CharacterBody3D` (afecta a física: `up_direction`, slides, etc.).
- La **cámara** orbita sobre un pivot (`CameraPivot`) hijo del jugador; el `SpringArm3D` resuelve la colisión contra el mundo.

### Una cámara activa por viewport

`Camera3D.current` marca la cámara activa. Solo una por `Viewport`. Si hay varias cámaras en escena, exactamente una debe tener `current = true` (o llamar `make_current()`).

## API relevante

Todo verificado en Godot 4.7.

### CharacterBody3D (física)

| API | Tipo | Nota |
|---|---|---|
| `velocity` | `Vector3` | m/s. **NO** multiplicar por delta (advertencia oficial en la prop). |
| `move_and_slide()` | `-> bool` | Mueve según `velocity`, desliza en colisiones, modifica `velocity`. Solo en `_physics_process`. `true` si colisionó. |
| `is_on_floor()` / `is_on_floor_only()` | `-> bool` | Válido tras `move_and_slide()`. |
| `is_on_wall()` / `is_on_ceiling()` | `-> bool` | Idem. |
| `get_floor_normal()` / `get_floor_angle()` / `get_wall_normal()` | `-> Vector3/float` | Para lerp de cámara, detección de pendientes, etc. |
| `floor_max_angle` | `float` (45°) | Pendientes hasta 45° cuentan como suelo. |
| `floor_snap_length` | `float` (0.1) | Aísla al cuerpo del suelo; sin snap no se pega a pendientes bajando. No se aplica si hay velocidad hacia `up_direction` (salto). |
| `floor_constant_speed` | `bool` | `true`: misma velocidad en pendientes (requiere `floor_snap_length`). |
| `safe_margin` | `float` (0.001) | Margen de recuperación de colisión. Subir si el personaje se atasca al entrar en geometría. |
| `max_slides` | `int` (6) | Slides máximos por `move_and_slide()`; bajo → atascos en esquinas. |
| `up_direction` | `Vector3` (0,1,0) | Se normaliza; no puede ser ZERO. |
| `motion_mode` | enum | `MOTION_MODE_GROUNDED` (default) / `MOTION_MODE_FLOATING` (espacio). |
| `slide_on_ceiling`, `floor_stop_on_slope`, `floor_block_on_wall` | `bool` | Comportamiento fino de slides. |
| `platform_floor_layers` / `platform_on_leave` | — | Plataformas móviles: propagación de velocidad. |
| `get_last_slide_collision()` | `-> KinematicCollision3D` | `null` si no colisionó. |
| `get_real_velocity()` / `get_position_delta()` | `Vector3` | Velocidad real (diagonal en pendientes) vs `velocity` (la pedida). |
| `get_rid()` | `RID` | Para excluir el cuerpo de casts (ej. spring arm). |

**NO existen en 4.x (verificado en 4.0 y 4.7):** señales `floor_entered/floor_exited/wall_entered/wall_exited/ceiling_entered/ceiling_exited/moved` (polling de `is_on_floor()` o `Area3D` como alternativa) y propiedades `floor_friction` / `wall_min_slide_speed` (API de Godot 3; en 4.x la fricción se resuelve con `floor_constant_speed`, `PhysicsMaterial3D` o amortiguación manual).

### Camera3D + SpringArm3D (cámara)

| API | Tipo | Nota |
|---|---|---|
| `Camera3D.current` | `bool` | Cámara activa del viewport. |
| `Camera3D.fov` | `float` (75) | FOV en grados (proyección perspectiva). |
| `Camera3D.near` / `Camera3D.far` | `float` (0.05 / 4000) | Planos de recorte. |
| `Camera3D.make_current()` / `clear_current()` | — | Cambios de cámara (transiciones). |
| `Camera3D.cull_mask` | `int` | Capas visibles (visual, no física). |
| `SpringArm3D.spring_length` | `float` (1.0) | Longitud del brazo; el cast usa esta longitud. |
| `SpringArm3D.margin` | `float` (0.01) | Resta al largo en colisión para no pegar la cámara al muro. |
| `SpringArm3D.shape` | `Shape3D` | Si es `<empty>` y la cámara es **hijo directo** → usa la pirámide del plano near de la cámara. Si no hay shape y la cámara no es hijo directo → **raycast impreciso (no recomendado, dice la docs)**. |
| `SpringArm3D.collision_mask` | `int` (1) | Capas físicas contra las que choca el brazo. |
| `SpringArm3D.add_excluded_object(rid)` | — | Excluye un `PhysicsBody3D` del cast (ej. el propio jugador). |
| `SpringArm3D.get_hit_length()` | `-> float` | Longitud actual (útil para debug: si es menor a `spring_length`, hay colisión). |

**NO existe (verificado en 4.0 y 4.7):** smoothing incorporado de posición/rotación en `Camera3D` (eso existe en `Camera2D`, no en 3D). El suavizado 3D es manual (lerp/damping con `delta`).

### AnimationTree + AnimationMixer (animación)

| API | Nota |
|---|---|
| `AnimationTree.anim_player` | `NodePath` al `AnimationPlayer` que **debe ser HERMANO** del `AnimationTree` (o de la escena que contiene al mixer). |
| `AnimationTree.tree_root` | `AnimationRootNode` (BlendTree, BlendSpace, StateMachine). |
| `AnimationTree.get("parameters/playback")` | Devuelve `AnimationNodeStateMachinePlayback` (patrón oficial). |
| `state_machine.travel("Estado")` | Viaje a un estado; los tiempos de transición se configuran en el editor (`AnimationNodeStateMachineTransition`). |
| `AnimationTree.set("parameters/<Nodo>/<param>", v)` | Parámetros de blend (ej. `blend_position`), OneShots, etc. Sintaxis alternativa: `tree["parameters/..."] = v`. |
| `AnimationTree.get("parameters/OneShot/active")` | Lectura de estado de un OneShot. |
| `AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE` / `ONE_SHOT_REQUEST_ABORT` | Disparo/aborto de OneShot. |
| `AnimationMixer.root_motion_track` + `root_motion_local` | Raíz de motion: path al track de hueso. |
| `get_root_motion_position()` / `get_root_motion_rotation()` (+ `_accumulator`) | Deltas de root motion (patrón oficial para aplicar a `CharacterBody3D`). |
| `AnimationMixer.callback_mode_process` | `ANIMATION_CALLBACK_MODE_PROCESS_PHYSICS` (0) recomendado al animar physics bodies; `..._IDLE` (1, default); `..._MANUAL` (2). |
| `AnimationMixer.active` | Si es `false`, nada se anima. |
| Señales (AnimationMixer): `animation_started`, `animation_finished` | Para acoplar eventos de juego (no se emiten si la animación es loop). |
| `AnimationNodeStateMachine.state_machine_type` | `STATE_MACHINE_TYPE_ROOT` (0) / `..._NESTED` (1) — 4.7. |

**Deprecado en 4.7:** `AnimationTree.set_process_callback()` / `get_process_callback()` → usar `AnimationMixer.callback_mode_process`.
**Nota oficial:** con `AnimationTree` enlazado, varias props/métodos del `AnimationPlayer` no funcionan como se espera; el `AnimationPlayer` queda solo para editar/guardar animaciones.

### Input (input)

| API | Nota |
|---|---|
| `Input.get_vector(neg_x, pos_x, neg_y, pos_y, deadzone=-1.0)` | `-> Vector2`. `deadzone=-1` usa el del proyecto; pasa `0.2` si quieres deadzone propio del stick. |
| `Input.is_action_just_pressed(action)` / `is_action_just_released(action)` | Uno-por-frame. En `_physics_process` funcionan por paso de física. |
| `Input.is_action_pressed(action)` | Estado continuo. |
| `Input.get_action_strength(action)` | 0..1 (análogo: sticks/triggers). |
| `Input.get_axis(neg, pos)` | `strength(pos) - strength(neg)`. |
| `Input.mouse_mode` | `MOUSE_MODE_VISIBLE(0)` / `HIDDEN(1)` / `CAPTURED(2)` / `CONFINED(3)` / `CONFINED_HIDDEN(4)`. |
| `InputEventMouseMotion.screen_relative` | Deltas de ratón **independientes de la resolución** (recomendado por el tutorial oficial para sensibilidad). |
| `Input.get_joy_axis(device, JOY_AXIS_...)` | Para sticks de cámara (pollear en `_process`). |
| `Input.action_press(action)` / `action_release(action)` | Simulación (no dispara `_input`; para tests). |

**NO existe en 4.7 (verificado):** señales `action_just_pressed` / `action_released` del singleton `Input` (la única señal es `joy_connection_changed`). Usar polling o `_unhandled_input`.

### Shader / material (aspecto)

| API | Nota |
|---|---|
| `StandardMaterial3D` | `albedo_color`/`albedo_texture`, `metallic`/`metallic_texture`, `roughness`/`roughness_texture`, `normal_enabled`/`normal_texture`, `emissive_enabled`/`emissive`/`emissive_texture`. Material PBR default. |
| `ShaderMaterial` + shader `shader_type spatial` | Built-ins estándar: `ALBEDO`, `ALPHA`, `UV`, `TEXTURE`, `EMISSIVE`, `NORMAL`, `VERTEX`, `TIME`, etc. `texture(TEXTURE, UV)` para muestrear la textura albedo. |
| `ShaderMaterial.set_shader_parameter(name, value)` | Uniforms por material (a todos los que lo comparten). |
| `GeometryInstance3D.set_instance_shader_parameter(name, value)` | Uniform **por instancia** (el uniform debe declararse `instance uniform` en el shader). Ideal: N enemigos con material compartido pero highlight independiente. |
| `GeometryInstance3D.material_override` / `material_overlay` | Reemplazar/encimar material de todo el nodo. |
| `GeometryInstance3D.transparency` | 0..1; `>0` fuerza pipeline transparente (más lento). **Solo Forward+**; en Mobile/Compatibility se ignora. |
| `GeometryInstance3D.cast_shadow` | `SHADOW_CASTING_SETTING_OFF` para apagar sombras en LODs lejanos. |

### LOD (verificado en 4.7)

**No hay nodo `LOD` nativo en el core 4.7** (verificado: no existe `LOD.xml`, ni `scene/3d/lod.cpp`, ni página de clase). No inventarlo. Mecanismos oficiales:

| API | Nota |
|---|---|
| `GeometryInstance3D.visibility_range_begin` / `visibility_range_end` | Distancias de culling (0 = deshabilitado). |
| `visibility_range_begin_margin` / `visibility_range_end_margin` | Hysteresis (fade desactivado) o distancia de fade. |
| `visibility_range_fade_mode` | `FADE_DISABLED(0)` hysteresis (más rápido); `FADE_SELF(1)` y `FADE_DEPENDENCIES(2)` — **solo Forward+**. |
| `Node3D.visibility_parent` | Jerarquías de visibilidad (HLOD): un nodo controla la visibilidad de sus dependencias. |
| `GeometryInstance3D.lod_bias` | `0` fuerza LOD más bajo, `1` default, `>1` mantiene LODs altos más lejos. "Útil para probar transiciones de LOD". |
| `ArrayMesh.add_surface_from_arrays(primitive, arrays, blend_shapes, lods, flags)` | `lods`: `Dictionary` {float (coef. de distancia) → `PackedInt32Array` (índices de ese nivel de LOD)}. LOD **por superficie** del mesh. |
| `Mesh.surface_get_lods(surf_idx)` | Lectura del dict de LODs de una superficie. |

## Arquitectura recomendada

### Escena del jugador (`player.tscn`)

```text
Player (CharacterBody3D)                      [player.gd]
├── CollisionShape3D
│   └── CapsuleShape3D   (radius 0.4, height 1.8)  → posición y = 0.9
├── Model (Node3D)  [unique: %Model]                 pivot SOLO para rotar el visual
│   └── MeshInstance3D  (personaje .glb: Skeleton3D + mesh skinned)
├── AnimationPlayer   (animaciones: idle, walk, run, jump)   ← HERMANO del AnimationTree
├── AnimationTree  [unique: %AnimationTree]
│   └── tree_root: AnimationNodeStateMachine
│       ├── Idle  → AnimationNodeAnimation ("idle")
│       ├── Move  → AnimationNodeBlendSpace1D (walk → run, eje = velocidad)
│       └── Jump  → AnimationNodeAnimation ("jump")
└── CameraPivot (Node3D)  [unique: %CameraPivot]   posición (0, 1.6, 0)
    └── SpringArm3D  [unique: %SpringArm3D]        spring_length 4.0, margin 0.03,
        └── Camera3D  [unique: %Camera3D]          current = true, fov 60          collision_mask = capa MUNDO
```

Decisiones y porqué:

1. **`Model` como hijo pivot**: rotar el `CharacterBody3D` rompería física (`up_direction`, slides, `get_floor_normal`). El visual rota, el cuerpo no.
2. **`AnimationPlayer` hermano de `AnimationTree`**: `anim_player` es un `NodePath` relativo al `AnimationTree`. Con personajes importados (.glb) el patrón del demo oficial TPS es: instanciar la escena importada (que trae su `AnimationPlayer`) como hija, y crear `AnimationTree` en TU escena apuntando a ese `AnimationPlayer`.
3. **`Camera3D` hijo DIRECTO de `SpringArm3D`**: así el brazo usa la pirámide del plano near de la cámara. Si insertas un nodo intermedio sin `shape`, el brazo cae a raycast (impreciso, no recomendado según la docs).
4. **`SpringArm3D.collision_mask` solo capa MUNDO** + `add_excluded_object(get_rid())` en `_ready`: doble seguridad para que la cámara no choque contra el propio jugador.
5. **Nombres únicos (`%Name`)**: referencia robusta sin cadenas de `get_node` (anti-pattern).
6. **Capas de colisión sugeridas**: capa 1 = mundo, capa 2 = jugador, capa 3 = interactables, capa 4 = cámara (si se usa shape propio en el brazo).

### Input Map (Project Settings → Input Map)

| Acción | Teclado | Ratón | Gamepad | Uso |
|---|---|---|---|---|
| `move_left` | A / ← | — | stick izq. X− / pad izq. | `get_vector` |
| `move_right` | D / → | — | stick izq. X+ / pad der. | `get_vector` |
| `move_forward` | W / ↑ | — | stick izq. Y− | `get_vector` |
| `move_back` | S / ↓ | — | stick izq. Y+ | `get_vector` |
| `jump` | Espacio | — | X (South) / B | `is_action_just_pressed` |
| `sprint` | Shift izq. | — | A (East) / LB | `is_action_pressed` |
| cámara | — | botón der. (capture), rueda (zoom), motion (orbitar) | stick der. (orbitar) | `_unhandled_input` + `_process` |

La lógica de gameplay **solo usa nombres de acción**: cambiar de dispositivo no toca el código (requisito de la arquitectura de input).

## Implementación mínima

Un solo script (principio de mínima complejidad). Escena de arriba + Input Map de arriba.

```gdscript
# player.gd — Godot 4.7 (APIs verificadas)
extends CharacterBody3D
## Tercera persona mínima: movimiento relativo a cámara, salto con coyote time,
## cámara orbital con SpringArm3D y locomoción con AnimationTree.

# --- Tuning (m/s, m/s²). Valores MEDIUM (prácticos, no oficiales) ---
@export var walk_speed := 4.5
@export var run_speed := 7.5
@export var jump_velocity := 5.5
@export var gravity := 18.0          # gravedad propia (independiente de la del proyecto)
@export var ground_accel := 24.0
@export var air_accel := 10.0
@export var deceleration := 30.0
@export var turn_speed := 12.0       # rad/s de suavizado de giro del modelo
@export var mouse_sensitivity := 0.0035
@export_range(0.0, 89.0) var tilt_limit := 75.0   # grados

const COYOTE_TIME := 0.1             # s de gracia tras saltar de una acera
const JUMP_BUFFER := 0.12            # s que se recuerda un salto prematuro

@onready var _model: Node3D = %Model
@onready var _camera_pivot: Node3D = %CameraPivot
@onready var _spring_arm: SpringArm3D = %SpringArm3D
@onready var _anim_tree: AnimationTree = %AnimationTree

var _state_machine: AnimationNodeStateMachinePlayback
var _current_state := &""
var _time_since_floor := 10.0
var _time_since_jump := 10.0

func _ready() -> void:
	# Patrón oficial: playback de la máquina de estados por parámetro.
	_state_machine = _anim_tree.get("parameters/playback")
	# La cámara no debe chocar contra el colisionador del propio jugador.
	_spring_arm.add_excluded_object(get_rid())

func _physics_process(delta: float) -> void:
	# 1) Suelo + gravedad (delta del paso de física)
	if is_on_floor():
		_time_since_floor = 0.0
	else:
		_time_since_floor += delta
		velocity.y -= gravity * delta

	# 2) Salto con buffer + coyote time
	_time_since_jump += delta
	if Input.is_action_just_pressed("jump"):
		_time_since_jump = 0.0
	if _time_since_jump <= JUMP_BUFFER and _time_since_floor <= COYOTE_TIME:
		velocity.y = jump_velocity
		_time_since_jump = 10.0
		_time_since_floor = 10.0

	# 3) Input → dirección relativa a la cámara (solo YAW de la cámara)
	var input_vec := Input.get_vector("move_left", "move_right", "move_forward", "move_back")
	var direction := Vector3.ZERO
	if input_vec != Vector2.ZERO:
		var cam_yaw := Basis.from_euler(Vector3(0.0, _camera_pivot.rotation.y, 0.0))
		direction = (cam_yaw * Vector3(input_vec.x, 0.0, input_vec.y)).normalized()

	# 4) Velocidad horizontal con aceleración/amortiguación
	var target_speed := run_speed if Input.is_action_pressed("sprint") else walk_speed
	var accel := ground_accel if is_on_floor() else air_accel
	var flat := Vector2(velocity.x, velocity.z)
	if direction != Vector3.ZERO:
		flat = flat.move_toward(Vector2(direction.x, direction.z) * target_speed, accel * delta)
	elif is_on_floor():
		flat = flat.move_toward(Vector2.ZERO, deceleration * delta)
	velocity.x = flat.x
	velocity.z = flat.y

	# 5) Mover. move_and_slide() usa el delta de física internamente.
	#    velocity está en m/s: NO hacer velocity = velocity * delta.
	move_and_slide()

	# 6) Orientar el visual (arco más corto) + animación
	if direction != Vector3.ZERO:
		var target_yaw := -atan2(direction.x, direction.z)
		_model.rotation.y = lerp_angle(_model.rotation.y, target_yaw, 1.0 - exp(-turn_speed * delta))
	_update_animation()

func _unhandled_input(event: InputEvent) -> void:
	# Ratón: orbitar (solo con captura), zoom con rueda.
	if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
		var mm := event as InputEventMouseMotion
		_camera_pivot.rotation.x = clampf(
			_camera_pivot.rotation.x - mm.screen_relative.y * mouse_sensitivity,
			deg_to_rad(-tilt_limit), deg_to_rad(tilt_limit))
		_camera_pivot.rotation.y -= mm.screen_relative.x * mouse_sensitivity
	elif event is InputEventMouseButton:
		var mb := event as InputEventMouseButton
		if mb.pressed and mb.button_index == MOUSE_BUTTON_RIGHT:
			Input.mouse_mode = Input.MOUSE_MODE_CAPTURED
		if mb.pressed and mb.button_index == MOUSE_BUTTON_WHEEL_UP:
			_spring_arm.spring_length = maxf(_spring_arm.spring_length - 0.5, 2.0)
		if mb.pressed and mb.button_index == MOUSE_BUTTON_WHEEL_DOWN:
			_spring_arm.spring_length = minf(_spring_arm.spring_length + 0.5, 8.0)
	elif event is InputEventKey:
		var key := event as InputEventKey
		if key.pressed and key.keycode == KEY_ESCAPE:
			Input.mouse_mode = Input.MOUSE_MODE_VISIBLE

func _update_animation() -> void:
	# Máquina de estados: Idle / Move / Jump (nombres EXACTOS de los estados).
	var flat_speed := Vector2(velocity.x, velocity.z).length()
	var target := &"Idle"
	if not is_on_floor():
		target = &"Jump"
	elif flat_speed > 0.5:
		target = &"Move"
	if target != _current_state:
		_state_machine.travel(target)
		_current_state = target
	# Blend de locomoción: 0 = walk, 1 = run (BlendSpace1D dentro del estado Move).
	_anim_tree.set("parameters/Move/blend_position", clampf(flat_speed / run_speed, 0.0, 1.0))
```

Configuración del editor para que funcione:

1. `AnimationTree.anim_player` → path al `AnimationPlayer` hermano (ej. `../AnimationPlayer`).
2. En la state machine, transiciones con `cross_fade_time` ~0.1–0.25 s (Idle→Move→Jump→Idle).
3. `AnimationTree.callback_mode_process` = `ANIMATION_CALLBACK_MODE_PROCESS_PHYSICS` (se anima durante física: coherente con `move_and_slide`; docs: "especially useful when animating physics bodies").
4. El estado `Move` es un `AnimationNodeBlendSpace1D` con `walk` en 0 y `run` en 1.

## Implementación recomendada

Cuando el proyecto crece (principio de escalabilidad): **separar** controlador, cámara y tuning; añadir señales para otros sistemas.

### 1. Tuning como Resource (data-driven)

```gdscript
# character_tuning.gd
class_name CharacterTuning
extends Resource
## Tuning del personaje. Instanciar una copia .tres por tipo de personaje.

@export var walk_speed := 4.5
@export var run_speed := 7.5
@export var jump_velocity := 5.5
@export var gravity := 18.0
@export var ground_accel := 24.0
@export var air_accel := 10.0
@export var deceleration := 30.0
@export var turn_speed := 12.0
@export var coyote_time := 0.1
@export var jump_buffer := 0.12
```

`player.gd` usa `@export var tuning: CharacterTuning = preload("res://resources/player_tuning.tres")` en vez de los `@export` sueltos.

### 2. Cámara como nodo propio (`third_person_camera.gd` en `CameraPivot`)

```gdscript
# third_person_camera.gd — Godot 4.7 (APIs verificadas)
extends Node3D
## Cámara en tercera persona: orbita sobre este pivot.
## Ratón (captura + screen_relative) y stick derecho de gamepad.

signal captured_changed(captured: bool)

@export var mouse_sensitivity := 0.0035
@export var gamepad_sensitivity := 1.4      # rad/s por unidad de stick
@export_range(0.0, 89.0) var tilt_limit := 75.0
@export var min_length := 2.0
@export var max_length := 8.0
@export var zoom_step := 0.5

@onready var _spring_arm: SpringArm3D = %SpringArm3D

func _ready() -> void:
	_spring_arm.spring_length = max_length * 0.5

func _process(delta: float) -> void:
	# Stick derecho: pollear cada frame (no es un "evento").
	var stick := Vector2(Input.get_joy_axis(0, JOY_AXIS_RIGHT_X),
		Input.get_joy_axis(0, JOY_AXIS_RIGHT_Y))
	if stick.length() > 0.15:
		_rotate(stick * gamepad_sensitivity * delta)

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventMouseButton:
		var mb := event as InputEventMouseButton
		if mb.pressed and mb.button_index == MOUSE_BUTTON_RIGHT:
			set_captured(Input.mouse_mode != Input.MOUSE_MODE_CAPTURED)
		if mb.pressed and mb.button_index == MOUSE_BUTTON_WHEEL_UP:
			_zoom(-zoom_step)
		if mb.pressed and mb.button_index == MOUSE_BUTTON_WHEEL_DOWN:
			_zoom(zoom_step)
	elif event is InputEventKey:
		var key := event as InputEventKey
		if key.pressed and key.keycode == KEY_ESCAPE:
			set_captured(false)
	elif event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
		var mm := event as InputEventMouseMotion
		_rotate(Vector2(mm.screen_relative.x, mm.screen_relative.y) * mouse_sensitivity)

func _rotate(amount: Vector2) -> void:
	rotation.x = clampf(rotation.x - amount.y, deg_to_rad(-tilt_limit), deg_to_rad(tilt_limit))
	rotation.y -= amount.x

func _zoom(d: float) -> void:
	_spring_arm.spring_length = clampf(_spring_arm.spring_length + d, min_length, max_length)

func set_captured(captured: bool) -> void:
	Input.mouse_mode = Input.MOUSE_MODE_CAPTURED if captured else Input.MOUSE_MODE_VISIBLE
	captured_changed.emit(captured)

func is_clipping() -> bool:
	# Debug/UX: el brazo está comprimido por geometría.
	return _spring_arm.get_hit_length() < _spring_arm.spring_length - 0.05
```

El `player.gd` ya no contiene nada de cámara; lee `@onready var _camera_pivot := %CameraPivot` solo para el yaw del movimiento.

### 3. Eventos de juego (señales) + ataques con OneShot + root motion

```gdscript
# Añadir en player.gd (v2):
signal state_changed(from: StringName, to: StringName)
signal landed(fall_speed: float)
signal attack_started()

var _was_on_floor := true    # true si el jugador spawnea en suelo (evita emitir al empezar)
var _max_fall_speed := 0.0

# En _physics_process(), justo DESPUÉS de move_and_slide():
	if not is_on_floor():
		_max_fall_speed = maxf(_max_fall_speed, -velocity.y)
	elif not _was_on_floor:
		landed.emit(_max_fall_speed)   # m/s de caída acumulada durante el vuelo
	_max_fall_speed = 0.0
	_was_on_floor = is_on_floor()

# En _update_animation(), donde se llama a travel():
	if target != _current_state:
		_state_machine.travel(target)
		if _current_state != &"":
			state_changed.emit(_current_state, target)
		_current_state = target

# OneShot de ataque (nodo AnimationNodeOneShot "Attack" en el blend tree):
func start_attack() -> void:
	attack_started.emit()
	_anim_tree.set("parameters/Attack/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)

# Aborto (ej. al iniciar otro ataque):
# _anim_tree.set("parameters/Attack/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_ABORT)
```

**Root motion** (opcional; para animaciones cuyo desplazamiento de cadera debe mover al cuerpo — patrón oficial de `AnimationMixer.get_root_motion_position`):

```gdscript
# 1) En la AnimationTree: root_motion_track = path al track POSITION_3D del hueso (p. ej. "root").
# 2) En _physics_process(), en vez de (o junto a) la velocidad manual:
	var vel: Vector3 = _anim_tree.get_root_motion_position() / delta
	velocity = vel
	move_and_slide()
# Con rotación:
# set_quaternion(get_quaternion() * _anim_tree.get_root_motion_rotation())
# root_motion_local = true → el delta es en espacio local del nodo.
```

> El root motion coexiste con el control manual: típicamente lo usas solo para ataques/estocadas y devuelves el control de locomoción a la lógica de input al terminar (la animación de root motion es OneShot y vuelve al estado de locomoción).

### 4. Otros jugadores / enemigos con LOD

```gdscript
# enemy_lod.gd — ejemplo de LOD "custom" para un NPC lejano (híbrido con el nativo)
extends Node3D

@export var player: CharacterBody3D     # referenciado en el editor (o inyectado por código)
@export var lod_high_distance := 40.0   # más lejos de esto → LOD bajo
@export var hysteresis := 8.0           # banda muerta: evita flip-flop

@onready var _mesh: MeshInstance3D = %EnemyMesh

func _process(_delta: float) -> void:
	# Usar esto solo si el mecanismo nativo (visibility_range_*) no cubre el caso.
	if player == null:
		return
	var d := global_position.distance_to(player.global_position)
	if _mesh.cast_shadow == GeometryInstance3D.SHADOW_CASTING_SETTING_ON and d > lod_high_distance + hysteresis:
		_mesh.cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF
	elif _mesh.cast_shadow == GeometryInstance3D.SHADOW_CASTING_SETTING_OFF and d < lod_high_distance - hysteresis:
		_mesh.cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_ON
```

Preferir el mecanismo nativo cuando cubra el caso (sin script, sin coste por frame):

```gdscript
# Configurar UNA VEZ (editor o _ready) para NPCs/props:
_enemy_mesh.visibility_range_end = 80.0          # oculta más allá de 80 m
_enemy_mesh.visibility_range_end_margin = 10.0   # hysteresis / fade
_enemy_mesh.visibility_range_fade_mode = GeometryInstance3D.VISIBILITY_RANGE_FADE_DEPENDENCIES  # solo Forward+
```

## Ejemplo práctico

**Juego**: RPG 3D de exploración. Jugador con WASD + ratón (desktop) y gamepad (console):

1. **Nivel**: mundo estático con `StaticBody3D` + `WorldEnvironment`. Jugador instanciado en el nivel como nodo hijo (no autoload: un solo jugador, ciclo de vida con la escena).
2. **Movimiento**: `walk 4.5 / run 7.5 m/s`, gravedad 18 m/s², coyote 0.1 s → sensación responsive en plataformas; en pendientes >45° el cuerpo trata la superficie como muro (`floor_max_angle` default).
3. **Cámara**: orbita con ratón capturado (tilt limitado ±75°), zoom 2–8 m con rueda; en pasadizos el `SpringArm3D` se acorta al chocar (la pirámide de la cámara es la forma del cast); `is_clipping()` muestra aviso "geometría cerca" en el HUD.
4. **Animación**: glb con `idle/walk/run/jump/attack`; state machine Idle/Move/Jump + OneShot Attack; `blend_position` = velocidad corrida/7.5; transiciones 0.15 s. Al atacar (R ratón / X gamepad → `attack` en el Input Map): `travel` a "Move" se mantiene, el OneShot se dispara por encima (el OneShot no cambia el estado base).
5. **Aspecto**: `StandardMaterial3D` PBR; al entrar en zona de diálogo, `set_shader_parameter("highlight_amount", 0.6)` en un `ShaderMaterial` con outline suave (ver shader abajo).
6. **Rendimiento**: en la escena con 200 NPCs, el FPS cae de 144 a 96. Medición: `RENDER_TOTAL_DRAW_CALLS_IN_FRAME` sube de 800 → 2400. Optimización medida: LOD nativo (`visibility_range_end = 60` + `FADE_DEPENDENCIES` con `visibility_parent`) y `cast_shadow OFF` por distancia → draw calls 900, FPS 140. **Se midió antes y después** (regla de rendimiento).

### Shader de ejemplo (destacado de selección)

```glsl
// highlight.shader — Godot 4.x, shader_type spatial
shader_type spatial;

// instance uniform: permite un valor por enemigo con material compartido.
instance uniform float highlight_amount : range(0.0, 1.0) = 0.0;
uniform vec4 highlight_color : source_color = vec4(1.0, 0.85, 0.2, 1.0);

void fragment() {
	vec4 base = texture(TEXTURE, UV);
	ALBEDO = mix(base.rgb, highlight_color.rgb, highlight_amount);
	ALPHA = base.a;
}
```

```gdscript
# Desde GDScript, por instancia (GeometryInstance3D verificado en 4.7):
_enemy_mesh.set_instance_shader_parameter("highlight_amount", 0.6)
# ... o por material compartido (ShaderMaterial):
_enemy_mat.set_shader_parameter("highlight_amount", 0.6)
```

## Integración

### Escenas
- Instancia `player.tscn` en el nivel. **No** poner el jugador en un autoload (su ciclo de vida es el de la escena de juego).
- Si hay múltiples personajes (jugador + NPCs con el mismo rig), hereda o instancia la misma escena base y cambia `tuning` / material / animaciones por instancia.

### Nodos y scripts
- `player.gd` en el `CharacterBody3D` raíz; `third_person_camera.gd` en `CameraPivot` (o en el `Player` si prefieres un solo script en juegos pequeños).
- Nombres únicos para todo lo que se referencea por código (`%Model`, `%AnimationTree`, `%SpringArm3D`, `%Camera3D`).
- `NavigationAgent3D` como hijo del `Player` cuando el mundo tenga navegación (skill `godot-navigationagent3d`, pendiente).

### Señales (contratos hacia otros sistemas)
| Señal | Emite | Consume |
|---|---|---|
| `state_changed(from, to)` | player | HUD (ícono de estado), audio (pasos por estado), IA (el jugador está en Jump → no lo persigue) |
| `landed(fall_speed)` | player | Audio (sonido de aterrizaje), partícula de polvo, cámara (micro shake) |
| `attack_started()` | player | Combat system, audio, input (lock breve de movimiento) |
| `captured_changed(captured)` | camera | Cursor UI (mostrar/ocultar), pausas |
| `AnimationMixer.animation_finished(anim)` | anim tree | Gameplay (fín de stinger de ataque → reanudar control) |

### Resources
- `CharacterTuning` (Resource) por tipo de personaje → data-driven, sin tocar código.
- `AnimationLibrary` importada con el glb; la escena del jugador referencia las animaciones por nombre — al renombrar una animación, el BlendSpace se rompe: validar al importar.

### Otros sistemas
- **Combate**: `attack_started()` + OneShot; damage con `Area3D` (skill pendiente `godot-recipe-melee-combat`).
- **Misiones/HUD**: `state_changed` para UI contextual.
- **Save/load**: posicionar al jugador al cargar (`global_position`, `rotation.y` del `Model`); la cámara no se guarda (se re-deriva).
- **Multiplayer**: el rig se replica en remotos sin input local; cada remoto usa su propio `player.tscn` en modo "remote" (skill pendiente `godot-recipe-multiplayer-lobby`).

## Errores frecuentes

Ordenados por frecuencia en proyectos reales (casi todos verificados contra la docs 4.7):

1. **`velocity = velocity * delta`** — `velocity` está en m/s; multiplicar por delta lo convierte en metros. Advertencia literal en la docs de la propiedad. Sintoma: el personaje se mueve a una fracción minúscula de la velocidad esperada (y depende del FPS).
2. **`move_and_slide()` en `_process`** — la docs: debe ir en `_physics_process` porque usa el delta de física; en `_process` la simulación corre a velocidad incorrecta. Sintoma: movimiento que varía con el FPS / en pantallas de 120+ Hz.
3. **Conectar señales `floor_entered`/`wall_entered`/`moved`** — **no existen en CharacterBody3D 4.x** (verificado en 4.0 y 4.7; no hay sección de señales). Error: "signal not found" en runtime. Solución: polling de `is_on_floor()` en `_physics_process` (baratísimo) o `Area3D` para detección de zonas.
4. **Usar `floor_friction` / `wall_min_slide_speed`** — **no existen en 4.x** (verificado ausentes en 4.0 y 4.7; son de Godot 3). Sintoma: "Invalid get index 'floor_friction'". Solución: `floor_constant_speed = true` para velocidad uniforme en pendientes, `PhysicsMaterial3D` (friction) en el suelo para física de otros cuerpos, o amortiguación manual (`move_toward`, como en la implementación de esta skill).
5. **Buscar "smoothing" en `Camera3D`** — **no existe** en 4.0 ni 4.7 (verificado). Es de `Camera2D`. En 3D: damping manual con `lerp`/`exp(-k*delta)` (ver `third_person_camera.gd`).
6. **Conectar `Input.action_just_pressed` / `action_released`** — **no existen en 4.7** (la única señal del singleton es `joy_connection_changed`; verificado). Solución: `is_action_just_pressed()` o manejar el evento en `_unhandled_input`.
7. **`AnimationTree` como HIJO del `AnimationPlayer`** — `anim_player` es un `NodePath` relativo al `AnimationTree`; debe apuntar a un `AnimationPlayer` **hermano** (o en otra escena, con el patrón del demo TPS). Sintoma: la tree no reproduce nada, a veces sin error claro.
8. **`travel("estado")` con nombre incorrecto o sin transición** — los nombres de estado son EXACTOS (mayúsculas incluidas); con `state_machine_type = ROOT`, `travel` al nodo END sale de la máquina. Sintoma: la animación no cambia, o cambia y vuelve sola.
9. **`SpringArm3D` con cámara que NO es hija directa y sin `shape`** — el brazo cae a raycast, "impreciso para colisiones de cámara y no recomendado" (docs). Sintoma: la cámara atraviesa geometría fina. Solución: cámara hija directa (usa la pirámide del plano near) o `shape = SphereShape3D` pequeño.
10. **La cámara choca contra el propio jugador** — falta `add_excluded_object(player.get_rid())` y/o el `collision_mask` incluye la capa del jugador. Sintoma: la cámara pegada al cuerpo dentro de muros/cuerpos.
11. **Rotar el `CharacterBody3D` para orientar el personaje** — afecta a `up_direction`, slides y a `get_floor_normal`. Sintoma: se desliza al girar, se cae de pendientes. Solución: pivot `Model` hijo.
12. **Giro del modelo sin arco más corto** — `lerp(model.rotation.y, target, t)` con ángulos en radianes absolutos → el personaje hace giros de 350° al cruzar 0/2π. Solución: `lerp_angle(...)` (o `-atan2(x, z)` + `lerp_angle`, como en la skill).
13. **`is_action_just_pressed` para movimiento continuo** — es un evento (un solo frame). Para mantener velocidad usar `is_action_pressed` / `get_vector`; para saltar, `just_pressed`.
14. **`floor_snap_length = 0`** — el cuerpo no se pega a pendientes al bajar: "flota" y `is_on_floor()` parpadea. Default 0.1; típico 0.05–0.2.
15. **`safe_margin` demasiado alto (ej. 0.5)** — el personaje salta/pulsa al entrar en geometría. Default 0.001; subir a 0.01–0.05 solo si hay atascos reales (y medir).
16. **`AnimationTree` con `callback_mode_process = MANUAL`** (o `AnimationMixer.active = false`) — nada se anima; el editor (que avanza la tree en modo edit) puede ocultar el bug hasta ejecutar.
17. **Animar un physics body en IDLE con movimiento rápido** — la animación se aplica entre pasos de física → jitter de piernas/muñones contra el suelo. Solución: `callback_mode_process = PHYSICS`.
18. **Sensibilidad de cámara con `event.relative` en múltiples monitores/resoluciones** — usar `screen_relative` (recomendación del tutorial oficial: resolución-independiente).
19. **Múltiples `Camera3D` con `current = true`** — solo una puede estar activa; la última ganadora manda. Para transiciones, `clear_current(false)` + `make_current()`.
20. **Pollear `Input.get_joy_axis` en `_unhandled_input`** — los sticks no son eventos; pollearlos en `_process` (ver `third_person_camera.gd`).

## Anti-patrones

- **Giant script**: un `player.gd` de 1500 líneas con movimiento + cámara + combate + inventario + audio + save. → Separar por nodo/sistema (la escalada mínima→recomendada de esta skill es el mapa).
- **Giant autoload "GameManager"** que contiene la lógica del jugador. → El jugador vive en su escena; los autoloads son servicios (tiempo, audio, save), no personajes.
- **`get_node("A/B/C/D")` en cadena cada frame**. → `@onready` + nombres únicos.
- **Crear/destrozar nodos de cámara o `AnimationTree` por frame** (ej. "resetear" la cámara instanciando otra). → Mutar propiedades (`spring_length`, `rotation`) y reutilizar.
- **`Input.get_vector` cada frame en `_process` para saltos/jumps** (buffer propio manual sin límite de tiempo) → usar buffer acotado (`JUMP_BUFFER`).
- **Lerp de cámara sin `delta`** (`rotation += amount` con factor fijo) → dependiente de FPS; usar `1.0 - exp(-k * delta)` o `lerp_angle`.
- **`StandardMaterial3D` duplicado por enemigo** (duplicar el material solo para cambiar un color) → un material compartido + `set_instance_shader_parameter` (uniform por instancia) o `material_overlay`.
- **Cientos de `RayCast3D` por jugador para "sentir el suelo"** → `is_on_floor()` + `floor_snap_length` ya lo hacen dentro de `move_and_slide()`.
- **LOD con `visible = distance < X` sin hysteresis por frame** → flip-flop visual y pico de CPU; usar `visibility_range_*` nativo (con margen) o hysteresis.
- **`process_mode` / `_process` activo en enemigos lejanos sin razón** → `set_process(false)` por distancia (ver `enemy_lod.gd`) o `visibility_range`.
- **Optimizar el shader del personaje sin medir** (p. ej. "paso a un shader más barato porque parece lento") → medir con `TIME_FPS` / `RENDER_TOTAL_PRIMITIVES_IN_FRAME` antes y después; el shader del jugador puede no ser el cuello de botella.
- **Poner al jugador en un grupo y iterar `get_tree().get_nodes_in_group()` cada frame** para "encontrar al jugador" → referencia directa (`@onready var player := get_tree().root.get_node("Player")` una vez, o inyección por señal).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.** Nunca optimizar "porque parece más rápido".

### Herramientas oficiales verificadas (4.7)

1. **Editor → Debugger → Monitor**: los mismos monitores que la API `Performance` (abajo). Incluye FPS, tiempos de process/physics, draw calls, primitivas, memoria VRAM, objetos, physics 3D, navigation.
2. **Editor → Debugger → Profiler**: tiempo por función de GDScript (top callers).
3. **`Performance.get_monitor(monitor)`** (uso oficial documentado):

```gdscript
# Godot 4.7 (APIs verificadas). Los monitores pueden tener hasta 1 s de delay
# y algunos solo existen en debug (ver notas de la docs de Performance).
print("FPS: ", Performance.get_monitor(Performance.TIME_FPS))
print("ms/frame (process): ", Performance.get_monitor(Performance.TIME_PROCESS) * 1000.0)
print("ms/physics: ", Performance.get_monitor(Performance.TIME_PHYSICS_PROCESS) * 1000.0)
print("draw calls: ", Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME))
print("primitivas: ", Performance.get_monitor(Performance.RENDER_TOTAL_PRIMITIVES_IN_FRAME))
print("objetos render: ", Performance.get_monitor(Performance.RENDER_TOTAL_OBJECTS_IN_FRAME))
print("VRAM: ", Performance.get_monitor(Performance.RENDER_VIDEO_MEM_USED) / 1048576.0, " MB")
print("objetos físicos 3D: ", Performance.get_monitor(Performance.PHYSICS_3D_ACTIVE_OBJECTS))
print("pares de colisión: ", Performance.get_monitor(Performance.PHYSICS_3D_COLLISION_PAIRS))
```

   Monitores verificados (constantes de `Performance.Monitor`): `TIME_FPS(0)`, `TIME_PROCESS(1)`, `TIME_PHYSICS_PROCESS(2)`, `TIME_NAVIGATION_PROCESS(3)`, `MEMORY_STATIC(4, debug)`, `OBJECT_COUNT(7)`, `OBJECT_RESOURCE_COUNT(8)`, `OBJECT_NODE_COUNT(9)`, `OBJECT_ORPHAN_NODE_COUNT(10, debug)`, `RENDER_TOTAL_OBJECTS_IN_FRAME(11)`, `RENDER_TOTAL_PRIMITIVES_IN_FRAME(12)`, `RENDER_TOTAL_DRAW_CALLS_IN_FRAME(13)`, `RENDER_VIDEO_MEM_USED(14)`, `RENDER_TEXTURE_MEM_USED(15)`, `PHYSICS_3D_ACTIVE_OBJECTS(20)`, `PHYSICS_3D_COLLISION_PAIRS(21)`.
4. **Monitores propios** (aparecen en la pestaña Monitor del editor):

```gdscript
var _move_ms := 0.0

func _ready() -> void:
	Performance.add_custom_monitor("Player/MoveMs", _move_ms_monitor)

func _move_ms_monitor() -> float:
	return _move_ms * 1000.0

# En _physics_process(), medir el bloque de movimiento:
# var t0 := Time.get_ticks_usec()
# ... (bloque de movimiento) ...
# _move_ms = (Time.get_ticks_usec() - t0) / 1_000_000.0
```

5. **`RenderingServer.frame_drawn` / `frame_post_draw`** (señales) para instrumentar en torno al render; `Engine.get_frames_per_second()` para FPS en código.
6. **3D viewport → "visible collisions"** (gizmos del editor) para ver capas/shapes al depurar "no choca".

### Cuadro de diagnóstico (bottleneck)

| Síntoma medido | Causa probable | Qué hacer (medido) |
|---|---|---|
| `TIME_FPS` bajo + `TIME_PROCESS` alto | CPU en scripts | Profiler: top funciones; cache de nodos; evitar work por frame en inactivos |
| `TIME_FPS` bajo + `TIME_PHYSICS_PROCESS` alto | Física | `PHYSICS_3D_*`: pares/objetos; reducir shapes dinámicos; `continuous collision` solo donde toca; cuerpos dormidos |
| `TIME_FPS` bajo + process/physics bajos | GPU-bound | `RENDER_TOTAL_DRAW_CALLS_IN_FRAME` / `PRIMITIVES` / `VIDEO_MEM`; ver sección shaders/LOD |
| Draw calls altos | Muchos materiales/meshes | Batching (mismo material), `MultiMesh` (skill pendiente), reducir submallas, LOD nativo |
| `RENDER_VIDEO_MEM_USED` alto | Texturas | Compresión (VRAM), mipmaps, atlases (skill pendiente) |
| `OBJECT_ORPHAN_NODE_COUNT` crece (debug) | Fugas de nodos | `queue_free` / ownership; revisar spawn dinámico |
| Pipeline stutters al cargar | `PIPELINE_COMPILATIONS_*` | Compilación de pipelines (normal en primer arranque de build) |

### Costes específicos de este sistema (teóricos — medir antes de tocar)

- `move_and_slide()`: coste constante y bajo por tick; el coste crece con `max_slides` y la complejidad de la geometría colindante. No añadir raycasts propios "por si acaso".
- `SpringArm3D`: 1 motion cast por tick de física → despreciable.
- `AnimationTree`: CPU proporcional a la complejidad del blend tree y al número de tracks; mantener la locomoción simple (BlendSpace1D + estados) y evitar árboles profundos por personaje.
- Cámara: damping en `_process` → despreciable (unas cuantas operaciones por frame).
- Shader del personaje: coste **por píxel en pantalla** (el jugador ocupa mucha pantalla en 3ª persona: un fragment caro se paga caro). Overdraw de transparencias/outline dobla o tripla el coste. Medir con `TIME_FPS` antes/después del cambio de shader.
- LOD: `visibility_range_*` nativo no cuesta nada en script (lo resuelve el motor); un script de distancia por frame × N objetos sí cuesta.
- Memory: materiales/meshes/animaciones son Resources compartidos — instanciar la misma escena los comparte; duplicar recursos (p. ej. `resource.duplicate()`) multiplica VRAM/RAM.

## Debugging

Procedimiento sistemático (ver futura `godot-debugging-workflow`): reproducir → aislar → identificar subsistema → instrumentar → medir → hipótesis → corregir → verificar.

### "El personaje no se mueve"

```text
1. ¿La acción existe y está mapeada?  → Input Map (Project Settings). Prueba:
   print(Input.get_action_strength("move_forward"))  # debe ser 1 al pulsar W
2. ¿El código lee la acción?          → print(input_vec) en _physics_process.
3. ¿Colisiona con el mundo?           → capas: CollisionShape3D del jugador (layer 2,
   mask 1) vs StaticBody3D del suelo (layer 1). "visible collisions" en el 3D viewport.
4. ¿Está dentro de geometría?         → safe_margin bajo + inicio dentro de un muro;
   mover la escena de spawn.
5. ¿velocity * delta?                 → revisar el error #1.
6. ¿move_and_slide en _process?       → revisar el error #2.
7. ¿is_on_floor() false permanente?   → floor_snap_length, up_direction, floor_max_angle.
8. ¿motion_mode FLOATING?             → en ese modo no hay "suelo".
```

### "La cámara atraviesa muros"

```text
1. ¿SpringArm.collision_mask incluye la capa del mundo? (default = solo capa 1)
2. ¿Camera3D es HIJO DIRECTO del SpringArm (o hay shape)?
3. ¿margin = 0? → subirla a 0.01–0.05 (la cámara queda en el punto exacto de colisión).
4. Debug: print(_spring_arm.get_hit_length(), _spring_arm.spring_length)
   → si hit < length, hay colisión activa (correcta). Si hit == length siempre y se
     atravasa → el cast no toca nada (capas) o la forma es raycast (hijo no directo).
```

### "La animación no cambia"

```text
1. ¿anim_player apunta a un AnimationPlayer HERMANO con las animaciones? (pestaña AnimationTree)
2. ¿Los nombres travel() == nombres de los estados (exactos)?
   print(_state_machine)  # ¿es null? → "parameters/playback" no existe (tree_root mal)
3. ¿Existe la transición entre estados? (grafo de la state machine)
4. ¿callback_mode_process = MANUAL? → poner PHYSICS/IDLE.
5. ¿AnimationMixer.active = false?
6. ¿La animación importada es loop / se importó? (AnimationPlayer: lista de animaciones)
```

### "Cae a través del suelo / flota"

```text
1. floor_snap_length = 0 → poner 0.1 (default).
2. Gravedad demasiado baja / se anula velocity.y en tierra.
3. Plataforma móvil: platform_floor_layers no incluye la capa de la plataforma.
4. Geometría del suelo sin colisión (StaticBody3D sin CollisionShape3D).
```

### "La cámara rota al personaje / el personaje rota con la cámara"

```text
1. ¿El script escribe rotation.y del CharacterBody3D? → moverlo al pivot %Model.
2. ¿CameraPivot es hijo del Model? → debe ser hijo del Player (que no rota).
```

### Errores de consola típicos

| Error | Causa |
|---|---|
| `Node not found: "%Model"` | Falta el flag "unique" en el nodo, o el script está en otra escena. |
| `AnimationTree: invalid path "parameters/playback"` | tree_root sin state machine. |
| `Invalid get index 'floor_friction'` | API de Godot 3 (error #4). |
| `Signal 'floor_entered' not found` | API inexistente en 4.x (error #3). |
| `Non-uniform scale on CharacterBody3D` | Escalado no uniforme: usar scale uniforme y redimensionar el shape. |

## Compatibilidad

- **Verificado: Godot 4.7 (stable)** — toda la API de esta skill existe en 4.7 (ver *Referencias oficiales*).
- **4.3+ (compatibilidad alta, con matices verificados por versión)**:
  - `AnimationTree` hereda de `AnimationMixer` y `callback_mode_process` existe (verificado en 4.7; la refactorización AnimationMixer y las diferencias 4.0→4.3 están cubiertas por el artículo oficial *"Migrating Animations from Godot 4.0 to 4.3"*).
  - En 4.0–4.2 el equivalente a `callback_mode_process` es `AnimationTree.set_process_callback(...)` (funcional; **deprecado en 4.7**).
  - `AnimationNodeStateMachine.state_machine_type` (verificado en 4.7); en versiones anteriores la prop equivalente tiene otro nombre — consultar el artículo de migración oficial.
- **LOD por superficie de mesh** (`lods` en `ArrayMesh.add_surface_from_arrays`): verificado en 4.7; en 4.x anteriores verificar por versión antes de usarlo.
- **No existe nodo `LOD` core en 4.7** (verificado) — si un tutorial menciona un nodo "LOD", es de un add-on de AssetLib, no del motor.
- **Godot 3 → 4 (migración)**: `KinematicBody3D` → `CharacterBody3D`; `move_and_slide(safe_margin)` → `move_and_slide()` (safe_margin es propiedad); `floor_friction`/`wall_min_slide_speed` no existen en 4.x; signals de `Input` eliminados; `Input.MOUSE_MODE_*` renombrados respecto a 3.x.
- **Plataformas**:
  - **Windows / Linux / macOS**: objetivo principal; todo verificado por docs.
  - **Android**: sin ratón → orbitar con touch (drag) y joystick virtual (skills `godot-input` y `godot-mobile` pendientes); verificar `emulate_touch_from_mouse` para testear en desktop.
  - **Web**: pointer lock con restricciones de navegador; rendimiento muy limitante (considerar renderer Compatibility y `rendering` settings reducidos — skill pendiente `godot-web`).

## Dependencias

Skills que esta compuesta integra (todas **PENDIENTE** en Lote 0; hasta su creación, esta skill contiene el contenido operativo integrado de cada una):

| ID | Categoría | Estado | Contenido que aporta aquí |
|---|---|---|---|
| `godot-characterbody3d` | 3D · Física | Pendiente | Cuerpo, `move_and_slide`, suelo/paredes/techo, propiedades de slide |
| `godot-character-controller` | 3D · Characters | Pendiente | Aceleración, fricción, salto, coyote, buffer, movimiento relativo a cámara |
| `godot-input` | Input | Pendiente | Input Map, `get_vector`, `just_pressed`, mouse/gamepad, captura |
| `godot-animationtree` | Animation | Pendiente | State machine, BlendSpace, OneShot, root motion, `callback_mode_process` |
| `godot-camera3d` | 3D · Cámaras | Pendiente | `Camera3D`, `SpringArm3D`, orbit, colisión, zoom, (falta smoothing nativo) |
| `godot-shader-spatial` | Shaders | Pendiente | `shader_type spatial`, uniforms, per-instance uniforms, `StandardMaterial3D` |
| `godot-rendering-performance` | Performance | Pendiente | Monitores `Performance`, diagnóstico CPU/GPU/VRAM, "measure first" |
| `godot-lod` | 3D · Performance | Pendiente | `visibility_range_*`, `lod_bias`, mesh LOD, sin nodo LOD core |

## Skills relacionadas

- `godot-recipe-third-person-controller` (receta pura de movimiento, sin cámara/anim) — pendiente
- `godot-recipe-first-person-controller` — pendiente
- `godot-navigationagent3d` (NPCs que persiguen a este jugador) — pendiente
- `godot-physics` (capas, tipos de cuerpo, troubleshooting general) — disponible (Lote 4)
- `godot-physics-materials` (fricción/rebote) — disponible (Lote 4)
- `godot-raycast3d` / `godot-area3d` (queries y detección) — disponibles (Lote 4)
- `godot-error-character-not-moving` (árbol de diagnóstico dedicado) — pendiente
- `godot-error-collision-not-working` — pendiente
- `godot-error-animation-not-playing` — pendiente
- `godot-root-motion` (profundización; aquí solo el patrón mínimo) — pendiente

## Referencias oficiales

Todas verificadas accesibles en la rama `stable` (4.7) el 2026-09-15:

- CharacterBody3D: https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html
- Camera3D: https://docs.godotengine.org/en/stable/classes/class_camera3d.html
- SpringArm3D: https://docs.godotengine.org/en/stable/classes/class_springarm3d.html
- Tutorial "Third-person camera with spring arm": https://docs.godotengine.org/en/stable/tutorials/3d/spring_arm.html
- Input: https://docs.godotengine.org/en/stable/classes/class_input.html
- AnimationTree: https://docs.godotengine.org/en/stable/classes/class_animationtree.html
- AnimationMixer: https://docs.godotengine.org/en/stable/classes/class_animationmixer.html
- AnimationNodeStateMachine: https://docs.godotengine.org/en/stable/classes/class_animationnodestatemachine.html
- Tutorial "Using AnimationTree": https://docs.godotengine.org/en/stable/tutorials/animation/animation_tree.html
- GeometryInstance3D (LOD/visibilidad/materials): https://docs.godotengine.org/en/stable/classes/class_geometryinstance3d.html
- ArrayMesh (LOD por superficie): https://docs.godotengine.org/en/stable/classes/class_arraymesh.html
- Mesh: https://docs.godotengine.org/en/stable/classes/class_mesh.html
- Performance (monitores): https://docs.godotengine.org/en/stable/classes/class_performance.html
- Tutorial "First 3D Game — Moving the player with code": https://docs.godotengine.org/en/stable/getting_started/first_3d_game/03.player_movement_code.html
- Artículo oficial "Migrating Animations from Godot 4.0 to 4.3": https://godotengine.org/article/migrating-animations-from-godot-4-0-to-4-3/
- Demo oficial TPS (patrón AnimationTree + personaje): https://godotengine.org/asset-library/asset/2710
- Demo oficial Platformer 3D: https://godotengine.org/asset-library/asset/2748
- Demo oficial Kinematic Character 3D: https://godotengine.org/asset-library/asset/2739
