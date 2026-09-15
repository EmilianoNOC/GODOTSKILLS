# godot-lod

## ID

`godot-lod`

## Categoría

3D · Performance · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la class reference oficial 4.7 (`docs.godotengine.org/en/stable/`): `GeometryInstance3D.xml`, `Mesh.xml`, `ArrayMesh.xml` (rama `4.7-stable` de `godotengine/godot`) y `tutorials/performance/optimizing_3d_performance.html`.

## Confidence

**HIGH** para la API verificada (visibility ranges, `lod_bias`, `lods` de `ArrayMesh`, `cast_shadow`, `VisibleOnScreen*3D`).
**"No existe (verificado)"**: **no hay nodo `LOD` nativo en el core 4.7** (verificado: no existe `LOD.xml`, ni `class_lod.html`, ni `scene/3d/lod.cpp` en la rama 4.7-stable). Todo lo que un tutorial llame "LOD node" es un add-on de AssetLib.
**"Custom implementation using Godot primitives"**: los scripts de distancia (ej. `cast_shadow` por distancia) son implementación propia con primitivas nativas.

## Nivel

**BASE** (sin prerrequisitos; se integra con `godot-rendering-performance` para medir).

## Propósito

Proporcionar el conocimiento operativo completo de **Level of Detail** en Godot 4.x: las tres estrategias oficiales (LOD de mesh en import, *visibility ranges* en el nodo, *distance fade* en decals/luces), cómo funcionan, cuándo se combinan, y qué **no** existe (nodo `LOD` core) para no inventarlo.

## Cuándo utilizarla

- Muchos objetos repetidos (árboles, rocas, props) que a distancia no necesitan detalle.
- Reducir triángulos/draw calls en mundos grandes sin que el jugador lo note.
- Controlar qué se renderiza (o qué calidad) según la distancia a la cámara/jugador.
- Apagar sombras de objetos lejanos (el mayor ahorro habitual).

## Cuándo NO utilizarla

- **Pocos objetos en pantalla** (menos de ~50–100 con detalle) → el LOD no aporta (mínima complejidad primero); medir antes de añadirlo.
- **El jugador está siempre cerca del objeto** (vehículo propio, HUD 3D) → no hay distancia que aproveche el LOD.
- **El cuello de botella no es geometría/visibilidad** (es CPU de scripts, o fill rate de transparencias) → el LOD no lo resuelve (ver `godot-rendering-performance` para diagnosticar primero).

## Conceptos fundamentales

### Las tres estrategias oficiales (verificadas en `optimizing_3d_performance.html`)

> "Godot 4 offers several ways to control level of detail:"

| Estrategia | Dónde | Qué hace |
|---|---|---|
| **1. Mesh LOD en import** | Asset (import del mesh) | Versiones de baja poligonía del mesh, por distancia. El motor elige la versión. (Ref. oficial: `doc_mesh_lod`.) |
| **2. Visibility ranges** | `GeometryInstance3D` (nodo) | Ocultar (o fundir) el objeto según distancia a la cámara. El patrón "impostor" (billboard a lo lejos). |
| **3. Distance Fade** | `Decal3D` / `Light3D` (y sombras) | Las decals y luces "desfaden" con la distancia (menos coste a lo lejos). |

> "While they can be used independently, these approaches are most effective when used together." (docs)

### LOD vs culling vs impostor

- **LOD** = cambiar la **calidad** (menos triángulos, menos materiales) según distancia. El objeto sigue siendo "el mismo".
- **Culling** = dejar de renderizar **lo que no se ve** (frustum/occlusión) o **lo que está lejos** (visibility range).
- **Impostor/billboard** = a distancia, reemplazar el objeto por algo **muy barato** (un quad con textura pre-renderizada). Se monta con *visibility ranges* (ver abajo).

### Por qué el LOD importa (el coste que reduce)

