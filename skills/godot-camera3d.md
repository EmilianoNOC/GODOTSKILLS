# godot-camera3d

## ID

`godot-camera3d`

## Categoría

3D · Cámaras · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la class reference oficial 4.7 (`docs.godotengine.org/en/stable/`) y el XML de clases de la rama `4.7-stable` (`Camera3D.xml`).

## Confidence

**HIGH** — API verificada directamente en la docs 4.7 (contraste 4.0 en `GODOT_VERSION_MATRIX.md`).
**"No existe (verificado)"**: smoothing nativo de `Camera3D` (es de `Camera2D`, no de 3D — verificado ausente en 4.0 y 4.7).

## Nivel

**BASE** (sin prerrequisitos).

## Propósito

Proporcionar el conocimiento operativo completo de `Camera3D` en Godot 4.x: proyecciones (perspectiva/ortográfica/frustum), planos de recorte, capas de culling, offsets, cambio de cámara activo, y las utilidades de proyección (`project_position`, `unproject_position`, `project_ray_*`, `is_position_in_frustum`) — más el patrón de cámara orbital con `SpringArm3D` (la receta oficial del tutorial de spring arm).

## Cuándo utilizarla

- Cualquier vista 3D del juego (la cámara por defecto).
- TPS/FPS: rig de cámara orbital o en primera persona.
- Recortar capas (no renderizar la capa del jugador con su cámara, o el "debug" por capas).
- Proyectar puntos 3D a pantalla (marcadores, UI anclada, miras) e inverso (click → rayo 3D).
- Transiciones de cámara (`make_current` / `clear_current`).

## Cuándo NO utilizarla

- **Juegos 2D** → `Camera2D` (sí tiene smoothing nativo, `zoom`, limits).
- **Vistas de mini-mapa / debug** → un `SubViewport` con su propia `Camera3D` (el patrón es el mismo, pero aislado).
- **Cinematografía compleja** (tracks, keyframes de cámara) → `AnimationPlayer` animando la `Camera3D` (o `PathFollow3D`); la cámara es solo un nodo animable.
- **XVR (VR)** → las cámaras las gestiona el sistema HMD (`XRBodyTracker`/`XRServer`), no una `Camera3D` manual (skill pendiente `godot-xr`).

## Conceptos fundamentales

### Una cámara activa por viewport

`Camera3D.current` (bool) marca la cámara que renderiza el `Viewport`. **Solo una** por viewport. Cambiar de cámara en runtime: `clear_current()` en la vieja + `make_current()` en la nueva (o `current = true`). La cámara de la escena que se está cargando normalmente lleva `current = true`.

### Proyecciones

| `projection` | Valor | Uso |
|---|---|---|
| `PROJECTION_PERSPECTIVE` | 0 (default) | El estándar 3D. `fov` (default 75°) controla el ángulo vertical. |
| `PROJECTION_ORTHOGONAL` | 1 | Sin profundidad por tamaño; `size` (default 1.0) = altura del view frustum en metros. |
| `PROJECTION_FRUSTUM` | 2 | Custom: `frustum_near`/`frustum_left`/`frustum_right`/`frustum_top`/`frustum_bottom` (para cámaras compuestas/VFX). |

`set_perspective(fov)` / `set_orthogonal(size)` cambian de proyección y configuran en una llamada (verificado 4.7).

### Planos de recorte

`near` (default 0.05) / `far` (default 4000): lo que **no** se renderiza. `near` demasiado grande → objetos "cortados" cerca (cabeza del jugador en FPS); `far` demasiado grande → más precisión de Z desperdiciada + culling de frustum menos útil. Ajustar `far` al tamaño real del mundo.

### Capas de culling (visuales)

`cull_mask` (int, default 1048575 = todas las capas): qué capas **de geometría renderizada** ve la cámara (independiente de las capas de física). El caso típico: la cámara del jugador no renderiza la capa del propio jugador (el interior del cuerpo) → `cull_mask` sin el bit del jugador.

### Offsets

`h_offset` / `v_offset` (float, en fracciones del viewport): desplazar la proyección (cámara "descentrada" — p. ej. mira en un hombro, o UI que ocupa parte de la pantalla).

### Sin smoothing nativo

**Verificado (4.0 y 4.7): `Camera3D` no tiene propiedades de smoothing/interpolación** (eso es de `Camera2D`: `position_smoothing_enabled`, etc.). El suavizado 3D es **manual**: lerp/damping con `delta` en `_process` (patrón abajo).

## API relevante

Todo verificado en Godot 4.7.

