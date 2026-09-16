# godot-characterbody3d

## ID

`godot-characterbody3d`

## Categoría

3D · Física · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra:

- Class reference oficial: `docs.godotengine.org/en/stable/` (rama 4.7)
- XML de clases del motor, rama `4.7-stable` de `godotengine/godot` (`CharacterBody3D.xml`, `CollisionObject3D.xml`)
- Contraste 4.0 vs 4.7 documentado en `GODOT_VERSION_MATRIX.md`

## Confidence

**HIGH** — toda la API usada está verificada directamente en la documentación oficial 4.7 (incluido el contraste con 4.0).
Excepciones:

- **MEDIUM**: valores de tuning (rango típico de `safe_margin`, `floor_snap_length`).
- **"No existe (verificado)"**: APIs que **NO existen** en 4.x y que esta skill documenta explícitamente para que el agente no las invente.

## Nivel

**BASE** (prerrequisito de `godot-character-controller`, `godot-recipe-third-person-controller`, `godot-third-person-character`).

## Propósito

Proporcionar al agente el conocimiento operativo completo de `CharacterBody3D` en Godot 4.x: el cuerpo físico kinemático para personajes, el ciclo `velocity` + `move_and_slide()`, detección de suelo/pared/techo, propiedades de slide, capas de colisión heredadas y los límites reales de la API (qué no existe y qué alternativas usar).

## Cuándo utilizarla

- Crear o modificar cualquier personaje/entidad que se mueva colisionando con el mundo en 3D.
- Depurar "el personaje no se mueve / se atasca / flota / atraviesa".
- Configurar capas y máscaras de colisión para un cuerpo kinemático.
- Entender qué información devuelve `move_and_slide()` (suelo, pared, colisión de slide).

## Cuándo NO utilizarla

- **Juegos 2D** → `CharacterBody2D` (API paralela con diferencias: skill pendiente `godot-characterbody2d`).
- **Cuerpos con física completa** (caídas, botes, fuerzas) → `RigidBody3D` (skill pendiente).
- **Plataformas móviles** → el cuerpo kinemático puede ir sobre una plataforma, pero la propagación de velocidad se gestiona con `platform_*` (ver abajo) o un `RigidBody3D` de plataforma.
- **Vehículos** → `VehicleBody3D` (skill pendiente).
- **Movimiento por path/navmesh** (NPCs) → `NavigationAgent3D` sobre un `CharacterBody3D` (skill pendiente) — el cuerpo sigue siendo kinemático, pero la dirección la da el agente de navegación.

## Conceptos fundamentales

### Qué es

`CharacterBody3D` es un `RigidBody3D` kinemático especializado: **tú** le dices la velocidad (`velocity`), el motor resuelve la colisión contra el mundo deslizándolo (slides) y actualiza su estado (suelo, pared, techo). No reacciona a fuerzas ni a gravedad (todo es tu responsabilidad).

### El contrato `move_and_slide()`

| Paso | Qué ocurre |
|---|---|
| 1. Escribes `velocity` | En **m/s** (no metros por frame). El motor **no** multiplica por delta. |
| 2. Llamas `move_and_slide()` | Solo dentro de `_physics_process(delta)`. Usa el delta del paso de física internamente. Mueve el cuerpo según `velocity`, resuelve colisiones con slides, y **modifica** `velocity` (p. ej. anula la componente contra un muro). |
| 3. Lees el estado | `is_on_floor()`, `get_floor_normal()`, `is_on_wall()`, `get_last_slide_collision()` — válidos **tras** la llamada. |

**Reglas duras (docs oficial):**

1. `velocity` está en m/s. `velocity = velocity * delta` es el error más común (convierte m/s a metros).
2. `move_and_slide()` **siempre** en `_physics_process` (la simulación usaría un delta incorrecto en otro sitio).
3. `move_and_slide()` devuelve `bool` (`true` si colisionó) y **puede** modificar `velocity`. Nunca asumir que `velocity` es lo que escribiste.

### Suelo: snapping