- **Triángulos** (primitivas): en mobile es relevante (ver `godot-rendering-performance`); en desktop menos (vertex cost bajo).
- **Draw calls**: menos materiales/submallas por distancia = menos state changes (la palanca principal en desktop).
- **Sombras**: `cast_shadow = OFF` en objetos lejanos ahorra el render de la shadow map (el ahorro más habitual y medible).
- **Animación/skinning**: reducir o pausar la animación de objetos lejanos (el skinning es caro en algunas plataformas, verificado).

## API relevante

Todo verificado en Godot 4.7.

### `GeometryInstance3D` (visibility ranges + LOD)

| API | Tipo / default | Nota |
|---|---|---|
| `visibility_range_begin` | float (0) | A partir de esta distancia **deja de ser visible** (0 = deshabilitado). |
| `visibility_range_end` | float (0) | Más allá de esta distancia **deja de ser visible** (0 = deshabilitado). El patrón habitual: `end` (ocultar lejos). |
| `visibility_range_begin_margin` | float (0) | Hysteresis/fade de entrada. |
| `visibility_range_end_margin` | float (0) | Hysteresis/fade de salida. **Banda muerta**: evita flip-flop en el borde. |
| `visibility_range_fade_mode` | enum | `VISIBILITY_RANGE_FADE_DISABLED(0)` (hysteresis, más rápido) / `FADE_SELF(1)` (fade solo este) / `FADE_DEPENDENCIES(2)` (fade con dependencias) — **los dos fade, solo Forward+** (verificado). |
| `visibility_parent` | bool | Si la visibilidad la controla un ancestro (jerarquías HLOD: un nodo controla a sus hijos). |
| `lod_bias` | float (1.0) | Sesgo del LOD de mesh: `0` = forzar el LOD más bajo, `1` = default, `>1` = mantener LODs altos más lejos. "Útil para probar transiciones de LOD" (verificado). |
| `cast_shadow` | enum | `SHADOW_CASTING_SETTING_OFF` para apagar sombras (el ahorro de LOD más habitual). |
| `gi_mode` | enum | `GI_MODE_DISABLED(0)` para apagar GI en el objeto (ej. personajes, props lejanos). |

### Mesh LOD por superficie (`ArrayMesh`)

| API | Nota |
|---|---|
| `ArrayMesh.add_surface_from_arrays(primitive, arrays, blend_shapes = [], lods = {}, flags = 0)` | `lods`: `Dictionary` de `{float (coef. de distancia) → PackedInt32Array (índices de ese LOD)}`. **LOD por superficie** del mesh (verificado). |
| `Mesh.surface_get_lods(surf_idx)` | Leer el dict de LODs de una superficie. |

> **No existe `Mesh.lods` como propiedad** (verificado en 4.7). El LOD de mesh va **por superficie** (dict de `add_surface_from_arrays`) o se configura en el **import** (estrategia 1, `doc_mesh_lod`).

### `VisibleOnScreen*` (controlar por visibilidad en pantalla)

| Nodo | Señales/métodos | Uso |
|---|---|---|
| `VisibleOnScreenNotifier3D` | `body_entered`/`body_exited`/`area_entered`… + `is_on_screen()` | Detectar que el objeto **se ve** (para activar desactivar lógica/animación). |
| `VisibleOnScreenEnabler3D` | Activa/desactiva sus hijos según visibilidad | Pausar animación/lógica de lo que no se ve (verificado en el tutorial de 3D performance: "reduce the animation rate for distant or occluded meshes, or pause the animation entirely"). |

### No existe (verificado 4.7)

| API | Estado | Alternativa |
|---|---|---|
| **Nodo `LOD`** (core) | **No existe** en 4.7 (no hay `LOD.xml`/`class_lod.html`/`scene/3d/lod.cpp`) | Las 3 estrategias de arriba; o un add-on de AssetLib (marcarlo como tal). |
| `Mesh.lods` (prop) | No existe (verificado) | `ArrayMesh.add_surface_from_arrays(lods=...)` por superficie. |
| "LOD automático" sin configurar | No hay (verificado) | Se configura (import o visibility ranges); no es automático. |

