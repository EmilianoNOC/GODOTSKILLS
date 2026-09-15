# GODOT SKILLS INDEX

> Índice global de la biblioteca de skills para Godot 4.x (spec §55).
> **Estado: LOTE 5** — 2026-09-15. Fuente de verdad de la API: `docs.godotengine.org/en/stable/` (rama **4.7**).

## Resumen

| Métrica | Valor |
|---|---|
| Skills creadas | **17** (1 compuesta + 16 base) |
| Skills pendientes | ver plan de lotes (lotes 1, 2, 6–20) |
| Skills actualizadas | 0 en el Lote 5 |
| Skills descartadas por duplicación | 0 |
| Versión de referencia verificada | Godot 4.7 (stable) |

## Cómo leer este índice

- **Estado**: `Verified` = API verificada contra la docs oficial en la versión indicada. `Pendiente` = skill programada, aún no generada.
- **Nivel**: BEGINNER / INTERMEDIATE / ADVANCED / EXPERT (conocimiento del motor requerido, spec §56).
- Las skills compuestas (spec §51) agrupan skills base; desde el Lote 5, las 16 bases existen y la compuesta las referencia.

## 3D · Characters

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-third-person-character` | 3D · Characters (compuesta) | Intermedio | 4.7 | Verified |
| `godot-characterbody3d` | 3D · Física | Principiante | 4.7 | Verified |
| `godot-character-controller` | 3D · Characters | Intermedio | 4.7 | Verified |

## 3D · Cámaras

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-camera3d` | 3D · Cámaras | Intermedio | 4.7 | Verified |

## 2D

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-node2d` | 2D · Base | Principiante | 4.7 | Verified |
| `godot-tilemap` | 2D · Base (mapas) | Principiante | 4.7 | Verified |
| `godot-platformer-2d` | 2D · Base (juego) | Principiante | 4.7 | Verified |
| `godot-camera2d` | 2D · Base (cámara) | Principiante | 4.7 | Verified |

## Animation

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-animationtree` | Animation | Intermedio | 4.7 | Verified |

## Input

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-input` | Input | Principiante | 4.7 | Verified |

## Physics

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-physics` | Física · Base | Principiante | 4.7 | Verified |
| `godot-raycast3d` | Física · Base (queries) | Intermedio | 4.7 | Verified |
| `godot-area3d` | Física · Base (detección) | Principiante | 4.7 | Verified |
| `godot-physics-materials` | Física · Base (materiales) | Principiante | 4.7 | Verified |