El "suelo" no es un raycast tuyo: `move_and_slide()` aplica un **snap** (`floor_snap_length`, default 0.1) que mantiene al cuerpo pegado al suelo al bajar por pendientes. Sin snapping, el cuerpo "flota" sobre rampas y `is_on_floor()` parpadea.

### Capas de colisión (heredado de `CollisionObject3D`)

`collision_layer` / `collision_mask` son **ints** de 32 bits: 32 capas. `collision_layer` = "contra qué puedo chocar yo (que otros detecten)"; `collision_mask` = "qué detección de mí respondo". Para `CharacterBody3D`: la máscara dice qué cuerpos/estáticos detona al moverse. Helperes: `get_collision_layer_value(n)`, `set_collision_layer_value(n, b)`, `get_collision_mask_value(n)`, `set_collision_mask_value(n, b)` con `n` en 1..32.

### Prioridad de colisión

`collision_priority` (float, default 1.0): cuando dos cuerpos kinemáticos se solapan, **gana el de mayor prioridad** (el otro se desliza). Documentado en `CollisionObject3D`.

### Sin señales

`CharacterBody3D` **no tiene señales** en 4.0 ni 4.7 (verificado). "¿Cayó al suelo?", "¿Tocó una pared?" → polling en `_physics_process` (baratísimo) o `Area3D` para zonas de detección.

## API relevante

Todo verificado en Godot 4.7 (contraste 4.0 en `GODOT_VERSION_MATRIX.md`).

### Propiedades

| API | Tipo | Default | Nota |
|---|---|---|---|
| `velocity` | `Vector3` | (0,0,0) | m/s. **NO** multiplicar por delta (advertencia literal en la docs). Modificado por `move_and_slide()`. |
| `up_direction` | `Vector3` | (0,1,0) | Se normaliza; no puede ser `Vector3.ZERO`. Cambiarlo = gravedad invertida/ejes no estándar. |
| `floor_max_angle` | `float` (grados) | 45.0 | Pendientes ≤ 45° cuentan como suelo; > 45° como muro. |
| `floor_snap_length` | `float` (m) | 0.1 | Pegado al suelo. 0 = no se pega a rampas al bajar (flota, `is_on_floor()` parpadea). No se aplica si hay velocidad hacia `up_direction` (salto). |
| `floor_stop_length` | `float` (m) | 0.1 | Parada suave en bordes de pendientes (rango típico 0.01–1.0). |
| `floor_stop_on_slope` | `bool` | false | No deslizar al llegar a una pendiente con `floor_stop_length`. |
| `floor_block_on_wall` | `bool` | false | Muros bajos bloquean en lugar de dejar deslizar. |
| `floor_constant_speed` | `bool` | false | Misma velocidad en pendientes (requiere `floor_snap_length` > 0). |
| `slide_on_ceiling` | `bool` | false | Deslizar por el techo en lugar de frenar. |
| `wall_min_slide_angle` | `float` (grados) | 0.0 | Ángulo mínimo de pared para que cuente como pared en slides. |
| `max_slides` | `int` | 6 | Slides máximos por `move_and_slide()`. Bajarlo ahorra CPU pero produce atascos en esquinas. |
| `safe_margin` | `float` (m) | 0.001 | Margen de recuperación al estar dentro de geometría. Subir (0.01–0.05) solo con atascos reales; demasiado alto → pulsos al entrar en geometría. |
| `motion_mode` | enum | `MOTION_MODE_GROUNDED` | `MOTION_MODE_GROUNDED` (default, suelo+slides) / `MOTION_MODE_FLOATING` (espacio: sin suelo, sin snapping; control 6DOF propio). |
| `platform_floor_layers` | `int` (máscara de capas) | — | Capas de plataformas móviles cuya velocidad se propaga al cuerpo. |
| `platform_on_leave` | `PlatformOnLeave` enum | `PLATFORM_ON_LEAVE_KEEPS_VELOCITY` | Qué conservar al abandonar una plataforma. |
| `platform_up_direction` | `Vector3` | (0,1,0) | "Arriba" local de las plataformas. |