## Arquitectura recomendada

### Jerarquía típica (props del mundo)

```text
World
└── Props (Node3D)  [visibility_parent = false — el propio nodo controla]
    ├── Tree_A (MeshInstance3D)   visibility_range_end = 80, end_margin = 10,
    │                              fade_mode = FADE_DEPENDENCIES (Forward+)
    ├── Tree_B (MeshInstance3D)   idem
    └── …
```

Decisiones:

1. **`visibility_range_end` + `end_margin` (hysteresis)** en cada prop: oculta lejos, con banda muerta (evita flip-flop). `FADE_DISABLED` si no se puede usar Forward+ (los fade son solo Forward+, verificado).
2. **Mesh LOD en el import** para los assets que lo necesitan (árboles, rocas): versiones 100%/50%/25%. El motor las elige; `lod_bias` para sesgar por escena.
3. **`cast_shadow = OFF` por distancia** para los props lejanos (el ahorro más medible) — script *custom* (abajo) o dejarlo si el mesh LOD ya reduce el coste de sombra.
4. **`VisibleOnScreenEnabler3D`** en NPC/props con animación: pausar la animación cuando no se ven (verificado).
5. **Impostor/billboard** para densidades altas (bosque): un `MeshInstance3D` de quad con la textura pre-renderizada, `visibility_range_begin` (aparece lejos) mientras el árbol real tiene `visibility_range_end` (desaparece lejos). El cruce es el "impostor" (verificado como patrón oficial).

## Implementación mínima

Configurar visibility range **una vez** (editor o `_ready`) — sin script por frame:

```gdscript
# config_lod.gd — Godot 4.7 (APIs verificadas)
extends Node
## Configurar visibility ranges de un conjunto de props UNA VEZ.
## (El motor hace el culling por distancia sin coste de script por frame.)

@export var props: Array[MeshInstance3D]     # arrastrar los props en el editor
@export var end_distance := 80.0
@export var hysteresis := 10.0
@export var fade := true                      # requiere Forward+

func _ready() -> void:
	for p in props:
		p.visibility_range_end = end_distance
		p.visibility_range_end_margin = hysteresis
		p.visibility_range_fade_mode = (
			GeometryInstance3D.VISIBILITY_RANGE_FADE_DEPENDENCIES
			if fade else GeometryInstance3D.VISIBILITY_RANGE_FADE_DISABLED
		)
		# El ahorro más habitual: apagar sombras a distancia se hace con el script
		# de *Implementación recomendada* (custom), no con visibility range.
```

## Implementación recomendada

### 1. `cast_shadow` por distancia (custom, con hysteresis)

```gdscript
# shadow_lod.gd — Godot 4.7 (custom implementation using Godot primitives)
extends MeshInstance3D
## Apagar sombras más allá de una distancia (el ahorro de LOD más medible).
## Hysteresis para no flip-flopear en el borde.

@export var shadow_off_distance := 40.0
@export var hysteresis := 8.0

@export var target: Node3D                    # la cámara o el jugador

func _process(_delta: float) -> void:
	if target == null:
		return
	var d := global_position.distance_to(target.global_position)
	match cast_shadow:
		GeometryInstance3D.SHADOW_CASTING_SETTING_ON:
			if d > shadow_off_distance + hysteresis:
				cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_OFF
		GeometryInstance3D.SHADOW_CASTING_SETTING_OFF:
			if d < shadow_off_distance - hysteresis:
				cast_shadow = GeometryInstance3D.SHADOW_CASTING_SETTING_ON
```

> Preferir el mecanismo **nativo** (`visibility_range_*`) cuando cubra el caso (sin script por frame). El script de arriba solo para lo que el nativo no hace (apagar sombras, no ocultar).

