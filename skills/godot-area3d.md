# godot-area3d

## ID

`godot-area3d`

## Categoría

Física · Base (detección)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Area3D` (tabla completa de propiedades/métodos/señales/enums) y tutorial `physics_introduction.html` (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — propiedades (`monitoring/monitorable`, overrides de gravedad/damping/viento/audio, `priority`, enum `SpaceOverride`), señales (`area_*`, `body_*`, `*_shape_*`), métodos (`get_overlapping_bodies/areas`, `overlaps_body/area`), los 3 usos oficiales, warning de `ConcavePolygonShape3D` hueco, warning de objects creados por `PhysicsServer3D`: verificados directamente en la docs 4.7.
**MEDIUM** — patrones de diseño de pickups/contadores (no son reglas oficiales; son la aplicación estándar de la API verificada).
**"No existe en la tabla oficial 4.7 (verificado)"**: señales `object_entered`/`object_exited` (la tabla 4.7 lista solo `area_*` y `body_*` y sus variantes `_shape_*`); reportes de overlap contra `SoftBody3D` (la docs dice explícitamente que el motor no los soporta y no emite la señal).

## Nivel

**BASE** (depende de `godot-physics` para layers/masks y el ciclo de física).

## Propósito

Proporcionar el conocimiento operativo de `Area3D`: el objeto de **detección e influencia** de la física 3D. Cubre las señales de entrada/salida (body/area/shape), el tracking de overlap (`get_overlapping_bodies`), los overrides locales de física (gravedad, damping, viento) y audio (bus/reverb), y los patrones típicos: pickups, triggers, zonas de daño, checkpoints, "¿el jugador está dentro?".

## Cuándo utilizarla

- Detectar "X entró / salió / está adentro" (pickups, portales, zonas de daño, checkpoints).
- Contar objetos dentro de una zona (bots, enemigos en sector).
- **Influir** en la física local: gravedad distinta, damping, viento (la docs: areas "can also locally alter or override physics parameters").
- Enrutar audio por zona (bus de audio/reverb por región).

## Cuándo NO utilizarla

- **Algo debe ser SÓLIDO** (que se choque contra) → `StaticBody3D`/`RigidBody3D` (`godot-physics`); un Area no detiene a nadie.
- **Overlap instantáneo en el mismo tick** (necesitas el estado YA, no en el próximo tick) → `ShapeCast3D` con `target_position = (0,0,0)` + `force_shapecast_update()` (patrón oficial; ver `godot-raycast3d`). La docs de `ShapeCast3D` lo cita explícitamente como la solución a esta limitación de Area3D.
- **Línea de visión / ray** → `godot-raycast3d`.
- **2D** → `Area2D` (skill pendiente; espejo casi exacto).

## Conceptos fundamentales

### DetECCIÓN + INFLUENCIA (los dos papeles)

`Area3D` (hereda `CollisionObject3D` **directamente**, no `PhysicsBody3D`) es una región del espacio definida por uno o varios `CollisionShape3D`/`CollisionPolygon3D` hijos. La docs la define con dos funciones:

1. **Detección**: saber cuándo otros `CollisionObject3D` entran/salen, y llevar la cuenta de los que **aún no salieron** (overlap). Emite señales y expone `get_overlapping_*`.
2. **Influencia**: modificar parámetros físicos dentro de la región: gravedad (valor/dirección/puntual), damping lineal/angular, viento, y audio (bus/reverb).

Los tres usos principales (lista oficial):

- Override de parámetros físicos (gravedad, etc.) en una región.
- Detectar entradas/salidas o qué bodies están dentro.
- Comprobar overlap entre areas (`area_entered`/`get_overlapping_areas`).

**Detalle de input (verificado)**: las areas también reciben input de mouse/touch por defecto (igual que los bodies, vía `input_ray_pickable` de `CollisionObject3D`).

### `monitoring` vs `monitorable` (los dos bools que confunden)

| Propiedad | Default | Significado |
|---|---|---|
| `monitoring` | `true` | **Este area emite señales** (detecta a otros). Las señales `body_entered` etc. **requieren `monitoring = true`** (notado en cada señal). |
| `monitorable` | `true` | **Este area puede ser detectado por OTROS** (areas y bodies que lo monitoreen). |

Casos típicos:

- **Trigger puro** (solo emite): `monitorable = false` (nada necesita detectarlo a él) → menos trabajo.
- **Zona detectable** (otros quieren saber "¿estoy en un area?"): `monitoring = false` + `monitorable = true` (no paga señales propias).
- **Área de física** (override de gravedad): no necesita señales ni ser detectada → `monitoring = false`, `monitorable = false`.

### El overlap es por tick, no continuo

Un `Area3D` detecta en el paso de física: su información de colisión **no es inmediata** dentro del tick (por eso existe el patrón de `ShapeCast3D` overlap-instantáneo, verificado). Para "¿está dentro AHORA?" tras un movimiento en el mismo tick: `get_overlapping_bodies()` tras `move_and_slide()` del body (el body ya resolvió su movimiento) o el patrón de shapecast.

### Influence: gravedad, damping, viento, audio

- **Gravedad**: `gravity` (default `9.8`), `gravity_direction` (default `(0,-1,0)`), `gravity_point` (gravedad puntual: atrae hacia `gravity_point_center`, con `gravity_point_unit_distance` para la falloff). Cada override se rige por `gravity_space_override`.
- **Damping**: `linear_damp`/`angular_damp` (default `0.1`) + `linear_damp_space_override`/`angular_damp_space_override`. Recuerda el lado del body: `linear_damp_mode` `COMBINE` (suma al del area/default) o `REPLACE` (ver `godot-physics`).
- **`SpaceOverride`** (enum verificado): `DISABLED (0)`, `COMBINE (1)`, `COMBINE_REPLACE (2)`, `REPLACE (3)`, `REPLACE_COMBINE (4)` — resuelven la jerarquía cuando varios areas se solapan, **en orden de `priority`** (default `0`; mayor prioridad se aplica después).
- **Viento**: `wind_force_magnitude`, `wind_attenuation_factor`, `wind_source_path` (`NodePath` a un nodo fuente, p. ej. un `AudioStreamPlayer` o un nodo propio para leer dirección).
- **Audio**: `audio_bus_override` + `audio_bus_name` (deruta el audio a un bus distinto dentro de la zona) y `reverb_bus_enabled/name/amount/uniformity` (reverb por zona).

## API relevante

Todo verificado en Godot 4.7.

### Propiedades

| Propiedad | Default | Nota |
|---|---|---|
| `monitoring` | `true` | Emite señales (requisito de TODAS las señales). |
| `monitorable` | `true` | Puede ser detectado. |
| `priority` | `0` | Orden de overrides cuando se solapan (mayor = después). |
| `gravity` | `9.8` | Módulo de la gravedad local. |
| `gravity_direction` | `(0,-1,0)` | Dirección. |
| `gravity_point` | `false` | Gravedad puntual (hacia `gravity_point_center`). |
| `gravity_point_center` | `(0,-1,0)` | Centro (relativo al area). |
| `gravity_point_unit_distance` | `0.0` | Distancia de unidad de la falloff puntual. |
| `gravity_space_override` | `SPACE_OVERRIDE_DISABLED (0)` | Modo del override de gravedad. |
| `linear_damp` / `angular_damp` | `0.1` / `0.1` | Damping local. |
| `linear_damp_space_override` / `angular_damp_space_override` | `DISABLED (0)` | Modo de cada override. |
| `wind_force_magnitude` / `wind_attenuation_factor` | `0.0` | Fuerza / caída con distancia. |
| `wind_source_path` | `NodePath("")` | Nodo fuente del viento. |
| `audio_bus_override` / `audio_bus_name` | `false` / `"Master"` | Reruteo de audio. |
| `reverb_bus_enabled` / `reverb_bus_name` / `reverb_bus_amount` / `reverb_bus_uniformity` | `false` / `"Master"` / `0.0` / `0.0` | Reverb por zona. |

### Señales (todas requieren `monitoring = true`)

| Señal | Argumentos | Nota |
|---|---|---|
| `body_entered(body: Node3D)` / `body_exited(body)` | `body` puede ser `PhysicsBody3D`, `SoftBody3D` o `GridMap` | **No se emite para `SoftBody3D`** (el motor no reporta overlaps con soft bodies — nota oficial). `GridMap` solo si su `MeshLibrary` tiene collision shapes. |
| `area_entered(area: Area3D)` / `area_exited(area)` | — | Area dentro de area. |
| `body_shape_entered(body_rid, body, body_shape_index, local_shape_index)` / `body_shape_exited(…)` | RID + índices de shape | Para saber **qué forma** tocó. Recuperar el nodo `CollisionShape3D` (ejemplo oficial): `body.shape_find_owner(body_shape_index)` → `body.shape_owner_get_owner(owner)`. |
| `area_shape_entered(area_rid, area, area_shape_index, local_shape_index)` / `area_shape_exited(…)` | igual, para areas | — |

### Métodos

| Método | Devuelve | Nota |
|---|---|---|
| `get_overlapping_bodies()` | `Array[Node3D]` | Los bodies que **están dentro ahora**. |
| `get_overlapping_areas()` | `Array[Area3D]` | Las areas solapadas ahora. |
| `has_overlapping_bodies()` / `has_overlapping_areas()` | `bool` | Chequeo barato (sin array). |
| `overlaps_body(body: Node)` / `overlaps_area(area: Node)` | `bool` | "¿Este object está dentro de mí?". |

### Warning oficial: objects creados por `PhysicsServer3D`

"Areas and bodies created with `PhysicsServer3D` might not interact as expected with `Area3D`s, and might not emit signals or track objects correctly" → **crear los bodies por escena/nodos** (no por server raw) si van a interactuar con areas.

### Warning oficial: `ConcavePolygonShape3D` en areas

Un trimesh (concave) dentro de un `CollisionShape3D` de un Area **puede dar resultados inesperados porque el shape es hueco**. Si se necesita: dividir en convexes/primitivas, o usar `CollisionPolygon3D` (la opción que la docs sugiere).

## Arquitectura recomendada

### Escena: un Area por responsabilidad

```text
Pickup (Area3D)                    # layer 5 (interactables), mask 2 (player)
├─ CollisionShape3D (SphereShape3D)
└─ MeshInstance3D (visual)
   # monitoring = true, monitorable = false  (solo emite)