### Métodos

| API | Devuelve | Nota |
|---|---|---|
| `move_and_slide()` | `bool` | El corazón. Solo en `_physics_process`. `true` si colisionó. |
| `is_on_floor()` | `bool` | Válido tras `move_and_slide()`. El caso más común (suelo "casi"). |
| `is_on_floor_only()` | `bool` | En suelo y no tocando nada más (más estricto). |
| `is_on_wall()` | `bool` | Válido tras `move_and_slide()`. |
| `is_on_ceiling()` | `bool` | Válido tras `move_and_slide()`. |
| `get_floor_normal()` | `Vector3` | Normal del suelo (para lerp de cámara, orientación en rampas). |
| `get_floor_angle()` | `float` (rad) | Ángulo entre la normal del suelo y `up_direction`. |
| `get_wall_normal()` | `Vector3` | Normal de la pared en el último slide. |
| `get_slide_count()` | `int` | Slides realizados en el último `move_and_slide()`. |
| `get_last_slide_collision()` | `KinematicCollision3D` o `null` | `null` si no colisionó. La colisión da `get_collider()`, `get_collider_shape()`, `get_position()`, `get_normal()`, `get_travel()`. |
| `get_real_velocity()` | `Vector3` | Velocidad real del cuerpo (diagonal en pendientes) vs `velocity` (la pedida). |
| `get_position_delta()` | `Vector3` | Desplazamiento real del último paso de física. |
| `get_rid()` | `RID` | Para excluir el cuerpo de casts ajenos (p. ej. `SpringArm3D.add_excluded_object()`). |

### Heredado de `CollisionObject3D` (verificado 4.7)

| API | Nota |
|---|---|
| `collision_layer` / `collision_mask` | `int` (32 capas), default 1 (solo capa 1). |
| `collision_priority` | float, default 1.0. Mayor = penetra sobre otros con misma/menor prioridad. |
| `disable_mode` | `DISABLE_MODE_REMOVE(0)` / `MAKE_STATIC(1)` / `KEEP_ACTIVE(2)` — qué hace el cuerpo cuando su `process_mode` es `DISABLED`. |
| `input_ray_pickable` | bool, default `true`. `InputEventScreenDrag`/raycasts de input lo tocan. |
| `input_capture_on_drag` | bool, default `false`. |
| Señales: `input_event(camera, event, event_position, normal, shape_index)` | Clicks/teclado sobre el cuerpo (requiere cámara y `input_ray_pickable`). |
| Señales: `mouse_entered` / `mouse_exited` | **Caveat oficial**: sin CCD para el ratón; pueden no dispararse si el ratón es muy rápido o el objeto se mueve mucho. |
| `get/set_collision_layer_value(n, b)` / `get/set_collision_mask_value(n, b)` | `n` en 1..32. |
| Shape owners: `create_shape_owner()`, `shape_owner_add_shape()`, `shape_owner_get_shape_count()`, `shape_owner_get_shape()`, `shape_owner_set_shape_transform()`, `shape_owner_set_shape_disabled()`, `shape_find_owner()` | Múltiples shapes dinámicos por cuerpo (p. ej. box adicional para un escudo). |

### NO existen en 4.x (verificado en 4.0 Y 4.7 — no inventar)

| API inexistente | Época | Alternativa 4.x |
|---|---|---|
| Señales `floor_entered` / `floor_exited` / `wall_entered` / `wall_exited` / `ceiling_entered` / `ceiling_exited` / `moved` | 3.x (ni siquiera en 3.x: nunca existieron) | Polling `is_on_floor()`/`is_on_wall()` en `_physics_process`; `Area3D` para zonas. |
| `floor_friction` (prop) | 3.x | `floor_constant_speed = true`; `PhysicsMaterial3D` (fricción) en el suelo; amortiguación manual con `move_toward`. |
| `wall_min_slide_speed` (prop) | 3.x | Tuning propio (p. ej. `wall_sliding_speed` en el controlador). |
| Argumento de `move_and_slide(safe_margin)` | 3.x | `safe_margin` es propiedad. |
| `KinematicBody3D` (clase) | 3.x | Renombrada a `CharacterBody3D` en 4.0. |
| `move_and_collide()` | 3.x | En 4.x es `move_and_collide(local_transform, collision, margin)` con firma distinta; para un personaje, `move_and_slide()` es lo correcto. |

