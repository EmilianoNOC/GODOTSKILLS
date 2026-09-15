# godot-node2d

## ID

`godot-node2d`

## Categoría

2D · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Node2D` (tabla completa), `Camera2D`, `CharacterBody2D` (defaults que confirman la convención de eje Y) y la lista *Inherited By* (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — propiedades y métodos de `Node2D` (tabla completa verificada), la lista de subclases, el hecho de que `pivot_offset`/`top_level` **no** existen en 4.7, `z_index`/`visible` heredados de `CanvasItem`: verificados directamente en la docs 4.7.
**MEDIUM** — convención de proyecto (estructura de escenas 2D, qué mover con `_process` vs `_physics_process` cuando no hay física).
**"No existe en la tabla oficial 4.7 (verificado)"**: `Node2D.pivot_offset` (prop de 3.x), `Node2D.top_level` (prop de 3.x) — la tabla 4.7 lista exactamente 12 propiedades y no están.

## Nivel

**BASE** (fundación del stack 2D: `godot-tilemap`, `godot-platformer-2d`, `godot-camera2d` y todo lo 2D dependen de estos conceptos).

## Propósito

Proporcionar el conocimiento operativo de `Node2D`: el nodo raíz de TODO lo 2D en Godot 4 (sprites, cuerpos físicos, cámaras, tilemaps, partículas). Cubre el transform (posición/rotación/escala/skew, local vs global), las conversiones de coordenadas (`to_global`/`to_local`), el orden de render (`z_index` vía `CanvasItem`), y las trampas de versión (lo que existía en 3.x y ya no está). Es la capa mínima sin la que ningún otro skill 2D se entiende.

## Cuándo utilizarla

- Entender la escena 2D: qué hereda de `Node2D` (casi todo).
- Mover/rotar/escalar nodos 2D y entender local vs global.
- Convertir entre coordenadas de pantalla/world/scene (`to_global`/`to_local`).
- Migrar de Godot 3 a 4 en código 2D (`pivot_offset`, `top_level`, escala).
- Depurar "el nodo no está donde lo puse" (global vs local, padre escalado/rotado).

## Cuándo NO utilizarla

- **UI** (botones, paneles, anchos/alturas) → `Control` (skill pendiente; hereda de `CanvasItem`, no de `Node2D`).
- **Movimiento con colisión** → `godot-platformer-2d` (`CharacterBody2D` hereda `Node2D` pero el movimiento va por `move_and_slide`).
- **Mapas por tiles** → `godot-tilemap`.
- **Cámara** → `godot-camera2d`.
- **2D vs 3D**: nada de esto aplica a `Node3D` (otra jerarquía, otro sistema de coordenadas).

## Conceptos fundamentales

### `Node2D` es la raíz de todo lo 2D

- Hereda de `CanvasItem` (y de `Node`). **Todos** los nodos 2D de Godot heredan de `Node2D` (verificado en la lista *Inherited By* 4.7), entre ellos:
  - Render: `Sprite2D`, `AnimatedSprite2D`, `Polygon2D`, `Line2D`, `MeshInstance2D`, `MultiMeshInstance2D`, `Light2D`, `LightOccluder2D`.
  - Física: `CollisionObject2D` (→ `CharacterBody2D`, `RigidBody2D`, `StaticBody2D`, `Area2D`), `CollisionShape2D`, `CollisionPolygon2D`, `RayCast2D`, `ShapeCast2D`, `Joint2D`.
  - Mundo: `Camera2D`, `TileMap`, `TileMapLayer`, `AudioStreamPlayer2D`, `AudioListener2D`.
  - Estructura: `Bone2D`, `Skeleton2D`, `Path2D`, `PathFollow2D`, `Parallax2D`, `CanvasGroup`, `RemoteTransform2D`, `Marker2D`, `VisibleOnScreenNotifier2D`, `GPUParticles2D`/`CPUParticles2D`, `TouchScreenButton`, `NavigationRegion2D`/`NavigationObstacle2D`/`NavigationLink2D`.
- Propósito (texto oficial): "has a position, rotation, scale, and skew"; usarlo como padre para mover/escalar/rotar hijos en conjunto, y dar control del **render order**.
- **Convención de ejes**: Y apunta **hacia abajo** en pantalla. Se confirma en la API: `CharacterBody2D.up_direction` default es `Vector2(0, -1)` ("arriba" = -Y, verificado). Por eso "saltar" es velocidad Y negativa.
- **`CanvasItem` compartido**: `Node2D` y `Control` comparten `CanvasItem.z_index`, `CanvasItem.visible` y demás (nota oficial). El orden de render se controla con `z_index`/`z_as_relative` (en `CanvasItem` — la skill de `CanvasItem` está pendiente; aquí solo el uso).

### Transform: local vs global (la causa nº1 de bugs 2D)

| Prop (local) | Prop (global) | Qué es |
|---|---|---|
| `position` (default `(0,0)`) | `global_position` | Relativa al **padre** vs al **mundo** (canvas). |
| `rotation` (rad) / `rotation_degrees` | `global_rotation` / `global_rotation_degrees` | — |
| `scale` (default `(1,1)`) | `global_scale` | — |
| `skew` (rad) | `global_skew` | Skew (raro; para efectos). |
| `transform` (`Transform2D`) | `global_transform` | La combinación completa. |

Reglas:

1. **`global_*` incluye a los padres**: si el padre está rotado/escalado, `position` ≠ `global_position`.
2. **No escalar nodos con hijos 2D "a ciegas"**: una escala no uniforme en un padre corrompe rotaciones y raycasts de los hijos (la docs de física 2D/3D advierte sobre escala no uniforme en bodies).
3. Para posicionar "en el mundo" (p. ej. spawn en coordenadas absolutas) → `global_position`. Para mover "relativo a mí" → `position`/`translate`.

### `pivot_offset` y `top_level`: no existen en 4.7 (trampa de migración)

- **`Node2D.pivot_offset` (3.x) no existe en la tabla 4.7** (verificado). En 4.x el punto de pivote de un sprite lo da `Sprite2D.pivot_offset` (prop del sprite, no del Node2D genérico).
- **`Node2D.top_level` (3.x) no existe en 4.7** (verificado). Si un nodo 3D/2D "se fijaba" al canvas, en 4.x el mecanismo es distinto (en 2D, el canvas del viewport; `CanvasGroup` cambia el canvas).
- Cualquier tutorial 3.x que toque estas dos props → migrar antes de usar.

### Procesar: `_process` vs `_physics_process`

- Movimiento **sin física** (decoración, cámara simple, parallax): `_process(delta)` basta.
- Movimiento **con física** (bodies, raycasts, queries al space): **`_physics_process(delta)`** — fuera, el space 2D puede estar *locked* (misma regla que 3D, ver `godot-physics`).
- `delta` SIEMPRE (tasa de physics 60 Hz por defecto — verificado en `godot-physics`).

## API relevante

Todo verificado en Godot 4.7.

### Propiedades (tabla completa: 12)

| Propiedad | Default | Nota |
|---|---|---|
| `position` | `Vector2(0, 0)` | Local (padre). |
| `rotation` | `0.0` | Radianes, local. |
| `rotation_degrees` | — | Helper en grados (mismo valor, otra unidad). |
| `scale` | `Vector2(1, 1)` | Local. Escala no uniforme = cuidado (rotaciones/raycasts de hijos). |
| `skew` | `0.0` | Radianes (efectos; poco común). |
| `transform` | — | `Transform2D` local completo. |
| `global_position` | — | Posición en el canvas/mundo. |
| `global_rotation` / `global_rotation_degrees` | — | Radianes / grados. |
| `global_scale` | — | — |
| `global_skew` | — | — |
| `global_transform` | — | `Transform2D` global completo. |
| (heredadas de `CanvasItem`) `z_index`, `z_as_relative`, `visible`, … | — | Orden de render y visibilidad (compartidas con `Control`, nota oficial). |

### Métodos (tabla completa)

| Método | Devuelve | Nota |
|---|---|---|
| `to_global(local_point)` | `Vector2` | Pto local → world. |
| `to_local(global_point)` | `Vector2` | Pto world → local. |
| `translate(offset)` | — | Mueve **en coordenadas locales** (si el nodo está rotado, "x" no es +X world). |
| `global_translate(offset)` | — | Mueve en world (independiente de la rotación). |
| `rotate(radians)` | — | Suma rotación (local). |
| `apply_scale(ratio: Vector2)` | — | Multiplica la escala actual. |
| `look_at(point)` | — | Apunta al punto (gira el nodo). |
| `get_angle_to(point)` | `float` | Ángulo (rad) hacia el punto. |
| `move_local_x(delta)` / `move_local_y(delta)` | — | Mueve solo en X/Y local (con `scaled=false` por defecto; `scaled=true` aplica la escala). |
| `get_relative_transform_to_parent(parent)` | `Transform2D` | Transform relativo a otro ancestor. |

## Arquitectura recomendada

### Estructura típica de escena 2D

```text
World (Node2D)                     ← coordenas del mundo (Y abajo)
├─ Ground (TileMapLayer)           ← godot-tilemap
├─ Player (CharacterBody2D)        ← godot-platformer-2d
│   └─ Sprite (Sprite2D)           ← el visual va DENTRO (se mueve con el body)
├─ Camera2D (hija del player)      ← godot-camera2d (follow automático)
└─ FX (Node2D)                     ← partículas, líneas
```

1. **El visual nunca es el body**: `CharacterBody2D`/`StaticBody2D` con `Sprite2D`/`CollisionShape2D` hijos (patrón oficial de todos los demos 2D).
2. **Un `Node2D` "pivot" para rotar un conjunto** (p. ej. arma que mira al cursor): rota el pivot con `look_at`, no cada sprite.
3. **Coordenadas de design en world** (editor) y conversión a local solo donde hace falta.
4. **`z_index` por capas de render** (suelo 0, entities 10, UI-en-mundo 20, FX 30) — números propios de proyecto (MEDIUM).

## Implementación mínima

**Mover un nodo 2D por input (sin física)** — todo API verificada:

```gdscript
extends Node2D