```

Reglas:

1. **El shape del Area no es el visual**: el visual es `MeshInstance3D` (sin colisión); el Area lleva su propio `CollisionShape3D` (normalmente más grande que el mesh: el "magnet" del pickup).
2. **La interacción la define la mask del Area** (qué lo detecta): pickup → mask solo `player`; zona de daño de enemigo → mask `player`; sensor del jugador → mask `enemies`.
3. **`monitorable = false` en triggers** que nadie necesita detectar (menos pares, ver *Performance*).
4. **Área de influencia sin señales**: `monitoring = false` (no pagas señales que nadie escucha).
5. **Varios solapados**: `priority` + modos `SpaceOverride` para decidir quién gana (gravedad de un area dentro de otro).

### Patrón de señales vs polling

| Necesidad | Mecanismo |
|---|---|
| "Reaccionar a la entrada/salida" | Señales (`body_entered/exited`) — el camino por defecto. |
| "¿Quién está dentro AHORA?" (una vez) | `get_overlapping_bodies()`. |
| "¿Está dentro este object?" (chequeo puntual) | `overlaps_body(player)`. |
| "Contador" | `+1` en `body_entered`, `-1` en `body_exited` (o `get_overlapping_bodies().size()` si no confías en el par de señales). |

## Implementación mínima

**Pickup** (la pieza más repetida de cualquier juego):

```gdscript
# pickup.gd — Area3D, layer 5 (interactables), mask 2 (player)
extends Area3D