## Arquitectura recomendada

### Escena mínima

```text
Player (CharacterBody3D)
└── CollisionShape3D
    └── CapsuleShape3D   (radius 0.4, height 1.8; position y = 0.9 → pies en 0)
```

Decisiones:

1. **Capsula, no box**: la cápsula desliza mejor en esquinas y rampas (slides suaves), es barata en colisión y "perdona" la desalineación del mesh. Box = atascos en esquinas.
2. **El shape es el "cuerpo lógico"**: el mesh puede ser distinto y más grande (visual). Rotar el visual → pivot hijo (`Model`), nunca el `CharacterBody3D` (rompe `up_direction`, slides, `get_floor_normal`).
3. **Capas sugeridas** (proyecto estándar): capa 1 = MUNDO (estático), capa 2 = JUGADOR, capa 3 = ENEMIGOS/props físicos, capa 4 = INTERACTABLES (áreas). Jugador: layer 2, mask 1 (choca contra el mundo; normalmente no contra otros jugadores: añadir 3 si quieres).
4. **`up_direction` solo si el mundo no es Y-arriba** — en casi todo el juego no se toca.

### Patrón de escritura (qué va en `_physics_process`)

```gdscript
# 1) Suelo/gravedad  2) Salto  3) Input→dirección  4) Aceleración/frenado
# 5) move_and_slide()  6) Estado post-slide (animación, señales, audio)
```

El orden importa: leer `is_on_floor()` **antes** de mover (estado del tick anterior) y **después** (estado nuevo) según el uso.

## Implementación mínima

Personaje que camina y salta sin cámara (la versión con cámara está en `godot-character-controller`):

```gdscript
# capsule_player.gd — Godot 4.7 (APIs verificadas)
extends CharacterBody3D
## Cuerpo kinemático base: caminar (WASD), salto (espacio), gravedad propia.

@export var walk_speed := 4.5      # m/s (MEDIUM: tuning práctico)
@export var jump_velocity := 5.5   # m/s
@export var gravity := 18.0        # m/s²

func _physics_process(delta: float) -> void:
	# 1) Gravedad (solo en el aire)
	if not is_on_floor():
		velocity.y -= gravity * delta
	else:
		velocity.y = minf(velocity.y, 0.0)  # evita "rebote" residual al salir de caída

	# 2) Salto (evento, uno por frame)
	if is_on_floor() and Input.is_action_just_pressed("jump"):
		velocity.y = jump_velocity

	# 3) Dirección horizontal (WASD; sin cámara)
	var input_vec := Input.get_vector("move_left", "move_right", "move_forward", "move_back")
	var target := Vector3(input_vec.x, 0.0, input_vec.y) * walk_speed

	# 4) Acelerar/frenar hacia el objetivo
	var flat := Vector2(velocity.x, velocity.z)
	flat = flat.move_toward(target.xz if input_vec != Vector2.ZERO else Vector2.ZERO, 20.0 * delta)
	velocity.x = flat.x
	velocity.z = flat.y

	# 5) Mover (usa el delta de física; velocity en m/s)
	move_and_slide()
```

## Implementación recomendada

Cuando el proyecto escala (ver `godot-character-controller` para la versión completa con cámara):

1. **Estado post-slide para eventos de juego**:

```gdscript
signal landed(fall_speed: float)

var _was_on_floor := true
var _max_fall_speed := 0.0

# En _physics_process(), DESPUÉS de move_and_slide():
	if not is_on_floor():
		_max_fall_speed = maxf(_max_fall_speed, -velocity.y)
	elif not _was_on_floor:
		landed.emit(_max_fall_speed)   # m/s de caída
	_max_fall_speed = 0.0
	_was_on_floor = is_on_floor()
```

