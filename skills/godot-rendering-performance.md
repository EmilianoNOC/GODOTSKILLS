# godot-rendering-performance

## ID

`godot-rendering-performance`

## Categoría

Performance · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: `class_performance.html` (monitores), `tutorials/performance/gpu_optimization.html`, `tutorials/performance/optimizing_3d_performance.html` (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** para los monitores `Performance` (API verificada 4.7) y para las directrices de GPU/3D (tutoriales oficiales 4.7, sintetizados).
**MEDIUM** para los valores de "cuánto es mucho" (nº de draw calls, VRAM) — orientativos; la decisión se toma con medición.
**"No verificado"**: nombres exactos de los *Project Settings* de `rendering/*` (MSAA/TAA/scale/shadows) — la UI existe (Project Settings → Rendering), pero esta skill **no afirma rutas de setting sin verificar**; ver abajo.

## Nivel

**BASE** (sin prerrequisitos; es la skill de "medir y diagnosticar").

## Propósito

Proporcionar al agente el **método** de optimización de rendering en Godot 4.x: medir con los monitores oficiales (`Performance`), identificar el bottleneck (CPU / GPU / VRAM / draw calls), y aplicar las optimizaciones que documenta la docs oficial (materiales, batching, instancing, LOD, sombras, transparencias, texturas, culling) — **siempre midiendo antes y después**.

## Cuándo utilizarla

- "El juego va lento" → diagnosticar qué sube (FPS, process, physics, draw calls, VRAM).
- Antes y después de cualquier cambio de rendering (shader, LOD, sombras, materiales).
- Escalar un mundo (N objetos, N luces, N materiales) sin perder FPS.
- Elegir la estrategia de optimización correcta según la plataforma (desktop vs mobile).

## Cuándo NO utilizarla

- **Bottleneck de CPU en lógica de juego** (no rendering) → profiler de GDScript (aunque el cuadro de diagnóstico de esta skill incluye el split CPU/GPU).
- **Física pura** (pares de colisión, cuerpos) → `godot-physics` (la sección de monitores de física de esta skill sirve para detectar, no para resolver).
- **Optimización de assets** (reducir polígonos/texturas en el editor) → es diseño de asset; esta skill indica **qué** medir y **qué** ajuste de engine aplica.

## Conceptos fundamentales

### La regla (spec §60)

**MEASURE → IDENTIFY BOTTLENECK → OPTIMIZE → MEASURE AGAIN.** Nunca optimizar "porque parece más rápido" o "porque en otro juego funcionó". Cada cambio de rendering se mide antes y después con los monitores oficiales.

### Los tres mundos del bottleneck

| Mundo | Síntoma medido | Herramienta |
|---|---|---|
| **CPU (lógica)** | `TIME_PROCESS` alto | Profiler de GDScript (top funciones) |
| **CPU (física)** | `TIME_PHYSICS_PROCESS` alto | `PHYSICS_3D_*` (pares, objetos) |
| **GPU** | `TIME_FPS` bajo + process/physics bajos | `RENDER_TOTAL_*` (draw calls, primitivas, VRAM) + truco de ventana |

### Draw calls vs primitivas vs VRAM (GPU)

- **Draw calls**: coste de "enviar instrucciones" a la GPU + **state changes**. Reducir **materiales distintos** (el renderer agrupa por material/shader) es la palanca principal. (Verificado: "The fewer different materials in the scene, the faster the rendering will be".)
- **Primitivas** (triángulos): coste de vértice. En desktop es bajo (verificado: "vertex cost is low"); en mobile importa más el **fill rate** (píxeles) que el vértice.
- **VRAM**: texturas. La compresión de VRAM reduce **bandwidth** (no solo tamaño). (Verificado.)

### Fill rate (el enemigo en mobile y en 4K)

- **Overdraw**: pintar el mismo píxel varias veces (transparencias, partículas, outlines). En mobile (tile-based) es el coste dominante. (Verificado: "avoid concentration of vertices in small parts of the screen", "transparency is particularly expensive where multiple transparent objects overlap".)
- **Truco de detección (verificado)**: V-Sync off, comparar FPS con ventana grande vs ventana muy pequeña. Si el FPS sube bastante con ventana pequeña → **fill rate limited**.

### Instancing (agrupar draw calls)

- **Automático** (verificado): `MeshInstance3D` con **mismo mesh + mismo material** se instancian solos. **Solo Forward+** (no Mobile/Compatibility). Requiere material **opaco o alpha-test** (no alpha-blend ni depth prepass).
- **Manual**: `MultiMesh` (miles de objetos idénticos: hierba, enjambres, partículas). (Verificado: "ideal for flocks, grass, particles".)

### Culling (no renderizar lo que no se ve)

- **Frustum** (automático, verificado): Godot recorta lo fuera del viewport.
- **Oclusión** (manual con occluders, verificado): `OccluderInstance3D` — "Godot offers an approach to occlusion culling using occluder nodes".
- **Visibilidad por distancia**: `GeometryInstance3D.visibility_range_*` (ver `godot-lod`).
- **Culling de capas**: `Camera3D.cull_mask`.

## API relevante

Todo verificado en Godot 4.7.

### Monitores `Performance` (los que importan para rendering)

| Monitor | Valor | Qué mide |
|---|---|---|
| `TIME_FPS` | 0 | FPS (actualización ~1×/s). |
| `TIME_PROCESS` | 1 | s de `_process` por frame. |
| `TIME_PHYSICS_PROCESS` | 2 | s de `_physics_process` por frame. |
| `OBJECT_NODE_COUNT` | 9 | Nº de nodos. |
| `OBJECT_ORPHAN_NODE_COUNT` | 10 | Nodos huérfanos (**debug only**). |
| `RENDER_TOTAL_OBJECTS_IN_FRAME` | 11 | Objetos renderizados en el frame. |
| `RENDER_TOTAL_PRIMITIVES_IN_FRAME` | 12 | Triángulos en el frame. |
| `RENDER_TOTAL_DRAW_CALLS_IN_FRAME` | 13 | Draw calls en el frame. |
| `RENDER_VIDEO_MEM_USED` | 14 | VRAM (bytes). |
| `RENDER_TEXTURE_MEM_USED` | 15 | VRAM de texturas (bytes). |
| `RENDER_BUFFER_MEM_USED` | 16 | VRAM de buffers (bytes). |
| `PHYSICS_3D_ACTIVE_OBJECTS` | 20 | Objetos físicos 3D activos. |
| `PHYSICS_3D_COLLISION_PAIRS` | 21 | Pares de colisión 3D. |
| `PIPELINE_COMPILATIONS_*` | 34–38 | Compilación de pipelines (stutters al cargar). |

Notas oficiales (verificadas):

- `get_monitor(monitor)` actualiza **~1 vez por segundo** (y algunos monitores pueden tener hasta 1 s de delay).
- **Monitores de debug** (p. ej. `OBJECT_ORPHAN_NODE_COUNT`) **devuelven 0 en release** — no son válidos en build.
- **Custom monitors**: `Performance.add_custom_monitor(id, callable)` — aparecen en la pestaña Monitor del editor. El callable no toma argumentos y devuelve float.

```gdscript
# Godot 4.7 (APIs verificadas):
print("FPS: ", Performance.get_monitor(Performance.TIME_FPS))
print("ms/frame: ", Performance.get_monitor(Performance.TIME_PROCESS) * 1000.0)
print("draw calls: ", Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME))
print("primitivas: ", Performance.get_monitor(Performance.RENDER_TOTAL_PRIMITIVES_IN_FRAME))
print("objetos render: ", Performance.get_monitor(Performance.RENDER_TOTAL_OBJECTS_IN_FRAME))
print("VRAM: ", Performance.get_monitor(Performance.RENDER_VIDEO_MEM_USED) / 1048576.0, " MB")
```

### Herramientas del editor (oficiales)

- **Debugger → Monitor**: los mismos monitores que la API `Performance` (incluye FPS, tiempos, draw calls, VRAM, physics 3D).
- **Debugger → Profiler**: tiempo por función de GDScript (top callers).
- **3D viewport → "visible collisions"** (gizmos) para ver colisiones.
- **RenderingServer.frame_drawn / frame_post_draw** (señales) para instrumentar en torno al render.
- **`Engine.get_frames_per_second()`** para FPS en código.

## Arquitectura recomendada

### Flujo de diagnóstico (el "método")

```text
1. REPRODUCIR: la escena real, en la plataforma real (no el editor si el target difiere).
2. MEDIR baseline: TIME_FPS, TIME_PROCESS, TIME_PHYSICS_PROCESS,
   RENDER_TOTAL_DRAW_CALLS_IN_FRAME, RENDER_TOTAL_PRIMITIVES_IN_FRAME, RENDER_VIDEO_MEM_USED.
3. CLASSIFICAR:
   - TIME_PROCESS alto            → CPU lógica (Profiler)
   - TIME_PHYSICS_PROCESS alto    → física (PHYSICS_3D_*)
   - Ambos bajos y FPS bajo       → GPU (RENDER_TOTAL_* + truco de ventana)
4. OPTIMIZAR UNA COSA A LA VEZ (ver el menú de palancas de abajo).
5. MEDIR DE NUEVO: comparar los mismos monitores. Si no mejora, deshacer y probar la siguiente.
```

### Menú de palancas (qué tocar según el síntoma, todas verificadas)

| Síntoma medido | Palancas (de la docs oficial) |
|---|---|
| **Draw calls altos** | 1) Reutilizar materiales (el renderer agrupa por material; "20,000 objetos con 100 materiales es mucho más rápido que 20,000 materiales"). 2) Instancing automático (Forward+, mismo mesh+material, opaco/alpha-test). 3) `MultiMesh` para miles de idénticos. 4) Batching **pre-join** de meshes estáticos (arriba en el editor, no en runtime: "join meshes ahead of time"). |
| **Primitivas altas (triángulos)** | LOD (mesh LOD + `visibility_range_*`, ver `godot-lod`). En mobile: evitar concentración de vértices en zonas pequeñas de pantalla. |
| **VRAM alta** | Compresión de VRAM en import (reduce bandwidth, verificado). Mipmaps. Atlases (reducir nº de texturas). Tamaños de textura acordes al uso. |
| **Fill rate / overdraw** | Reducir transparencias (el overdraw es el enemigo en mobile y 4K; "try to use as few transparent objects as possible"). Simplificar shaders (apagar opciones caras de `StandardMaterial3D`). `vertex_lighting` en partículas en móvil. Variable rate shading (VRS) en hardware que lo soporte. |
| **Sombras caras** | **Reducir tamaño de shadowmap** ("reducing the size of shadowmaps can increase performance"). Apagar sombras en luces/objetos que no las necesitan ("turn shadows off for as many lights and objects as possible"). Bake lighting (lightmaps) — especialmente en mobile; lights `bake_mode = Static` para omnis/spots (mantener `Dynamic` en `DirectionalLight3D`). |
| **Post-proceso caro** | Medir el impacto por efecto (bloom, SSAO, SSR). En tile-based (mobile): "Be very careful to test the performance of shaders, viewport textures and post processing". |
| **Mundo grande** | Occlusion culling (occluders). Tile-loading (mundo en piezas que se cargan a demanda). Coordenadas de mundo grande para errores de float (o orientar el mundo alrededor del jugador / shift de origen periódico). |
| **Instancias dinámicas (skinning/morph)** | Reducir polígonos de modelos animados; limitar nº en pantalla; reducir tasa de animación o pausar si el jugador no lo nota (`VisibleOnScreenNotifier3D`/`VisibleOnScreenEnabler3D`, verificados). |

