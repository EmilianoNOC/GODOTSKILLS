# godot-raycast3d

## ID

`godot-raycast3d`

## Categoría

Física · Base (queries)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: páginas de clase (`RayCast3D`, `ShapeCast3D`, `PhysicsDirectSpaceState3D`, `PhysicsRayQueryParameters3D`) y el tutorial `ray-casting.html` (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — propiedades/métodos de `RayCast3D` y `ShapeCast3D`, `PhysicsDirectSpaceState3D.intersect_ray/intersect_shape/intersect_point/collide_shape/cast_motion` (incl. claves de los Dictionary de resultado), `PhysicsRayQueryParameters3D` (`create()`, `exclude`, `collision_mask`), regla de *space locked* fuera de `_physics_process`, excepciones, picking con `project_ray_origin/normal`: verificados directamente en la docs 4.7.
**MEDIUM** — recomendaciones de "cuántos raycasts son muchos" (la docs documenta que shapecast es más caro que raycast; el resto es decisión de proyecto).
**"No existe en la tabla oficial 4.7 (verificado)"**: `RayCast3D.cast_to` (en 4.x es `target_position`; `cast_to` era 3.x), `PhysicsDirectSpaceState3D.test_motion` (la consulta 3D es `cast_motion`), clave `metadata` en el dict de `intersect_ray` (la tabla 4.7 lista `collider, collider_id, normal, position, face_index, rid, shape`).

## Nivel

**BASE** (depende de `godot-physics` para layers/masks y el ciclo de física).

## Propósito

Proporcionar el conocimiento operativo completo de las queries de línea/volumen en física 3D: el nodo `RayCast3D` (resultado por tick), `ShapeCast3D` (barrido de un shape, varios resultados, overlap instantáneo), y las consultas directas vía `PhysicsDirectSpaceState3D` (`intersect_ray` y compañía) con su regla de thread (solo en `_physics_process`), excepciones y máscaras. Cubre los patrones: detectar piso, picking con mouse, línea de visión de IA, "¿hay obstáculo?".

## Cuándo utilizarla

- Detectar el piso/pared frente al personaje (RayCast3D hija con `exclude_parent`).
- Picking 3D con mouse (ray de cámara a world).
- Línea de visión / alcance de habilidad / "¿el jugador me ve?".
- Barrido de volumen (laser ancho, snap de un shape al suelo) → `ShapeCast3D`.
- Queries puntuales desde código sin crear nodos (`intersect_ray` directo).

## Cuándo NO utilizarla

- **Detección de overlap continuo (objeto adentro de zona)** → `Area3D` (`godot-area3d`); el raycast es instantáneo, no "está dentro".
- **Choque/respuesta física** (que algo rebote contra una pared) → bodies (`godot-physics`); un raycast no empuja nada.
- **Movimiento de personaje** → `move_and_slide` (`godot-characterbody3d`) resuelve el slide completo; raycast + lógica propia es más código para el mismo problema.
- **2D** → `RayCast2D`/`PhysicsDirectSpaceState2D` (skill pendiente; la API espejo).

## Conceptos fundamentales

### RayCast3D: el ray por tick

- Un `RayCast3D` (hereda `Node3D`, **no** `CollisionObject3D`) es un rayo **desde su origen (posición del nodo) hasta `target_position`** (local, default `(0, -1, 0)`).
- **Calcula la intersección cada frame de física y conserva el resultado hasta el siguiente** (verificado). Leerlo en `_process` da el último resultado de física (normal); para resultado inmediato o varios cambios en el mismo tick físico: `force_raycast_update()`.
- Encuentra el **primer** objeto a lo largo del camino; los filtra con capas (`collision_mask`), tipos (`collide_with_bodies`/`collide_with_areas`) y excepciones.
- Para barrer una **región** (no una línea): múltiples `RayCast3D` aproximando, o `ShapeCast3D` (la docs da las dos opciones).
- El debug visual está integrado: `debug_shape_custom_color` + `debug_shape_thickness` (se ve en el editor y en runtime con los debug shapes del viewport).

### ShapeCast3D: el barrido (sweep)

- Mismo modelo que `RayCast3D` pero **barrida un `Shape3D`** a lo largo de `target_position`: detecta **múltiples** objetos (`get_collision_count()`, `get_collider(index)`), útil para "laser ancho" o "snap de un shape al piso" (casos de uso oficiales).
- **Más caro que raycast** (aviso oficial: "Shape casting is more computationally expensive than ray casting").
- Truco oficial de **overlap instantáneo**: `target_position = Vector3.ZERO` + `force_shapecast_update()` dentro del mismo tick físico — resuelve la limitación de `Area3D` de que su información de colisión **no es inmediata** (útil cuando necesitas "¿toco algo YA?", sin esperar al próximo tick).
- `max_results` (default 32) limita cuántos colliders reporta; `margin` (default 0.0) es margen de la sweep.

### Queries directas: `PhysicsDirectSpaceState3D`

- Accesos: `get_world_3d().direct_space_state` (verificado, recomendado) o `PhysicsServer3D.space_get_direct_state(get_world_3d().space)`.
- **Regla de thread (verificada)**: la física puede correr en otro thread; acceder al space **solo es seguro durante `_physics_process()`** — desde `_input()`/`_process()` el space puede estar *locked* y dar error. Patrón oficial para picking: recibir el click en `_input` y **disparar la query en el siguiente `_physics_process`**.
- `intersect_ray(PhysicsRayQueryParameters3D)` → `Dictionary`; **vacío si no tocó nada** (el chequeo más común: `if result:`).
- Claves del dict (tabla 4.7): `collider` (Object), `collider_id` (ObjectID), `normal` (world), `position` (punto, world), `face_index` (solo válido con `ConcavePolygonShape3D`, si no `-1`), `rid`, `shape` (índice de shape del collider).
- `PhysicsRayQueryParameters3D.create(from, to, collision_mask=4294967295, exclude=[])` — **coords GLOBALES** (no locales al nodo; comentario explícito del tutorial). Default de mask = todas las capas.
- **Excepciones**: `exclude` acepta **objetos o RIDs**. La docs da la regla de elección: para excluir el propio body, excepciones; **para listas grandes/dinámicas, la collision mask es mucho más eficiente** (oficial).
- El resto de consultas de la clase (verificadas): `intersect_shape(params, max_results=32)` (dicts con `collider, collider_id, rid, shape`; **ignora `motion`**), `intersect_point(PhysicsPointQueryParameters3D, max_results=32)`, `collide_shape(params, max_results=32)` (pares de puntos de contacto: primero el de tu shape, segundo el del espacio; ignora `motion`), `cast_motion(params)` → `PackedFloat32Array [safe, unsafe]` (`[1.0, 1.0]` = sin colisión; **ignora shapes con los que ya estés colisionando** — usar `collide_shape` para verlos), `get_rest_info(params)` (dict `collider_id, linear_velocity, normal, point, rid, shape`; `linear_velocity` es `(0,0,0)` si el collider es Area).

### Picking con cámara (verificado en el tutorial)

- `Camera3D.project_ray_origin(mouse_pos)` → origen del ray; `project_ray_normal(mouse_pos)` → dirección (normalizado).
- `to = from + direction * RAY_LENGTH` (el tutorial usa 1000 con la convención 1 unidad = 1 m).
- Funciona en proyección perspective **y** orthogonal (por eso hay que pedir ambas: en orthogonal cambia el origen, en perspective la normal).
- Alternativa sin query: `CollisionObject3D.input_event` (el tutorial: "there is not much need to do this" — para clicks en objetos con `input_ray_pickable`).

## API relevante

Todo verificado en Godot 4.7.

### `RayCast3D` (hereda `Node3D`)

| Propiedad | Default | Nota |
|---|---|---|
| `target_position` | `(0, -1, 0)` | Fin del ray, **local** al nodo. (Era `cast_to` en 3.x.) |
| `enabled` | `true` | Ray activo. |
| `collide_with_bodies` | `true` | `PhysicsBody3D`/`RigidBody3D`/etc. |
| `collide_with_areas` | `false` | `Area3D` (off por defecto). |
| `collision_mask` | `1` | Bitmask de capas. |
| `exclude_parent` | `true` | Ignora al padre (evita auto-detección; el patrón "ray del personaje" funciona sin más). |
| `hit_back_faces` | `true` | ¿Choca con caras traseras? |
| `hit_from_inside` | `false` | ¿Choca si el origen está DENTRO del shape? |
| `debug_shape_custom_color` / `debug_shape_thickness` | `(0,0,0,1)` / `2` | Vista de debug. |

| Método | Devuelve | Nota |
|---|---|---|
| `is_colliding()` | `bool` | ¿Tocó algo en el último tick? |
| `get_collider()` | `Object` | El objeto (o null). |
| `get_collider_rid()` | `RID` | — |
| `get_collider_shape()` | `int` | Índice de shape tocado. |
| `get_collision_point()` | `Vector3` | Punto (world). |
| `get_collision_normal()` | `Vector3` | Normal (world). |
| `get_collision_face_index()` | `int` | Solo `ConcavePolygonShape3D`, si no `-1`. |
| `force_raycast_update()` | — | Recalcula YA (mismo tick) o tras reconfigurar. |
| `add_exception(node)` / `add_exception_rid(rid)` / `clear_exceptions()` | — | Excepciones. |
| `get/set_collision_mask_value(layer, value)` | — | Bit a bit. |

### `ShapeCast3D` (hereda `Node3D`)

| Propiedad | Default | Nota |
|---|---|---|
| `shape` | — | El `Shape3D` barrido (obligatorio para que haga algo). |
| `target_position` | `(0, -1, 0)` | Dirección/fin de la sweep (local). |
| `collision_mask` | `1` | Bitmask. |
| `collide_with_bodies` / `collide_with_areas` | `true` / `false` | Como RayCast3D. |
| `max_results` | `32` | Techo de colliders reportados. |
| `margin` | `0.0` | Margen de la sweep. |
| `enabled` / `exclude_parent` | `true` / `true` | — |
| `collision_result` | `[]` | Array de resultados (lectura). |

Métodos: `get_collision_count()`, `get_collider(index)`, `get_collider_rid(index)`, `get_collider_shape(index)`, `get_closest_collision_safe_fraction()`, `get_closest_collision_unsafe_fraction()`, `force_shapecast_update()`, `add_exception/add_exception_rid/clear_exceptions`, `get/set_collision_mask_value`.

### `PhysicsDirectSpaceState3D` (no instanciar; `get_world_3d().direct_space_state`)

| Método | Devuelve |
|---|---|
| `intersect_ray(params: PhysicsRayQueryParameters3D)` | `Dictionary` (vacío = sin hit). |
| `intersect_shape(params, max_results=32)` | `Array[Dictionary]` (`collider, collider_id, rid, shape`). |
| `intersect_point(params, max_results=32)` | `Array[Dictionary]` (¿un punto está dentro de qué shapes?). |
| `collide_shape(params, max_results=32)` | `Array[Vector3]` (pares de puntos de contacto). |
| `cast_motion(params)` | `PackedFloat32Array` `[safe, unsafe]`. |
| `get_rest_info(params)` | `Dictionary` (el collider más cercano + velocity/normal/punto). |

### `PhysicsRayQueryParameters3D` (hereda `RefCounted`)

| Propiedad | Default | Nota |
|---|---|---|
| `from` / `to` | `(0,0,0)` | **Coords globales**. |
| `collision_mask` | `4294967295` (todas) | Bitmask. |
| `collide_with_bodies` / `collide_with_areas` | `true` / `false` | — |
| `exclude` | `[]` | `Array[RID]` (o objetos; verificado). |
| `hit_back_faces` / `hit_from_inside` | `true` / `false` | — |
| `create(from, to, collision_mask=…, exclude=[])` | static | Factory. |

## Arquitectura recomendada

### Elegir el mecanismo

```text
¿El rayo es fijo en la escena (mismo origen/dirección cada tick)?
├─ SÍ → RayCast3D como nodo hijo (se ve en el editor, debug gratis,
│       exclude_parent resuelve el auto-hit)
└─ NO (orígenes cambiantes: mouse, IA, varias direcciones)
   ├─ 1 ray por consulta → intersect_ray() directo en _physics_process
   └─ volumen (no línea) → ShapeCast3D (nodo) o cast_motion/intersect_shape
```

### Reglas de diseño

1. **Nodo hijo + `exclude_parent`** para "mis" raycasts (piso, pared): el padre no se auto-detecta (default `true`, verificado).
2. **Masks, no listas de excepciones** cuando la lista crece (recomendación oficial): una capa `no_hit_by_rays` y quitar esa capa de la mask del ray.
3. **Guardar el click, query en `_physics_process`** (regla de space locked; verificado).
4. **Longitud de ray explícita** (constante `RAY_LENGTH`): un ray infinito toca la luna.
5. **`collide_with_areas` solo si hace falta**: off por defecto; activarlo cambia qué objetos son alcanzables.

## Implementación mínima

**Detectar el piso del personaje** (RayCast3D hija, patrón estándar):

```text
Player (CharacterBody3D)
└─ FloorRay (RayCast3D)      # target_position = (0, -1.0, 0) local
                             # exclude_parent = true (default)
                             # collision_mask = capa world
```

```gdscript
# floor_check.gd — en el CharacterBody3D (o en el RayCast3D)
@onready var floor_ray: RayCast3D = $FloorRay

func _physics_process(_delta: float) -> void:
	if floor_ray.is_colliding():
		var ground: Object = floor_ray.get_collider()
		var normal: Vector3 = floor_ray.get_collision_normal()
		var point: Vector3 = floor_ray.get_collision_point()
		# ... (is_on_floor_custom, ángulo de pendiente, etc.)
```

## Implementación recomendada

### 1. Picking con mouse (click → objeto del mundo)

```gdscript
extends Node3D

const RAY_LENGTH := 1000.0

var _pending_pick: Vector2 = Vector2.ZERO
var _has_pending_pick := false

func _input(event: InputEvent) -> void:
	# _input: el space puede estar locked (verificado) → solo registrar.
	if event is InputEventMouseButton and event.pressed \
			and event.button_index == MOUSE_BUTTON_LEFT:
		_pending_pick = event.position
		_has_pending_pick = true

func _physics_process(_delta: float) -> void:
	if not _has_pending_pick:
		return
	_has_pending_pick = false

	var camera: Camera3D = get_viewport().get_camera_3d()
	var from: Vector3 = camera.project_ray_origin(_pending_pick)
	var to: Vector3 = from + camera.project_ray_normal(_pending_pick) * RAY_LENGTH

	var query := PhysicsRayQueryParameters3D.create(from, to)
	query.collision_mask = 0b00000000_00000000_00000000_00000011  # world + player
	query.exclude = [get_rid()]  # objetos o RIDs (verificado)

	var result: Dictionary = get_world_3d().direct_space_state.intersect_ray(query)
	if result:
		print("Tocado: ", result.collider, " en ", result.position)
	# if result == {} → no tocó nada (dict vacío, verificado)
```

> Alternativa sin código de ray: `CollisionObject3D.input_event` (señal oficial, ver `godot-physics`); el tutorial la menciona como la vía de "menos necesidad".

### 2. Línea de visión de IA (con excepción del propio body)

```gdscript
extends CharacterBody3D

@export var los_range := 20.0

func can_see_target(target: Node3D) -> bool:
	var query := PhysicsRayQueryParameters3D.create(
		global_position, target.global_position, collision_mask, [get_rid()]
	)
	var result: Dictionary = get_world_3d().direct_space_state.intersect_ray(query)
	if not result:
		return true
	return result.collider == target  # lo primero que toca es el target
```

(Ejemplo oficial: `PhysicsRayQueryParameters3D.create(global_position, target, collision_mask, [self])` — la docs usa `[self]` en 2D; en 3D el patrón equivalente con `get_rid()` o el nodo funciona igual, ya que `exclude` acepta objetos o RIDs.)

### 3. Snap de un shape al suelo (ShapeCast3D)

```text
Crate (RigidBody3D)
└─ SnapCast (ShapeCast3D)    # shape = el mismo BoxShape3D del crate
                             # target_position = (0, -1, 0)
```

```gdscript
# En el RigidBody3D, para "aterrizar" suavemente sobre la primera superficie:
@onready var snap: ShapeCast3D = $SnapCast

func _physics_process(_delta: float) -> void:
	snap.target_position = velocity * _delta  # barrer hacia donde voy
	snap.force_shapecast_update()
	if snap.get_collision_count() > 0:
		var collider: Object = snap.get_collider(0)
		var safe: float = snap.get_closest_collision_safe_fraction()
		# global_position += velocity * _delta * safe  (mover solo hasta el hit)
```

### 4. Overlap instantáneo (la limitación de Area3D, resuelta oficialmente)

```gdscript
# Cuando un Area3D no basta porque su estado no es inmediato en el tick:
# ShapeCast3D con target_position = Vector3.ZERO + force_shapecast_update()
# en el MISMO tick físico (patrón documentado en la página de ShapeCast3D).
@onready var overlap: ShapeCast3D = $OverlapProbe

func check_overlap_now() -> void:
	overlap.target_position = Vector3.ZERO
	overlap.force_shapecast_update()
	print("tocando: ", overlap.get_collision_count())
```

### 5. ¿Estoy dentro de un shape? (`intersect_point`)

```gdscript
# "¿El jugador está dentro de esta forma?" (p. ej. ¿la caja está dentro
# del área de la bomba?):
var point_params := PhysicsPointQueryParameters3D.new()
point_params.position = player.global_position
point_params.collision_mask = 0b10  # capa de "zonas"
var inside: Array[Dictionary] = get_world_3d().direct_space_state.intersect_point(point_params)
if inside.size() > 0:
	print("dentro de: ", inside[0].collider)
```

## Ejemplo práctico

**Juego**: shooter third-person con 12 enemigos y mundo de 200 m.

1. **Piso/pendiente del jugador**: `RayCast3D` hija (`target_position=(0,-1.2,0)`) + `exclude_parent`; el control de movimiento (`godot-characterbody3d`) consulta `is_colliding()`/`get_collision_normal()` en `_physics_process`.
2. **Puntero**: el `mouse_event` se guarda en `_input`; la query de cámara corre en `_physics_process` (space locked — verificado); la mask del ray es world+enemies (los propios projectiles en su capa, fuera de la mask).
3. **Línea de visión IA**: 1 `intersect_ray` por enemigo **solo cuando cambia de estado** (no por frame por enemigo: 12 raycasts/frame son baratos, 12× por frame por enemigo no lo son — medir con el FPS y no por intuición).
4. **Laser de habilidad ancho**: `ShapeCast3D` con `CapsuleShape3D` (no 8 RayCast3D: el shapecast lo resuelve en una query; la docs documenta el coste mayor que un ray, pero 1 sweep < 8 rays por código y por consistencia).
5. **Debug**: `debug_shape_custom_color` por ray para ver exactamente qué toca en runtime (con *Debug > Visible Collision Shapes* / debug del viewport).

## Integración

- **`godot-physics`**: layers/masks, ciclo de ticks, `disable_mode` (un body deshabilitado con `REMOVE` no es alcanzable por raycasts).
- **`godot-characterbody3d`**: los raycasts de piso/escalera/muro del personaje viven aquí.
- **`godot-area3d`**: lo que el ray NO es (overlap continuo); el overlap instantáneo de `ShapeCast3D` complementa la limitación de Area (documentado).
- **`godot-rendering-performance`**: picking NO va en `_process` por física; no confundir con el coste de render del ray visual (línea dibujada = otro tema).
- **`godot-physics-materials`**: el ray NO devuelve el material (no es parte del dict verificado); si necesitas fricción del punto tocado, es decisión de tu código (leer el collider y su `physics_material_override`).

## Errores frecuentes

1. **`cast_to` (3.x) en código 4.x** → no existe en 4.7 (verificado: la propiedad es `target_position`). Migrar.
2. **Hacer `intersect_ray` desde `_input()`/`_process()`** → el space puede estar *locked* (error; verificado). → guardar y disparar en `_physics_process`.
3. **`target_position` local mal entendido** → es **local al nodo** `RayCast3D` (default `(0,-1,0)` = "abajo 1 m"). Para coords del mundo, ponerlo en el nodo o usar query directa con coords globales (el tutorial remarca "use global coordinates").
4. **El ray se choca con su propio personaje** → `exclude_parent` (nodo, default `true`) o `exclude = [get_rid()]` (query directa). Si la lista crece: **mask** (recomendación oficial).
5. **`collide_with_areas` esperando detectar un Area** → off por defecto (verificado: `false` en nodo y en query params).
6. **Leer `get_collider()` tras mover el ray en el mismo `_process`** → el resultado es del último tick de física; si reconfiguraste, `force_raycast_update()` (o hacerlo todo en `_physics_process`).
7. **Esperar `metadata` en el dict de `intersect_ray` 4.7** → la tabla 4.7 no lista esa clave (la lista 4.7: `collider, collider_id, normal, position, face_index, rid, shape`). No depender de ella.
8. **`face_index` en shapes que no son `ConcavePolygonShape3D`** → devuelve `-1` (verificado); no es un "triángulo 0".
9. **Ray infinito** (`to = from + dir * 1e9`) → toca cualquier cosa lejana; longitud explícita.
10. **`test_motion` en 3D** → no existe en la tabla 4.7 (verificado); la consulta de movimiento es `cast_motion` (devuelve `[safe, unsafe]`).
11. **Muchos RayCast3D "baratos"** → el shapecast es documentadamente más caro que el ray; un abanico de 16 rays por frame por actor empieza a contar — medir (ver *Performance*).
12. **Usar raycast como overlap** ("¿la moneda está dentro?") → el ray es una línea instantánea; para volumen continuo es `Area3D` (`godot-area3d`).

## Anti-patrones

- **Un RayCast3D por cada dirección de cada personaje, siempre encendidos** → apagar (`enabled = false`) cuando no se usan; cada ray activo es trabajo de física por tick.
- **`get_overlapping_bodies()`-style polling por raycasts** (re-disparar `intersect_ray` 60×/s para 50 cosas que no cambian) → cachear el resultado y re-query al moverse/cambiar estado.
- **Picking en `_input` "porque es inmediato"** → el espacio físico está desactualizado/locked (verificado); la query corre en `_physics_process` y el resultado se usa en el siguiente frame (imperceptible).
- **Abanico de 20 raycasts para "una forma"** → `ShapeCast3D` (1 sweep) o primitivas; la docs: para barrer una región, múltiples rays SOLO como aproximación.
- **Excepciones en vez de capas para "que no me toque nada mío"** → la lista de excepciones crece; la mask es el mecanismo correcto para listas grandes/dinámicas (oficial).
- **Raycasts en `_process` "porque el mouse es de frame"** → mixing de ticks: el resultado físico tiene latencia de un tick; si el diseño lo exige, así se documenta (no se esconde).
- **Confundir `hit_from_inside` con bug** → un ray cuyo origen está dentro de un shape NO choca con él a menos que `hit_from_inside = true` (default `false`, verificado).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Coste documentado**: shapecast > raycast (aviso oficial en `ShapeCast3D`). Raycasts son de las queries más baratas, pero no gratis: cada ray activo por tick de física escanea los shapes candidatos según su mask.
- **Qué medir**: FPS del juego con el sistema de raycasts ON vs OFF (mismo escenario), y `Performance.PHYSICS_3D_COLLISION_PAIRS` (índice 21, ver `godot-physics`) si la escena tiene muchos colliders. El raycast NO aparece como "objeto activo" — su coste está en el escaneo, que se refleja en el tiempo de tick.
- **Optimizaciones (solo contra un cuello medido)**:
  1. `enabled = false` cuando no aplica (el más fácil y el que más da).
  2. Menos raycasts por frame (cacheo, re-query por evento, no por tick).
  3. Masks restrictivas (menos candidates a escanear).
  4. `ShapeCast3D.max_results` bajo si solo necesitas el primero.
  5. Reemplazar abanicos por 1 sweep (ShapeCast3D) o por `Area3D` si el requisito es "estar dentro" y no "línea".
- **No**: "12 raycasts son pocos, no mido nada" (la regla es medir, no estimar por nº de objetos).

## Debugging

### "El ray no toca lo que debería"

```text
1. ¿enabled?  (y ¿el nodo está activo/visible en la escena?)
2. ¿target_position apunta hacia el objeto? (local al nodo: imprimir
   global_position + global_transform * target_position)
3. ¿collision_mask intersecta con el collision_layer del objetivo?
4. ¿El objetivo es Area? (collide_with_areas = true?)
5. ¿El origen está DENTRO del objetivo? (hit_from_inside = false lo ignora)
6. ¿exclude_parent/exclude lo está excluyendo?
7. Visual: debug_shape_custom_color + Visible Collision Shapes (viewport)
   → ver el ray EXACTO que se lanza.
```

### "El ray se toca a sí mismo / a su padre"

```text
1. RayCast3D: exclude_parent (default true; ¿lo desactivaste?)
2. Query directa: exclude = [get_rid()]
3. Si la lista crece → capa "no-ray" y quitarla de la mask (oficial).
```

### "A veces la query falla con error / el espacio está locked"

```text
1. ¿La query corre en _physics_process? (requisito oficial, verificado)
2. _input solo registra; _physics_process dispara.
3. Si la física corre multithread, fuera de _physics_process NO es seguro
   (documentado en ray-casting.html).
```

### "El resultado llega un frame tarde"

```text
1. Normal: el resultado del nodo es del último tick de física.
2. ¿Necesitas inmediato? force_raycast_update() / force_shapecast_update()
   dentro del tick.
3. ¿Necesitas el estado de Area YA? → ShapeCast3D overlap instantáneo
   (patrón oficial) en vez de esperar al signal del Area.
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tablas de clase + tutorial (lista en *Referencias*).
- **3.x → 4.x (trampas, verificado contra tablas 4.7)**:
  - `cast_to` → `target_position` (3.x usaba `cast_to`).
  - `test_motion` no existe en `PhysicsDirectSpaceState3D` 4.7 → `cast_motion` (devuelve `[safe, unsafe]`).
  - Dict de `intersect_ray` 4.7: `collider, collider_id, normal, position, face_index, rid, shape` (sin `metadata` en la tabla 4.7; los tutoriales de rama master la mencionan → tratar como **API no verificada** para 4.7).
  - `exclude` acepta objetos **o** RIDs (documentado) — en 3.x era solo RIDs.
- **4.0 ↔ 4.7**: tablas estables para `RayCast3D`/`ShapeCast3D` en lo consultado.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-physics` | Base: layers/masks, ciclo de física, debug shapes |
| `godot-characterbody3d` | Consumidor principal (ray de piso, escalera, muro) |
| `godot-area3d` | Complemento: overlap continuo (lo que el ray no es) |
| `godot-rendering-performance` | Diagnóstico conjunto del FPS |
| `godot-raycast2d` (pendiente) | Espejo 2D |

## Skills relacionadas

- `godot-recipe-mouse-picking-3d` — receta (pendiente; germen en *Implementación recomendada* #1).
- `godot-recipe-ai-line-of-sight` — pendiente (germen #2).
- `godot-error-raycast-hits-self` — pendiente (germen en *Debugging*).
- `godot-error-physics-space-locked` — pendiente (germen #3).

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Ray-casting tutorial (space, thread rule, `direct_space_state`, `intersect_ray`, excepciones, mask vs exceptions, picking con cámara, `_input`→`_physics_process`): https://docs.godotengine.org/en/stable/tutorials/physics/ray-casting.html
- RayCast3D (props/métodos, tick por tick, `force_raycast_update`): https://docs.godotengine.org/en/stable/classes/class_raycast3d.html
- ShapeCast3D (sweep, multi-result, overlap instantáneo, coste mayor que raycast): https://docs.godotengine.org/en/stable/classes/class_shapecast3d.html
- PhysicsDirectSpaceState3D (`intersect_ray/intersect_shape/intersect_point/collide_shape/cast_motion/get_rest_info`, claves de los dicts): https://docs.godotengine.org/en/stable/classes/class_physicsdirectspacestate3d.html
- PhysicsRayQueryParameters3D (`create()`, `exclude`, `collision_mask`, defaults): https://docs.godotengine.org/en/stable/classes/class_physicsrayqueryparameters3d.html
- Physics introduction (layers/masks, 60 Hz): https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html