2. **Colisión contra entidades (ejemplo de uso de `get_last_slide_collision`)**:

```gdscript
func _physics_process(delta: float) -> void:
	# ... movimiento ...
	if move_and_slide():
		var c := get_last_slide_collision()
		if c and c.get_collider() is CharacterBody3D:
			var otro := c.get_collider() as CharacterBody3D
			# p. ej.: empuje suave, daño, sonido
```

3. **Plataformas móviles** (sin `RigidBody3D` de plataforma):

```gdscript
# En el jugador:
# platform_floor_layers = capa de las plataformas (bit 1 → 1)
# Al salir de una plataforma, la velocidad se conserva según platform_on_leave
# (default PLATFORM_ON_LEAVE_KEEPS_VELOCITY: llevas la velocidad de la plataforma).
```

4. **Excluir al cuerpo de casts ajenos** (cámara, raycasts de UI):

```gdscript
@onready var _spring_arm: SpringArm3D = %SpringArm3D

func _ready() -> void:
	_spring_arm.add_excluded_object(get_rid())
```

5. **Múltiples shapes por cuerpo** (p. ej. escudo delantero): API de shape owners verificada arriba (`create_shape_owner()`, `shape_owner_add_shape(shape, transform)`).

## Ejemplo práctico

**Juego**: plataformer 3D. Jugador con 2 plataformas móviles (un `RigidBody3D` con `lock_rotation` en cada una, capa 16):

1. `platform_floor_layers = 16` → al salir de una plataforma en movimiento, el jugador lleva su velocidad (default `PLATFORM_ON_LEAVE_KEEPS_VELOCITY`): no se "quedar atrás".
2. Rampas de 30°: `floor_max_angle` default 45° → se trata como suelo. Con `floor_constant_speed = true` el jugador no acelera en la rampa (mismo m/s que en plano).
3. Esquina con 90° y `max_slides = 6`: el jugador redondea la esquina en 2 slides; `get_slide_count()` = 2. Bajar a 1 → se atasca en la esquina (diagnóstico medible).
4. Caída de 3 m: `landed.emit(~7.3)` (m/s, `maxf` de `-velocity.y` durante el vuelo) → sonido de aterrizaje + partícula.
5. Atasco al spawn dentro de un muro: `safe_margin = 0.01` (de 0.001) resuelve la recuperación; si aparece un "pulso" visible al entrar en pasillos, bajar de vuelta a 0.001 (medido: no hay atascos sin pulso).

## Integración

- **Controlador**: `godot-character-controller` construye sobre esta skill (aceleración, cámara, coyote, buffer).
- **Animación**: `godot-animationtree` consume `is_on_floor()`, velocidad plana, `get_floor_angle()` (para estados de pendiente) vía señales del controlador.
- **Cámara**: `godot-camera3d` usa `get_rid()` para excluir al jugador del cast del `SpringArm3D`.
- **Física**: `godot-physics` (capas, shapes, tipos de cuerpo) y `godot-physics-materials` (`PhysicsMaterial`, no `PhysicsMaterial3D`) — disponibles (Lote 4).
- **Navegación**: `NavigationAgent3D` como hijo (la dirección del body la aporta el agente; `velocity` sigue el mismo contrato).

## Errores frecuentes

