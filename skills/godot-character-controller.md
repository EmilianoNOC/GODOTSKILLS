# godot-character-controller

## ID

`godot-character-controller`

## Categoría

3D · Characters · Base (patrón)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la class reference oficial 4.7 (`docs.godotengine.org/en/stable/`) y el XML de clases de la rama `4.7-stable`.

## Confidence

**HIGH** para la API usada (verificada en 4.7).
**MEDIUM** para todos los valores de tuning (velocidades, aceleraciones, tiempos de coyote/buffer, sensaciones) — son recomendaciones prácticas del patrón, **no** especificación oficial.
**"Custom implementation using Godot primitives"**: el wall slide y el air control son implementación propia construida con la API nativa (Godot no trae wall slide para `CharacterBody3D`).

## Nivel

**BASE** (requiere `godot-characterbody3d`).

## Propósito

Proporcionar el **patrón de control de personaje** en 3D: convertir input (teclado/ratón/gamepad) en `velocity` de un `CharacterBody3D` con sensación ajustable — aceleración, fricción, salto con coyote time y jump buffer, control en el aire, movimiento relativo a cámara, y wall slide opcional — manteniendo la sensación consistente entre dispositivos y plataformas.

## Cuándo utilizarla

- Construir el "feel" de movimiento de un personaje 3D (caminar/correr/saltar/volar opcional).
- Unificar sensación teclado + gamepad (mismos tiempos, mismos arcos de aceleración).
- Añadir coyote time / jump buffer (estándar de la industria en plataformas).
- Implementar wall slide / wall jump (custom, pero con primitivas nativas verificadas).

## Cuándo NO utilizarla

- **Personaje simple que solo camina** → la *implementación mínima* de `godot-characterbody3d` basta (mínima complejidad primero).
- **Movimiento 6DOF puro (espacio)** → `motion_mode = MOTION_MODE_FLOATING` + control directo; el patrón de suelo/salto de esta skill no aplica.
- **NPCs guiados por navmesh** → `NavigationAgent3D` (pendiente); el tuning de feel es distinto (velocidad constante, sin input).
- **Vehículos** → `VehicleBody3D` (pendiente).

## Conceptos fundamentales

### Separación: input → dirección → velocidad → cuerpo

```text
Input (acciones)        → Vector2 de dirección (normalizado)
Dirección + cámara      → Vector3 mundial de destino (yaw de cámara)
Destino + tuning        → target_speed (walk/run)
target_speed + aceleración → velocity.x/z (move_toward con delta)
velocity                → move_and_slide()  (godot-characterbody3d)
```

Cada capa es independiente: cambiar dispositivos no toca el tuning; cambiar el tuning no toca el input; cambiar el body no toca ninguno de los dos.

### Tiempos acotados (coyote + buffer)

- **Coyote time**: ventana (típico 0.1 s) tras saltar/salir de un borde durante la cual el salto aún funciona. Se mide con un timer que se reinicia cuando `is_on_floor()`.
- **Jump buffer**: ventana (típico 0.12 s) durante la cual un "saltar" pulsado justo antes de tocar suelo se "recuerda" y se ejecuta al aterrizar.
- Ambos son **implementación propia** (Godot no los trae): dos timers flotantes. MEDIUM (valores de feel).

### Movimiento relativo a cámara (TPS)

El vector de input se rota por el **yaw de la cámara** (solo rotación Y). El tilt (pitch) de la cámara **no** entra: con la cámara mirando hacia arriba, "adelante" sigue siendo el suelo.

### Wall slide (custom)

Godot no trae wall slide. Patrón propio: detectar `is_on_wall()` + normal de la pared + input empujando contra la pared → limitar `velocity.y` a un máximo (ej. -2 m/s) + opcional wall jump con impulso `wall_normal * k`. Todo con la API nativa verificada (`is_on_wall()`, `get_wall_normal()`, `Input.get_axis`).

## API relevante

Todo verificado en 4.7 (en `godot-characterbody3d` el detalle completo del body):