| API | Tipo / default | Nota |
|---|---|---|
| `current` | bool | Cámara activa del viewport. |
| `make_current()` | — | Hacerla activa. |
| `clear_current(without_fallback = true)` | — | Quitar la activez (`without_fallback` = no forzar a la "default camera" del viewport). |
| `is_current()` | `-> bool` | — |
| `projection` | enum (0) | `PERSPECTIVE(0)` / `ORTHOGONAL(1)` / `FRUSTUM(2)`. |
| `fov` | float (75) | Grados, solo perspectiva (vertical). |
| `near` / `far` | float (0.05 / 4000) | Planos de recorte. |
| `size` | float (1.0) | Solo ortográfica: altura del frustum (m). |
| `frustum_near/left/right/top/bottom` | float | Solo `FRUSTUM`. |
| `set_perspective(fov)` / `set_orthogonal(size)` | — | Cambio de proyección + parámetro en una llamada. |
| `cull_mask` | int (1048575) | Capas de geometría visibles (32 bits). |
| `h_offset` / `v_offset` | float | Offset de proyección (fracciones). |
| `position_smoothing_enabled` | — | **No existe** en `Camera3D` (verificado). Es de `Camera2D`. |
| `project_position(world_pos)` | `-> Vector2` | Pto 3D → píxeles (pantalla). Para UI anclada/marcadores. |
| `unproject_position(screen_pos)` | `-> Vector3` | Píxel → pto 3D **en el plano Z de la cámara** (cuidado: no es un rayo). |
| `project_ray_origin(screen_pos)` | `-> Vector3` | Origen del rayo (posición de la cámara). |
| `project_ray_normal(screen_pos)` | `-> Vector3` | Dirección del rayo (pantalla → mundo). Para click-to-object. |
| `is_position_in_frustum(world_pos)` | `-> bool` | ¿El pto es visible (sin oclusión)? |
| `get_camera_projection()` | `-> Projection` | La matriz de proyección actual. |
| `get_final_transform()` | `-> Transform3D` | Transform global final (para shaders). |
| `doppler_shift` | float | Efecto Doppler (sonido de pasados). |
| `velocity` | Vector3 | Para el Doppler (velocidad de la cámara). |

### `SpringArm3D` (la colisión de cámara — tutorial oficial)

| API | Nota |
|---|---|
| `spring_length` (float, 1.0) | Longitud del brazo. |
| `margin` (float, 0.01) | Se resta en colisión (no pegar al muro). |
| `shape` (Shape3D, `<empty>`) | Si `<empty>` **y la `Camera3D` es hija directa** → usa la pirámide del plano near de la cámara. Sin shape y cámara no hija directa → **raycast impreciso, no recomendado (docs)**. |
| `collision_mask` (int, 1) | Capas físicas contra las que choca. |
| `add_excluded_object(rid)` / `remove_excluded_object(rid)` | Excluir cuerpos del cast (el propio jugador). |
| `get_hit_length()` | Longitud actual del cast (debug: `< spring_length` = hay colisión). |

## Arquitectura recomendada

### Rig TPS (receta oficial del tutorial spring arm)

```text
Player (CharacterBody3D)
└── CameraPivot (Node3D)  position (0, 1.6, 0)      ← rota (yaw + pitch limitado)
    └── SpringArm3D       spring_length 4.0, margin 0.03
        └── Camera3D      current = true, fov 60     ← HIJA DIRECTA (usa la pirámide)
```

Decisiones:

1. **`Camera3D` hija DIRECTA de `SpringArm3D`** → el cast usa la pirámide del near plane (mejor que una esfera pequeña, peor que el near plano real). Si necesitas un nodo intermedio, pon `shape = SphereShape3D` pequeño en el brazo.
2. **El pivot rota, no la cámara**: el yaw/pitch van en `CameraPivot` (así el `SpringArm3D` apunta siempre "hacia atrás" del pivot).
3. **Pitch limitado** (±75° típico) en el pivot; `SpringArm3D.collision_mask` = capa MUNDO; `add_excluded_object(player.get_rid())`.
4. **`cull_mask`**: quitar el bit de la capa del jugador para no renderizar el interior del cuerpo (en TPS el mesh del cuerpo se recorta por `near` o por capa).
5. **Un solo rig por "modo" de cámara**; cambios de modo (pista, cinemática) = `clear_current()` + `make_current()`.

### Suavizado manual (lo que no trae el motor)

```gdscript
# Smoothing de posición/rotación de cámara (patrón standard, 4.7)
var _target_position := Vector3.ZERO
var _target_basis := Basis.IDENTITY
@export var smooth_speed := 8.0   # 1/s de damping

func _process(delta: float) -> void:
	# Damping independiente de FPS:
	global_position = global_position.lerp(_target_position, 1.0 - exp(-smooth_speed * delta))
	# para rotación:
	rotation = slerp(rotation, _target_basis.angles, 1.0 - exp(-smooth_speed * delta))
```