1. **`velocity = velocity * delta`** — m/s × s = metros. El personaje se mueve a fracciones mínimas y depende del FPS. (Advertencia literal en la docs de `velocity`.)
2. **`move_and_slide()` en `_process`** — la docs manda `_physics_process`. Síntoma: velocidad distinta según Hz de pantalla (120 Hz = más rápido).
3. **Conectar `floor_entered`/`wall_entered`/`moved`** — no existen en 4.x (verificado 4.0 y 4.7). "signal not found". → polling o `Area3D`.
4. **`floor_friction` / `wall_min_slide_speed`** — no existen en 4.x (3.x). "Invalid get index". → `floor_constant_speed`, `PhysicsMaterial3D`, amortiguación manual.
5. **Leer `is_on_floor()` antes de `move_and_slide()` del tick actual y actuar como si fuera "ahora"** — el estado es del tick anterior (normal, pero la causa de "salta en el borde y no cuenta").
6. **`floor_snap_length = 0`** — flota sobre rampas, `is_on_floor()` parpadea. → 0.1 (default).
7. **`safe_margin` alto "por seguridad" (0.5)** — pulsos/saltitos al entrar en geometría. → 0.001 (default); subir solo con atascos medidos.
8. **Asumir que `velocity` sigue siendo lo que escribiste tras `move_and_slide()`** — el motor la modifica (componente contra muro a 0). Recalcular lo que necesites tras la llamada.
9. **Rotar el `CharacterBody3D` para orientar el visual** — rompe slides/`up_direction`. → pivot `Model` hijo.
10. **Escalado no uniforme en el body** — la docs advierte: escala no uniforme rompe la colisión. → escala uniforme + redimensionar el shape.
11. **Esperar señales de colisión** — no las hay: el retorno `bool` de `move_and_slide()` + `get_last_slide_collision()` es toda la información.
12. **`motion_mode = MOTION_MODE_FLOATING` sin darse cuenta** — al ponerlo, desaparece el suelo: `is_on_floor()` siempre false. Es un modo para espacio/6DOF.

## Anti-patrones

- **Raycasts propios para "sentir el suelo"** (2–3 `RayCast3D` por jugador por frame) → `is_on_floor()` + `floor_snap_length` ya lo resuelven dentro de `move_and_slide()`.
- **`CharacterBody3D` con `RigidBody3D` de "fuerza extra"** (añadir un rígido al jugador para "mejor peso") → duplicado de física, jitter. El kinemático + tuning propio es el patrón.
- **Un `CharacterBody3D` gigante con 10 shapes de box para "ajustar la forma"** → cápsula + 1 box extra solo si el juego lo exige (y con `shape_owner_add_shape`).
- **`max_slides = 1` para "rendimiento"** → atascos en esquinas garantizados; el coste real es despreciable (medir antes de tocarlo).
- **Mover el body en `_process` "porque va más suave"** → la física es 60 Hz fija; el suavizado visual es del `Model`, no del cuerpo.
- **Polling de `get_tree()` cada frame para "encontrar al suelo"** → la colisión ya te lo da (`get_last_slide_collision().get_collider()`).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

### Coste (teórico — medir antes de optimizar)

- `move_and_slide()`: coste bajo y constante por tick de física; crece con `max_slides` y con la complejidad de la geometría colindante.
- `is_on_floor()`/`get_floor_normal()`: O(1) — son lecturas del estado resuelto; **gratis** frente a raycasts propios.
- El cuello de botella de un personaje rara vez es el body: suele ser animación (CPU), shader (GPU por píxel) o los cientos de instancias (ver `godot-rendering-performance`).

### Medición

```gdscript
# Godot 4.7 (APIs verificadas):
print("ms/physics: ", Performance.get_monitor(Performance.TIME_PHYSICS_PROCESS) * 1000.0)
print("pares colisión 3D: ", Performance.get_monitor(Performance.PHYSICS_3D_COLLISION_PAIRS))
print("objetos 3D activos: ", Performance.get_monitor(Performance.PHYSICS_3D_ACTIVE_OBJECTS))
```

(Ver `godot-rendering-performance` para el cuadro completo de monitores.)

## Debugging

### "El personaje no se mueve"

```text
1. ¿La acción existe y está mapeada?      → Project Settings → Input Map;
   print(Input.get_action_strength("move_forward"))  # debe ser 1 al pulsar
2. ¿move_and_slide() está en _physics_process?
3. ¿velocity * delta?                     → error #1
4. ¿Colisiona?                            → capas: body (mask) vs mundo (layer);
   3D viewport → "visible collisions"
5. ¿Start dentro de geometría?            → mover el spawn; safe_margin 0.001
```

### "Se atasca en esquinas / muros"