signal picked_up(pickup: Area3D)

func _ready() -> void:
	monitorable = false   # nadie necesita detectarme; solo yo detecto
	body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node3D) -> void:
	if body is CharacterBody3D:          # o chequear su capa/grupo
		picked_up.emit(self)
		queue_free()
```

## Implementación recomendada

### 1. Zona de daño con cooldown (sin que dispare 60 veces por segundo)

```gdscript
# damage_zone.gd — Area3D: monitoring=true, monitorable=false
extends Area3D

@export var damage_per_second := 10.0

var _victims: Array[Node3D] = []

func _ready() -> void:
	body_entered.connect(_on_body_entered)
	body_exited.connect(_on_body_exited)

func _physics_process(delta: float) -> void:
	for victim in _victims:
		if is_instance_valid(victim) and victim.has_method("take_damage"):
			victim.take_damage(damage_per_second * delta)

func _on_body_entered(body: Node3D) -> void:
	if body is CharacterBody3D and not _victims.has(body):
		_victims.append(body)

func _on_body_exited(body: Node3D) -> void:
	_victims.erase(body)
```

Patrón clave: **las señales mantengan la lista; el daño corre en `_physics_process`** (daño continuo = trabajo de tick, no de señal).

### 2. Checkpoint / estado del jugador ("¿estoy dentro?")

```gdscript
# checkpoint.gd — Area3D
extends Area3D

