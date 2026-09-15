# AGENT_ROUTING

> Enseña al agente de código a decidir qué skills consultar antes de implementar, modificar, depurar u optimizar (spec §50, §68).
> **Estado: LOTE 4** — 2026-09-15. Creadas: `godot-third-person-character` (compuesta) + 12 skills base (incluido el cluster físico). El resto se enrutará cuando se creen (ver `GODOT_SKILLS_INDEX.md`).

## Proceso de decisión

```text
USER REQUEST
  ↓
1. ANALYZE: ¿qué subsistemas toca la petición? (física, input, animación, cámara, rendering, datos, red…)
  ↓
2. SEARCH: buscar en GODOT_SKILLS_INDEX.md por categoría o por "pregunta".
  ↓
3. LOAD: cargar la skill que corresponda. Si la petición es un sistema completo → la compuesta;
   si toca un subsistema concreto → la skill base (más profunda). La compuesta referencia a las bases.
  ↓
4. DEPENDENCIES: seguir el grafo (SKILLS_GRAPH.md) y cargar dependencias marcadas como obligatorias.
  ↓
5. VERIFY: cualquier API dudosa → consultar la documentación oficial (docs.godotengine.org/en/stable/, rama 4.7). NUNCA inventar APIs: si no se encuentra, marcar "API no verificada".
  ↓
6. PLAN → IMPLEMENT → TEST → DEBUG → OPTIMIZE ONLY IF NECESSARY (medir antes y después, spec §60).
```

## Reglas de oro del router

1. **Sistema completo → compuesta; subsistema → base.** "Quiero un personaje TPS completo" → `godot-third-person-character`. "El salto no se siente bien" → `godot-character-controller`. "El shader se ve lavado" → `godot-shader-spatial`.
2. **Siempre la skill de error correspondiente ante un fallo.** `godot-error-<problema>` (cuando existan); mientras, la sección *Debugging* de la skill del subsistema.
3. **Rendimiento = medir primero.** Ante "pocos FPS / lento / se traba" → `godot-rendering-performance` (§ Flujo de diagnóstico); si el cuello es de **física** → `godot-physics` (§ Performance: monitores `PHYSICS_3D_ACTIVE_OBJECTS`/`COLLISION_PAIRS`). Prohibido optimizar por intuición (spec §60).
4. **Nunca mezclar versiones.** Toda skill indica su versión Godot; si la skill dice 4.7, el código es 4.7. Diferencias entre versiones → `GODOT_VERSION_MATRIX.md`.
5. **Mínima complejidad.** Usar la solución más simple que resuelva (spec §58); escalar (separar scripts/manager) solo cuando la skill indique el punto de ruptura (spec §59).
6. **Desconfiar de APIs "que deberían existir".** Las skills documentan también lo que **NO** existe (verificado): señales de floor/wall en `CharacterBody3D`, `floor_friction`/`wall_min_slide_speed`, smoothing de `Camera3D`, señales de acciones en `Input`, nodo `LOD` core, `get_spatial_lighting` (3.x), **`RigidBody3D.motion_mode` (CCD es `continuous_cd`), `StaticBody3D.force_recompute_shapes`, `PhysicsMaterial3D` (la clase es `PhysicsMaterial`), enums `*_combine_mode` de materiales, señales `object_entered/exited` en `Area3D`, `RayCast3D.cast_to` (es `target_position`), `test_motion` en 3D (es `cast_motion`)**. Si el usuario/tutorial las menciona → la skill da la alternativa.
7. **Código que toca estado físico va en `_physics_process`.** Leer/mover bodies, raycasts directos y queries al space: fuera de ese callback el espacio de física puede estar *locked* (verificado). Regla transversal del cluster físico.

## Tabla de enrutamiento (Lote 4)

### Personaje y movimiento

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "Quiero un personaje en tercera persona" (sistema completo) | `godot-third-person-character` | Compuesta: integra las bases |
| "¿Cómo funciona `move_and_slide`?" / "el cuerpo se atasca" / "no choca" | `godot-characterbody3d` | § API, § Errores, § Debugging |
| "Que el personaje salte mejor" / "coyote time" / "jump buffer" / "wall slide" / "air control" | `godot-character-controller` | § Implementación mínima/recomendada (tuning) |
| "El personaje no se mueve" | `godot-characterbody3d` → § Debugging ("no se mueve") | + `godot-input` § Debugging ("el input no llega") |
| "Migrar de Godot 3 a 4" (movimiento) | `godot-characterbody3d` + `godot-character-controller` → § Compatibilidad + `GODOT_VERSION_MATRIX.md` | |