```text
1. ¿max_slides bajo?         → 6 (default)
2. ¿Cápsula?                 → box en esquinas se atasca; cápsula desliza
3. ¿safe_margin alto?        → 0.5+ produce pulsos (error #7)
4. print(get_slide_count(), get_last_slide_collision())  # ver cuántos slides y contra qué
```

### "Flota sobre rampas / is_on_floor() parpadea"

```text
1. floor_snap_length = 0?    → 0.1
2. floor_max_angle bajo?     → 45 default; rampas > 45° son muros (correcto)
3. ¿Gravedad aplicada?       → velocity.y -= gravity * delta solo en el aire
```

### "Se cae a través del suelo al correr rápido"

```text
1. Velocidad vs paso de física: v (m/s) / 60 > grosor del suelo → atraviesa.
   → motion_mode + CCD no existe en CharacterBody3D 4.7 (verificado: solo los modos
     GROUNDED/FLOATING); reducir velocidad máxima o engrosar la geometría fina.
2. Suelo sin colisión (StaticBody3D sin CollisionShape3D).
```

### Consola

| Error | Causa |
|---|---|
| `Invalid get index 'floor_friction'` | API 3.x (error #4) |
| `Signal 'floor_entered' not found` | API inexistente (error #3) |
| `Non-uniform scale on ...` warning | Escalado no uniforme (error #10) |
| `move_and_slide() should be called in _physics_process` (warning) | error #2 |

## Compatibilidad

- **Verificado: 4.7 (stable)** — API completa verificada contra `CharacterBody3D.xml` (4.7) y la docs stable.
- **4.0 ↔ 4.7**: tabla de propiedades **idéntica** entre 4.0 y 4.7 para `velocity`, `up_direction`, `floor_max_angle`, `floor_snap_length`, `floor_constant_speed`, `floor_stop_on_slope`, `floor_block_on_wall`, `max_slides`, `motion_mode`, `platform_*`, `safe_margin`, `slide_on_ceiling`, `wall_min_slide_angle` (verificado en ambas ramas; `GODOT_VERSION_MATRIX.md`).
- **Sin señales en ninguna 4.x** (verificado 4.0 y 4.7).
- **3 → 4**: `KinematicBody3D` → `CharacterBody3D`; `move_and_slide(safe_margin)` → `move_and_slide()` + prop; `floor_friction`/`wall_min_slide_speed` eliminadas.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-physics` (capas, shapes, tipos de cuerpo) | Base compartida — disponible (Lote 4) |
| `godot-physics-materials` (fricción/rebote de superficies) | Base disponible (Lote 4) |
| `godot-character-controller` | **Depende de esta skill** (la construye encima) |
| `godot-input` | `Input.get_vector`/`is_action_just_pressed` en los ejemplos |
| `godot-recipe-third-person-controller` | Receta completa — pendiente |
| `godot-third-person-character` | Skill compuesta que integra esta — creada (Lote 0) |

## Skills relacionadas

- `godot-characterbody2d` — equivalente 2D (pendiente)
- `godot-error-character-not-moving` — árbol de diagnóstico dedicado (pendiente)
- `godot-error-collision-not-working` — pendiente
- `godot-physics-layers` — convenciones de capas (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- CharacterBody3D: https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html
- CollisionObject3D (capas, prioridad, disable_mode, señales de input, shape owners): https://docs.godotengine.org/en/stable/classes/class_collisionobject3d.html
- KinematicCollision3D: https://docs.godotengine.org/en/stable/classes/class_kinematiccollision3d.html
- PhysicsBody3D / RigidBody3D (herencia y `motion_mode`): https://docs.godotengine.org/en/stable/classes/class_rigidbody3d.html
- Tutorial "First 3D Game — Moving the player with code": https://docs.godotengine.org/en/stable/getting_started/first_3d_game/03.player_movement_code.html
- Demo oficial Kinematic Character 3D: https://godotengine.org/asset-library/asset/2739
- Demo oficial Platformer 3D: https://godotengine.org/asset-library/asset/2748