func _ready() -> void:
	monitorable = false
	body_entered.connect(func(body: Node3D):
		if body is CharacterBody3D:
			Game.checkpoint_reached(self)
			monitoring = false   # ya cumplió: apagar el detector (menos trabajo)
	)
```

### 3. Zona de gravedad media (influencia, sin señales)

```text
Area3D:
  monitoring = false, monitorable = false
  gravity = 4.9, gravity_space_override = SPACE_OVERRIDE_REPLACE
  linear_damp_space_override = DISABLED (dejar el default del proyecto)
```

Los `RigidBody3D`/`CharacterBody3D` adentro sienten `4.9` en vez de `9.8`. Si el body tiene `linear_damp_mode = COMBINE` (default), el damping del area se **suma** al del body (ver `godot-physics`).

### 4. Contador de intrusos por sector (con `body_shape_entered` para precisión)

```gdscript
# sector.gd — Area3D con 3 CollisionShape3D hijos (sectores A/B/C)
extends Area3D

func body_shape_entered(_body_rid: RID, body: Node3D, _body_shape: int, local_shape: int) -> void:
	# local_shape = índice del shape de ESTE area (0,1,2 = sectores A/B/C)
	sectors[local_shape].add(body)
```

### 5. Viento que empuja partículas de papel (rigids ligeros)

```text
Area3D (grande, sin colisión visible):
  monitoring = false, monitorable = false
  wind_force_magnitude = 3.0
  wind_attenuation_factor = 0.1
  wind_source_path = NodePath("../WindSource")   # nodo propio con dirección
```

### 6. Audio por zona (reverb en la cueva)

```text
Area3D:
  monitoring = false, monitorable = false
  reverb_bus_enabled = true
  reverb_bus_name = &"CaveReverb"
  reverb_bus_amount = 0.7
  reverb_bus_uniformity = 0.5