@export var speed := 200.0  # px/s

func _process(delta: float) -> void:
	var dir := Input.get_vector("move_left", "move_right", "move_up", "move_down")
	global_translate(dir * speed * delta)  # world, no depende de la rotación
```

## Implementación recomendada

### 1. Apuntar al mouse (turret 2D)

```gdscript
extends Node2D

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventMouseMotion:
		var world_mouse := get_global_mouse_position()  # CanvasItem
		look_at(world_mouse)                            # Node2D (verificado)
```

### 2. Spawn en coordenadas absolutas del mundo

```gdscript
# El spawner puede estar en cualquier rama de la escena:
spawn.global_position = Vector2(1600, -32)  # world, no importa el padre
# o leer un punto de un nodo hermano:
var p: Vector2 = $Marker2D.global_position
```

### 3. Convertir pantalla ↔ mundo (lo más usado)

```gdscript
# pantalla → mundo (p. ej. dónde clickear):
var world_pos: Vector2 = get_global_mouse_position()     # si el nodo es del canvas raíz
var world_pos2: Vector2 = some_node.to_local(screen_pos) # si some_node tiene transform

# mundo → pantalla (p. ej. tooltip sobre una entidad):
var screen_pos: Vector2 = get_canvas_transform().affine_inverse() * entity.global_position
```

> `get_canvas_transform()` (CanvasItem) es la vía para "a pantalla"; la conversión inversa es `to_local` cuando el nodo ES el canvas raíz.

### 4. Rotar un conjunto (padre pivot)

```gdscript
# Arma que sigue a la mira: pivot gira, el sprite hijo se dibuja donde quieras.
# No rotar el sprite con escala no uniforme (los hijos se deforman).
$TurretPivot.look_at(mouse_world)
```

### 5. Parallax simple (sin Parallax2D)

```gdscript
# Fondo que se mueve menos que el mundo:
# background.global_position = camera_screen_center * -parallax_factor  (MEDIUM: patrón)
# Para parallax con capas reales → nodo Parallax2D (existe en 4.7, verificado en Inherited By;
# su API propia no se cubre aquí).
```

## Ejemplo práctico

**Juego**: runner 2D con mundo de 12,000 px, jugador, monedas y cámara.

1. `World (Node2D)` raíz; todo en coords world (Y abajo: el piso está en `y = 0` y lo "alto" es `y < 0`).
2. Jugador: `CharacterBody2D` + `Sprite2D` hijo + `CollisionShape2D` (ver `godot-platformer-2d`). El sprite se anima, el body se mueve.
3. Turret de debug que mira al mouse: `look_at(get_global_mouse_position())`.
4. Spawn de monedas por datos de level: `coin.global_position = tile_pos` (world, verificado `to_local`/`local_to_map` en `godot-tilemap` para el caso de tiles).
5. **Debug** "el nodo no está donde lo puse": imprimir `position` y `global_position` — si difieren, el padre tiene transform (rotación/escala/posición).

## Integración

- **`godot-platformer-2d`**: `CharacterBody2D` es un `Node2D`; su `position` la mueve `move_and_slide` (no setear a mano — ver `godot-physics`).
- **`godot-tilemap`**: `TileMapLayer` es un `Node2D`; `local_to_map`/`map_to_local` hacen la conversión tile↔coordenada local.
- **`godot-camera2d`**: la cámara es un `Node2D`; su `global_position` **no** es la posición real de la pantalla (smoothing/limits — ver esa skill).
- **`godot-physics`**: la regla `_physics_process` para todo lo físico 2D es la misma que 3D.

## Errores frecuentes

1. **Esperar `pivot_offset` en un `Node2D` (3.x)** → no existe en 4.7 (verificado); el pivot es de `Sprite2D`.
2. **Usar `top_level` (3.x)** → no existe en 4.7 (verificado).
3. **`position` vs `global_position`** → el clásico "se mueve raro": si el padre está rotado/escalado/posicionado, `position` es relativo. Usar `global_*` para world.
4. **`translate` en un nodo rotado** → mueve en ejes LOCALES (no world). Para world: `global_translate`.
5. **Escalar un padre no uniformemente** → rotaciones y raycasts de los hijos se corrompen (la docs de física lo advierte para bodies; aplica a cualquier hijo con transform).
6. **Esperar que `rotation` esté en grados** → es radianes (`rotation_degrees` para grados, verificado).
7. **Mover bodies físicos en `_process`** → space 2D puede estar *locked*; usar `_physics_process` (misma regla que 3D).
8. **Posicionar UI como si fuera `Node2D`** → la UI es `Control` (anchors/anchors, otro sistema; skill pendiente).
9. **`look_at` esperando que "mire por +X" vs "-X"** → `look_at` apunta el eje local hacia el punto; el "frente" del sprite depende del asset (rotar el sprite en el editor si hace falta).
10. **Confundir `Node2D` con `Node3D`** (código 3.x portado) → distinta jerarquía y distinto sistema de coordenadas (Y abajo en 2D, Y arriba en 3D).

## Anti-patrones

- **Un `Node2D` gigante con 200 hijos "para rotarlo"** cuando solo 3 se deben rotar → pivot chico para el conjunto que rota.
- **Guardar `global_position` y luego leerlo en otro frame** sin re-leerlo (el mundo puede moverse: plataformas, cámara, scrolling).
- **Convertir a world cada frame "por si acaso"** en bucles grandes → `to_global`/`to_local` son baratas, pero si el padre no cambia, cachear el transform es legítimo (medir, no por intuición).
- **Escenas 2D con nodos 3D mezclados** (un `Node3D` dentro del mundo 2D "para un efecto") → sistemas distintos; en 2D usar `Node2D`/`MeshInstance2D`/shaders 2D.
- **`position = global_position`** (asignar global a local) → se desplaza doble (una vez por el padre).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **`Node2D` en sí no es un cuello**: el transform es 2D barato. El coste real aparece en:
  - **Nº de CanvasItems visibles** (fill rate 2D — ver `godot-rendering-performance`; la ventana pequeña detecta fill-rate).
  - **Bodies/raycasts 2D activos** → monitores `Performance.PHYSICS_2D_ACTIVE_OBJECTS` / `PHYSICS_2D_COLLISION_PAIRS` (la familia de monitores existe; los índices exactos 2D: **no verificados en esta revisión** — ver `Performance.xml` antes de citar índices).
  - **Conversions por frame en bucles enormes** → medir con `TIME_PROCESS`.
- **No optimizar por intuición**: "son solo nodos 2D" NO es una medida; el FPS con/without el sistema lo es.

## Debugging

### "El nodo no está donde lo puse / se mueve raro"

```text
1. Imprimir position y global_position: ¿difieren? → el padre tiene transform.
2. ¿El padre está rotado/escalado? (global_transform vs transform)
3. ¿Estás usando translate (local) cuando querías global?
4. ¿Asignaste global_position a position? (desplazamiento doble)
```

### "Lo que dibujo en pantalla no coincide con el mundo"

```text
1. ¿Cámara activa? (Camera2D desplaza el canvas; get_screen_center_position()
   da la posición REAL — ver godot-camera2d)