### 2. Impostor / billboard (bosque)

```text
Tree (MeshInstance3D)        visibility_range_end = 30   (desaparece a 30 m)
Impostor (QuadMesh)          visibility_range_begin = 30 (aparece a 30 m)
                             visibility_range_end   = 200
                             material: billboard (render_mode billboard) con la
                             textura pre-renderizada del árbol
```

El cruce a 30 m es el "impostor" (verificado como patrón oficial en `optimizing_3d_performance.html`). Coste: 1 quad (4 triángulos) en vez del árbol (miles).

### 3. Pausar animación de lo oculto (verificado)

```text
NPC (CharacterBody3D)
└── VisibleOnScreenEnabler3D     (envuelve a lo que debe pausarse al no verse)
    ├── AnimationPlayer / AnimationTree
    └── …
```

"reduce the animation rate for distant or occluded meshes, or pause the animation entirely if the player is unlikely to notice" (docs, verificado).

### 4. Jerarquía HLOD con `visibility_parent`

```gdscript
# Un nodo "contenedor" controla la visibilidad de todos sus hijos (HLOD):
$Chunk.visibility_parent = false   # el contenedor decide
# (los hijos con visibility_parent = true heredan la decisión del ancestro)
```

Útil para cargar/descargar chunks de mundo: ocultar el `Chunk` oculta a todos sus props de una vez (verificado: `visibility_parent` en `GeometryInstance3D`).

## Ejemplo práctico

**Juego**: mundo abierto con 500 árboles + 200 rocas + 30 NPC.

1. **Baseline**: FPS 96 (target 144), 2400 draw calls, 1.8M tris. (Medido, regla §60.)
2. **Mesh LOD en import**: árboles 100%/50%/25% (distancias 0/30/60) → **medir**: 900k tris.
3. **Visibility range**: árboles/rocas `end = 80`, `end_margin = 10`, `FADE_DEPENDENCIES` (Forward+) → **medir**: 900 draw calls (los que no se ven, no se renderizan).
4. **`cast_shadow OFF` por distancia** (script, 40 m + hysteresis) → **medir**: shadow map reducido, FPS sube.
5. **NPC**: `VisibleOnScreenEnabler3D` → la animación de los 20 NPC se pausa cuando no se ven.
6. **Resultado**: FPS 142, draw calls 700, tris 500k. **Cada paso se midió** (antes/después) — la regla cumplida.

## Integración

- **`godot-rendering-performance`**: aquí se miden los ahorros (draw calls, tris, VRAM, FPS).
- **`godot-shader-spatial`**: el impostor usa un material (billboard); el fade de material es `FADE_*`.
- **`godot-character*`**: los NPC usan `VisibleOnScreenEnabler3D` + (opcional) `cast_shadow` por distancia al jugador.
- **Mundo por chunks**: `visibility_parent` (HLOD) para ocultar/cargar chunks.

## Errores frecuentes