```

(El bus `CaveReverb` debe existir en el `AudioBusLayout` con un efecto `Reverb` — eso es audio, fuera de scope físico.)

## Ejemplo práctico

**Juego**: metroidvania 3D con 40 pickups, 3 zonas de peligro y cuevas con eco.

1. **Pickups** (40×): `Area3D` + `SphereShape3D` (radio > mesh), layer `interactables`, mask `player`, `monitorable = false`. Señal `body_entered` → spawn/recoger. Coste marginal: cada area es un detector barato; 40 areas es trivial (ver *Performance*).
2. **Zonas de peligro** (lava/vacío): `monitoring = true` solo; lista de víctimas + daño en `_physics_process` (patrón #1). Si la lava también mata a los `RigidBody3D` enemigos, la mask incluye esa capa.
3. **Gravedad de la cueva flotante**: `SPACE_OVERRIDE_REPLACE` con `gravity = 3.0` (el jugador se siente en la luna; los objetos que caen rebotan distinto).
4. **Reverb**: area de la cueva con `reverb_bus_*` (el jugador no nota la física, solo el audio).
5. **Debug**: `debug_color` en el `CollisionShape3D` del area para ver el volumen real en el editor (y en runtime con los debug shapes del viewport).

## Integración

- **`godot-physics`**: layers/masks (la detección la decide el cruce layer/mask), ciclo de ticks, `disable_mode` (un body `REMOVE` no es detectable).
- **`godot-characterbody3d`**: el personaje es el `body` de las señales (`body is CharacterBody3D`); su `move_and_slide()` actualiza el estado de overlap para el `get_overlapping_bodies()` del mismo tick.
- **`godot-raycast3d`**: el overlap instantáneo que Area3D no da (`ShapeCast3D` con `target_position = (0,0,0)`) — la docs de `ShapeCast3D` lo documenta como la solución a esta limitación de Area3D.
- **`godot-physics-materials`**: complementario: el material define la respuesta de choque; el Area define la zona (gravedad/damping) que la modula.
- **Audio**: `audio_bus_override`/`reverb_bus_*` puentean a `AudioStreamPlayer`/`AudioListener3D` (skills de audio pendientes).

## Errores frecuentes

1. **Esperar señales con `monitoring = false`** → ninguna señal sale (requisito explícito de cada señal, verificado). Lo típico: apagarlo "para optimizar" y preguntar "¿por qué no entra?".
2. **El Area "no detecta" al jugador** → cruce de capas: el `collision_layer` del jugador no está en el `collision_mask` del area (o viceversa). Es el error nº1 (ver *Debugging*).
3. **Un Area como suelo** (esperando que detenga) → los areas no son sólidos; es `StaticBody3D`.
4. **Pickup que dispara N veces al solaparse** → `body_entered` dispara una sola vez por entrada, pero si el object entra/sale de un shape hueco (`ConcavePolygonShape3D`) o rebota en el borde, puede parpadear: usar primitivas de shape (warning oficial de concave hueco) o cooldown/`queue_free` inmediato.
5. **Daño continuo en la señal** (aplicar `damage * 60` en `body_entered`) → el daño continuo va en `_physics_process` con `delta` (patrón #1).
6. **Polling `get_overlapping_bodies()` por frame "por seguridad"** → las señales ya dan el evento; el polling es para consultas puntuales ("¿quién está adentro ahora?"). 40 areas × polling por frame = trabajo inútil.
7. **`SoftBody3D` dentro de un area y no hay señal** → no es bug: el motor no reporta overlaps con soft bodies (nota oficial).
8. **Body creado por `PhysicsServer3D` raw no interactúa con areas** → warning oficial: crear por escena/nodos.
9. **Varios areas de gravedad solapados y "gana el que quiero"** → `priority` + modos `SpaceOverride` (verificado); con todo en `COMBINE` los valores se suman en orden de prioridad.
10. **`wind_source_path` vacío esperando viento global** → el viento usa la fuente del path (default `NodePath("")` = ninguna fuente).
11. **Trimesh del mesh como shape del Area** → resultados inesperados (shape hueco, warning oficial) → primitivas/convexes o `CollisionPolygon3D`.
12. **Mover el Area por `_process` y esperar detección inmediata** → la detección es por tick de física (y el state de overlap se actualiza en el paso de física).

## Anti-patrones

- **Un Area gigante "de todo el mundo" para detectar "cualquier cosa"** → 1 detector con mask de 32 capas escanea todo; areas pequeñas y específicas con masks restrictivas.
- **Areas como muros/escalones** (sólidos de verdad) → `StaticBody3D`; el Area es detección, no geometría.
- **100 Areas idénticas donde alcanza una GridMap/colisionador compuesto + lógica** (p. ej. "tiles con daño") → un solo body/area con `body_shape_entered` por shape o datos de grid (`godot-physics` → colisionador compuesto).
- **`monitorable = true` + `monitoring = true` en triggers que nadie observa** → cada flag activo es pares de colisión a mantener; apagar lo que no se usa.
- **Recrear el Area por `SceneInstance` en runtime por cada pickup** cuando alcanza `add_child` de una escena ligera, o un pool — decisión de diseño (medir, no asumir).
- **Confiar en `body_exited` para limpiarez con objects que se quitan del árbol** → si el body se libera sin emitir `exited` (edge cases de quita/teleport), el contador se queda; limpiar con `NOTIFICATION_EXIT_TREE` o validación `is_instance_valid` en el `_physics_process`.
- **Área de influencia con `monitoring = true` "para debug"** y dejarlo en producción → señales que nadie escucha = trabajo por tick.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El coste de un Area está en los pares**: cada area con `monitoring`/`monitorable` activo genera pares de colisión con los objects de las capas que se cruzan. Eso se ve en `Performance.PHYSICS_3D_COLLISION_PAIRS` (índice 21, ver `godot-physics`).
- **Qué medir**: pair count y FPS con N areas vs N/2 (mismo escenario). "Son areas, no cuestan nada" NO es una medida.
- **Optimizaciones (contra un cuello medido)**:
  1. `monitoring = false` / `monitorable = false` donde no se usan (el de mayor impacto).
  2. Masks restrictivas (menos pairs por area).
  3. Apagar el área cuando cumple (checkpoint: `monitoring = false` tras el primer hit — patrón #2).
  4. Menos, mayores, bien colocadas (1 area por sector > 50 areas por tile; ver anti-patrón del grid).
  5. Shapes simples (primitivas) en vez de trimesh (más caro de escanear).
- **Lo barato de verdad**: un área con una primitiva y mask de 1 capa es ruido comparado con un `RigidBody3D` activo o un shape concave.

## Debugging

### "El area no detecta"

```text
1. ¿monitoring = true? (requisito de TODAS las señales — verificado)
2. ¿El body tiene shape y el area tiene shape? (los dos lados necesitan
   al menos un CollisionShape3D con shape)