## Implementación mínima

Cámara fija que sigue al jugador (sin orbitar):

```gdscript
# follow_camera.gd — Godot 4.7 (APIs verificadas)
extends Camera3D
## Cámara que sigue a un objetivo con damping manual (no hay smoothing nativo en 3D).

@export var target: Node3D            # el jugador (o cualquier Node3D)
@export var offset := Vector3(0.0, 3.0, 6.0)
@export var smooth_speed := 8.0       # 1/s

func _process(delta: float) -> void:
	if target == null:
		return
	var desired := target.global_transform.origin + target.global_transform.basis * offset
	global_position = global_position.lerp(desired, 1.0 - exp(-smooth_speed * delta))
	look_at(target.global_transform.origin + target.global_transform.basis * Vector3(0.0, 1.0, 0.0))
```

## Implementación recomendada

### 1. Orbitación con ratón capturado + gamepad (patrón completo)

```gdscript
# orbit_camera.gd — en CameraPivot (Godot 4.7, APIs verificadas)
extends Node3D

@export var mouse_sensitivity := 0.0035
@export var gamepad_speed := 1.4        # rad/s
@export_range(0.0, 89.0) var tilt_limit := 75.0
@export var zoom_range := Vector2(2.0, 8.0)

@onready var _spring: SpringArm3D = $SpringArm3D

func _ready() -> void:
	_spring.spring_length = zoom_range.x + (zoom_range.y - zoom_range.x) * 0.5

func _process(delta: float) -> void:
	# Stick derecho: polling de ejes (los sticks NO son eventos)
	var stick := Vector2(Input.get_joy_axis(0, JOY_AXIS_RIGHT_X),
		Input.get_joy_axis(0, JOY_AXIS_RIGHT_Y))
	if stick.length() > 0.15:
		_rotate(stick * gamepad_speed * delta)

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventMouseButton:
		var mb := event as InputEventMouseButton
		if mb.pressed and mb.button_index == MOUSE_BUTTON_RIGHT:
			Input.mouse_mode = Input.MOUSE_MODE_CAPTURED
		if mb.pressed and mb.button_index == MOUSE_BUTTON_WHEEL_UP:
			_zoom(-0.5)
		if mb.pressed and mb.button_index == MOUSE_BUTTON_WHEEL_DOWN:
			_zoom(0.5)
	elif event is InputEventKey:
		var k := event as InputEventKey
		if k.pressed and k.keycode == KEY_ESCAPE:
			Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
	elif event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
		var mm := event as InputEventMouseMotion
		if not mm.screen_relative.is_zero_approx():   # puede emitirse sin movimiento
			_rotate(Vector2(mm.screen_relative.x, mm.screen_relative.y) * mouse_sensitivity)

func _rotate(amount: Vector2) -> void:
	rotation.x = clampf(rotation.x - amount.y, deg_to_rad(-tilt_limit), deg_to_rad(tilt_limit))
	rotation.y -= amount.x

func _zoom(d: float) -> void:
	_spring.spring_length = clampf(_spring.spring_length + d, zoom_range.x, zoom_range.y)

func is_clipping() -> bool:
	return _spring.get_hit_length() < _spring.spring_length - 0.05
```

### 2. Proyección a pantalla (marcadores de objetivos)

```gdscript
# En _process() o cuando el objetivo se mueve:
var screen_pos: Vector2 = $Camera3D.project_position($Objetivo.global_position)
$HUD/Marker.position = screen_pos
# ¿Está visible (sin oclusión)?
if $Camera3D.is_position_in_frustum($Objetivo.global_position):
	$HUD/Marker.show()
else:
	$HUD/Marker.hide()
```

### 3. Click → rayo 3D (seleccionar objetos)

```gdscript
func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventMouseButton and event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
		var origin: Vector3 = $Camera3D.project_ray_origin(event.position)
		var dir: Vector3 = $Camera3D.project_ray_normal(event.position)
		# Y tu propio raycast (o Input.is_action + raycast del input):
		var space := get_world_3d().direct_space_state
		var q := PhysicsRayQueryParameters3D.create(origin, origin + dir * 100.0)
		var hit := space.intersect_ray(q)
		if hit:
			print("Tocó: ", hit.collider)
```

### 4. Transiciones de cámara (pista → cinemática)