### Ajustes de proyecto de rendering (MSAA/TAA/scale/shadows)

**No verificado en esta revisión** (la UI existe: Project Settings → Rendering, con secciones de anti-aliasing, sombras, tonemap, texturas…). **No afirmar nombres de setting** (`rendering/anti_aliasing/quality/msaa_3d`, `rendering/temporal_aa/*`, etc.) sin verificarlos en la rama actual. Cuando se necesite:

```gdscript
# Leer un setting por nombre (ProjectSettings, API verificada 4.7):
var msaa := ProjectSettings.get_setting("rendering/anti_aliasing/quality/msaa_3d", null)
print("MSAA 3D: ", msaa)   # si es null, el nombre no existe en esta versión → verificar
```

La palanca segura y verificada: **los ajustes de la UI de Project Settings → Rendering** (MSAA 3D/2D, TAA, tamaño de shadowmap, tonemap, compresión de texturas) se prueban **midiendo antes/después** con los monitores de arriba.

## Implementación mínima

Un "perf HUD" mínimo (para medir en juego, 4.7, APIs verificadas):

```gdscript
# perf_hud.gd — Godot 4.7 (APIs verificadas)
extends CanvasLayer
## HUD de rendimiento: los monitores oficiales, actualizados a 1 Hz (el ritmo del
## propio Performance.get_monitor).

@onready var _label: Label = $Label

func _ready() -> void:
	set_process(false)   # actualizamos por timer, no por frame

func _process(_delta: float) -> void:
	# get_monitor se actualiza ~1×/s; leer cada frame es suficiente (barato).
	var fps := Performance.get_monitor(Performance.TIME_FPS)
	var ms := Performance.get_monitor(Performance.TIME_PROCESS) * 1000.0
	var dc := Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)
	var prim := Performance.get_monitor(Performance.RENDER_TOTAL_PRIMITIVES_IN_FRAME)
	var vram := Performance.get_monitor(Performance.RENDER_VIDEO_MEM_USED) / 1048576.0
	_label.text = "FPS %.0f | %.1f ms | DC %d | %d tris | %.0f MB VRAM" % [fps, ms, dc, prim, vram]
```