## Shaders

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-shader-spatial` | Shaders | Intermedio | 4.7 | Verified |

## Performance

| ID | Categoría | Nivel | Godot | Estado |
|---|---|---|---|---|
| `godot-rendering-performance` | Performance | Avanzado | 4.7 | Verified |
| `godot-lod` | 3D · Performance | Avanzado | 4.7 | Verified |

## Mapas por pregunta (spec §2 — cobertura al cierre del Lote 5)

| Pregunta | Skill a consultar |
|---|---|
| ¿Cómo implementar un personaje 3D (tercera persona) completo? | `godot-third-person-character` (flujo completo) |
| ¿Cómo funciona `CharacterBody3D` / `move_and_slide`? | `godot-characterbody3d` |
| ¿Cómo ajustar el feel de movimiento (aceleración, salto, coyote, buffer, wall slide)? | `godot-character-controller` |
| ¿Cómo configurar input (teclado/ratón/gamepad, remape, captura)? | `godot-input` |
| ¿Cómo utilizar AnimationTree para locomoción / one-shots / root motion? | `godot-animationtree` |
| ¿Cómo crear una cámara 3D (orbital, spring arm, proyección, transiciones)? | `godot-camera3d` |
| ¿Cómo escribir shaders 3D (render modes, uniforms, per-instance, sRGB)? | `godot-shader-spatial` |
| ¿Cómo medir/diagnosticar rendimiento de rendering (CPU/GPU/VRAM/draw calls)? | `godot-rendering-performance` |
| ¿Cómo aplicar LOD (mesh, visibility ranges, impostor, sombras)? | `godot-lod` |
| ¿Qué tipo de body usar (Static/Rigid/Character/Area)? ¿Cómo configuro layers y masks? | `godot-physics` |
| ¿Cómo hacer raycasts / shapecasts / picking con mouse en 3D? | `godot-raycast3d` |
| ¿Cómo hacer pickups / triggers / zonas de daño / override de gravedad? | `godot-area3d` |
| ¿Cómo hacer superficies resbaladizas, con rebote, de alto agarre? | `godot-physics-materials` |
| ¿El objeto atraviesa el piso / la pila se va / el FPS se va a 1 (física)? | `godot-physics` (§ Debugging/Troubleshooting) |
| ¿Cómo hacer un platformer 2D (saltos, pendientes, plataformas una-vía/móviles)? | `godot-platformer-2d` |
| ¿Cómo armar mapas por tiles (colisión, terreno, datos por tile, isométrico/hex)? | `godot-tilemap` |
| ¿Cómo hacer una cámara 2D (follow, smoothing, límites, zoom, lookahead)? | `godot-camera2d` |
| ¿Transform/posición/rotación de nodos 2D (local vs global, `to_global`)? | `godot-node2d` |
| ¿Mi juego tiene pocos FPS? | `godot-rendering-performance` (§ Flujo de diagnóstico) + `godot-physics` (si es física) |
| ¿El personaje no se mueve / no salta bien / la cámara choca / la animación no cambia? | La skill del subsistema (§ Debugging) |

## Plan de lotes (spec §65 — prioridad)

| Lote | Área | Estado |
|---|---|---|
| 0 | Skill compuesta `godot-third-person-character` + infraestructura (index/router/graph/matrix) | ✅ Hecho (2026-09-15) |
| 1 | Fundamentos (editor, project settings, nodos, escenas, recursos, señales, autoloads, lifecycle) | Pendiente |
| 2 | GDScript (sintaxis, tipos, classes, signals, await, export, tool, performance) | Pendiente |
| 3 | Las 8 skills base del Lote 0 (characterbody3d, character-controller, input, animationtree, camera3d, shader-spatial, rendering-performance, lod) | ✅ Hecho (2026-09-15) |
| 4 | Physics (capas, shapes, raycast/shapecast, areas, materiales, interpolación) | ✅ Hecho (2026-09-15): `godot-physics`, `godot-raycast3d`, `godot-area3d`, `godot-physics-materials` |
| 5 | 2D (nodos, tilemap, plataformas, cámaras 2D) | ✅ Hecho (2026-09-15): `godot-node2d`, `godot-tilemap`, `godot-platformer-2d`, `godot-camera2d` |
| 6 | Animation (BlendSpace, IK, retargeting, Skeleton3D) | Pendiente |
| 7 | UI (containers, theme, focus, localización) | Pendiente |
| 8 | Rendering (renderers, MSAA, sombras, culling, profiling GPU) | Pendiente |
| 9 | Shaders (canvasitem, post-process, efectos) | Pendiente |
| 10 | Navigation + IA (NavigationServer3D, FSM, behavior trees) | Pendiente |
| 11 | Audio (buses, efectos, spatial, music manager) | Pendiente |
| 12 | Networking (MultiplayerAPI, RPC, spawner, lobby) | Pendiente |
| 13 | Save/Load + inventarios/data-driven | Pendiente |
| 14 | Performance general (pooling, threading, memory, profiling) | Pendiente |
| 15 | Export + plataformas (win/linux/mac/android/web) | Pendiente |
| 16 | Tools (EditorPlugin, @tool, custom inspectors) | Pendiente |
| 17 | Anti-patrones + troubleshooting (`godot-error-*`) | Pendiente |
| 18 | Recetas restantes (combat, quests, weather, day-night, streaming…) | Pendiente |
| 19 | 3D avanzado (open world, chunking, vegetación, HLOD) | Pendiente |
| 20 | Procedural generation + testing + mobile + web + VR | Pendiente |

## APIs "no verificadas" declaradas (Lotes 3–5)

Marques explícitas dejadas por las skills (no afirmar como API hasta verificar en la rama actual):

| Skill | API | Motivo |
|---|---|---|
| `godot-shader-spatial` | Funciones de ruido builtin (`noise_*`, `cellular_*`, `perlin_*`, `value_*`) y `rand*` | La sección no aparece en `shader_functions.html` 4.7; verificar antes de usar (hay hash propio como alternativa) |
| `godot-shader-spatial` | Built-in `TEXTURE` (albedo) y `TEXTURE_SCREEN` | La vía robusta (`uniform sampler2D : source_color / hint_screen_texture`) sí está verificada; el nombre exacto de los built-ins, verificar por versión |
| `godot-rendering-performance` | Rutas exactas de `ProjectSettings` `rendering/*` (MSAA/TAA/scale/shadows) | La UI existe; los nombres de setting no se afirman sin verificar (hay `ProjectSettings.get_setting` como vía de comprobación) |
| `godot-physics` | Clave exacta de `ProjectSettings` para la interpolación global de física (feature 4.3+) | `PhysicsBody3D` 4.7 NO tiene prop per-body (verificado: solo los 6 `axis_lock_*`); la clave del setting no se afirma sin verificar — revisar *Physics → Common* |
| `godot-raycast3d` | Clave `metadata` en el dict de `intersect_ray` | La tabla 4.7 lista `collider, collider_id, normal, position, face_index, rid, shape` (sin `metadata`); aparece en el tutorial de rama master → tratar como no verificada para 4.7 |
| `godot-area3d` | Señales `object_entered` / `object_exited` | No están en la tabla de señales 4.7 (solo `area_*`/`body_*` y `_shape_*`) |
| `godot-physics-materials` | Clase `PhysicsMaterial3D` / enums `*_combine_mode` | La clase 4.7 es `PhysicsMaterial` (una sola; la página 3D da 404) y la combinación la rigen `rough`/`absorbent` |
| `godot-tilemap` | Getter exacto de custom data en `TileData` (`TileData.custom_data[…]/get_custom_data(…)`) | El mecanismo de capas de datos está verificado en `TileSet`/`TileMapLayer`; el acceso exacto al valor en `TileData` no se verificó en esta revisión — verificar `TileData` antes de usar |
| `godot-node2d` / `godot-platformer-2d` | Índices exactos de los monitores `Performance.PHYSICS_2D_ACTIVE_OBJECTS`/`PHYSICS_2D_COLLISION_PAIRS` | La familia de monitores 2D existe; los índices numéricos no se citan sin verificar (los de 3D, 20/21, sí están verificados) |