### Física y colisión (nuevo en Lote 4)

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "¿Qué body uso?" (Static/Rigid/Character/Area) / "configurar capas y masks" | `godot-physics` | § Conceptos (4 tipos, layers/masks, ticks 60 Hz) |
| "La caja debe caer y rebotar" / "empujar un objeto" / "fuerza e impulso" | `godot-physics` | § Implementación (`apply_force/impulse`, `_integrate_forces`) |
| "El objeto atraviesa el piso" / "la pila se va" / "FPS a 1 de golpe" | `godot-physics` → § Debugging/Troubleshooting | `continuous_cd` (CCD), Jolt, ticks, *spiral of death* (todo oficial) |
| "El objeto se duerme / no responde" / "¿con quién choco?" | `godot-physics` | Sleep (`can_sleep`) + contact reporting (`contact_monitor`/`max_contacts_reported`) |
| "Raycast para detectar el piso" / "picking con mouse" / "línea de visión IA" | `godot-raycast3d` | RayCast3D (nodo) o `intersect_ray` (directo, solo en `_physics_process`) |
| "Laser ancho" / "sweep" / "¿toco algo YA?" | `godot-raycast3d` | `ShapeCast3D` (overlap instantáneo oficial: target 0 + `force_shapecast_update`) |
| "Pickup" / "zona de daño" / "checkpoint" / "gravedad distinta en una zona" | `godot-area3d` | Señales `body_entered/exited`, `monitoring/monitorable`, `SpaceOverride` |
| "Suelo resbaladizo" / "trampolín" / "pelota que rebota" / "más fricción" | `godot-physics-materials` | `PhysicsMaterial` (friction/bounce/rough/absorbent) en `physics_material_override` |
| "La cámara proyecta un ray desde el mouse" | `godot-raycast3d` | `project_ray_origin/normal` (verificado en el tutorial oficial) |

### Input

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "Configurar input para teclado + gamepad" / "mapear acciones" | `godot-input` | § Arquitectura (Input Map) + § Implementación |
| "Remapear controles en runtime" / "perfiles de controles" | `godot-input` | § Implementación recomendada #2 |
| "El input no llega / el ratón va mal / el gamepad no se detecta" | `godot-input` → § Debugging | |
| "Cursor capturado" / "orbitar con ratón" | `godot-input` + `godot-camera3d` | `screen_relative` (verificado) + `mouse_mode` |

### Animación

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "Animar la locomoción con AnimationTree" / "blend walk/run" / "máquina de estados" | `godot-animationtree` | § Implementación mínima (Idle/Move/Jump) |
| "Ataque con OneShot" / "el ataque no rompe la locomoción" | `godot-animationtree` | § Implementación recomendada #1 (OneShot + `FADE_OUT` 4.7) |
| "Root motion" (el clip mueve al personaje) | `godot-animationtree` | § Implementación recomendada #2 (`get_root_motion_position`) |
| "La animación no cambia / no se reproduce" / "jitter de piernas" | `godot-animationtree` → § Debugging | |

### Cámara

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "Hacer una cámara third person" / "orbitar" / "zoom" | `godot-camera3d` | § Implementación recomendada #1 (orbit + spring arm) |
| "La cámara se mete en las paredes" | `godot-camera3d` → § Debugging | `SpringArm3D` + `add_excluded_object` |
| "Proyectar un punto 3D a la pantalla" / "click sobre un objeto" | `godot-camera3d` | § Implementación recomendada #2/#3 (`project_position`, `project_ray_*`) |
| "Transición de cámara" / "cámara cinemática" | `godot-camera3d` | § Implementación recomendada #4 (`make_current`/`clear_current`) |

### Shaders

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "Efecto visual 3D" / "highlight" / "disolución" / "vertex animation" | `godot-shader-spatial` | § Implementación mínima/recomendada |
| "El shader se ve lavado / los colores no son" | `godot-shader-spatial` → § Debugging | `source_color` (causa nº1) |
| "Un valor por enemigo con 1 material" / "N objetos, 1 material" | `godot-shader-spatial` | § Uniforms por instancia (`instance uniform`, máx 16) |
| "Viento en todo el mundo" / "1 parámetro para todos los shaders" | `godot-shader-spatial` | § Uniforms globales (`global uniform` + `RenderingServer.global_shader_parameter_set`) |
| "El shader no compila" / "usar SCREEN_TEXTURE" / "get_spatial_lighting" | `godot-shader-spatial` → § Errores | APIs 3.x eliminadas (verificado) |

### Rendimiento