## Implementación recomendada

### 1. Monitores custom (los que el editor no tiene)

```gdscript
# Godot 4.7 (API verificada: add_custom_monitor(id, callable)).
# El callable no toma argumentos y devuelve float.
var _spawn_ms := 0.0

func _ready() -> void:
	Performance.add_custom_monitor("Game/SpawnMs", _spawn_ms_monitor)

func _spawn_ms_monitor() -> float:
	return _spawn_ms * 1000.0

# En el código de spawn, medir el bloque:
# var t0 := Time.get_ticks_usec()
# ... spawn ...
# _spawn_ms = (Time.get_ticks_usec() - t0) / 1_000_000.0
```

### 2. Truco de fill rate (verificado en `gpu_optimization.html`)

```text
1. V-Sync OFF (que el FPS no esté limitado).
2. Ejecutar con ventana GRANDE → anotar FPS.
3. Ejecutar con ventana MUY PEQUEÑA → anotar FPS.
4. Si el FPS sube bastante con ventana pequeña → fill rate limited:
   reducir overdraw/transparencias/shaders/post-proceso.
   (También: reducir el tamaño de shadowmap y re-probar.)
```

### 3. Instancing: verificar que está ocurriendo (Forward+)

```text
1. Mismo mesh + mismo material en N MeshInstance3D (opaco o alpha-test).
2. Comparar RENDER_TOTAL_DRAW_CALLS_IN_FRAME con y sin las N instancias:
   si el draw calls casi no sube → el instancing automático está agrupando (Forward+).
   Si sube lineal → no es Forward+, o el material es alpha-blend (no se instancia).
3. Si no puedes usar Forward+ (mobile/compatibility) → MultiMesh para los N idénticos.
```