1. **Buscar un nodo "LOD" nativo** → **no existe en el core 4.7** (verificado). Si un tutorial lo muestra, es un add-on de AssetLib (marcarlo como tal; no es del motor).
2. **`visible = distance < X` por frame sin hysteresis** → flip-flop visual en el borde (parpadeo) + pico de CPU. → `visibility_range_*` nativo (con margen) o hysteresis (arriba).
3. **`visibility_range_end` sin `end_margin`** → flip-flop en el borde (el objeto aparece/desaparece al cruzar). El margen es la banda muerta.
4. **`FADE_SELF`/`FADE_DEPENDENCIES` en Mobile/Compatibility** → los fade son **solo Forward+** (verificado); en Compatibility usar `FADE_DISABLED` (hysteresis, sin fundido).
5. **Esperar que el LOD sea "automático"** → no hay LOD automático (verificado): se configura en el import (mesh) o en el nodo (visibility range).
6. **`Mesh.lods` como propiedad** → no existe (verificado); el LOD de mesh es por superficie (`add_surface_from_arrays(lods=...)`) o en el import.
7. **Aplicar `lod_bias` "para mejorar" sin entenderlo** → `0` fuerza el LOD más bajo (¡más barato, menos detalle!), `>1` mantiene LODs altos más lejos (¡más caro!). Usar para **probar** transiciones (docs), no como dial de calidad.
8. **Impostor sin cruce limpio** (el árbol y el quad visibles a la vez) → los `visibility_range_end` (árbol) y `begin` (impostor) deben coincidir (con margen).
9. **LOD en un objeto que siempre está cerca** (vehículo propio, HUD 3D) → no hay distancia que aproveche; el LOD no aporta (y añade complejidad).
10. **Pausar animación de lo oculto con `set_process(false)` a mano** → `VisibleOnScreenEnabler3D` hace exactamente eso de forma nativa (verificado); no reinventarlo.

## Anti-patrones

- **Un script de distancia por prop × 500 props** (cada uno con `_process` midiendo distancia) → el coste de 500 `_process` por frame supera el ahorro. Usar `visibility_range_*` nativo (sin script) o un **único** script que itere (con hysteresis) solo para lo que el nativo no cubre (sombras).
- **LOD para "ahorrar" en un objeto que ocupa toda la pantalla** → no hay distancia; el LOD no lo reduce (el LOD actúa por distancia, no por tamaño en pantalla).
- **5 niveles de mesh LOD** (100/80/60/40/20%) → la diferencia entre niveles adyacentes es invisible; 3 niveles (100/50/25) cubren el caso. Menos niveles = menos memoria + transiciones más limpias.
- **Impostor que "se ve impostor"** (quad que parpadea/brilla) → la textura del impostor debe ser coherente con el lighting (o usar un material que la iguale); medir el cruce a ojo y a distancia real.
- **`cast_shadow OFF` en el jugador/objetos cercanos** → las sombras cercas son las que se notan; apagarlas solo en lo **lejos** (con hysteresis).
- **Optimizar el LOD sin medir** (la violación de la regla) → medir draw calls/tris/FPS antes/después (ver `godot-rendering-performance`).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Coste del LOD nativo (visibility ranges)**: **cero de script** — lo resuelve el motor en el culling (el ahorro es el de no renderizar). Es la opción preferida cuando cubre el caso.
- **Coste del LOD por script** (cast_shadow por distancia, HLOD manual): 1 operación de distancia por objeto por frame — con **500 props** ya suma; preferir el nativo.
- **Ahorro típico (medir, no asumir)**:
  - Mesh LOD: triángulos (primitivas) — en mobile, relevante; en desktop, menos.
  - Visibility range: draw calls + tris (lo que no se ve, no se renderiza).
  - `cast_shadow OFF` por distancia: el render de la shadow map (a menudo el mayor ahorro medible).
  - `VisibleOnScreenEnabler3D`: CPU de animación de lo oculto.
- **Medición**: `RENDER_TOTAL_DRAW_CALLS_IN_FRAME`, `RENDER_TOTAL_PRIMITIVES_IN_FRAME`, `RENDER_TOTAL_OBJECTS_IN_FRAME`, `TIME_FPS` (ver `godot-rendering-performance`).

## Debugging

### "El objeto parpadea a cierta distancia"

```text
1. ¿Falta el margen (hysteresis)? → visibility_range_end_margin > 0 (error #3)
2. ¿visible = distancia < X por frame sin hysteresis? → error #2
3. ¿Impostor y original con rangos que se solapan? → el cruce debe ser limpio (error #8)
4. ¿Fade en Compatibility? → FADE_* solo Forward+ (error #4); usar FADE_DISABLED
```

### "El LOD no se aplica / no ahorra"