| API | Uso en el controlador |
|---|---|
| `CharacterBody3D.velocity` | La escribimos (m/s) y la leemos tras `move_and_slide()`. |
| `CharacterBody3D.move_and_slide()` | Único punto de movimiento. `_physics_process`. |
| `is_on_floor()` / `is_on_wall()` | Estados para coyote, buffer, wall slide. |
| `get_floor_normal()` | Orientar el modelo en rampas; detectar "casi en borde". |
| `get_wall_normal()` | Dirección del wall jump. |
| `get_floor_angle()` | Opcional: reducir velocidad en rampas empinadas. |
| `floor_max_angle` (45°) | Límite rampa vs muro — condiciona el diseño de niveles. |
| `floor_constant_speed` | `true` = misma velocidad en rampas (recomendado para TPS). |
| `floor_snap_length` (0.1) | No tocar salvo debug de "flota en rampas". |
| `Input.get_vector(neg_x, pos_x, neg_y, pos_y, deadzone)` | Stick/WASD unificados (`deadzone = -1` → la del proyecto). |
| `Input.is_action_just_pressed` / `is_action_pressed` | Salto (evento) / esprintar (continuo). |
| `Input.get_action_strength` | Análogos (triggers) si se mapean. |
| `Input.get_joy_axis(device, axis)` | Stick derecho de cámara (en `_process`; ver `godot-input`). |
| `InputEventMouseMotion.screen_relative` | Orbitar con ratón capturado (resolución-independiente; recomendado por el tutorial oficial). |

**No existe (verificado 4.0 y 4.7):** señales de floor/wall/ceiling en `CharacterBody3D` (polling), `floor_friction`/`wall_min_slide_speed` (3.x), smoothing de cámara (es de `Camera2D`).

## Arquitectura recomendada

```text
Player (CharacterBody3D)  [player.gd — SOLO movimiento: usa CharacterController.gd]
├── CharacterTuning (Resource)  ← data-driven: un .tres por tipo de personaje
└── (cámara, anim, model: ver godot-third-person-character)
```

Decisiones:

1. **Tuning como `Resource`**: el feel vive en datos (editable en editor, por personaje, sin tocar código).
2. **Un solo script de movimiento** por jugador; cámara y animación en scripts propios (la compuesta Lote-0 muestra la escalada).
3. **Estado mínimo** (5 timers/flags): `_time_since_floor`, `_time_since_jump`, `_was_on_floor`, `_max_fall_speed`, `_wall_slide_direction`. Nada más.
4. **Señales hacia afuera** (`landed(fall_speed)`, `state_changed(from, to)`, `wall_slide_changed(on)`) — el controlador **informa**, no controla (HUD/audio/IA reaccionan).

## Implementación mínima

```gdscript
# character_tuning.gd
class_name CharacterTuning
extends Resource
## Tuning de movimiento (MEDIUM: valores de feel, no oficiales).

@export var walk_speed := 4.5          # m/s
@export var run_speed := 7.5           # m/s
@export var jump_velocity := 5.5       # m/s
@export var gravity := 18.0            # m/s²
@export var ground_accel := 24.0       # m/s²
@export var air_accel := 10.0          # m/s² (control en el aire)
@export var ground_decel := 30.0       # m/s² (frenado en suelo)
@export var air_decel := 2.0           # m/s² (inercia en el aire)
@export var coyote_time := 0.1         # s
@export var jump_buffer := 0.12        # s
@export var max_fall_speed := 26.0     # m/s (terminal)
@export var wall_slide_speed := 2.0    # m/s de caída max en wall slide
@export var wall_jump_impulse := 6.0   # m/s perpendicular a la pared
```