```gdscript
func go_cinematic() -> void:
	$CinematicCamera.make_current()      # la nueva cámara
	# La vieja:
	$Camera3D.clear_current()            # without_fallback = true (default): no forcea default

func go_back() -> void:
	$Camera3D.make_current()
	$CinematicCamera.clear_current()
```

## Ejemplo práctico

**Juego**: TPS con pasillos y combate.

1. **Rig**: pivot (0,1.6,0) + `SpringArm3D` (4 m, margin 0.03) + `Camera3D` hija directa (fov 60, near 0.1).
2. **Culling**: la cámara tiene `cull_mask` sin la capa del jugador → no se renderiza el interior del cuerpo; el near plane recorta el resto.
3. **Pasillo**: el brazo se acorta al chocar (pirámide del near plane = cast); `is_clipping()` → HUD "geometría cerca".
4. **Mira de objetivos**: `project_position` ancla un marcador UI sobre enemigos visibles; `is_position_in_frustum` los oculta cuando están de espaldas al frustum.
5. **Click derecho = interactuar**: `project_ray_origin/normal` + `PhysicsRayQueryParameters3D.create` → el objeto bajo el cursor.
6. **Transición**: al morir → cámara lenta de la cinemática (`make_current` en la `CinematicCamera` animada por `AnimationPlayer`).

## Integración

- **Personaje**: `godot-characterbody3d` — `get_rid()` para excluir del cast del brazo.
- **Input**: `godot-input` — `_unhandled_input` (motion/rueda) + `get_joy_axis` (stick derecho).
- **UI**: `project_position` → anclar marcadores; `is_position_in_frustum` → ocultar.
- **Audio**: `Camera3D.velocity` + `doppler_shift` para el efecto Doppler de pasados.

## Errores frecuentes

1. **Buscar "smoothing" en `Camera3D`** — **no existe en 4.0 ni 4.7** (verificado; es de `Camera2D`). → damping manual `lerp(..., 1 - exp(-k*delta))` (patrón *Implementación mínima*).
2. **`SpringArm3D` con cámara no hija directa y sin `shape`** → el brazo cae a raycast "impreciso, no recomendado" (docs). → hija directa o `shape` explícito.
3. **La cámara choca contra el propio jugador** → falta `add_excluded_object(player.get_rid())` y/o `collision_mask` incluye la capa del jugador.
4. **Múltiples `Camera3D` con `current = true`** → "la última manda" (el viewport usa una sola). → `clear_current()` en la vieja.
5. **`fov` en grados pensando en radianes** (o al revés) → `fov` es **grados** (default 75). Para radianes: `deg_to_rad` solo mentalmente; la prop ya es grados.
6. **`unproject_position` creyendo que devuelve un rayo** → devuelve un punto 3D a lo largo de la línea de vista (en la distancia del punto de pantalla); para click→mundo usar `project_ray_origin` + `project_ray_normal`.
7. **`project_position` para "¿está mirando al objeto?"** → `project_position` siempre devuelve algo (extrapolado fuera de pantalla). Para visibilidad: `is_position_in_frustum` (sin oclusión) o un raycast (con oclusión).
8. **`near` por defecto (0.05) en FPS** → la cabeza del jugador se corta al girar rápido (el near recorta a 5 cm). Subir a 0.1–0.3 en primera persona.
9. **`far` = 4000 en un mundo de 200 m** → no rompe nada, pero desperdicia precisión de Z; ajustar `far` al mundo real (bajar puede dar "pops" de culling si hay niebla/objetos lejanos: medir).
10. **Rotar la `Camera3D` en vez del pivot en TPS** → el `SpringArm3D` apunta donde apunta la cámara (no "hacia atrás del jugador"); la órbita se rompe. El pivot rota, la cámara mira "hacia atrás" del pivot.
11. **`cull_mask` = 0** (sin darse cuenta) → la cámara no renderiza nada. Default 1048575 (todo).
12. **Doppler sin `velocity`** → `doppler_shift` con `velocity = 0` no hace nada (el efecto depende de la velocidad de la cámara).

## Anti-patrones