```text
1. ¿Se midió? (draw calls/tris antes/después) → sin medición, no hay "no ahorra"
2. ¿El objeto está dentro de visibility_range? → print(visibility_range_end)
3. ¿Mesh LOD configurado en el import? (verificar las versiones en el import)
4. ¿El bottleneck es otro? (CPU/fill rate) → el LOD no lo resuelve (ver
   godot-rendering-performance)
5. ¿visibility_parent = true en un ancestro que lo oculta/visible? → la jerarquía
   HLOD puede anular el range propio.
```

### "Las sombras de lejos siguen costando"

```text
1. ¿cast_shadow = OFF en lo lejos? (el script de *Implementación recomendada* #1)
2. ¿El shadow map es grande? → reducir el tamaño (ver godot-rendering-performance)
3. ¿Muchas luces proyectando? → apagar sombras en luces/objetos que no las
   necesitan (verificado en gpu_optimization)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — `GeometryInstance3D.xml` (visibility ranges, lod_bias, cast_shadow, gi_mode, visibility_parent), `ArrayMesh.xml` (lods en `add_surface_from_arrays`), `VisibleOnScreen*3D`, y `optimizing_3d_performance.html` (las 3 estrategias + impostor + instancing).
- **Nodo `LOD` core: no existe en 4.7** (verificado; ni en 4.0 — la ausencia es estable en la serie 4.x).
- **Mesh LOD por superficie** (`lods` en `add_surface_from_arrays`): verificado en 4.7; en 4.x anteriores, verificar por versión antes de usarlo (la estrategia de **import** es la estable).
- **`FADE_SELF`/`FADE_DEPENDENCIES`: solo Forward+** (verificado) — en Mobile/Compatibility, `FADE_DISABLED`.
- **3 → 4**: en Godot 3.x no había `visibility_range_*` con los mismos nombres (el LOD de 3.x era por `LOD` node de add-on / `visibility_range` en GeometryInstance con sintaxis distinta). No portar código 3.x tal cual.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-rendering-performance` | **Requerida** para medir los ahorros |
| `godot-shader-spatial` | El impostor usa un material (billboard) |
| `godot-third-person-character` | Compuesta que integra esta skill (Lote 0) |
| `godot-physics` | Si el LOD afecta a colisión (normalmente no: el LOD es visual) — disponible (Lote 4) |

## Skills relacionadas

- `godot-recipe-forest-lost` — bosque con impostors (pendiente; germen en *Implementación recomendada* #2)
- `godot-recipe-hlod-chunks` — mundo por chunks con `visibility_parent` (pendiente)
- `godot-error-lod-flicker` — diagnóstico de parpadeo (pendiente)
- `godot-multimesh` — instancing para densidades altas (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- GeometryInstance3D (visibility_range_*, lod_bias, cast_shadow, gi_mode, visibility_parent): https://docs.godotengine.org/en/stable/classes/class_geometryinstance3d.html
- ArrayMesh (add_surface_from_arrays con lods): https://docs.godotengine.org/en/stable/classes/class_arraymesh.html
- Mesh (surface_get_lods; sin prop `lods`): https://docs.godotengine.org/en/stable/classes/class_mesh.html
- Optimizing 3D performance (las 3 estrategias de LOD, impostor, instancing, skinning, mundos grandes): https://docs.godotengine.org/en/stable/tutorials/performance/optimizing_3d_performance.html
- VisibleOnScreenNotifier3D: https://docs.godotengine.org/en/stable/classes/class_visibleonscreennotifier3d.html
- VisibleOnScreenEnabler3D: https://docs.godotengine.org/en/stable/classes/class_visibleonscreenenabler3d.html
- Páginas referenciadas por el tutorial de arriba (buscar por nombre en el índice de la docs si la URL cambia entre versiones): *Mesh LOD* (`doc_mesh_lod`), *Visibility Ranges* (`doc_visibility_ranges`).