### 4. Baking de luces (mobile)

```text
1. Identificar las luces que no son dinámicas (omnis/spots de ambiente).
2. bake_mode = Static (verificado: "set lights' bake mode to Static ... skip real-time
   lighting on meshes that have baked lighting").
3. Mantener Dynamic SOLO en la DirectionalLight3D (sol) — "a good balance" (verificado).
   Downside oficial: las luces Static no proyectan sombras sobre meshes con lightmap.
```

## Ejemplo práctico

**Juego**: mundo abierto 3D, desktop + mobile.

1. **Baseline (desktop)**: FPS 144, 6.5 ms, 2400 draw calls, 1.2M tris, 900 MB VRAM.
2. **Diagnóstico**: `TIME_PROCESS` bajo → no es CPU lógica. `RENDER_TOTAL_DRAW_CALLS_IN_FRAME` alto (2400) → **materiales/meshes**.
3. **Optimización 1 (materiales)**: unificar 400 materiales de props a 60 (atlases + mismo StandardMaterial3D) → **medir**: 900 draw calls, FPS 144 (sin perder, pero con margen).
4. **Optimización 2 (LOD)**: `visibility_range_end = 80` + `FADE_DEPENDENCIES` en props + mesh LOD en árboles → **medir**: 400k tris, FPS 144.
5. **Optimización 3 (sombras, mobile)**: en mobile, shadowmap 2048→1024 + `cast_shadow OFF` en props lejanos + luces Static → **medir**: FPS 60 (de 38), fill rate reducido (truco de ventana: ventana pequeña ya no sube tanto el FPS).
6. **Cada paso se midió antes/después** — la regla cumplida. Si un paso no mejoraba, se deshacía.

## Integración