| Si el usuario pide… | Consultar | Notas |
|---|---|---|
| "Mi juego tiene pocos FPS" / "se traba" | `godot-rendering-performance` | § Flujo de diagnóstico (medir → clasificar → optimizar → medir) |
| "Pocos FPS y el cuello es de física" (muchos bodies, pares de colisión) | `godot-physics` | § Performance (monitores 20/21, Jolt, shapes, ticks) |
| "Reducir draw calls" / "muchos materiales" | `godot-rendering-performance` | § Menú de palancas (materiales, instancing, MultiMesh) |
| "La VRAM está alta" / "las texturas pesan" | `godot-rendering-performance` | § Menú de palancas (VRAM) |
| "Aplicar LOD" / "los árboles/props a distancia" / "impostor" | `godot-lod` | § Implementación (visibility ranges, mesh LOD, impostor, sombras) |
| "Los props parpadean a lo lejos" | `godot-lod` → § Debugging | Hysteresis (`end_margin`) |
| "Optimizar para móvil" | `godot-rendering-performance` | § Conceptos (fill rate, truco de ventana, baking) + `godot-lod` |

## Peticiones fuera de cobertura del Lote 4

Si la petición cae en un área sin skill creada (2D, UI, red, save/load, audio, IA, vehículos, export…), el agente debe:

1. Decir explícitamente que no hay skill para ese sistema en estos lotes (no fingir cobertura).
2. Ir directo a la documentación oficial:
   - Class reference: https://docs.godotengine.org/en/stable/classes/index.html
   - Tutorials: https://docs.godotengine.org/en/stable/tutorials/index.html
3. Responder con la misma disciplina: APIs verificadas en la rama 4.7, marcar "API no verificada" si no se confirma, no inventar, no mezclar 3.x/4.x.
4. Registrar el tema como pendiente en `GODOT_SKILLS_INDEX.md` (plan de lotes) para el siguiente lote.

## Ejemplos de razonamiento (spec §50)

- **"Quiero que el personaje pueda saltar"** → subsistema: movimiento → `godot-character-controller` (§ Implementación mínima: coyote + buffer) + `godot-input` (acción `jump`) + `godot-animationtree` (estado `Jump`).
- **"El jugador salta al borde de la plataforma aunque pulse tarde"** → `godot-character-controller` (§ Errores #2, #7: coyote/buffer sin límite de tiempo) — ajustar los timers.
- **"En el aire el personaje no frena"** → `godot-character-controller` (§ Errores #4: `air_accel`/`air_decel` delirantes) — el dial de air control.
- **"La cámara rota con el personaje"** → `godot-third-person-character` (§ Errores #11) o `godot-characterbody3d` (§ Errores #9) — rotar el pivot `Model`, no el body.
- **"Quiero enemigos que persigan al jugador"** → subsistemas: IA + navegación → NO cubierto (Lote 10) → docs oficiales (`NavigationAgent3D`, `NavigationServer3D`) + marcar pendiente `godot-navigationagent3d`.
- **"Mi juego tiene pocos FPS"** → `godot-rendering-performance` (§ Flujo de diagnóstico): medir `TIME_FPS`/`TIME_PROCESS`/`TIME_PHYSICS_PROCESS`/`RENDER_TOTAL_DRAW_CALLS_IN_FRAME` → clasificar (CPU lógica / física / GPU) → optimizar una cosa → medir de nuevo.
- **"El jugador atraviesa las paredes al correr rápido"** → `godot-physics` → § Debugging (tunneling): `continuous_cd = true`, pared más gruesa, shape ∝ velocidad, ticks 120+ (soluciones oficiales) — NO "subir la velocidad de la física a ciegas" (medir).
- **"Hacer un pickup que se recoge al tocarlo"** → `godot-area3d` (§ Implementación mínima): `Area3D` + `body_entered` + `queue_free`; `monitorable = false`; la interacción la define la mask del area (player).
- **"Quiero que el piso de hielo resbale"** → `godot-physics-materials` (§ Implementación recomendada #4): `PhysicsMaterial {friction=0.03, rough=false}` en `physics_material_override` del `StaticBody3D` — recordar la regla de `rough` (el mínimo gana).
- **"Detectar con el mouse qué objeto toca"** → `godot-raycast3d` (§ Implementación recomendada #1): click en `_input`, query en `_physics_process` (space locked — verificado), `project_ray_origin/normal`, `intersect_ray` + dict de resultado.
- **"Poner LOD a los árboles"** → `godot-lod` (§ Implementación mínima: `visibility_range_end` + margen) + `godot-rendering-performance` (medir draw calls/tris antes/después). Recordar: NO existe nodo `LOD` core en 4.7 (verificado).
- **"Un highlight distinto por enemigo sin duplicar materiales"** → `godot-shader-spatial` (§ Uniforms por instancia: `instance uniform` + `set_instance_shader_parameter`).
