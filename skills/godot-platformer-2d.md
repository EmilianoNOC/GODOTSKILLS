# godot-platformer-2d

## ID

`godot-platformer-2d`

## Categoría

2D · Base (juego)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: páginas de clase `CharacterBody2D` (tabla completa: props/métodos/enums) y `CollisionShape2D` (one-way), tutorial `using_character_body_2d.html` (patrón oficial) y `physics_introduction.html` (ejemplos 2D con código) (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — toda la tabla de `CharacterBody2D` (16 props con defaults 2D, 18 métodos, enums `MotionMode`/`PlatformOnLeave`), `move_and_slide()` **void** en 2D, `one_way_collision`/`direction`/`margin` de `CollisionShape2D`, "move_and_slide es especialmente útil en platformers" (oficial), patrón de gravedad (no multiplicar `velocity` por `delta`, sí la gravedad), `get_slide_collision_count` "solo cuenta cambios de dirección" (nota oficial), `CharacterBody2D` para plataformas móviles/proyectiles (tip oficial): verificados directamente en la docs 4.7.
**MEDIUM** — valores de feel (jump velocity, gravedad, coyote/buffer) — no son reglas oficiales; son puntos de partida para iterar.
**Diferencias 2D↔3D verificadas**: `move_and_slide()` → **void (2D)** vs **bool (3D)**; defaults distintos (`safe_margin` 0.08 vs 0.001, `floor_snap_length` 1.0 vs 0.1, `max_slides` 4 vs 6, `up_direction` (0,-1) vs (0,1,0)).

## Nivel

**BASE** (depende de `godot-node2d` y `godot-physics`; se apoya en `godot-character-controller` para el feel).

## Propósito

Proporcionar el conocimiento operativo para construir un platformer 2D en Godot 4.7: `CharacterBody2D` + `move_and_slide()`, el patrón de gravedad oficial, saltos, pendientes, plataformas una-vía (`one_way_collision`), plataformas móviles (`platform_*`), y el diagnóstico de los fallos clásicos (se queda pegado, atraviesa pisos, no salta). Es la skill de juego más repetida de la biblioteca 2D.

## Cuándo utilizarla

- Hacer un personaje de platformer 2D (saltos, gravedad, slide).
- Plataformas una-vía (puedes pasarlas por abajo).
- Plataformas móviles (elevadores, cintas) con `platform_floor_layers`.
- Depurar "el personaje se atasca / atraviesa / no salta".
- Proyectiles con rebote/ricochet (`move_and_collide`, tip oficial).

## Cuándo NO utilizarla

- **Top-down** → `MOTION_MODE_FLOATING` (sin piso/techo; verificado: "suitable for top-down games") o `godot-character-controller` (feel).
- **Personaje 3D** → `godot-characterbody3d`/`godot-character-controller` (misma familia, defaults distintos).
- **Física libre (cajas, balas físicas)** → `RigidBody2D` (`godot-physics`); no CharacterBody.
- **El feel del control** (coyote time, jump buffer, air control, curvas de aceleración) → `godot-character-controller` (la skill de tuning; esta da la base física).
- **IA de enemigos con pathing** → `NavigationRegion2D`/`NavigationAgent2D` (Lote 10 pendiente).

## Conceptos fundamentales

### `CharacterBody2D`: detecta, no es afectado

Texto oficial (4.7): "They are **not affected by physics at all**, but they **affect other physics bodies** in their path." — la gravedad/fricción **no** las aplica el motor; las programas tú (tip oficial: "A CharacterBody2D can be affected by gravity and other forces, but you must calculate the movement in code").

- Moverlo: **`move_and_slide()`** (slide automático; "especially useful in platformers or top-down games" — oficial) o **`move_and_collide(velocity * delta)`** (sin respuesta automática; devuelve `KinematicCollision2D`; útil para **ricochet de balas** — oficial).
- **Nunca setear `position` a mano** (oficial): usar `move_and_slide`/`move_and_collide`.
- En `_physics_process` **siempre** (warning oficial).
- Tip oficial: sirve también para **plataformas móviles** y **proyectiles complejos** (movimiento manual preciso). Para plataformas sin lógica: `AnimatableBody2D` es más simple (oficial).

### El patrón de movimiento (oficial, de `physics_introduction` 4.7)

```gdscript
extends CharacterBody2D

var run_speed := 250.0    # px/s (MEDIUM: punto de partida)
var jump_speed := -400.0  # negativo = arriba (Y abajo, ver godot-node2d)
var gravity := 1000.0     # px/s² (MEDIUM)

func _physics_process(delta: float) -> void:
	velocity.y += gravity * delta          # gravedad SÍ se multiplica por delta
	if Input.is_action_just_pressed("ui_up") and is_on_floor():
		velocity.y = jump_speed
	var dir := Input.get_axis("ui_left", "ui_right")
	velocity.x = dir * run_speed
	move_and_slide()                        # NO multiplicar velocity por delta (oficial)
```

**Warning oficial**: `move_and_slide()` **ya incluye el timestep** en su cálculo → **no** multiplicar `velocity` por `delta`; la gravedad **sí** (es aceleración). (Mismo que en 3D — verificado en ambas docs.)

### Piso, pared, techo: la trinidad de `GROUNDED`

- `MOTION_MODE_GROUNDED (0)` (default): "suitable for **sided games like platformers**" (oficial) → piso/techo/paredes tienen sentido; reacciona a pendientes (acelera/baja en pendientes).
- `MOTION_MODE_FLOATING (1)`: sin piso/techo; **toda** colisión se reporta como `on_wall`; slide a velocidad constante (top-down).
- Querys: `is_on_floor()` / `is_on_wall()` / `is_on_ceiling()` (+ `_only` variants). Nombres verificados; **no hay señales** de piso/pared en `CharacterBody2D` (ni en 4.0 ni en 4.7 — verificado en la matriz de versiones).
- Propiedades clave (defaults 2D verificados): `up_direction (0,-1)`, `floor_max_angle 45°` (0.7853982), `floor_snap_length 1.0` (px que se "pegás" al piso), `wall_min_slide_angle 15°` (0.2617994), `floor_stop_on_slope true` (no resbalar en pendiente parado), `floor_block_on_wall true` (no caminar sobre paredes), `floor_constant_speed false` (velocidad variable en pendientes; con `true` + `floor_snap_length` mantiene velocidad), `max_slides 4`, `safe_margin 0.08`, `slide_on_ceiling true`.

### Colisiones de slide: `get_slide_collision_count()`

- Tras `move_and_slide()`, puede haber varias colisiones (el slide cambia de dirección). Iterar: `for i in get_slide_collision_count(): var c = get_slide_collision(i)`.
- **Nota oficial**: el count "only counts times the body has collided **and changed direction**" (no cada micro-contacto).
- `KinematicCollision2D`: `get_collider()`, `get_normal()`, `get_position()`, etc. (mismo contrato que 3D).

### Plataformas: una-vía y móviles

- **Una-vía**: `CollisionShape2D.one_way_collision = true` (verificado) — solo detecta colisión por un lado (arriba por defecto: `one_way_collision_direction = (0,1)`). `one_way_collision_margin` (1.0 px): "Higher values will make the shape thicker, and work better for colliders that enter the shape at a high velocity" (oficial). **Nota oficial**: no tiene efecto si el `CollisionShape2D` es hijo de un `Area2D`.
- **Móviles**: `platform_floor_layers` (default todas las capas: 4294967295) y `platform_wall_layers` (default 0) marcan qué cuerpos cuentan como plataforma (te llevan al moverse); `platform_on_leave`: `ADD_VELOCITY (0)` / `ADD_UPWARD_VELOCITY (1)` ("keep full jump height even when the platform is moving down" — oficial) / `DO_NOTHING (2)`; `get_platform_velocity()` para leer la velocidad heredada.

## API relevante

Todo verificado en Godot 4.7.

### `CharacterBody2D` (hereda `PhysicsBody2D` → `CollisionObject2D` → `Node2D`)

Props (16, con defaults **2D**):

| Prop | Default (2D) | Nota |
|---|---|---|
| `velocity` | `(0,0)` | px/s. `move_and_slide` la modifica al chocar. |
| `up_direction` | `(0,-1)` | Qué cara es "piso". |
| `motion_mode` | `MOTION_MODE_GROUNDED (0)` | `FLOATING (1)` = top-down. |
| `floor_max_angle` | `0.7853982` (45°) | Angulo máx. para contar "piso". |
| `floor_snap_length` | `1.0` | Pegado al piso (px). |
| `floor_constant_speed` | `false` | Vel. constante en pendientes (con `floor_snap_length`). |
| `floor_stop_on_slope` | `true` | No resbalar parado en pendiente. |
| `floor_block_on_wall` | `true` | No caminar sobre paredes. |
| `wall_min_slide_angle` | `0.2617994` (15°) | Ángulo mín. para slide. |
| `slide_on_ceiling` | `true` | Slide en techo. |
| `max_slides` | `4` | Colisiones máx. por slide. |
| `safe_margin` | `0.08` | Margen de seguridad (px). |
| `platform_floor_layers` | `4294967295` | Capas que cuentan como plataforma (piso). |
| `platform_wall_layers` | `0` | Capas que cuentan como plataforma (pared). |
| `platform_on_leave` | `ADD_VELOCITY (0)` | Qué pasa al dejar la plataforma. |

Métodos (18):

| Método | Devuelve | Nota |
|---|---|---|
| `move_and_slide()` | **void** | El patrón (no args en 4.x). |
| `apply_floor_snap()` | — | Re-aplicar snap (tras saltar). |
| `is_on_floor()` / `is_on_floor_only()` | `bool` | Piso. |
| `is_on_wall()` / `is_on_wall_only()` | `bool` | Pared. |
| `is_on_ceiling()` / `is_on_ceiling_only()` | `bool` | Techo. |
| `get_floor_normal()` / `get_wall_normal()` | `Vector2` | Normales (world). |
| `get_floor_angle(up_direction=(0,-1))` | `float` | Ángulo de la pendiente. |
| `get_slide_collision_count()` | `int` | "solo cuenta cambios de dirección" (oficial). |
| `get_slide_collision(idx)` / `get_last_slide_collision()` | `KinematicCollision2D` | Detalle. |
| `get_last_motion()` | `Vector2` | Último movimiento aplicado. |
| `get_position_delta()` | `Vector2` | Delta del último movimiento. |
| `get_real_velocity()` | `Vector2` | Velocidad "real" (con plataformas). |
| `get_platform_velocity()` | `Vector2` | Velocidad de la plataforma bajo el body. |

### `CollisionShape2D` (para una-vía; tabla completa 4.7)

| Prop | Default | Nota |
|---|---|---|
| `shape` | — | `Shape2D`. |
| `one_way_collision` | `false` | Solo un lado (oficial: sin efecto en hijos de `Area2D`). |
| `one_way_collision_direction` | `(0,1)` | Dirección del lado sólido. |
| `one_way_collision_margin` | `1.0` | px; más = mejor para entradas a alta velocidad (oficial). |
| `disabled` | `false` | Cambiar con `set_deferred` (recomendación oficial). |
| `debug_color` | — | Ver el shape (editor + runtime debug). |

## Arquitectura recomendada

### Escena del platformer

```text
Level (Node2D)
├─ Ground (TileMapLayer)          # godot-tilemap (physics layer pintado)
├─ Platform (StaticBody2D)        # plataforma móvil/una-vía
│   └─ CollisionShape2D (RectangleShape2D)
│      # one_way_collision = true para una-vía
└─ Player (CharacterBody2D)
   ├─ Sprite2D
   └─ CollisionShape2D (RectangleShape2D 12×14)   # el shape, no el mesh
      Camera2D (hija del player)                   # godot-camera2d
```

1. **El shape del player es más chico que el sprite** (feels más preciso; el sprite "salta" visualmente).
2. **Piso = TileMapLayer o StaticBody2D**; plataformas especiales = `StaticBody2D` (o `AnimatableBody2D` si se animan y deben empujar — oficial).
3. **Una capa de física para el mundo** (capa 1) y el player (capa 2, mask 1): el cruce layer/mask lo resuelve `godot-physics`.
4. **Feel en la skill separada**: esta da la física; `godot-character-controller` da coyote/buffer/curvas (reutilizarla, no re-inventarla).

## Implementación mínima

**Platformer base** (el patrón oficial de `physics_introduction` 4.7, comentado):

```gdscript
# player.gd — CharacterBody2D (todo verificado 4.7)
extends CharacterBody2D

@export var run_speed := 250.0
@export var jump_speed := -400.0
@export var gravity := 1000.0

func _physics_process(delta: float) -> void:
	velocity.y += gravity * delta          # aceleración → × delta
	if Input.is_action_just_pressed("jump") and is_on_floor():
		velocity.y = jump_speed
	velocity.x = Input.get_axis("left", "right") * run_speed
	move_and_slide()                        # sin × delta (oficial)
```

## Implementación recomendada

### 1. Plataforma una-vía (cruzás por abajo)

```text
Platform (StaticBody2D)
└─ CollisionShape2D (RectangleShape2D 64×8)
   one_way_collision = true
   one_way_collision_direction = (0, 1)   # sólido desde arriba (default)
   one_way_collision_margin = 2.0         # subirla si a velocidad alta atravesás (oficial)
```

- Con `floor_snap_length` del player (1.0 default), el snap te deja caer a través si no estás "pegado".

### 2. Plataforma móvil que lleva al player

```gdscript
# elevator.gd — AnimatableBody2D (o StaticBody2D movido por código)
extends AnimatableBody2D

@export var range := 120.0
@export var speed := 1.2
var t := 0.0

func _physics_process(delta: float) -> void:
	t += delta * speed
	position = position + Vector2(0, cos(t) * range * 0.01)
# El player (CharacterBody2D) con platform_floor_layers = capa del elevador
# se mueve con él; al saltarse, platform_on_leave decide (ADD_UPWARD_VELOCITY
# mantiene la altura de salto aunque la plataforma baje — oficial).
```

### 3. Doble salto simple

```gdscript
var jumps_left := 2

func _physics_process(delta: float) -> void:
	velocity.y += gravity * delta
	if Input.is_action_just_pressed("jump"):
		if is_on_floor():
			jumps_left = 2
		elif jumps_left > 0:
			velocity.y = jump_speed
			jumps_left -= 1
	velocity.x = Input.get_axis("left", "right") * run_speed
	move_and_slide()
```

### 4. Bala con ricochet (`move_and_collide`, tip oficial)

```gdscript
extends CharacterBody2D   # sí: "moving platforms or complex projectiles" (oficial)

@export var speed := 600.0
var dir := Vector2.RIGHT

func _physics_process(delta: float) -> void:
	var collision := move_and_collide(dir * speed * delta)   # SÍ × delta (oficial)
	if collision:
		dir = dir.bounce(collision.get_normal())             # ricochet
		if bounce_count >= max_bounces:
			queue_free()
```

### 5. Pendientes: sentirse bien

```gdscript
# En el player:
floor_max_angle = deg_to_rad(60.0)   # pendientes hasta 60° cuentan como piso
floor_constant_speed = true           # velocidad constante en pendientes
floor_snap_length = 2.0               # necesario para "pegarse" en pendiente descendente (oficial)
# Leer la pendiente para animar/rotar el sprite:
var slope := get_floor_angle()        # rad (verificado)
```

## Ejemplo práctico

**Juego**: platformer 2D con 8 levels, plataformas una-vía y elevadores.

1. **Player**: receta base + `floor_constant_speed=true`, `floor_max_angle=60°`, `floor_snap_length=2.0` (MEDIUM: iterar con el feel).
2. **Coyote/buffer**: `godot-character-controller` (la capa de feel sobre esta base) — no re-implementar aquí.
3. **Pisos**: `TileMapLayer` con physics pintado (`godot-tilemap`); plataformas especiales: `StaticBody2D` una-vía (receta #1).
4. **Elevadores**: `AnimatableBody2D` (receta #2) en capa 3; el player tiene `platform_floor_layers` con la capa 3.
5. **Mecánica de bala**: `move_and_collide` ricochet (receta #4).
6. **Debug**: "atravieso la plataforma una-vía corriendo" → subir `one_way_collision_margin` (oficial: alta velocidad); "me quedo pegado en la pendiente" → `floor_snap_length`/`floor_constant_speed`; "no detecto el piso en saltos altos" → `is_on_floor()` en `_physics_process`, no en `_process`.

## Integración

- **`godot-character-controller`**: el feel (coyote, buffer, air control) se construye sobre esta base; esta skill no duplica su tuning.
- **`godot-tilemap`**: el level (physics de tiles); el player choca con el physics layer del `TileSet`.
- **`godot-physics`**: layers/masks, `_physics_process` (space locked), troubleshooting (tunneling, ticks).
- **`godot-physics-materials`**: fricción/rebote de superficies (`PhysicsMaterial` en `StaticBody2D.physics_material_override` / physics layer del TileSet).
- **`godot-camera2d`**: la cámara sigue al player (smoothing oficial de Camera2D).
- **`godot-area3d` (espíritu)**: pickups/zonas = `Area2D` (skill pendiente `godot-area2d`; espejo de la 3D).

## Errores frecuentes

1. **Multiplicar `velocity` por `delta` en `move_and_slide()`** → se acelera con el framerate (warning oficial). La gravedad sí se multiplica.
2. **Setear `position` a mano** → rompe el state del slide (oficial: usar `move_and_slide`/`move_and_collide`).
3. **`move_and_slide` en `_process`** → space 2D locked / estado inconsistente (oficial: `_physics_process`).
4. **Esperar señales `floor_entered`/`wall_entered`** → no existen en 4.7 (verificado en la matriz); polling `is_on_floor()` en `_physics_process`.
5. **`move_and_slide(safe_margin)` (3.x)** → en 4.x no toma argumentos (la prop `safe_margin` existe; default 2D 0.08).
6. **Atravesar una-vía a alta velocidad** → `one_way_collision_margin` (oficial: "work better for colliders that enter the shape at a high velocity").
7. **Una-vía en un `Area2D`** → no tiene efecto (nota oficial: solo en bodies).
8. **El player no "se pega" al piso en pendiente descendente** → `floor_snap_length` (necesario con `floor_constant_speed`, oficial).
9. **Top-down usando `GROUNDED`** → usar `MOTION_MODE_FLOATING` (oficial: todo se reporta `on_wall`).
10. **`get_slide_collision_count()` esperando todos los contactos** → "solo cuenta cambios de dirección" (nota oficial); es un conteo de colisiones que cambiaron el slide.
11. **Esperar que la gravedad/fricción del motor aplique** → CharacterBody **no** es afectado por física (oficial); todo va en código.
12. **2D defaults usando valores 3D** → `safe_margin` 0.08 (2D) vs 0.001 (3D), `floor_snap_length` 1.0 vs 0.1, `max_slides` 4 vs 6 (verificados en ambas).

## Anti-patrones

- **Re-implementar coyote/buffer/air control "a la antigua"** → usar `godot-character-controller` (la skill de feel) sobre esta base.
- **Un `RigidBody2D` para el player** (y luego "controlarlo") → CharacterBody es el tipo para control manual (oficial).
- **Plataforma móvil con `StaticBody2D` + animación esperando que empuje** → se teletransporta (no empuja); `AnimatableBody2D` si debe empujar (oficial).
- **`is_on_floor()` cacheado "por si acaso"** → leer en el tick que lo usás (el estado cambia cada tick).
- **Hardcodear capas en vez de usar `platform_floor_layers`** (para plataformas) → la API existe para eso (verificada).
- **Shape del player = mesh exacto** → shape más chico que el sprite (feels preciso; el sprite es visual).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El coste del platformer está en la física 2D**, no en el código: cuerpos activos + pairs de colisión. Monitores `Performance.PHYSICS_2D_*` (familia existe; **índices exactos no verificados en esta revisión** — verificar en `Performance.xml`).
- **Qué medir**: FPS y pair count con/without el sistema de plataformas (elevadores, una-vías); `max_slides` alto multiplica las colisiones por tick.
- **Optimizaciones (contra un cuello medido)**:
  1. Menos plataformas activas (las de fuera de pantalla: `enabled=false` del body o `disabled` del shape con `set_deferred`).
  2. `max_slides` acorde (4 default; subirlo multiplica trabajo).
  3. Shapes simples (rectángulo para player; no polygons complejos).
  4. Ticks: el remedio oficial de inestabilidad/velocidad es el tick rate (`physics_ticks_per_second` 120/180/240 — ver `godot-physics`), no más `max_slides`.
- **No**: "el platformer es simple, no mide" — el pair count de 20 plataformas móviles + tiles lo cambia todo.

## Debugging

### "El player no salta / salta débil"

```text
1. ¿is_on_floor() es true en el tick del salto? (imprimirlo; leer en
   _physics_process)
2. ¿jump_speed negativo? (Y abajo: arriba = negativo)
3. ¿El salto se pisa con gravity * delta en el mismo tick? (orden:
   gravity primero, luego el salto — el patrón oficial lo muestra)
```

### "Atraviesa pisos/plataformas a alta velocidad"

```text
1. una-vía: one_way_collision_margin (oficial: alta velocidad)
2. Velocidad general: safe_margin (2D 0.08) y/o physics_ticks_per_second
   120+ (solución oficial de godot-physics)
3. floor_snap_length bajo → no se pega al piso en transiciones
```

### "Se queda pegado / no desciende de la pendiente"

```text
1. floor_constant_speed + floor_snap_length (oficial: se necesita para
   pegarse en pendiente descendente)
2. floor_max_angle (¿la pendiente es "piso" o "pared"?)
3. floor_stop_on_slope (true = no resbala parado; si querés que resbale,
   false)
```

### "La plataforma móvil no me lleva / me deja"

```text
1. ¿La plataforma está en una capa que está en platform_floor_layers?
   (default: todas; si la cambiaste, verificar)
2. platform_on_leave: ADD_VELOCITY (default) suma la velocidad al dejarla;
   ADD_UPWARD_VELOCITY ignora el movimiento hacia abajo (oficial)
3. ¿La plataforma es StaticBody3D/2D movida por código? → se teleporta
   (no empuja); para empujar: AnimatableBody2D (oficial)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tablas de `CharacterBody2D`/`CollisionShape2D` + tutoriales (lista en *Referencias*).
- **3.x → 4.x (trampas, verificado contra tablas 4.7)**:
  - `move_and_slide(safe_margin)` → `move_and_slide()` (sin args; prop `safe_margin`).
  - Señales `floor_entered/wall_entered/ceiling_entered` → **no existen** (4.0 y 4.7, verificado).
  - `KinematicBody2D` → `CharacterBody2D` (renombrado, ver matriz).
  - Defaults 2D vs 3D distintos (ver tabla *Confidence*).
- **4.0 ↔ 4.7**: la tabla consultada es estable para el uso cubierto.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-node2d` | Base (Y abajo, transform) |
| `godot-physics` | Layers/masks, ticks, troubleshooting, `_physics_process` |
| `godot-tilemap` | El level (physics de tiles) |
| `godot-character-controller` | El feel (coyote/buffer/air control) sobre esta base |
| `godot-physics-materials` | Fricción/rebote de superficies |
| `godot-camera2d` | La cámara que sigue al player |
| `godot-characterbody3d` (3D) | La contraparte 3D (misma familia, defaults distintos) |

## Skills relacionadas

- `godot-recipe-coyote-time` — pendiente (germen en `godot-character-controller`)
- `godot-recipe-moving-platform-2d` — pendiente (germen #2)
- `godot-error-platformer-clipping` — pendiente (germen en *Debugging*)
- `godot-area2d` (pendiente, Lote 5 ext.) — pickups/zonas del platformer

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- CharacterBody2D (props/métodos/enums completos, defaults 2D, `AnimatableBody2D`): https://docs.godotengine.org/en/stable/classes/class_characterbody2d.html
- CollisionShape2D (one_way_collision/direction/margin, `disabled` deferred): https://docs.godotengine.org/en/stable/classes/class_collisionshape2d.html
- Using CharacterBody2D/3D (patrón oficial: `move_and_slide`, `move_and_collide`, props de slide, `get_slide_collision_*`, tip de plataformas/proyectiles): https://docs.godotengine.org/en/stable/tutorials/physics/using_character_body_2d.html
- Physics introduction (código 2D de gravedad/salto, `move_and_slide` sin `delta`, `move_and_collide` con `delta`): https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html
- CharacterBody3D (contraste 3D: `move_and_slide() -> bool`, defaults 3D): https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html