- **`godot-lod`**: las palancas de distancia (visibility_range, mesh LOD, lod_bias) están allí; aquí se miden.
- **`godot-shader-spatial`**: el coste de fragment/overdraw se explica aquí y se aplica en el shader.
- **`godot-character*`**: el coste del personaje (anim + shader) se mide con esta skill.
- **`godot-physics`**: los monitores de física de aquí sirven para detectar; la resolución va en `godot-physics` (Lote 4, disponible).
- **UI/HUD**: el perf HUD (*Implementación mínima*) se monta como `CanvasLayer`.

## Errores frecuentes

1. **Optimizar sin medir** (la violación de la regla) → "parecía lento" no es un dato. Medir con `Performance.get_monitor` antes de tocar nada.
2. **Optimizar y no medir después** → no hay forma de saber si ayudó o empeoró. Siempre comparar los mismos monitores.
3. **Confiar en `OBJECT_ORPHAN_NODE_COUNT` en release** → los monitores de debug **devuelven 0 en release** (verificado). En build, no son válidos.
4. **Confiar en un solo frame de `get_monitor`** → se actualiza ~1×/s (y hasta 1 s de delay, verificado). Medir una **media** (varios segundos), no un frame.
5. **Optimizar polígonos en desktop "porque en mobile importa"** → en desktop el vertex cost es bajo (verificado: "vertex cost is low"); la palanca en desktop suele ser draw calls/VRAM/fill rate. No invertir el orden.
6. **Cargar el mundo en runtime para "ahorrar"** sin tile-culling → si todo está instanciado y visible, no se ahorra nada; el culling (frustum/occlusion/visibility_range) es lo que lo hace útil.
7. **Batching en runtime de meshes grandes** ("juntar en un ArrayMesh cada frame") → "combining large meshes in real-time is prohibitively expensive" (verificado). El batch estático se hace **ahead of time** (editor/add-on).
8. **Apagar el MSAA "porque es caro" sin medir** → en desktop moderno, el MSAA puede ser barato o caro según la escena; medir (truco de ventana + FPS) antes de decidir.
9. **Usar `RENDER_TOTAL_PRIMITIVES_IN_FRAME` como "polígonos del asset"** → son los triángulos **renderizados en el frame** (post-culling/LOD), no los del asset. Para el asset, usar el contador del editor.
10. **Medir en el editor y extrapolar al build** → el editor añade overhead (viewport, gizmos, stats). Medir en **ejecución** (F5) y, para el target real, en el build exportado.
11. **`PIPELINE_COMPILATIONS_*` altos al arrancar** y pensar que es un bug → es la compilación de pipelines (normal en el primer arranque de una build; verificado como categoría de monitor). Optimizar = warm-up/carga, no es un cuello de botella en gameplay.

## Anti-patrones