```gdscript
# character_controller.gd — Godot 4.7 (APIs verificadas)
extends Node
## Controlador de personaje: input → velocity de un CharacterBody3D.
## Colgar en un nodo hijo del body (o usarlo como base del script del body).

@export var tuning: CharacterTuning = preload("res://resources/tuning_default.tres")

signal landed(fall_speed: float)
signal wall_slide_changed(on: bool)

@onready var body: CharacterBody3D = get_parent() as CharacterBody3D

var _time_since_floor := 10.0
var _time_since_jump := 10.0
var _was_on_floor := true
var _max_fall_speed := 0.0
var _wall_sliding := false

# --- Inyectar desde fuera (cámara) ---
var camera_yaw_provider: Callable   # () -> float : yaw de la cámara (rad)

func _ready() -> void:
	if camera_yaw_provider.is_empty():
		camera_yaw_provider = func(): return 0.0

func _physics_process(delta: float) -> void:
	# 1) Estado de suelo + timers acotados
	if body.is_on_floor():
		_time_since_floor = 0.0
	else:
		_time_since_floor += delta

	# 2) Gravedad (con velocidad terminal)
	if not body.is_on_floor():
		body.velocity.y = maxf(body.velocity.y - tuning.gravity * delta, -tuning.max_fall_speed)

	# 3) Salto: buffer + coyote
	_time_since_jump += delta
	if Input.is_action_just_pressed("jump"):
		_time_since_jump = 0.0
	if _time_since_jump <= tuning.jump_buffer and _time_since_floor <= tuning.coyote_time:
		body.velocity.y = tuning.jump_velocity
		_time_since_jump = 10.0
		_time_since_floor = 10.0

	# 4) Input → dirección mundial (yaw de cámara)
	var input_vec := Input.get_vector("move_left", "move_right", "move_forward", "move_back")
	var direction := Vector3.ZERO
	if input_vec != Vector2.ZERO:
		var yaw := Basis.from_euler(Vector3(0.0, camera_yaw_provider.call(), 0.0))
		direction = (yaw * Vector3(input_vec.x, 0.0, input_vec.y)).normalized()

	# 5) Velocidad horizontal: acelera/frena hacia el objetivo
	var target_speed := tuning.run_speed if Input.is_action_pressed("sprint") else tuning.walk_speed
	var flat := Vector2(body.velocity.x, body.velocity.z)
	var target := Vector2(direction.x, direction.z) * target_speed
	var accel := tuning.ground_accel if body.is_on_floor() else tuning.air_accel
	var decel := tuning.ground_decel if body.is_on_floor() else tuning.air_decel
	if direction != Vector3.ZERO:
		flat = flat.move_toward(target, accel * delta)
	else:
		flat = flat.move_toward(Vector2.ZERO, decel * delta)
	body.velocity.x = flat.x
	body.velocity.z = flat.y

	# 6) Wall slide (custom)
	_update_wall_slide(input_vec)

	# 7) Mover
	body.move_and_slide()

	# 8) Eventos post-slide
	if not body.is_on_floor():
		_max_fall_speed = maxf(_max_fall_speed, -body.velocity.y)
	elif not _was_on_floor:
		landed.emit(_max_fall_speed)
	_max_fall_speed = 0.0
	_was_on_floor = body.is_on_floor()

func _update_wall_slide(input_vec: Vector2) -> void:
	# Custom implementation using Godot primitives (no existe wall slide nativo).
	var sliding := false
	if body.is_on_wall() and not body.is_on_floor() and input_vec.length() > 0.1:
		var wn := body.get_wall_normal()
		# Empujando contra la pared (input opuesto a la normal):
		var push := Vector2(-wn.x, -wn.z).normalized()
		if push.dot(input_vec.normalized()) > 0.7:
			sliding = true
			body.velocity.y = maxf(body.velocity.y, -tuning.wall_slide_speed)
	if sliding != _wall_sliding:
		_wall_sliding = sliding
		wall_slide_changed.emit(sliding)

	# Wall jump (opcional; llamarlo desde un "jump" mientras wall_sliding):
	# var wn := body.get_wall_normal()
	# body.velocity = body.velocity + Vector3(wn.x, 0.0, wn.z) * tuning.wall_jump_impulse
	# body.velocity.y = tuning.jump_velocity
```

Configuración: accionamiento de la cámara por `camera_yaw_provider` (p. ej. `func(): return $CameraPivot.rotation.y`).

## Implementación recomendada