- **Instanciar/destrozar cámaras para "resetear"** → mutar transform/`make_current`; la cámara es un recurso que se reutiliza.
- **Lerp de cámara sin `delta`** (factor fijo 0.1 por frame) → depende de la Hz; `1 - exp(-k*delta)` es lo correcto.
- **`project_position` en un bucle sobre 500 enemigos cada frame** → el coste de la proyección es bajo, pero 500 × (proyección + frustum + UI) por frame sí suma: solo proyectar los enemigos **visibles** (frustum first) y en un intervalo (10 Hz) si no es crítico.
- **Varias cámaras "por si acaso" con `current`** → una por viewport; las de reserva, `current = false`.
- **Animar la cámara con `AnimationPlayer` y también con script en el mismo eje** → doble escritura: elegir una sola fuente por transform (el script o la animación).
- **`SpringArm3D` con `shape` gigante (sphere 2 m)** → la cámara "huye" antes de tiempo; la pirámide del near plane (hija directa) es el balance correcto.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- La cámara casi nunca es el cuello de botella: su coste es CPU despreciable (transformaciones) + el **culling de frustum** que habilita (cuanto más ajustado el `far`, más útil).
- Lo que sí cuesta (GPU, por la cámara): `near`/`far` mal ajustados (precisión de Z), `cull_mask` renderizando capas innecesarias, y post-proceso (ver `godot-rendering-performance`).
- Medición: `Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)` y `RENDER_TOTAL_OBJECTS_IN_FRAME` antes/después de ajustar `far`/`cull_mask`.

## Debugging

### "La cámara atraviesa muros"

```text
1. ¿SpringArm.collision_mask incluye la capa del mundo? (default = solo capa 1)
2. ¿La Camera3D es hija DIRECTA del brazo (o hay shape)?
3. ¿margin = 0? → 0.01–0.05 (la cámara queda en el punto exacto de la colisión)
4. print(_spring.get_hit_length(), _spring.spring_length)
   → hit < length = colisión activa (correcto); hit == length y se atravasa →
     el cast no toca nada (capas) o es raycast (hijo no directo, sin shape).
```

### "La cámara se corta a sí misma / al jugador"

```text
1. FPS: near 0.05 → subir a 0.1–0.3.
2. TPS: cull_mask incluye la capa del jugador → quitar el bit.
```

### "La órbita se siente 'al revés' / la cámara no orbita"

```text
1. ¿Se rota el pivot o la cámara? → el pivot (error #10)
2. ¿Signo del yaw? → rotation.y -= amount.x (estándar: mover el ratón a la derecha
   gira la cámara a la izquierda del jugador)
3. ¿Tilt limitado? → clampf en rotation.x (error: sin clamp, la cámara da la vuelta
   por encima del jugador).
```

### "Los marcadores UI van al revés / fuera de pantalla"

```text
1. ¿El objeto está en el frustum? → is_position_in_frustum (ocultar si no)
2. ¿El HUD es en el mismo viewport? → project_position da coords del viewport de
   la cámara; si el HUD es en otro viewport, hay que mapear coords.
3. ¿La cámara tiene h_offset/v_offset? → afectan la proyección (el marcador se
   desplaza igual que la imagen: coherente).
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — `Camera3D.xml` (4.7) + docs stable.
- **4.0 ↔ 4.7**: API de `Camera3D` estable entre ambas ramas (verificado en `GODOT_VERSION_MATRIX.md`: `near/far/fov/projection/cull_mask/h_offset/v_offset/make_current/project_*` presentes en ambas).
- **Sin smoothing en ninguna 4.x** (verificado 4.0 y 4.7) — si un tutorial 3D menciona "position smoothing" en cámara 3D, es error del tutorial (es de `Camera2D`).
- **Godot 3 → 4**: `Camera3D` cambió poco (los métodos `project_ray_*` existían en 3.x con la misma idea); los valores de `near/far` defaults pueden diferir de 3.x (en 3.x `near = 0.1`? verificar por caso).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-input` | `_unhandled_input` + `get_joy_axis` para orbitar/zoom |
| `godot-characterbody3d` | `get_rid()` para excluir del cast del brazo |
| `godot-third-person-character` | Compuesta que integra este rig (Lote 0) |
| `godot-cinematic-camera` (pendiente) | Transiciones/cinematografía |
| `godot-recipe-first-person-controller` (pendiente) | Rig FPS (cámara en la cabeza) |

## Skills relacionadas

- `godot-error-camera-clipping` — diagnóstico dedicado (pendiente)
- `godot-springarm3d` — el brazo en detalle (aquí ya cubierto lo operativo; pendiente si hace falta)
- `godot-xr` — cámaras VR (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- Camera3D: https://docs.godotengine.org/en/stable/classes/class_camera3d.html
- SpringArm3D: https://docs.godotengine.org/en/stable/classes/class_springarm3d.html
- Tutorial "Third-person camera with spring arm" (receta oficial del rig): https://docs.godotengine.org/en/stable/tutorials/3d/spring_arm.html
- PhysicsRayQueryParameters3D (click → rayo): https://docs.godotengine.org/en/stable/classes/class_physicsrayqueryparameters3d.html
- Viewport (una cámara activa): https://docs.godotengine.org/en/stable/classes/class_viewport.html
