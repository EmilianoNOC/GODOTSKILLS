# godot-tilemap

## ID

`godot-tilemap`

## Categoría

2D · Base (mapas)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: páginas de clase `TileMapLayer` (tabla completa) y `TileSet` (props, métodos, enums), y tutorial `troubleshooting_physics_issues.html` (colisionador compuesto, 4.5+) (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — `TileMap` **deprecado** → `TileMapLayer` (texto oficial), props/métodos/señal `changed()` de `TileMapLayer`, props/métodos/enums de `TileSet` (`TILE_SHAPE_*`, `TILE_LAYOUT_*`, `TILE_OFFSET_AXIS_*`, `CellNeighbor`, `TerrainMode`), las capas de `TileSet` (physics/navigation/custom data/occlusion) y sus `add_*/remove_*/set_*`, warning de sources compartidos, límite de coordenadas 16-bit (wrap), batching de updates + `update_internals()`, `physics_quadrant_size` (default 16): verificados directamente en la docs 4.7.
**MEDIUM** — valores de tuning (tamaño de tile, cuántos layers de física) — decisión de proyecto.
**"No existe / cambió (verificado 4.7)"**: `TileMap` como nodo principal (deprecado oficialmente; usar `TileMapLayer` por capa), `TileMap.layers` (los layers separados son nodos `TileMapLayer`), API de 3.x (`set_cell` con layer index, `get_cell_atlas_coords` con layer, etc.).

## Nivel

**BASE** (depende de `godot-node2d`; la usan `godot-platformer-2d` y cualquier juego por tiles).

## Propósito

Proporcionar el conocimiento operativo de los mapas 2D por tiles en Godot 4.7: `TileMapLayer` (el nodo, una capa por nodo; `TileMap` está deprecado) + `TileSet` (la librería: sources de atlas/escenas, capas de física/navegación/custom data/oclusión, shapes isométrica/hexagonal, terrenos, patrones). Cubre: pintar tiles, conversión tile↔coordenadas, colisión por tiles, datos custom (collectibles), terreno automático, runtime editing y performance (batching, `changed()`, cuadrantes).

## Cuándo utilizarla

- Construir mundos 2D por tiles (plataformas, RPG, isométrico, hex).
- Dar **colisión** a tiles (physics layer del `TileSet`).
- **Datos por tile** (qué tile es collectible/door/door-id) con custom data layers.
- Editar el mapa en runtime (spawn, destruir suelo, generar chunks).
- Isométrico/hexagonal (shapes y layouts del `TileSet`).

## Cuándo NO utilizarla

- **Tiles con lógica compleja por celda** (cada tile con script/estado) → escenas por tile (`TileSetScenesCollectionSource`) o un sistema de grid propio; no meter lógica en el tilemap.
- **Niveles que NO son grid** (curvas, free-form) → `StaticBody2D`/`CollisionShape2D` normales.
- **2.5D/3D** → `TileMap` no cubre 3D (malla 3D por tiles es otra cosa; sin skill).
- **UI/panesles** → `Control`.

## Conceptos fundamentales

### `TileMap` está deprecado → `TileMapLayer` (la decisión 4.3+)

Texto oficial (página de `TileMapLayer` 4.7): "Unlike the **TileMap** node, which is **deprecated**, **TileMapLayer** has only one layer of tiles. You can use several **TileMapLayer** to achieve the same result as a **TileMap** node."

- 1 `TileMapLayer` = 1 capa de tiles. Fondo + colisión + decoración = 3 nodos `TileMapLayer` con el mismo `TileSet` (o distintos).
- El orden de render entre capas = orden de los nodos hijos (o `z_index`).
- **Migrar**: cada layer del viejo `TileMap` → un `TileMapLayer`; la API `set_cell(coords, layer, …)` 3.x/`TileMap` → `set_cell(coords, …)` del `TileMapLayer` (sin índice de layer — la capa ES el nodo).

### El `TileSet`: librería + capas de propiedades

- `TileSet` (hereda `Resource`) es la **librería** de tiles que usan uno o varios `TileMapLayer`.
- Maneja **sources**: `TileSetAtlasSource` (tiles cortados de una **textura**; soporta física, navegación, etc.) o `TileSetScenesCollectionSource` (**tiles-escena**: cada tile instancia un nodo).
- Un tile se referencia con **3 IDs** (verificado): **source ID** + **atlas coords** (`Vector2i`) + **alternative tile ID**.
- **Capas de propiedades** del `TileSet` (se agregan/quitan según necesidad — verificado):
  - **Physics layers** → "collision polygons to atlas tiles"; cada capa tiene `collision_layer`/`collision_mask`/`collision_priority` y un `PhysicsMaterial` (ver `godot-physics-materials`).
  - **Navigation layers** → zonas navegables por tile.
  - **Custom data layers** → propiedades custom por tile (tipo `Variant.Type`: int/float/string/bool/…); para "este tile es una puerta id 7".
  - **Occlusion layers** → polígonos de oclusión (luces 2D).
- **Warning oficial**: "A source cannot belong to two TileSets at the same time. If the added source was attached to another **TileSet**, it will be removed from that one." → no compartir sources entre TileSets.

### Coordenadas: tile ↔ local ↔ global (con límite 16-bit)

- `local_to_map(local_position)` / `map_to_local(map_position)` (verificados): convierten entre posición en el mundo **local al TileMapLayer** y coordenada de celda `Vector2i`.
- **Límite oficial**: las coordenadas serializadas son **enteros de 16 bits: `-32768..32767`**; tiles fuera de rango **se envuelven** ("tiles outside this range are wrapped") — para mundos más grandes, chunks/offsets por nodo, no coords gigantes.

### Batching y `changed()` (performance oficial)

- **Todas las updates de TileMap se hacen en batch al final del frame** (verificado). Consecuencia: scene tiles se inicializan **después** de su padre; si necesitas el update YA: `update_internals()`.
- Señal `changed()`: "may be emitted **very often** when batch-modifying a **TileMapLayer**. Avoid executing complex processing… consider delaying it to the end of the frame (i.e. `call_deferred()`)" (nota oficial).

### Colisión de tiles (física por celda)

- Se pinta la **colisión en el editor** sobre tiles del atlas (physics layer del `TileSet`), no en el nodo.
- Por physics layer del `TileSet`: `collision_layer`/`collision_mask`/`collision_priority`/`PhysicsMaterial` (verificado: `set_physics_layer_collision_layer(layer_index, layer)` etc.).
- Por `TileMapLayer`: `collision_enabled` (default `true`) enciende/apaga la colisión de toda la capa.
- **Colisionador compuesto** (verificado en troubleshooting, Lote 4): en Godot **4.5+** el TileMapLayer crea colisión agrupada por **`physics_quadrant_size`** (default **16** tiles/eje) — "Larger values provide more reliable collision, at the cost of slower updates when the TileMap is changed". No es un collider por tile.

## API relevante

Todo verificado en Godot 4.7.

### `TileMapLayer` (hereda `Node2D`)

Props:

| Prop | Default | Nota |
|---|---|---|
| `tile_set` | — | El `TileSet` de la capa. |
| `tile_map_data` | `PackedByteArray()` | Los datos serializados (bajo nivel). |
| `enabled` | `true` | `false` = "disables this TileMapLayer completely (rendering, collision, navigation, scene tiles, etc.)" (verificado). |
| `collision_enabled` | `true` | Enciende la colisión de la capa. |
| `collision_visibility_mode` | `DEBUG_VISIBILITY_MODE_DEFAULT (0)` | `FORCE_SHOW (1)` / `FORCE_HIDE (2)`. |
| `navigation_enabled` | `true` | Regiones de navegación. |
| `navigation_visibility_mode` | `0` | Igual que arriba. |
| `occlusion_enabled` | `true` | Oclusión de luces. |
| `physics_quadrant_size` | `16` | Tamaño (tiles/eje) del colisionador compuesto (4.5+). |
| `rendering_quadrant_size` | `16` | Quadrante de render (culling/updates). |
| `y_sort_origin` | `0` | Y-sort desde la fila N (isométrico). |
| `x_draw_order_reversed` | `false` | Orden de dibujo en X. |
| `use_kinematic_bodies` | `false` | (Scene tiles: kinematic vs static). |

Métodos (los esenciales):

| Método | Nota |
|---|---|
| `set_cell(coords, source_id=-1, atlas_coords=Vector2i(-1,-1), alternative_tile=0)` | Pintar. `source_id=-1` → borra la celda. |
| `get_cell_tile_data(coords)` → `TileData` | Leer el tile completo. |
| `get_cell_source_id(coords)` / `get_cell_atlas_coords(coords)` / `get_cell_alternative_tile(coords)` | Leer por ID. |
| `erase_cell(coords)` / `clear()` | Borrar. |
| `get_used_cells()` / `get_used_cells_by_id(source_id=-1, atlas_coords, alternative_tile=-1)` / `get_used_rect()` | Iterar lo pintado. |
| `local_to_map(local_position)` / `map_to_local(map_position)` | Celda ↔ local. |
| `get_surrounding_cells(coords)` / `get_neighbor_cell(coords, neighbor)` | Vecinos (`CellNeighbor` 0–15). |
| `set_cells_terrain_connect(cells, terrain_set, terrain, ignore_empty_terrains=true)` / `set_cells_terrain_path(…)` | Terrain automático (batch de celdas). |
| `set_pattern(position, pattern)` / `get_pattern(coords_array)` / `map_pattern(…)` | Patrones (`TileMapPattern`). |
| `update_internals()` | Forzar update YA (el batch es a fin de frame). |
| `fix_invalid_tiles()` | Limpia tiles huérfanos. |
| `get_navigation_map()` / `set_navigation_map(map)` | RID de navegación. |

Señal: `changed()` — ver nota de performance arriba.

### `TileSet` (hereda `Resource`)

Props:

| Prop | Default | Nota |
|---|---|---|
| `tile_shape` | `TILE_SHAPE_SQUARE (0)` | `ISOMETRIC (1)` (diamond; Y-sort), `HALF_OFFSET_SQUARE (2)`, `HEXAGON (3)`. |
| `tile_layout` | `TILE_LAYOUT_STACKED (0)` | `STACKED_OFFSET (1)`, `STAIRS_RIGHT (2)`, `STAIRS_DOWN (3)`, `DIAMOND_RIGHT (4)`, `DIAMOND_DOWN (5)`. |
| `tile_offset_axis` | `TILE_OFFSET_AXIS_HORIZONTAL (0)` | `VERTICAL (1)`; solo shapes half-offset. |
| `tile_size` | `Vector2i(16, 16)` | "the minimal cell size required in an atlas" (verificado). |
| `uv_clipping` | `false` | Clipping de UV al renderizar. |

Métodos (los esenciales; los `set_*/get_*` por layer verifican el mismo patrón):

| Método | Nota |
|---|---|
| `add_source(source, atlas_source_id_override=-1)` → `int` (source id o -1) | Warning: no puede pertenecer a 2 TileSets. |
| `get_source(source_id)` / `get_source_count()` / `get_source_id(index)` / `get_next_source_id()` / `has_source(id)` / `remove_source(id)` / `set_source_id(id, new_id)` | Sources. |
| `add_physics_layer(to_position=-1)` / `remove_physics_layer(i)` / `move_physics_layer(i, pos)` / `get_physics_layers_count()` | Capas de colisión. |
| `set_physics_layer_collision_layer(layer_index, layer)` / `…_collision_mask(…, mask)` / `…_collision_priority(…, float)` / `…_physics_material(…, PhysicsMaterial)` | Config por capa física. |
| `add_navigation_layer` / `get_navigation_layer_layers` / `set_navigation_layer_layers` / `…` | Navegación. |
| `add_custom_data_layer(to_position=-1)` / `get_custom_data_layers_count()` / `get_custom_data_layer_type(i)` / `get_custom_data_layer_name(i)` / `get_custom_data_layer_by_name(name)` / `set_custom_data_layer_name(type)` / `remove_custom_data_layer(i)` | Datos custom. |
| `add_occlusion_layer` / `…` | Oclusión. |
| `add_terrain_set(to_position=-1)` / `add_terrain(terrain_set, to_position=-1)` / `get_terrain_sets_count()` / `get_terrains_count(set)` / `set_terrain_name/color` / `set_terrain_set_mode(set, TerrainMode)` | Terrenos (`TERRAIN_MODE_MATCH_CORNERS_AND_SIDES (0)` / `MATCH_CORNERS (1)` / `MATCH_SIDES (2)`). |
| `add_pattern(pattern, index=-1)` / `get_patterns_count()` / `remove_pattern(index)` | Patrones. |
| Proxies de tile (`map_tile_proxy`, `set_*_tile_proxy`, `has_*`) | Renombrar/remapear tiles en masa (migra assets). |

## Arquitectura recomendada

### Escena: capas separadas

```text
Level (Node2D)
├─ TileBG   (TileMapLayer)   # z 0  — fondo/decoración (sin physics layer pintado)
├─ TileMain (TileMapLayer)   # z 10 — suelo, paredes (physics layer pintado)
├─ TileFG   (TileMapLayer)   # z 20 — foreground/parallax (collision_enabled = false)
└─ Entities (Node2D)
```

1. **1 `TileSet` por familia de assets** (no un TileSet gigante para todo el juego; tampoco uno por tile).
2. **Physics layer solo donde hay colisión**: si la capa BG no colisiona, no pints física en ella (y `collision_enabled = false` si de todos modos no se usa).
3. **Custom data layer para "semántica"**: capa `int` "interactable" (0 = nada, 1 = puerta, 2 = cofre…) en vez de hardcodear celdas.
4. **El TileSet es un `.tres` versionado** (resource): se edita en el editor y se commitea.

### Decidir shape/layout

| Estilo | `tile_shape` | `tile_layout` | Nota |
|---|---|---|---|
| Square (90%) | `SQUARE (0)` | `STACKED (0)` | El default. |
| Isométrico | `ISOMETRIC (1)` | `DIAMOND_* (4/5)` | "works best if all sibling TileMapLayers and their parent have Y-sort enabled" (nota oficial). |
| Hex | `HEXAGON (3)` | `STAIRS_* (2/3)` | Con `tile_offset_axis`. |
| Half-offset square | `HALF_OFFSET_SQUARE (2)` | `STACKED_OFFSET (1)` | Con `tile_offset_axis`. |

## Implementación mínima

**Pintar y leer un tile en runtime** (todo API verificada):

```gdscript
extends TileMapLayer

@export var floor_source := 0
@export var floor_atlas := Vector2i(0, 0)

func _ready() -> void:
	# Pintar una celda (source, atlas coords):
	set_cell(Vector2i(3, 2), floor_source, floor_atlas)
	# Borrar:
	# set_cell(coords)  (source_id=-1 por default)

func tile_at(world_pos: Vector2) -> Dictionary:
	var cell := local_to_map(world_pos - global_position)
	var data: TileData = get_cell_tile_data(cell)
	if data.source_id == -1:
		return {}
	return {"cell": cell, "source": data.source_id,
			"atlas": data.atlas_coords, "alt": data.alt_id}
```

## Implementación recomendada

### 1. Generar el suelo de un level por datos (runtime)

```gdscript
# level_gen.gd — Node2D hijo de la raíz
@onready var ground: TileMapLayer = $Ground

func build_from_rows(rows: Array[Array[int]], origin := Vector2i.ZERO) -> void:
	for y in rows.size():
		for x in rows[y].size():
			var tile := rows[y][x]
			if tile > 0:
				ground.set_cell(origin + Vector2i(x, y), 0, Vector2i(tile - 1, 0))
	# update_internals() solo si necesitas leerlo YA en el mismo frame
```

### 2. Collectibles por custom data layer

```gdscript
# En el TileSet (editor): add_custom_data_layer, name="interactable", type=INT.
# Pintar en el editor: tile de puerta → interactable=1, cofre → 2.

func pickup_near(cell: Vector2i) -> int:
	var data: TileData = ground.get_cell_tile_data(cell)
	# TileData expone los custom data (verificado el mecanismo de layers;
	# el getter exacto de TileData.custom_data: API no verificada en esta revisión —
	# leer TileData en la docs antes de usar el nombre exacto)
	return data.custom_data["interactable"] if data.source_id != -1 else 0
```

### 3. Destruir suelo (puzzle/ataque)

```gdscript
func destroy(cell: Vector2i) -> void:
	ground.erase_cell(cell)          # o set_cell(cell) (source_id=-1)
	# La colisión del quadrante (physics_quadrant_size=16) se reconstruye;
	# los cambios son batch a fin de frame (o update_internals() para YA).
```

### 4. Iterar tiles usados (auditoría/save)

```gdscript
func save_cells() -> Array[Dictionary]:
	var out: Array[Dictionary] = []
	for cell in ground.get_used_cells():
		var d: TileData = ground.get_cell_tile_data(cell)
		out.append({"c": cell, "s": d.source_id, "a": d.atlas_coords, "t": d.alt_id})
	return out
# por id (solo los tiles de una especie):
# for cell in ground.get_used_cells_by_id(0, Vector2i(0, 0)):
```

### 5. Terrains automáticos (bordes de tierra/agua)

```gdscript
# En el TileSet: add_terrain_set + add_terrain; pintar los 16 tiles del atlas
# con las combinaciones de terreno; set_terrain_set_mode(set, TERRAIN_MODE_MATCH_CORNERS_AND_SIDES).
# En código, pincelar un rect y dejar que el motor conecte:
ground.set_cells_terrain_connect(cells_rect, terrain_set=0, terrain=1)
```

## Ejemplo práctico

**Juego**: platformer 2D de 3 levels, mundo 2000×1200 px, tile 16 px.

1. **1 `TileSet`** `.tres` (atlas 32×32 tiles, `tile_size=(16,16)`, `TILE_SHAPE_SQUARE`): physics layer 0 (suelo/paredes) + custom data `interactable` (INT) + navigation layer (opcional).
2. **3 `TileMapLayer`**: BG (decoración, `collision_enabled=false`), Main (colisión pintada), FG (foreground, `collision_enabled=false`, `z_index=20`).
3. **Player** (`godot-platformer-2d`) choca con el physics layer 0: su `collision_mask` = capa del tile (p. ej. capa 1 del physics layer; el layer del body = capa 2).
4. **Puertas/cofres**: custom data `interactable` → el script del level lee `get_cell_tile_data(cell).custom_data` al detectar al player cerca.
5. **Save**: `get_used_cells()` + IDs (receta #4) → `PackedByteArray`/JSON.
6. **Debug**: `collision_visibility_mode = FORCE_SHOW` para ver la colisión real pintada; `physics_quadrant_size` si el player "atraviesa" bordes en mapas grandes (subir a 32 y medir los updates).

## Integración

- **`godot-node2d`**: `TileMapLayer` es un `Node2D` (transform global/local; `local_to_map` es local al nodo).
- **`godot-physics`**: la colisión de tiles son `CollisionShape2D` agrupados (quadrantes); layers/masks son los del physics layer del `TileSet` (verificado: `set_physics_layer_collision_layer/mask/priority`).
- **`godot-physics-materials`**: el physics layer del `TileSet` admite `PhysicsMaterial` (`set_physics_layer_physics_material`) — fricción/rebote por tiles.
- **`godot-platformer-2d`**: el level del platformer; una-way platforms son `CollisionShape2D.one_way_collision` (ver esa skill), no tiles.

## Errores frecuentes

1. **Usar `TileMap` en 4.7 "porque es el nodo que todos conocen"** → **deprecado** (verificado); `TileMapLayer` por capa.
2. **API de `TileMap` 3.x en `TileMapLayer`** (índice de layer en `set_cell`, `get_cell_atlas_coords(coords, layer)`, …) → en `TileMapLayer` no hay índice de layer (la capa es el nodo).
3. **Compartir un source entre 2 TileSets** → warning oficial: se **quita** del otro TileSet.
4. **Mundo con coords > 32767 tiles** → wrap de coordenadas (nota oficial 16-bit) → chunks con offset, no coords gigantes.
5. **Esperar el update de tile YA** → el batch es a fin de frame (verificado) → `update_internals()` si es crítico.
6. **Lógica pesada en `changed()`** → "may be emitted very often" (nota oficial) → `call_deferred()` / cachear.
7. **`tile_size` no es el tamaño visual exacto en isométrico/hex** → es "the minimal cell size required in an atlas" (verificado); el tile renderizado puede ocupar más.
8. **Pintar física en capas que no colisionan** (y dejar `collision_enabled=true`) → pares de colisión innecesarios.
9. **`local_to_map` con coords globales** → es **local al TileMapLayer**; restar `global_position` (o convertir con `to_local` primero).
10. **Esperar que el TileSet "sepa" de las celdas pintadas** → el TileSet es la librería; las celdas viven en el `TileMapLayer` (`tile_map_data`).

## Anti-patrones

- **Un solo `TileMapLayer` gigante para todo** (fondo + colisión + foreground) → capas separadas (render order, `collision_enabled`, Y-sort por capa).
- **Un TileSet por level** (cuando el mismo atlas sirve) → 1 TileSet por familia de assets, compartido.
- **Re-pintar el level entero cada frame** ("sincronizar" el tilemap con el mundo) → el tilemap ES el mundo; mutaciones puntuales (`set_cell`/`erase_cell`).
- **`update_internals()` por celda en un bucle** → el batch de fin de frame ya agrupa; `update_internals()` es para "necesito el resultado YA".
- **Lógica de juego en tiles** (estado por celda en el tilemap) → custom data para el mínimo; el estado va en escenas/entities.
- **`physics_quadrant_size` gigante "para fiabilidad"** → "slower updates when the TileMap is changed" (oficial); medir.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El tilemap no es "gratis por ser tiles"**: cada celda pintada con física genera colisión (agrupada por quadrante) y cada celda renderiza.
- **Qué medir**: `Performance.PHYSICS_2D_ACTIVE_OBJECTS`/`PHYSICS_2D_COLLISION_PAIRS` (monitores 2D; **índices exactos no verificados en esta revisión**) y FPS con/without capas de física; `rendering_quadrant_size` afecta el render.
- **Optimizaciones (contra un cuello medido)**:
  1. `collision_enabled = false` en capas sin colisión (el mayor impacto).
  2. Physics layer pintado **solo donde hay colisión** (no el atlas entero).
  3. `physics_quadrant_size`/`rendering_quadrant_size` acordes al level (16 default; subir = colisión más fiable pero updates más lentos — oficial).
  4. Menos tiles usados (`get_used_rect()` para auditar) — chunks fuera de pantalla en niveles enormes.
  5. `enabled = false` en capas completas que no se usan (apaga render+colisión+navegación+scene tiles, verificado).
- **No**: "hay 50,000 tiles, seguro pesa" sin medir (la agrupación por quadrante reduce el pair count respecto a colliders individuales — la docs de troubleshooting lo da como beneficio del compuesto).

## Debugging

### "La colisión no aparece / no choca"

```text
1. ¿El tile tiene physics pintado? (colisión se pinta en el TileSet/atlas,
   no en el nodo)
2. collision_enabled de la capa (default true; ¿la desactivaste?)
3. Layers: el physics layer del TileSet (set_physics_layer_collision_layer)
   vs la collision_mask del body (o viceversa)
4. collision_visibility_mode = FORCE_SHOW → ver la colisión real
5. ¿El body usa un shape que no toca? (debug_color del shape del body)
```

### "Atraviesa bordes en mapas grandes / alta velocidad"

```text
1. physics_quadrant_size (16 default): subir (32) = más fiable, updates más
   lentos (oficial) — medir.
2. El body muy rápido: ver godot-physics (tunneling; en 2D, aumentar el
   safe_margin / floor_snap_length o el tick rate).
3. ¿Coords cerca del límite 16-bit? (wrap oficial — ver error #4)
```

### "El tile pintado no se ve / se ve en otra parte"

```text
1. set_cell con source_id=-1 (borra, no pinta) — el default de source_id es -1.
2. local vs global (error #9): local_to_map espera local al TileMapLayer.
3. update_internals() si leíste antes del batch de fin de frame.
4. fix_invalid_tiles() si el asset del atlas cambió (tiles huérfanos).
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tablas de `TileMapLayer` y `TileSet` (lista en *Referencias*).
- **3.x / `TileMap` → 4.x (trampas, verificado 4.7)**:
  - `TileMap` → **deprecado** (texto oficial 4.7); `TileMapLayer` por capa.
  - `TileMap.set_cell(coords, layer, source, atlas, alt)` → `TileMapLayer.set_cell(coords, source=-1, atlas=(-1,-1), alt=0)` (sin layer).
  - Capas de navegación/física: el `TileSet` gestiona los layers (el `TileMap` 3.x tenía layers propios).
  - `physics_quadrant_size`: 4.5+ (verificado en troubleshooting; el default 16 está en la prop 4.7).
- **4.0 ↔ 4.7**: `TileMapLayer` llega en 4.3 (la deprecación de `TileMap` es de esa era); la API consultada es la de 4.7.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-node2d` | Base (transform, local/global) |
| `godot-physics` | Layers/masks de la colisión de tiles |
| `godot-physics-materials` | `PhysicsMaterial` por physics layer del TileSet |
| `godot-platformer-2d` | Consumidor (level del platformer) |
| `godot-navigation2d` (pendiente) | Navigation layers del TileSet |
| `godot-light2d` (pendiente) | Occlusion layers |

## Skills relacionadas

- `godot-recipe-procedural-dungeon-2d` — pendiente (germen en *Implementación recomendada* #1/#4).
- `godot-recipe-terrain-autotile` — pendiente (germen #5).
- `godot-error-tilemap-no-collision` — pendiente (germen en *Debugging*).

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- TileMapLayer (deprecación de TileMap, props, métodos, `changed()`, 16-bit wrap, batching, `update_internals`, `physics_quadrant_size`): https://docs.godotengine.org/en/stable/classes/class_tilemaplayer.html
- TileSet (props, sources, capas physics/navigation/custom data/occlusion, enums `TileShape`/`TileLayout`/`TileOffsetAxis`/`CellNeighbor`/`TerrainMode`, warning de sources, `tile_size`): https://docs.godotengine.org/en/stable/classes/class_tileset.html
- Troubleshooting physics issues (colisionador compuesto de tiles, `Physics Quadrant Size` 16, 4.5+): https://docs.godotengine.org/en/stable/tutorials/physics/troubleshooting_physics_issues.html
- Using Tilemaps (tutorial; listado en la página de clase): https://docs.godotengine.org/en/stable/tutorials/2d/using_tilemaps.html