2. Conversiones: get_global_mouse_position() / to_local / canvas transform.
3. ¿zoom? (Camera2D.zoom escala lo que se ve; las coords world no cambian)
```

### "El sprite rotado 'mira' al lado contrario"

```text
1. El asset apunta a -X o +Y según el artista; rotar el sprite en el editor
   (o el pivot) para que look_at coincida con el "frente".
2. look_at apunta el EJE local, no el "dibujo" (ver error #9).
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla completa de `Node2D` (lista en *Referencias*).
- **3.x → 4.x (trampas, verificado contra la tabla 4.7)**:
  - `Node2D.pivot_offset` → **no existe en 4.7**; el pivot es de `Sprite2D` (y `AnimatedSprite2D` etc.).
  - `Node2D.top_level` → **no existe en 4.7**.
  - `CanvasItem.z_index`/`z_as_relative`: estables (compartidas Node2D/Control, nota oficial 4.7).
- **4.0 ↔ 4.7**: la tabla de `Node2D` consultada (12 props, 11 métodos) es estable para el uso cubierto.

## Dependencias

| Skill | Relación |
|---|---|
| (ninguna base) | Fundación 2D |
| `godot-tilemap` | Usa `local_to_map`/`map_to_local` y el transform |
| `godot-platformer-2d` | `CharacterBody2D` es un `Node2D` |
| `godot-camera2d` | La cámara es un `Node2D` |
| `godot-physics` | La regla `_physics_process` (común 2D/3D) |
| `godot-control` (pendiente) | El otro hijo de `CanvasItem` (UI) |
| `godot-canvasitem` (pendiente) | `z_index`, drawing, `get_global_mouse_position` |

## Skills relacionadas

- `godot-recipe-spawn-system-2d` — pendiente (germen en *Implementación recomendada* #2/#3).
- `godot-error-2d-node-offset` — pendiente (germen en *Debugging* #1).

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Node2D (props, métodos, `CanvasItem` shared, *Inherited By*): https://docs.godotengine.org/en/stable/classes/class_node2d.html
- CharacterBody2D (default `up_direction = (0,-1)` — convención de ejes): https://docs.godotengine.org/en/stable/classes/class_characterbody2d.html
- Physics introduction (regla `_physics_process`, 60 Hz, no-determinismo — 2D y 3D): https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html
