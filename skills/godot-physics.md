# godot-physics

## ID

`godot-physics`

## Categoría

Física · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: páginas de clase (`RigidBody3D`, `StaticBody3D`, `PhysicsBody3D`, `CollisionShape3D`, `CollisionObject3D`) y tutoriales `physics_introduction.html`, `troubleshooting_physics_issues.html` (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — tipos de cuerpos, propiedades/métodos/señales de `RigidBody3D`/`StaticBody3D`/`PhysicsBody3D`, `CollisionShape3D`, layers/masks, sleep, contact reporting, `physics_ticks_per_second`, *spiral of death*, soluciones de troubleshooting (tunneling, pilas, cilindros, tiles, precisión alejada del origen), Jolt: verificados directamente en la docs 4.7.
**MEDIUM** — mapa de capas recomendado (convención de proyecto, no regla oficial) y valores de tuning (multiplicadores de ticks: la docs recomienda 120/180/240 pero el valor final es decisión de proyecto).
**"No existe en la tabla oficial 4.7 (verificado)"**: `RigidBody3D.motion_mode` (en 4.7 el CCD es el bool `continuous_cd`), `StaticBody3D.force_recompute_shapes`, propiedades de interpolación en `PhysicsBody3D`, `PhysicsMaterial3D` como clase aparte (la clase es `PhysicsMaterial`), enums `*_combine_mode` en materiales de física.

## Nivel

**BASE** (sin prerrequisitos; es la base de `godot-characterbody3d`, `godot-raycast3d`, `godot-area3d`, `godot-physics-materials` y de la compuesta `godot-third-person-character`).

## Propósito

Proporcionar el conocimiento operativo de la física 3D en Godot 4.7: qué tipo de cuerpo usar (Static/Rigid/Character), cómo corren los ticks de física, layers y masks, shapes de colisión, sleep/contactos, y el mapa completo de problemas típicos con sus soluciones oficiales (tunneling, inestabilidad, *spiral of death*, precisión de punto flotante). Es la capa sobre la que se construye cualquier sistema físico del juego.

## Cuándo utilizarla

- Elegir entre `StaticBody3D`, `RigidBody3D`, `CharacterBody3D` (y saber cuándo NO es física: `Area3D` puro o cinemática manual).
- Configurar layers/masks de colisión 3D.
- Depurar "el objeto atraviesa el piso / la pila se va / el FPS se va a 1".
- Entender el ciclo de física (60 Hz, `_physics_process`, `delta`) antes de escribir cualquier código de movimiento.
- Configurar `physics_ticks_per_second` o evaluar el motor Jolt.

## Cuándo NO utilizarla

- **Movimiento de personaje con slide/gravedad** → `godot-characterbody3d` (esta skill cubre el contexto, esa cubre el movimiento).
- **Raycasts y shapecasts** (nodos y consultas directas) → `godot-raycast3d`.
- **Detección/zonas (pickups, triggers)** → `godot-area3d`.
- **Fricción/rebote de superficies** → `godot-physics-materials`.
- **Vehículos** → `VehicleBody3D` (skill pendiente; esta skill da el contexto de RigidBody).
- **Soft bodies / fluidos** → fuera de scope (no hay skill).

## Conceptos fundamentales

### Colisión vs detección; los cuatro tipos de objeto

Godot ofrece cuatro tipos de objetos de colisión 3D (todos heredados de `CollisionObject3D`; los tres últimos además de `PhysicsBody3D`). La docs los describe con estas reglas de selección:

| Tipo | Qué hace | No hace | Caso típico |
|---|---|---|---|
| `StaticBody3D` | Participa en detección y respuesta; "sólido". Puede simular movimiento con `constant_linear_velocity`/`constant_angular_velocity` (cinta, plataforma giratoria). | No se mueve por la simulación. Al moverse por código/animación se **teletransporta** (no empuja a otros). | Pisos, muros, plataformas, cintas. |
| `RigidBody3D` | Simulación completa: gravedad, fuerzas, impactos, rotación, rebote. Se controla aplicando fuerzas, no moviendo. | No se mueve directamente (position/velocity directos → comportamiento impredecible, warning oficial). | Cajas que se tiran, balas físicas, objetos del mundo. |
| `CharacterBody3D` | Detección de colisiones; el movimiento y la respuesta **los programas tú** (`move_and_slide`/`move_and_collide`). | El motor no lo mueve ni aplica gravedad/fricción. | Jugador, NPCs. Ver `godot-characterbody3d`. |
| `Area3D` | **Detección + influencia**: señales de entrada/salida y override local de gravedad/damping/viento/audio. | No es sólido: nada choca contra un Area. | Pickups, zonas de daño, portales, zonas de gravedad. Ver `godot-area3d`. |

Decisiones oficiales que resuelven la mayoría de los casos:

- ¿El objeto se mueve solo por la física (gravedad, impactos)? → **RigidBody**.
- ¿Lo mueves tú con intención de juego (slide, salto, control)? → **CharacterBody**.
- ¿No se mueve o se mueve por animación/código? → **StaticBody** (si al moverlo debe **empujar** a otros, `AnimatableBody3D` — clase que hereda de `StaticBody3D`, verificada 4.7).
- ¿Solo necesitas saber "alguien está adentro"? → **Area** (no un body con colisión).

**Warning oficial**: la física de Godot **no es determinista** ("Physics in Godot… is not deterministic" — la determinación depende de muchos factores). No diseñes sistemas que asuman resultados idénticos en runs distintos (p. ej. repro exacto de demo) sin más.

### El ciclo de física: ticks fijos (60 Hz)

- El motor de física corre a tasa fija: **60 ticks/seg por defecto** (Project Settings `physics/common/physics_ticks_per_second`, verificado). El framerate de render varía y es independiente.
- `_physics_process(delta)` se llama **antes de cada paso de física**, con `delta` ≈ `0.01666…` (no siempre exacto: la docs pide usar `delta` SIEMPRE para que funcione si cambias la tasa de ticks o el equipo no sigue).
- Regla: **código que lee o cambia estado físico → `_physics_process`, no `_process`**. Leer `linear_velocity`/posición en `_process` da valores del último tick (desactualizado respecto al frame renderizado).
- Las consultas al space de física **solo son seguras durante `_physics_process`** (fuera puede estar *locked* y dar error — verificado en el tutorial de ray-casting).

### Layers y masks (32 bits)

- Cada `CollisionObject3D` tiene 32 capas. `collision_layer` = las capas en las que el objeto **aparece**; `collision_mask` = las capas que **escanea**. Si el otro no está en una de las capas de tu mask, lo ignoras.
- **Default: capa 1 en layer y en mask** (ambos `1` en bitmask, verificado en `CollisionObject3D`).
- Nombres de capas en **Project Settings → Layer Names → 3D Physics** (para el 2D es "2D Physics").
- En código, bitmasks:

```gdscript
# Capas 1, 3 y 4 activas (bit = 1 << (capa - 1)):
body.collision_mask = (1 << 1 - 1) | (1 << 3 - 1) | (1 << 4 - 1)  # = 0b1101 = 13
# o bit a bit (verificado):
body.set_collision_mask_value(1, true)
body.set_collision_mask_value(3, true)
body.set_collision_mask_value(4, true)
# Leer un bit:
if body.get_collision_mask_value(2):
    pass
# Todas las capas (queries que lo aceptan): 0xFFFFFFFF = 4294967295
```

El `<<` es más rápido que `pow()`; el `|` evita duplicar bits (ejemplo y justificación de la docs).

### Sleep (los rígidos "duermen")

- Un `RigidBody3D` en reposo sin movimiento **se duerme** (`sleeping = true`): actúa como estático, el motor deja de calcularle fuerzas. Se despierta con una colisión o fuerzas aplicadas por código.
- `can_sleep = false` mantiene el body siempre activo: la docs lo marca como posible override y **advierte del efecto negativo en performance**.
- **Mientras duerme, `_integrate_forces()` NO se llama** (verificado). Si un body dormido necesita lógica (propulsión continua), despiértalo o quítale `can_sleep` (con el coste).
- Señal `sleeping_state_changed()` (verificado). Nota oficial: cambiar `sleeping` a mano **no** emite la señal (solo la emite el motor, o `emit_signal("sleeping_state_changed")` manual).

### Motores de física 3D

- **GodotPhysics** es el motor 3D por defecto. **Jolt** es la alternativa (opción de Project Settings).
- La docs oficial (troubleshooting) recomienda Jolt en 3D para: estabilidad de **pilas de objetos**, confiabilidad de **formas cilíndricas** (GodotPhysics tiene bugs conocidos con cilindros, "many other physics engines don't even support them") y **rendimiento** general de simulación.

## API relevante

Todo verificado en Godot 4.7 salvo donde se marca.

### `CollisionObject3D` (base común; verificado en docs 4.7, Lote 0)

| Propiedad / método | Valor | Nota |
|---|---|---|
| `collision_layer` | `1` (bitmask 32 bits) | Capas en las que aparece. |
| `collision_mask` | `1` | Capas que escanea. |
| `collision_priority` | `1.0` | Mayor prioridad "penetra" en overlaps ambiguos. |
| `disable_mode` | `DISABLE_MODE_REMOVE (0)` | `0` = eliminar del space, `1` = `MAKE_STATIC` (queda sólido, sin dinámica), `2` = `KEEP_ACTIVE` (sigue simulando, sin detectar). |
| `input_ray_pickable` | — | ¿Recibe raycasts de input? (picking por mouse). |
| `get_rid()` | `RID` | Para excluir el body de queries (`exclude`). |
| Señales | `input_event(camera, event, position)`, `mouse_entered()`, `mouse_exited()` | Las areas/bodies reciben input por defecto. |

### `PhysicsBody3D` (base de los tres bodies; verificado 4.7)

| Propiedad / método | Valor | Nota |
|---|---|---|
| `axis_lock_linear_x/y/z` | `false` | Bloqueo de traslación por eje (ejes en `PhysicsServer3D.BodyAxis`: LINEAR_X…ANGULAR_Z). |
| `axis_lock_angular_x/y/z` | `false` | Bloqueo de rotación. |
| `set_axis_lock(axis: BodyAxis, lock: bool)` / `get_axis_lock(axis)` | — | Alternativa a las 6 props. |
| `move_and_collide(motion, test_only=false, safe_margin=0.001, recovery_as_collision=false, max_collisions=1)` | `KinematicCollision3D` | Mueve y resuelve; el body debe ser kinemático (CharacterBody/Rigid congelado). |
| `test_move(from, motion, collision=null, safe_margin=0.001, recovery_as_collision=false, max_collisions=1)` | `bool` | Prueba sin mover (para predecir). |
| `add_collision_exception_with(body)` / `remove_collision_exception_with(body)` | — | Excepciones de colisión entre bodies. |
| `get_gravity()` | `Vector3` | Gravedad efectiva (con overrides de areas). |

**Warning oficial**: escala no uniforme en un body → comportamiento inesperado; mantener scale igual en todos los ejes y ajustar el shape.

### `RigidBody3D` (verificado 4.7, tabla completa)

Propiedades:

| Propiedad | Default | Nota |
|---|---|---|
| `mass` | `1.0` | Masa. |
| `gravity_scale` | `1.0` | Escala la gravedad para este body (`0.0` = flota; `< 1` = menos gravedad). |
| `linear_velocity` / `angular_velocity` | `(0,0,0)` | Velocidades (rad/s la angular). **No setear a mano** (warning oficial): usar fuerzas. |
| `linear_damp` / `angular_damp` | `0.0` | Damping; default real viene de `physics/3d/default_linear_damp`/`default_angular_damp` o de Areas. |
| `linear_damp_mode` / `angular_damp_mode` | `DAMP_MODE_COMBINE (0)` | `COMBINE (0)` = se suma a lo del area/default; `REPLACE (1)` = reemplaza. |
| `constant_force` / `constant_torque` | `(0,0,0)` | Fuerza/torque permanentes (alternativa a `apply_*` en `_physics_process`). |
| `lock_rotation` | `false` | Solo traslación (caja que rueda por el piso). |
| `freeze` / `freeze_mode` | `false` / `FREEZE_MODE_STATIC (0)` | `KINEMATIC (1)`: se mueve por código/animación **chocando con otros a su paso** (cuerpo animado que empuja). |
| `can_sleep` / `sleeping` | `true` / `false` | Ver *Conceptos → Sleep*. |
| `continuous_cd` | `false` | **CCD** contra tunneling (ver *Troubleshooting*). Es el mecanismo 4.7 — en esta versión no hay `motion_mode` en RigidBody. |
| `contact_monitor` | `false` | Habilita las señales `body_entered`/`body_exited`/`body_shape_*`. |
| `max_contacts_reported` | `0` | Contacto reporting (memoria); `0` = desactivado. |
| `center_of_mass` / `center_of_mass_mode` | `(0,0,0)` / `CENTER_OF_MASS_MODE_AUTO (0)` | `CUSTOM (1)` usa `center_of_mass` relativo al origen. |
| `inertia` | `(0,0,0)` | Tensor de inercia (avanzado). |
| `physics_material_override` | — | `PhysicsMaterial` (ver `godot-physics-materials`). |
| `custom_integrator` | `false` | Habilita `_integrate_forces()` custom completo. |

Métodos:

| Método | Nota |
|---|---|
| `_integrate_forces(state: PhysicsDirectBodyState3D)` (virtual) | **El lugar correcto** para cambiar estado físico de un rígido (forces, velocities, transform) de forma segura. NO se llama si el body duerme. |
| `apply_force(force, position=0)` / `apply_central_force(force)` | Fuerza (con torque si `position ≠ 0`) / sin torque. |
| `apply_impulse(impulse, position=0)` / `apply_central_impulse(impulse)` | Impulso (cambio instantáneo de momentum). |
| `apply_torque(torque)` / `apply_torque_impulse(impulse)` | Torque / impulso angular. |
| `add_constant_force(force, position=0)` / `add_constant_central_force(force)` / `add_constant_torque(torque)` | Suman a la fuerza constante. |
| `set_axis_velocity(axis_velocity: Vector3)` | Velocity lineal por eje (world). |
| `get_colliding_bodies()` → `Array[Node3D]` | Bodies en contacto ahora mismo (requiere `max_contacts_reported > 0`). |
| `get_contact_count()` → `int` | nº de contactos (requiere `max_contacts_reported > 0`). |
| `get_inverse_inertia_tensor()` → `Basis` | Avanzado. |

Señales (todas requieren `contact_monitor = true` y `max_contacts_reported` suficientemente alto):

| Señal | Nota |
|---|---|
| `body_entered(body)` / `body_exited(body)` | Contacto comenzó/terminó (también `GridMap` con shapes). |
| `body_shape_entered(body_rid, body, body_shape_index, local_shape_index)` / `body_shape_exited(…)` | Nivel de shape (para saber QUÉ forma chocó). |
| `sleeping_state_changed()` | Ver *Sleep* (no emite al setear `sleeping` a mano). |

Ejemplo oficial adaptado a 3D (propulsión en `_integrate_forces`, patrón "Asteroids" de la docs):

```gdscript
extends RigidBody3D

@export var thrust := Vector3(0.0, 0.0, -500.0)
@export var torque := Vector3(0.0, 0.0, 200.0)

func _integrate_forces(state: PhysicsDirectBodyState3D) -> void:
	if Input.is_action_pressed("ui_up"):
		state.apply_force(thrust.rotated(global_transform.basis))
	if Input.is_action_pressed("ui_right"):
		state.apply_torque(-torque)
	if Input.is_action_pressed("ui_left"):
		state.apply_torque(torque)
```

### `StaticBody3D` (verificado 4.7, tabla completa)

| Propiedad | Default | Nota |
|---|---|---|
| `constant_linear_velocity` | `(0,0,0)` | No mueve el body, pero **afecta a los que tocan** como si se moviera (cinta transportadora). |
| `constant_angular_velocity` | `(0,0,0)` | Igual, para rotación (plataforma giratoria). |
| `physics_material_override` | — | `PhysicsMaterial`. |

- Descripción oficial: al moverse por código/`AnimationMixer`/`RemoteTransform3D` se **teletransporta** (no afecta a los cuerpos en su camino). Si necesitas que un body animado empuje: **`AnimatableBody3D`** (hereda de `StaticBody3D`, verificado).
- Con `AnimationMixer` para moverlo: usar `callback_mode = ANIMATION_CALLBACK_MODE_PROCESS_PHYSICS` (verificado en la página de la clase).

### `CharacterBody3D`

Cubierto en `godot-characterbody3d` (base del Lote 3, verificado 4.7). Solo el mínimo operativo aquí: `move_and_slide()` (NO multiplicar `velocity` por `delta`; sí la gravedad, que es aceleración — warning oficial), `is_on_floor()`, `velocity`. En 4.7 `motion_mode` es **solo `GROUNDED`/`FLOATING`** (no existe modo "space").

### `CollisionShape3D` (verificado 4.7)

| Propiedad | Default | Nota |
|---|---|---|
| `shape` | — | `Shape3D` (Box/Capsule/Sphere/Cylinder/ConvexPolygon/ConcavePolygon…). **Se requiere al menos un shape** para detectar colisiones. |
| `disabled` | `false` | Deshabilita el shape. **Cambiar con `set_deferred()`** (recomendación oficial, no asignación directa). |
| `debug_color` / `debug_fill` | `(0,0,0,0)`* / `true` | *el default real es `debug/shapes/collision/shape_color` (Project Settings); el `(0,0,0,0)` de la docs es placeholder. |
| Método `make_convex_from_sibling()` | — | Convierte la geometría convexa de los `MeshInstance3D` hermanos en un `ConvexPolygonShape3D`. |

**Warning oficial (×2)**: escala no uniforme en `CollisionShape3D`/body → colisiones raras. **Nunca escalar en el editor** (Scale queda `(1,1)`); cambiar el tamaño con los handles del shape. Si el shape es un `Resource` compartido, **Make Unique** (editor) o `duplicate()` (código) antes de cambiar tamaño, o el cambio se aplica a TODOS los usuarios del resource.

### Project Settings de física (verificados 4.7)

| Ajuste | Default | Uso |
|---|---|---|
| `physics/common/physics_ticks_per_second` | `60` | Tasa de simulación. La docs sugiere múltiplos (120/180/240) para estabilidad/veocidad; aumenta CPU (cuidado mobile/web). |
| Max Physics Steps per Frame | — | Techo de pasos por frame renderizado. Subirlo alivia el *spiral of death* (ver *Troubleshooting*). |
| `physics/3d/default_linear_damp` / `default_angular_damp` | — | Damping global de referencia (lo que usan los bodies con `DAMP_MODE_COMBINE`). |
| Layer Names → 3D Physics | — | Nombres de las 32 capas. |

### Monitores de performance (índices verificados en `Performance.xml`, Lote 0)

- `Performance.get_monitor(Performance.PHYSICS_3D_ACTIVE_OBJECTS)` = índice **20**.
- `Performance.get_monitor(Performance.PHYSICS_3D_COLLISION_PAIRS)` = índice **21**.

## Arquitectura recomendada

### Mapa de capas de ejemplo (MEDIUM — convención de proyecto, no oficial)

| Capa | Nombre | Qué va ahí | Mask típica |
|---|---|---|---|
| 1 | `world` | Pisos, muros (Static) | todos |
| 2 | `player` | CharacterBody del jugador | world + interactables |
| 3 | `enemies` | Bodies/areas de enemigos | world + player |
| 4 | `projectiles` | Balas (Rigid/Area) | world + enemies |
| 5 | `interactables` | Puertas, pickups (Areas) | (solo detectado) |
| 6 | `raycast` | Raycasts de personaje (si no son nodos) | world |

Regla de oro de la docs: **la capa define QUÉ ERES, la mask define A QUIÉN TOCAS**. Dos objects interactúan solo si el layer de A está en la mask de B (o viceversa).

### Flujo de decisión (mínima complejidad primero)

```text
¿Necesita sólido (algo choca contra él)?
├─ NO → Area3D (godot-area3d)
└─ SÍ
   ├─ ¿Lo mueve la simulación (gravedad/impactos)?
   │   ├─ SÍ → RigidBody3D (esta skill)
   │   └─ NO, lo muevo yo
   │       ├─ Con respuesta de slide/gravedad de juego → CharacterBody3D (godot-characterbody3d)
   │       ├─ Animación/código, debe empujar a otros → AnimatableBody3D
   │       └─ Animación/código, no importa empujar → StaticBody3D
```

### Plantilla de escena mínima

```text
World (Node3D)
├─ Floor (StaticBody3D)          # layer 1
│   └─ CollisionShape3D (BoxShape3D, espesor >= lo que el ojo no ve)
├─ Player (CharacterBody3D)      # layer 2, mask 1|5  (godot-characterbody3d)
│   └─ CollisionShape3D (CapsuleShape3D)
├─ Box (RigidBody3D)             # layer 1, mask 1
│   └─ CollisionShape3D (BoxShape3D)
└─ Pickup (Area3D)               # layer 5, mask 2  (godot-area3d)
    └─ CollisionShape3D (SphereShape3D)
```

## Implementación mínima

**Un rígido que reacciona y una cinta transportadora** (todo API verificada):

```gdscript
# box.gd — RigidBody3D (solo si necesita lógica; la gravedad/impactos son gratis)
extends RigidBody3D

func _physics_process(_delta: float) -> void:
	if Input.is_action_just_pressed("push"):
		apply_central_impulse(Vector3(0.0, 20.0, 0.0))
```

```gdscript
# conveyor.gd — StaticBody3D: el body no se mueve, pero arrastra lo que toca
extends StaticBody3D

@ready
func _ready() -> void:
	constant_linear_velocity = Vector3(2.0, 0.0, 0.0)  # 2 m/s hacia +X
```

## Implementación recomendada

### 1. Plataforma giratoria (estática, "como si" rotara)

```gdscript
# StaticBody3D: no animar el nodo; usar la velocidad constante (verificado:
# "does not rotate the body, but affects touching bodies, as if it were rotating")
constant_angular_velocity = Vector3(0.0, 2.0, 0.0)  # 2 rad/s
```

### 2. Cuerpo animado que EMPUJA (escaleras móviles, molienda)

```text
StaticBody3D → AnimatableBody3D (mismo código, ahora al moverlo empuja a los
bodies en su camino — verificado en la página de StaticBody3D).
```

### 3. Rígido que se mueve por animación sin física

```gdscript
# Opción A: congelado kinemático (se teleporta, pero empuja al moverse):
freeze = true
freeze_mode = RigidBody3D.FREEZE_MODE_KINEMATIC
# Opción B: deshabilitado estático (pierde toda dinámica):
# disable_mode = CollisionObject3D.DISABLE_MODE_MAKE_STATIC
```

### 4. CCD anti-tunneling (objetos rápidos y paredes finas)

```gdscript
# RigidBody3D rápido (balas, vehículos):
continuous_cd = true
```

Es la primera solución oficial contra tunneling; no siempre basta (ver *Troubleshooting*).

### 5. Contact reporting ("¿con quién estoy tocando?")

```gdscript
# En el inspector o código (AMBOS requisitos):
contact_monitor = true            # habilita señales body_entered/exited
max_contacts_reported = 4         # habilita get_colliding_bodies/get_contact_count

func body_entered(body: Node) -> void:
	print("toqué a ", body.name)

func _physics_process(_delta: float) -> void:
	var tocando: Array[Node3D] = get_colliding_bodies()
```

Por defecto (0) no hay tracking de contactos: la docs lo explica como decisión de memoria.

### 6. Subir la tasa de física (solo si MEDISTE el problema)

```text
Project Settings → physics/common/physics_ticks_per_second = 120
```

- La docs recomienda múltiplos del default (120/180/240) para "smooth appearance on most displays".
- Coste: CPU (no viable en todo mobile/web) — es el remedio oficial para pilas inestables, objetos delgados y vehículos rápidos, NO para "más FPS".

### 7. Colisionador compuesto (tiles 3D)

En vez de un `CollisionShape3D` por tile (bumps al cruzar bordes, verificado como bug conocido), un solo shape grande por isla de tiles. En 2D, `TileMapLayer` (Godot 4.5+) lo hace solo (*Physics Quadrant Size*, default 16 — verificado); en 3D es manual. La docs advierte: el shape compuesto es más complejo, así que **no siempre es net gain en performance** — medir.

## Ejemplo práctico

**Juego**: sandbox 3D con 30 cajas que se pueden empujar, una zona de gravedad media y un piso de concreto.

1. Piso: `StaticBody3D` + `BoxShape3D` (espesor mayor que la cara visible: la docs lo usa como solución de tunneling/wobble).
2. Cajas: `RigidBody3D` (`mass` según tamaño, `lock_rotation = true` si no deben rodar), `BoxShape3D`. Sin script: gravedad, apilado e impactos son "gratis" (beneficio citado por la docs para el estilo Angry Birds).
3. Empujón del jugador: `apply_central_impulse` (no setear `linear_velocity`).
4. Zona de gravedad media: `Area3D` con `gravity = 4.9` y `gravity_space_override` (ver `godot-area3d`).
5. El jugador empuja cajas: `CharacterBody3D` contra `RigidBody3D` — el slide del personaje choca con el rígido y lo mueve (los bodies rígidos son movibles por otros bodies).
6. **Medición**: antes de tocar nada, `PHYSICS_3D_ACTIVE_OBJECTS` y `PHYSICS_3D_COLLISION_PAIRS`; si al apilar 30 cajas el par de valores sube y el FPS baja → ahí está el cuello, no en el render (regla MEASURE-first).

## Integración

- **`godot-characterbody3d`**: el personaje es un `CharacterBody3D`; esta skill le da el contexto de ticks, layers y shapes.
- **`godot-raycast3d`**: raycasts desde el personaje (excepciones con `get_rid()`, mask).
- **`godot-area3d`**: todo lo que no es sólido (pickups, triggers, overrides de gravedad/damping/viento).
- **`godot-physics-materials`**: `physics_material_override` en Static/Rigid para fricción/rebote.
- **`godot-rendering-performance`**: el *spiral of death* de física se mide con los monitores físicos, no con los de render; ambos se miran juntos.
- **Compuesta `godot-third-person-character`**: sus bases pendientes (`godot-physics`) quedan cerradas por esta skill.

## Errores frecuentes

1. **Setear `position`/`linear_velocity` de un RigidBody a mano** → comportamiento impredecible (warning oficial). → `apply_force`/`apply_impulse`, o `_integrate_forces` para acceso directo seguro.
2. **Multiplicar `velocity` por `delta` en `move_and_slide()`** → el personaje se acelera con el framerate (warning oficial: `move_and_slide` ya incluye el timestep; la gravedad SÍ se multiplica).
3. **Escalar el CollisionShape3D en el editor** → colisiones raras (warning oficial ×2, pages de `CollisionShape3D` y `PhysicsBody3D`). → handles de tamaño del shape; Scale `(1,1)`.
4. **Shape compartido modificado sin `duplicate()`/Make Unique** → el tamaño cambia en TODOS los objetos que usan ese resource.
5. **Código de física en `_process`** → lee estado de un tick anterior; fuera de `_physics_process` el space puede estar *locked* (queries dan error). → `_physics_process` (y usar su `delta`).
6. **Objeto atraviesa el piso a alta velocidad (tunneling)** → `continuous_cd = true`; piso más grueso; shape más grande cuanto más rápido; `physics_ticks_per_second` 120+ (soluciones oficiales en orden).
7. **Pila de cajas que "humea"** → más ticks o Jolt (oficial); no "más mass" ni más CCD a ciegas.
8. **`_integrate_forces` "no corre"** → el body está durmiendo (no se llama al dormir, verificado). → fuerza ocasional, colisión, o `can_sleep = false` (con el coste de performance marcado por la docs).
9. **`get_colliding_bodies()` devuelve vacío** → falta `max_contacts_reported > 0` (y `contact_monitor` para las señales).
10. **Cilindros inestables con GodotPhysics** → bugs conocidos de cilindros (docs); Jolt los hace más fiables, o usar box/capsule para personajes (recomendación oficial).
11. **FPS a 1-2 tras un evento (spiral of death)** → el motor no da abasto con el nº de pasos; reducir trabajo físico simultáneo, subir *Max Physics Steps per Frame*, bajar ticks (oficial).
12. **Bumps al cruzar tiles** → un colisionador compuesto por isla, no uno por tile (bug conocido documentado).
13. **Mundo lejos del origen (10k+ unidades)** → errores de precisión de punto flotante (física y cámara "nerviosos"); la docs lo documenta (*large world coordinates*): re-origenar la escena.
14. **Asignar `disabled` del shape directamente** → la docs pide `set_deferred("disabled", true)` para el cambio limpio.
15. **Esperar que un StaticBody3D movido empuje** → se teleporta (no afecta a otros en su camino); usar `AnimatableBody3D`.
16. **Usar `motion_mode` en RigidBody3D (tutoriales viejos)** → no existe en la tabla oficial 4.7 (CCD = `continuous_cd`). Igual: `force_recompute_shapes` (Static) y interpolación per-body no aparecen en las tablas 4.7.

## Anti-patrones

- **RigidBody para todo** "porque es físico" → cada body activo cuesta (pairs, integración); un piso es Static, un trigger es Area, un personaje es Character. Medir `PHYSICS_3D_ACTIVE_OBJECTS`.
- **`can_sleep = false` "por si acaso" en todos los rígidos** → la docs marca el costo; dejar dormir a lo que no necesita acción continua.
- **100 raycasts/shapes por frame "son baratos"** → ver `godot-raycast3d` (shapecast más caro que raycast, documentado); medir `PHYSICS_3D_COLLISION_PAIRS` antes de añadir más.
- **Subir a 240 ticks "para que se vea más suave"** → el suavizado visual es del renderer/interpolación, no del tick rate; el tick rate es para estabilidad de simulación (pilas, velocidad). Medir CPU.
- **Trimesh concave (trimesh del mesh) como colisión de nivel** → la docs (troubleshooting) lo señala como causa de frame drops: usar colisión simplificada (boxes primitivas).
- **Convex auto-generado con "decenas o cientos" de sub-shapes** → la docs lo señala como causa de drops al tocar objetos: primitivas (box/sphere/capsule) en su lugar.
- **Diseñar con la física determinista** (reproducciones exactas, netcode sin sync explícito) → warning oficial de no-determinismo.
- **Depurar capas por adivinación** → imprimir `collision_layer`/`collision_mask` (o los names) y cruzar la tabla; es el error nº1 de "no chocan".

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Monitores oficiales**: `Performance.get_monitor(Performance.PHYSICS_3D_ACTIVE_OBJECTS)` (índice 20) y `Performance.PHYSICS_3D_COLLISION_PAIRS` (índice 21). El pair count es el predictor: coste ≈ pares de shapes que se pueden tocar, no "nº de objetos".
- **Coste medido en práctica típica** (lo que los monitores capturan):
  - Shapes complejos (trimesh/convex auto con muchos sub-shapes) → frame drops **al tocar** (docs).
  - Muchos bodies simultáneos activos (explosión, pile-up) → *spiral of death* (docs).
  - Ticks altos → CPU lineal (docs: "increases CPU utilization, may not be viable for mobile/web").
- **Optimizaciones oficiales (usar solo contra un cuello medido)**:
  1. Shapes primitivas en vez de trimesh/convex auto.
  2. Colisionador compuesto (menos pairs; pero shape más complejo — puede no ser net gain, medir).
  3. Sleep: dejar que duerma (no `can_sleep = false` sin motivo).
  4. Jolt en 3D (docs: mejora rendimiento y estabilidad).
  5. Ticks: subir solo para estabilidad; bajar para aliviar *spiral of death*.
- **No optimizar por intuición**: "poco peso", "pocos objetos" NO es medida; los monitores + el FPS con/without el sistema lo son.

## Debugging

### "No chocan dos objetos"

```text
1. ¿Ambos tienen al menos un CollisionShape3D con shape asignado?
2. Imprimir collision_layer de A y collision_mask de B (y viceversa):
   se intersectan los bits?  (el layer de A debe estar en la mask de B)
3. ¿Uno es Area? (Area NO es sólido: detecta, no choca)
4. ¿El shape está disabled? (o el body con disable_mode = REMOVE)
5. Ver shapes: View > Debug > Visible Collision Shapes (editor; verificado en
   la página de CollisionShape3D) o debug_color propio por shape.
```

### "Atraviesa paredes / pisos (tunneling)"

```text
1. continuous_cd = true en el objeto rápido.
2. Pared/piso: shape más grueso que lo visible (oficial).
3. ¿Muy rápido? El shape debe extenderse proporcionalmente a la velocidad (oficial).
4. physics_ticks_per_second = 120/180/240 (múltiplos; coste CPU — medir).
```

### "La pila vibra / los objetos delgados tiemblan"

```text
1. Shapes demasiado delgados (piso o body) → engrosar (oficial).
2. Subir ticks (oficial) o Jolt (oficial para pilas y cilindros).
3. Si es un cilindro: Jolt (bugs conocidos de cilindros en GodotPhysics).
```

### "El FPS se va al piso de golpe"

```text
1. ¿Coincide con un evento físico? (explosión, muchos bodies a la vez)
2. PHYSICS_3D_ACTIVE_OBJECTS / COLLISION_PAIRS antes y durante (regla MEASURE).
3. spiral of death (1-2 FPS sostenidos): subir Max Physics Steps per Frame y/o
   bajar ticks (oficial); pero primero reducir el trabajo físico concurrente.
4. Si es al TOCAR objetos: shape demasiado complejo (trimesh/convex auto).
```

### "La física se rompe lejos del origen"

```text
1. ¿La cámara/posición del mundo está en miles/millares de unidades?
2. Errores de precisión de float (docs): re-origenar la escena
   (ver large world coordinates en la docs).
```

### "El body 'congelado' no me responde a queries / no empuja"

```text
1. freeze + FREEZE_MODE_STATIC = no colisiona a su paso al moverse (oficial).
2. Para mover por animación Y empujar: FREEZE_MODE_KINEMATIC (Rigid) o
   AnimatableBody3D (Static).
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tablas de clase completas + tutoriales (lista en *Referencias*).
- **Trampas 3.x → 4.x (verificadas en las tablas 4.7)**:
  - `RigidBody3D.motion_mode` (enum DISCRETE/CONTINUOUS…): **no existe en la tabla 4.7**; el CCD es `continuous_cd: bool`.
  - `StaticBody3D.force_recompute_shapes`: **no existe en la tabla 4.7** (tutoriales viejos la citan; no usar).
  - Interpolación de física per-body: **no hay propiedad en `PhysicsBody3D` 4.7** (solo los 6 `axis_lock_*`). La interpolación global (introducida en 4.3) se configura por Project Settings: la clave exacta **no verificada en esta revisión** — revisar la sección *Physics → Common* de Project Settings 4.7.
  - `PhysicsMaterial3D` como clase: la clase de la docs 4.7 es **`PhysicsMaterial`** (una sola, para 2D y 3D; la página `class_physicsmaterial3d.html` da 404 en 4.7). Sin enums `friction_combine_mode`/`bounce_combine_mode`: la combinación se rige por `rough`/`absorbent` (ver `godot-physics-materials`).
  - `CharacterBody3D.motion_mode`: solo `GROUNDED`/`FLOATING` en 4.7 (ver `godot-characterbody3d`).
  - `move_and_slide()` sin argumentos en 4.x (las propiedades de slide del 3.x se movieron a propiedades del body).
- **4.0 ↔ 4.7**: la API de cuerpos de esta skill es estable en las tablas consultadas; los cambios documentados arriba son la excepción.

## Dependencias

| Skill | Relación |
|---|---|
| (ninguna base) | Skill fundacional de la familia física |
| `godot-characterbody3d` | Especializa el tipo Character (Lote 3, verificado) |
| `godot-raycast3d` | Usa layers/masks y `_physics_process` de esta skill |
| `godot-area3d` | El 4º tipo de objeto, cubierto con detalle aparte |
| `godot-physics-materials` | `physics_material_override` de Static/Rigid |
| `godot-rendering-performance` | Diagnóstico conjunto (física vs render en el FPS) |
| `godot-third-person-character` | Compuesta que la consume (Lote 0) |
| `godot-vehiclebody3d` (pendiente) | El RigidBody especializado para ruedas |

## Skills relacionadas

- `godot-recipe-interactive-rigidbody` — receta (pendiente; germen en *Implementación recomendada*).
- `godot-error-objects-passthrough` — tunneling (pendiente; germen en *Debugging*).
- `godot-error-physics-spiral-of-death` — pendiente (germen en *Performance*).
- `godot-error-objects-dont-collide` — pendientes: layers (germen en *Debugging* #1).

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Physics introduction (tipos de cuerpos, layers/masks, `_physics_process`, 60 Hz, no-determinismo, shapes, sleep, contact reporting, `_integrate_forces`): https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html
- Troubleshooting physics issues (tunneling/CCD, pilas, escala, delgados, cilindros/Jolt, vehículos, tiles/compuestos, frame drops, spiral of death, precisión de origen): https://docs.godotengine.org/en/stable/tutorials/physics/troubleshooting_physics_issues.html
- RigidBody3D (props/métodos/señales/enums, `_integrate_forces`, damping, CCD, freeze): https://docs.godotengine.org/en/stable/classes/class_rigidbody3d.html
- StaticBody3D (`constant_linear/angular_velocity`, teleportación, `AnimatableBody3D`): https://docs.godotengine.org/en/stable/classes/class_staticbody3d.html
- PhysicsBody3D (axis locks, `move_and_collide`/`test_move`, excepciones, `get_gravity`): https://docs.godotengine.org/en/stable/classes/class_physicsbody3d.html
- CollisionShape3D (`shape`, `disabled` deferred, debug, warning de escala): https://docs.godotengine.org/en/stable/classes/class_collisionshape3d.html
- CollisionObject3D (layers/masks/priority/disable_mode/input — Lote 0): https://docs.godotengine.org/en/stable/classes/class_collisionobject3d.html
- CharacterBody3D (Lote 3): https://docs.godotengine.org/en/stable/classes/class_characterbody3d.html
- Performance (monitores 20/21 — Lote 0): https://docs.godotengine.org/en/stable/classes/class_performance.html