1. **Esprintar como "recursos" opcionales**: el patrón de arriba usa `is_action_pressed("sprint")` (mantenido). Para esprint con stamina, añadir un `StaminaResource` y exponer `can_sprint()` — el controlador pregunta, no sabe.
2. **Aire: inercia** (`air_decel` bajo): el jugador conserva dirección en el aire; `air_accel` da control parcial. Este par (inercia vs control) es EL dial de feel del air control.
3. **Sensación de peso**: gravedad alta (18–25 m/s²) + `max_fall_speed` alto → caídas rápidas, plataformas ágiles. Gravedad baja (8–12) → floaty (estética deliberada en algunos juegos).
4. **Pendientes**: `floor_constant_speed = true` en el body para que la velocidad sea la misma en rampas; opcionalmente escalar `target_speed` por `floor_angle` (p. ej. ×0.85 en rampas >30°).
5. **Tres "presets" de feel** (MEDIUM):
   - *Responsive* (plataforma): walk 5, run 8, coyote 0.12, buffer 0.15, air_accel 14.
   - *Pesado* (acción): walk 3.5, run 6, coyote 0.08, buffer 0.1, air_accel 7.
   - *Corredor* (estilo Mirror's Edge): run 9, air_accel 12, `floor_constant_speed = true`.

## Ejemplo práctico

**Juego**: TPS de acción. Jugador en un nivel con rampas de 30° y muros de 3 m:

1. **Sensación**: walk 4.5 / run 7.5, coyote 0.1, buffer 0.12. El jugador pulsa saltar 80 ms antes de llegar al borde: **salta** (coyote). Pulsa 100 ms antes de aterrizar en una bala: **salta al tocar** (buffer).
2. **Rampas**: `floor_constant_speed = true` → el run es 7.5 m/s en plano Y en rampa (sin "acelerar" al subir).
3. **Wall slide** en el muro de 3 m: empujando contra la pared, la caída se limita a -2 m/s (`_update_wall_slide`); wall jump con impulso 6 m/s perpendicular → se sube. Señal `wall_slide_changed(true)` → UI muestra "¡Muro!".
4. **Debug de feel**: el diseñador edita `tuning_default.tres` en vivo (reinicio de escena) sin tocar código — por eso el tuning es `Resource`.

## Integración

- **Input**: `godot-input` — Input Map con `move_left/right/forward/back`, `jump`, `sprint` (teclado + gamepad + touch).
- **Cámara**: `godot-camera3d` — el yaw de la cámara se inyecta por `camera_yaw_provider` (el controlador no busca la cámara en el árbol).
- **Animación**: `godot-animationtree` — consume `landed()`, velocidad plana y `wall_slide_changed()` para estados (Move/Jump/WallSlide).
- **Body**: `godot-characterbody3d` — el controlador escribe `velocity` y lee el estado.
- **Audio/HUD**: señales del controlador (landed, wall_slide) → sonidos, partículas, UI.

## Errores frecuentes

1. **`velocity * delta` en el tuning** — el controlador entero se rompe (ver `godot-characterbody3d` error #1).
2. **Coyote/buffer sin límite de tiempo** (flag booleano "pulsé saltar" que nunca se limpia) → el jugador puede saltar 3 s después de la acción. Si siempre hay un timer flotante, hay límite.
3. **Movimiento con el pitch de la cámara** (usar la dirección completa de la cámara en vez del yaw) → al apuntar hacia arriba, "adelante" se va al cielo. Solución: `Basis.from_euler((0, yaw, 0))`.
4. **Air control idéntico al ground** (`air_accel == ground_accel`) → el personaje "frena en el aire" como un camión (o no frena, según el gusto): elegir `air_accel < ground_accel` deliberadamente.
5. **Wall slide sin comprobar que empujas contra la pared** → el personaje entra en wall slide al rozar un muro de paso (sensación de "pegajoso"). El dot > 0.7 contra la normal evita esto.
6. **`Input.get_vector` en `_process` y `move_and_slide` en `_physics_process`** → la dirección y el movimiento van en ticks distintos: jitter en 120 Hz. Leer input de movimiento en `_physics_process` (los `just_pressed` funcionan por paso de física).
7. **Coyote que se reinicia al aterrizar Y al estar en suelo** (doble condición) → el timer nunca envejece en el borde. La lógica correcta: `is_on_floor() → 0.0` (una línea), nada más.
8. **`sprint` con `is_action_just_pressed`** → el esprint dura un frame. Es continuo: `is_action_pressed`.
9. **Esperar `wall_entered` para el wall slide** — no existe (verificado). Polling `is_on_wall()` en `_physics_process`.
10. **Ajustar el feel con 40 constantes sueltas por script** → el tuning es `Resource` (arriba); 40 `@export` por clase de personaje = imposible de comparar/iterar.

## Anti-patrones

- **`Input.is_action_pressed` para saltar** → salta cada frame mientras se mantiene pulsado (el jugador "flota" saltando en bucle). Salto = `just_pressed` + buffer.
- **Girar el `CharacterBody3D` hacia la cámara/dirección** → rompe slides (ver `godot-characterbody3d` error #9). El visual rota en un pivot hijo.
- **Lerps de feel sin `delta`** (`velocity.x = lerp(velocity.x, target, 0.1)` con factor fijo) → depende de la Hz de la física; usar `move_toward` con aceleración × delta.
- **Un controlador distinto por personaje** (jugador, enemigo, vehiculo-caminante) duplicando 200 líneas → una clase de controlador + `Resource` de tuning por tipo.
- **Wall jump sin cooldown/buffer** → spam de wall jumps al rozar el muro; usar el mismo jump buffer del salto.
- **Medir el "feel" a ojo** → el feel es medible: grabar el tiempo de 0→run, la distancia de salto y el arco de la parábola (m/s y s) y comparar presets con números.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- El controlador es **CPU despreciable** (unas decenas de operaciones por tick de física). No es un lugar de optimización.
- El coste real del "personaje" está en: animación (ver `godot-animationtree`), shader por píxel (ver `godot-shader-spatial`), y en N instancias (ver `godot-lod` / `godot-rendering-performance`).
- Si `TIME_PHYSICS_PROCESS` crece con N jugadores: revisar pares de colisión (`PHYSICS_3D_COLLISION_PAIRS`) y `max_slides`, no el tuning.

```gdscript
# Verificación de que el movimiento no es el problema (4.7, APIs verificadas):
print("ms/physics: ", Performance.get_monitor(Performance.TIME_PHYSICS_PROCESS) * 1000.0)
print("pares: ", Performance.get_monitor(Performance.PHYSICS_3D_COLLISION_PAIRS))
```

## Debugging

### "El salto no se siente bien / salta doble"

```text
1. ¿El salto se dispara por just_pressed y no por pressed?      (error #10 de antipatrones)
2. ¿El buffer no se consume al saltar?                         (_time_since_jump = 10.0 tras saltar)
3. ¿El coyote se reinicia fuera de is_on_floor()?              (error #7)
4. ¿Dos scripts saltando (controlador + un "quick fix" aparte)? → un único punto de salto.
```

### "En el aire el personaje frena / no responde"

```text
1. ¿air_decel demasiado alto? → inercia baja (deliberado o no)
2. ¿air_accel = 0?            → sin control aéreo
3. ¿La dirección usa el pitch de la cámara? → error #3
```

### "Wall slide disparado a lo tonto"

```text
1. ¿Falta el dot contra la normal?  → error #5
2. ¿is_on_wall() en el borde de una rampa? (get_wall_normal ~ horizontal) → exigir
   input_vec.length() > 0.1 y no estar en suelo (ya está en el patrón).
```

### "Sensación distinta entre teclado y gamepad"

```text
1. ¿El stick tiene deadzone? → Input.get_vector con deadzone=-1 usa la del proyecto;
   verificar Project Settings → Input Devices → Joypads (deadzone 0.5 típico).
2. ¿El esprint gamepad es un trigger análogo? → is_action_pressed lo binariza (correcto).
3. ¿Velocidad del stick vs teclado? → get_vector normaliza (el teclado da 1.0).
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — `move_and_slide`, `velocity`, `is_on_floor/is_on_wall`, `get_wall_normal`, `floor_*`, `Input.*` verificados en 4.7 (y contraste 4.0 en `GODOT_VERSION_MATRIX.md`).
- **El patrón es estable entre 4.0–4.7**: la API de body no cambió entre ambas ramas (verificado).
- **3 → 4**: no usar `floor_friction`/`wall_min_slide_speed` (3.x); el wall slide de 3.x (que usaba `wall_min_slide_speed`) no tiene equivalente nativo en 4.x → custom como aquí.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-characterbody3d` | **Requerida** (body, `move_and_slide`, estados) |
| `godot-input` | **Requerida** (acciones, `get_vector`, `just_pressed`) |
| `godot-camera3d` | Para inyectar el yaw de la cámara (TPS) |
| `godot-animationtree` | Consumidor de `landed()`/`wall_slide_changed()` |
| `godot-third-person-character` | Compuesta que integra este controlador (Lote 0) |

## Skills relacionadas

- `godot-recipe-first-person-controller` — variante FPS (pendiente)
- `godot-recipe-air-control` — profundización en inercia/control aéreo (pendiente)
- `godot-error-jump-feels-off` — diagnóstico de feel (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- CharacterBody3D (todo el body y el contrato `move_and_slide`): https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html
- Input (get_vector, just_pressed, get_action_strength, get_joy_axis): https://docs.godotengine.org/en/stable/classes/class_input.html
- InputMap (deadzone de acciones): https://docs.godotengine.org/en/stable/classes/class_inputmap.html
- InputEventMouseMotion (`screen_relative`): https://docs.godotengine.org/en/stable/classes/class_inputeventmousemotion.html
- Tutorial "First 3D Game — Moving the player with code" (patrón velocity + move_and_slide): https://docs.godotengine.org/en/stable/getting_started/first_3d_game/03.player_movement_code.html
- Demo oficial Kinematic Character 3D (ejemplo de feel en el engine): https://godotengine.org/asset-library/asset/2739
- Demo oficial Platformer 3D: https://godotengine.org/asset-library/asset/2748