- **"El truco de otro juego"** (copiar una optimización de otro motor/juego sin medir) → la regla manda medir; lo que ayuda en un fill-rate-limited mobile puede ser irrelevante en un draw-call-limited desktop.
- **Optimizar 20 cosas a la vez** → no se puede atribuir la mejora (ni el empeoramiento). Una a la vez, medir, decidir.
- **Profiling con `print()` por frame** → el propio `print` es el cuello de botella (I/O). Usar `Performance.add_custom_monitor` + `Time.get_ticks_usec()`.
- **Un solo frame de medición** (arriba #4).
- **Desactivar V-Sync y medir "en el monitor de 144 Hz"** sin tener claro el target → el target define el presupuesto (60/120/144); medir contra el target.
- **Medir en la escena vacía** (y extrapolar) → medir en la **peor escena** del juego (la que el jugador va a jugar).

## Performance

(Esta skill **es** performance; su propio coste es despreciable.)

- `Performance.get_monitor`: lectura de un valor en memoria (actualizado ~1×/s) — **gratis** a efectos de medición.
- `Performance.add_custom_monitor`: el callable se ejecuta al ritmo de actualización del monitor (~1×/s) — despreciable.
- El coste real de la *medición* es el del bloque instrumentado con `Time.get_ticks_usec()` — dos llamadas de reloj por bloque, despreciable.

## Debugging

### "No sé qué es el bottleneck"

```text
1. Medir los 5 monitores (TIME_FPS, TIME_PROCESS, TIME_PHYSICS_PROCESS,
   RENDER_TOTAL_DRAW_CALLS_IN_FRAME, RENDER_VIDEO_MEM_USED) en la peor escena.
2. Aplicar la clasificación del *Flujo de diagnóstico* (CPU lógica / física / GPU).
3. Si es GPU: truco de ventana (fill rate) + RENDER_TOTAL_DRAW_CALLS vs PRIMITIVES.
```

### "Optimicé y no mejora"

```text
1. ¿Mediste los mismos monitores antes/después? (si no, no hay "no mejora": hay desconocido)
2. ¿La palanca ataca el bottleneck real? (ej. reducir polígonos en un fill-rate-limited
   no ayuda; el truco de ventana te lo dice)
3. ¿El cambio se aplicó? (el material/LOD/shadowmap se recargó; no hay un "cache"
   antiguo: reiniciar la escena)
4. ¿Mediste en el target real? (editor ≠ build; desktop ≠ mobile)
```

### "El VRAM crece y no baja"

```text
1. RENDER_VIDEO_MEM_USED por escena: si crece al cargar y no baja al descargar →
   los recursos no se liberan (refs/`preload`/escenas no `queue_free`).
2. Texturas: compresión de VRAM en import (verificado: reduce bandwidth).
3. Mipmaps: si no son necesarios (UI 2D), apagarlos ahorra VRAM.
```

### "Stutters al cargar / al entrar en una zona"

```text
1. PIPELINE_COMPILATIONS_* (34–38): compilación de pipelines (normal al arrancar).
2. Carga de assets (texturas/meshes) → preload/streaming; no cargar todo a la vez.
3. `OBJECT_ORPHAN_NODE_COUNT` (solo debug): si crece, hay fugas de nodos.
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — monitores `Performance` (constantes y notas de debug-only/delay) y los tutoriales `gpu_optimization.html` / `optimizing_3d_performance.html` en la rama stable (4.7).
- **4.0 ↔ 4.7**: los monitores de rendering (`RENDER_TOTAL_*`, `TIME_*`) son estables en la serie 4.x; los `PIPELINE_COMPILATIONS_*` (34–38) son de la era Forward+ (4.0+).
- **Instancing automático: solo Forward+** (verificado: "not Mobile or Compatibility") — en mobile/compatibility, la palanca es `MultiMesh` y el batching, no el instancing automático.
- **`VisibleOnScreenNotifier3D`/`VisibleOnScreenEnabler3D`**: verificados en 4.7 (referenciados por el tutorial de 3D performance).
- **Plataformas**: las diferencias mobile vs desktop (fill rate, tile-based, vertex_lighting) están documentadas arriba (verificado en `gpu_optimization.html`).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-lod` | Las palancas de distancia (aquí se miden) |
| `godot-shader-spatial` | El coste de fragment/overdraw (aquí se mide) |
| `godot-physics` | Los monitores de física (aquí se detectan) — disponible (Lote 4) |
| `godot-third-person-character` | Compuesta que usa esta skill para medir (Lote 0) |

## Skills relacionadas

- `godot-error-low-fps` — árbol de diagnóstico dedicado (pendiente)
- `godot-recipe-mesh-batching` — batching estático (pendiente)
- `godot-recipe-multimesh` — instancing manual (pendiente)
- `godot-mobile-performance` — deep-dive mobile (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- Performance (monitores, get_monitor, add_custom_monitor, notas de debug-only y delay): https://docs.godotengine.org/en/stable/classes/class_performance.html
- GPU optimization (draw calls, state changes, materials, fill rate, overdraw, texturas, sombras, transparencia, multi-plataforma, tile renderers): https://docs.godotengine.org/en/stable/tutorials/performance/gpu_optimization.html
- Optimizing 3D performance (culling, oclusión, transparencias, LOD, instancing, baking, skinning, mundos grandes): https://docs.godotengine.org/en/stable/tutorials/performance/optimizing_3d_performance.html
- Using MultiMesh (instancing manual): https://docs.godotengine.org/en/stable/tutorials/performance/using_multimesh.html
- Variable rate shading: https://docs.godotengine.org/en/stable/tutorials/3d/variable_rate_shading.html
- `GeometryInstance3D` (visibility ranges, lod_bias, cast_shadow): https://docs.godotengine.org/en/stable/classes/class_geometryinstance3d.html
- `ProjectSettings` (get_setting/set_setting/has_setting): https://docs.godotengine.org/en/stable/classes/class_projectsettings.html
- Páginas referenciadas por los tutoriales de arriba (buscar por nombre en el índice de la docs si la URL cambia entre versiones): *Occlusion culling* (`doc_occlusion_culling`), *Mesh LOD* (`doc_mesh_lod`), *Visibility Ranges* (`doc_visibility_ranges`), *Using Lightmap GI* (`doc_using_lightmap_gi`).