3. Cruzar capas: print del collision_layer del body vs collision_mask del
   area (y viceversa). ¿Se intersecta el bit?
4. ¿El body es SoftBody3D? (no se reportan — nota oficial)
5. ¿El body se quitó del tree / disable_mode = REMOVE?
6. Visual: debug_color del shape + Visible Collision Shapes (viewport)
   → ver el volumen EXACTO del area.
```

### "Dispara muchas veces al entrar"

```text
1. ¿Shape hueco (ConcavePolygonShape3D)? (warning oficial) → primitiva.
2. ¿El object rebota en el borde del area? (el body entra/sale/entra con
   la física) → agrandar el shape o cooldown.
3. ¿Conectaste el handler DOS veces? (p. ej. en _ready + inspector) →
   double signal.
4. En el handler: log de body.name + tiempo → ver el patrón real.
```

### "El overlap no es inmediato"

```text
1. Normal: la detección es por tick de física (no continuo).
2. ¿Necesitas "ahora mismo"? → ShapeCast3D target (0,0,0) +
   force_shapecast_update() (patrón oficial, ver godot-raycast3d).
3. ¿Tras un move_and_slide del mismo tick? → get_overlapping_bodies()
   tras la llamada (el body ya resolvió).
```

### "La gravedad del area no se aplica"

```text
1. ¿El body es RigidBody3D/CharacterBody3D afectado? (un Static no se
   mueve por gravedad de todos modos)
2. gravity_space_override = DISABLED (0)? (es el default: el area no
   afecta NADA hasta que elijas un modo)
3. ¿Varios areas solapados? → priority + modos (verificado).
4. ¿El body tiene gravity_scale = 0? (godot-physics)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla completa de la clase + tutorial (lista en *Referencias*).
- **3.x → 4.x (trampas)**:
  - Señales `object_entered`/`object_exited`: **no están en la tabla 4.7** (la tabla lista `area_*`/`body_*`/`*_shape_*`). Si un tutorial viejo las cita para 4.x, no usar sin verificar la versión.
  - El sistema de override de espacio es el enum `SpaceOverride` con 5 modos (verificado 4.7); en 3.x era `space_override`/`gravity_override` como bools.
  - Viento (`wind_*`) y reverb por area: API de 4.x (verificada 4.7).
- **4.0 ↔ 4.7**: la tabla consultada es estable para el uso cubierto aquí.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-physics` | Base: layers/masks, ticks, debug shapes |
| `godot-characterbody3d` | El "body" típico de las señales |
| `godot-raycast3d` | El overlap instantáneo que Area no da |
| `godot-physics-materials` | Respuesta de choque (complemento) |
| `godot-area2d` (pendiente) | Espejo 2D |
| `godot-audio-buses` (pendiente) | Los buses que el area enruta |

## Skills relacionadas

- `godot-recipe-pickup-system` — receta (pendiente; germen en *Implementación mínima* + #1).
- `godot-recipe-damage-zone` — pendiente (germen #1).
- `godot-error-area-does-not-detect` — pendiente (germen en *Debugging*).
- `godot-error-gravity-zone-not-working` — pendiente (germen #4).

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Area3D (props, señales, métodos, `SpaceOverride`, warnings de `PhysicsServer3D` y `ConcavePolygonShape3D`, soft bodies): https://docs.godotengine.org/en/stable/classes/class_area3d.html
- Physics introduction (los 3 usos de area, default de input de mouse/touch, layers/masks): https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html
- ShapeCast3D (la limitación de overlap-instantáneo de Area3D y su solución oficial): https://docs.godotengine.org/en/stable/classes/class_shapecast3d.html
- CollisionObject3D (layers/masks, `input_ray_pickable` — Lote 0): https://docs.godotengine.org/en/stable/classes/class_collisionobject3d.html
