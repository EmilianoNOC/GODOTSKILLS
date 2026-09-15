# GODOT_VERSION_MATRIX

> Diferencias de API entre versiones de Godot, **verificadas** en la documentación oficial (spec §62, §63).
> Referencias: rama `4.0` y rama `4.7` (stable) de `docs.godotengine.org`, y XML de clases de `godotengine/godot` rama `4.7-stable`.
> Última revisión: 2026-09-15.

## Reglas

- **Fuente de verdad**: la rama `stable` actual (**4.7**) es la versión objetivo de toda la biblioteca.
- Nunca mezclar APIs de 3.x y 4.x en el mismo código.
- Cada fila indica el estado en cada versión y la evidencia.
- "No existe (verificado)" = la API no aparece en la referencia de clases oficial de esa versión.

## Diferencias verificadas 4.0 → 4.7

| API | 4.0 | 4.7 (stable) | Diferencia / acción |
|---|---|---|---|
| `CharacterBody3D` (propiedades: `velocity`, `up_direction`, `floor_max_angle`, `floor_snap_length`, `floor_constant_speed`, `floor_stop_on_slope`, `floor_block_on_wall`, `max_slides`, `motion_mode`, `platform_*`, `safe_margin`, `slide_on_ceiling`, `wall_min_slide_angle`) | Existentes (tabla de propiedades idéntica) | Existentes (tabla de propiedades idéntica) | Sin cambios entre 4.0 y 4.7 en esta lista. |
| `CharacterBody3D.move_and_slide()` | `-> bool`, solo en `_physics_process` | `-> bool`, solo en `_physics_process` | Sin cambios. |
| Señales `floor_entered/floor_exited/wall_entered/wall_exited/ceiling_entered/ceiling_exited/moved` de `CharacterBody3D` | **No existen (verificado en 4.0)** | **No existen (verificado en 4.7)** | No usar en ninguna 4.x. Alternativa: polling `is_on_floor()` en `_physics_process` o `Area3D`. |
| `CharacterBody3D.floor_friction` / `wall_min_slide_speed` | **No existen (verificado en 4.0)** | **No existen (verificado en 4.7)** | Son API de **Godot 3.x**. En 4.x: `floor_constant_speed`, `PhysicsMaterial` (clase única, verificado 4.7) o amortiguación manual. |
| `Camera3D` smoothing incorporado (posición/rotación) | **No existe (verificado en 4.0)** | **No existe (verificado en 4.7)** | El smoothing existe en `Camera2D`, no en 3D. En 3D: damping manual con `delta`. |
| `Camera3D` (props `current`, `fov`, `near`, `far`, `cull_mask`, `projection`, `keep_aspect`, …) | Existentes | Existentes (misma lista + `attributes`/`compositor` presentes desde 4.0) | Estable. |
| `SpringArm3D` (`spring_length`, `margin`, `shape`, `collision_mask`, `add_excluded_object`, `get_hit_length`) | Existentes | Existentes | Estable 4.0→4.7. |
| Señales `Input.action_just_pressed` / `action_released` | — (no revisado en 4.0 en este lote) | **No existen (verificado en 4.7)**: la única señal del singleton es `joy_connection_changed` | Usar `is_action_just_pressed()` / `_unhandled_input`. |
| `Input.get_vector(neg_x, pos_x, neg_y, pos_y, deadzone=-1.0)` | Existente | Existente (verificado en 4.7) | Estable. |
| `Input.mouse_mode` (`VISIBLE/HIDDEN/CAPTURED/CONFINED/CONFINED_HIDDEN`) | Existente | Existente (verificado en 4.7, mismos valores) | Estable en 4.x (en 3.x los nombres eran distintos). |
| `AnimationTree` (clase) | Existe | Existe; hereda de `AnimationMixer` (verificado 4.7) | En 4.x la base `AnimationMixer` centraliza librerías/root motion/callbacks. |
| `AnimationTree.set_process_callback()` / `get_process_callback()` | Existente | **Deprecado en 4.7** → `AnimationMixer.callback_mode_process` | Migrar a `callback_mode_process` (`ANIMATION_CALLBACK_MODE_PROCESS_PHYSICS/IDLE/MANUAL`). |
| `AnimationMixer.root_motion_track`, `root_motion_local`, `get_root_motion_position/rotation/scale(+_accumulator)` | — | Existentes (verificado en 4.7) | Patrón oficial para root motion en `CharacterBody3D`. |
| `AnimationNodeStateMachine.state_machine_type` (`ROOT`/`NESTED`) | — | Existe (verificado en 4.7) | En 4.0–4.2 la prop equivalente es distinta; ver artículo oficial *"Migrating Animations from Godot 4.0 to 4.3"*. |
| `AnimationTree.get("parameters/playback")` + `state_machine.travel(...)` | Patrón vigente | Patrón oficial (verificado en 4.7) | Estable. |
| Nodo `LOD` (core) | — | **No existe (verificado en 4.7)**: sin `LOD.xml`, sin `scene/3d/lod.cpp`, sin página de clase | Los add-ons de AssetLib con nodo "LOD" no son core. Usar `visibility_range_*`, `lod_bias`, mesh LOD, o script propio. |
| `GeometryInstance3D.visibility_range_begin/end` + `_margin` + `visibility_range_fade_mode` | — | Existentes (verificado en 4.7; `FADE_SELF`/`FADE_DEPENDENCIES` solo Forward+) | Mecanismo nativo de culling/LOD por distancia. |
| `GeometryInstance3D.lod_bias` | — | Existe (verificado en 4.7) | Bias de transición de LOD de mesh. |
| `ArrayMesh.add_surface_from_arrays(..., lods, ...)` / `Mesh.surface_get_lods()` | — | Existentes (verificado en 4.7); `lods` = Dictionary {float → PackedInt32Array de índices} | LOD por superficie de mesh; verificar disponibilidad en 4.x < 4.7 antes de usar en otros targets. |
| `Performance.get_monitor(...)` (monitores `TIME_FPS`, `TIME_PROCESS`, `TIME_PHYSICS_PROCESS`, `RENDER_TOTAL_DRAW_CALLS_IN_FRAME`, `RENDER_TOTAL_PRIMITIVES_IN_FRAME`, `RENDER_VIDEO_MEM_USED`, `PHYSICS_3D_*`, …) + `add_custom_monitor` | — | Existentes (verificado en 4.7) | Algunos monitores solo en debug; delay de hasta 1 s (notas oficiales). |
| `RenderingServer.frame_drawn` / `frame_post_draw` (señales) | — | Existentes en 4.x (confianza alta; verificar al usar) | Instrumentación en torno al render. |
| `RigidBody3D` (tabla 4.7: `mass`, `gravity_scale`, `linear/angular_damp(+_mode)`, `constant_force/torque`, `lock_rotation`, `freeze(+_mode)`, `can_sleep`/`sleeping`, `continuous_cd`, `contact_monitor`, `max_contacts_reported`, `center_of_mass(+_mode)`, `inertia`, `physics_material_override`, `custom_integrator`; métodos `apply_force/impulse(+_central)`, `apply_torque(+_impulse)`, `add_constant_*`, `set_axis_velocity`, `get_colliding_bodies`, `get_contact_count`; señales `body_entered/exited`, `body_shape_entered/exited`, `sleeping_state_changed`; enums `FreezeMode STATIC/KINEMATIC`, `CenterOfMassMode`, `DampMode COMBINE/REPLACE`) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | CCD del rígido en 4.7 es el bool **`continuous_cd`**; NO hay `motion_mode` en la tabla 4.7 (eso es `CharacterBody3D`, con `GROUNDED/FLOATING`). |
| `RigidBody3D.motion_mode` | — | **No existe en la tabla 4.7 (verificado)** | No usar (confusión con `CharacterBody3D.motion_mode`). CCD = `continuous_cd`. |
| `StaticBody3D` (tabla 4.7: `constant_linear_velocity`, `constant_angular_velocity`, `physics_material_override`; hereda de `PhysicsBody3D`; `AnimatableBody3D` la hereda) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | Estable; al moverse por código se teleporta (no empuja) → `AnimatableBody3D` si debe empujar. |
| `StaticBody3D.force_recompute_shapes` | — | **No existe en la tabla 4.7 (verificado)** | Tutoriales viejos la citan; no usar en 4.7. |
| Clase de material de física: `PhysicsMaterial` (props `friction` 1.0, `bounce` 0.0, `rough`, `absorbent`) | — (no re-verificado en 4.0 en este lote) | `PhysicsMaterial` única (verificado 4.7); `class_physicsmaterial3d.html` **da 404** | La clase es UNA (2D y 3D). Sin enums `friction_combine_mode`/`bounce_combine_mode` (verificado 4.7): la combinación la rigen `rough` (mínimo/rough/máximo) y `absorbent` (resta rebote). |
| `CollisionShape3D` (tabla 4.7: `shape`, `disabled` — cambiar con `set_deferred()` —, `debug_color`, `debug_fill`; método `make_convex_from_siblings`) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | **Sin** `physics_material` (material por shape no disponible; va en `physics_material_override` del body). Warning oficial de escala no uniforme. |
| `Area3D` (props `monitoring`/`monitorable`/`priority`, overrides `gravity*`/`linear_damp`/`angular_damp` + `*_space_override` (enum `SpaceOverride` 5 modos), `wind_*`, `audio_bus_*`/`reverb_bus_*`; señales `body_entered/exited`, `area_entered/exited`, `body_shape_*`, `area_shape_*`; métodos `get_overlapping_bodies/areas`, `has_overlapping_*`, `overlaps_body/area`) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | Sin señales `object_entered`/`object_exited` en la tabla 4.7 (verificado). SoftBody3D no reporta overlaps (nota oficial). |
| `RayCast3D` / `ShapeCast3D` (`target_position`, no `cast_to`; `exclude_parent`, `hit_back_faces`, `hit_from_inside`, `force_*_update`, excepciones) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | `cast_to` es de 3.x. ShapeCast3D: overlap instantáneo = `target_position` 0 + `force_shapecast_update()` (patrón oficial). |
| `PhysicsDirectSpaceState3D` (`intersect_ray`, `intersect_shape`, `intersect_point`, `collide_shape`, `cast_motion`, `get_rest_info`) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | Dict de `intersect_ray` 4.7: `collider, collider_id, normal, position, face_index, rid, shape` (sin `metadata` en la tabla 4.7). `test_motion` **no existe** en 3D 4.7 → `cast_motion` (devuelve `[safe, unsafe]`). |
| `PhysicsBody3D` (tabla 4.7: solo `axis_lock_linear_x/y/z` + `axis_lock_angular_x/y/z`; `move_and_collide(motion, test_only, safe_margin, recovery_as_collision, max_collisions)`, `test_move(...)`, `add/remove_collision_exception_with`, `get_gravity`) | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | **Sin** prop per-body de interpolación de física en 4.7 (la interpolación global 4.3+ va por Project Settings; clave exacta no verificada en esta revisión). |
| Project Settings `physics/common/physics_ticks_per_second` (default 60) + Max Physics Steps per Frame | — (no re-verificado en 4.0 en este lote) | Existentes (verificado en 4.7) | Solución oficial (multiplied 120/180/240) para pilas/delgados/vehículos; subirlo alivia el *spiral of death* (coste CPU). |

### Lote 5 (2D) — verificado 4.7 el 2026-09-15

| API | 4.0 | 4.7 (stable) | Diferencia / acción |
|---|---|---|---|
| `CharacterBody2D.move_and_slide()` | `-> void` (sin args) | `-> void` (sin args) | **Asimetría verificada 2D↔3D**: 2D devuelve `void`; 3D devuelve `bool` (¿colisionó?). No usar el retorno en 2D. |
| `CharacterBody2D` defaults (2D): `up_direction (0,-1)`, `floor_max_angle 0.7853982`, `floor_snap_length 1.0`, `wall_min_slide_angle 0.2617994`, `max_slides 4`, `safe_margin 0.08`, `slide_on_ceiling true`, `floor_stop_on_slope true`, `floor_block_on_wall true`, `motion_mode GROUNDED` | — (no re-verificado en 4.0) | Existentes (verificado en 4.7) | **Difieren de los 3D**: `safe_margin` 0.08 (2D) vs 0.001 (3D), `floor_snap_length` 1.0 vs 0.1, `max_slides` 4 vs 6, `up_direction` (0,-1) vs (0,1,0). No portar valores entre dimensiones. |
| `CharacterBody2D` signals (`floor_entered/…`) | — | **No existen (verificado en 4.7)** | Igual que en 3D: polling `is_on_floor()`/`is_on_wall()`/`is_on_ceiling()` en `_physics_process`. |
| `Node2D` props (`position`, `rotation(_degrees)`, `scale`, `skew`, `transform` + `global_*`) | Existentes | Existentes (verificado en 4.7; 12 props) | **`pivot_offset` y `top_level` NO existen en 4.7** (verificado). Métodos: `to_global/to_local`, `translate`, `global_translate`, `rotate`, `apply_scale`, `move_local_x/y`, `get_angle_to`, `look_at`, `get_relative_transform_to_parent`. |
| `Node2D` Y-down / `up_direction (0,-1)` | Convención estable | Convención estable (verificado en 4.7) | Abajo = +Y; "arriba" (saltos) = Y negativa. |
| Nodo `TileMap` (4.0: el nodo de capas) | Existente (una capa = una sub-escena) | **DEPRECIADO en 4.7 → `TileMapLayer`** (verificado) | En 4.7: UNA capa = UN `TileMapLayer` (nodo). `TileMap` queda para compatibilidad; no crearlo en proyectos nuevos. |
| `TileMapLayer` (props: `tile_set`, `enabled`, `collision_enabled`, `navigation_enabled`, `occlusion_enabled`, `physics_quadrant_size 16`, `rendering_quadrant_size 16`, `y_sort_origin`, `use_kinematic_bodies`; métodos: `set_cell`, `erase_cell`, `clear`, `get_used_cells(_by_id)`, `local_to_map/map_to_local`, `update_internals`, `changed` (señal)) | — (nuevo con la refactor 4.3+) | Existentes (verificado en 4.7) | Coordenadas de celda **signed 16-bit**: fuera de rango WRAP (se envuelve). Updates batcheados a fin de frame → `update_internals()` para forzar. |
| `TileMapLayer.set_cell(coords, source_id=-1, atlas_coords=Vector2i(-1,-1), alternative_tile=0)` | — | Existente (verificado en 4.7) | El `alternative_tile` por defecto es **0** (no -1). |
| `TileSet` (tiles = 3 IDs: `source_id`, `atlas_coords` (`Vector2i`), `alternative_tile`; props `tile_shape`, `tile_layout`, `tile_offset_axis`, `tile_size`; APIs de layers: `add/remove/move_{physics,navigation,custom_data,occlusion}_layer`; `add_source` toma la ownership del source) | `TileSet` similar (4.0: `sources`) | Existentes (verificado en 4.7) | **Un `TileSetAtlasSource` no puede pertenecer a dos `TileSet`** (la warning oficial: se quita del otro). Enum verificados: `TileShape` SQUARE/ISOMETRIC/HALF_OFFSET_SQUARE/HEXAGON; `TileLayout` STACKED/STACKED_OFFSET/STAIRS_*/DIAMOND_*. |
| `CollisionShape2D` (`shape`, `one_way_collision`, `one_way_collision_direction (0,1)`, `one_way_collision_margin 1.0`, `disabled`, `debug_color`) | Existentes | Existentes (verificado en 4.7) | `one_way_collision` **no tiene efecto si el shape es hijo de un `Area2D`** (nota oficial). Margin: "Higher values … work better for colliders that enter the shape at a high velocity" (oficial). |
| `Camera2D` (`position_smoothing_*`, `rotation_smoothing_*`, `limit_*` ±10M + `limit_smoothed`, `zoom`, `offset`, `anchor_mode`, `drag_*`, `process_callback`, `custom_viewport`, `make_current`, `reset_smoothing`, `align`, `get_screen_center_position`, `get_screen_rotation`, `get_target_position`) | Existentes | Existentes (verificado en 4.7) | Una sola cámara activa **por viewport** (oficial). `global_position` ≠ posición real de pantalla (smoothing/limits) → `get_screen_center_position()`. `process_callback`: `PHYSICS (0)`/`IDLE (1)` (default IDLE). |
| `get_slide_collision_count()` (2D y 3D) | — | "Solo cuenta colisiones que cambiaron de dirección" (nota oficial, verificado 4.7) | No es un conteo de contactos: es el nº de rebotes del slide en el tick. |
| `AnimatableBody2D` / `AnimatableBody3D` | Existentes | Existentes (verificado 4.7) | Para cuerpos que se mueven por animación y deben **empujar**; `StaticBody*` movido por código se teleporta (no empuja). |

### Lote 6 (animación) — verificado 4.7 el 2026-09-15

| API | 4.0 | 4.7 (stable) | Diferencia / acción |
|---|---|---|---|
| `IKConstraint3D` / `InverseKinematics3D` (3.x) | — | **No existen en 4.7 (verificado 404)** | IK **removida** en el upgrade a 4.0 y **volvió en 4.6** (artículo oficial). La API 4.x = familia `SkeletonModifier3D`. No portar código 3.x de IK. |
| Familia `SkeletonModifier3D` (base + `active`/`influence` + `modification_processed`) | No existe (4.4 la introduce) | Existe (verificado 4.7) | Timeline oficial: base + `LookAtModifier3D` + `RetargetModifier3D` = **4.4**; `SpringBoneSimulator3D` + `BoneConstraint3D` (Aim/Copy/Convert) = **4.5**; `IKModifier3D` + 7 solvers, `BoneTwistDisperser3D`, `LimitAngularVelocityModifier3D` = **4.6**. El `influence` lo aplica el `Skeleton3D` (no en el código). |
| Solvers IK: `TwoBoneIK3D`, `ChainIK3D` (→ `SplineIK3D`, `IterateIK3D` → `FABRIK3D`/`CCDIK3D`/`JacobianIK3D`) | No existen (4.6) | Existentes (verificado 4.7) | `TwoBoneIK3D`/`SplineIK3D` siempre deterministas; `IterateIK3D` según `deterministic` (default `false`; `max_iterations 4`). `TwoBoneIK3D` requiere pole. Modificadores corren **post-`AnimationMixer`** (oficial). |
| `BoneConstraint3D` (reference bone **o Node3D**) | — | `Node3D` desde **4.6** (verificado) | En 4.5 solo refería bones; en 4.6+ puede referir un `Node3D` (twist/aiming post-IK). |
| `RetargetModifier3D` (retargeting runtime; `enable` flags, `profile`, `use_global_pose`) | No existe (4.4) | Existe (verificado 4.7) | Hijo del `Skeleton3D` destino; reescribe el pose en el update del skeleton. `use_global_pose=true` con bones unmapeados = problemas visuales (nota oficial). |
| Retargeting de **import** (`BoneMap` + `SkeletonProfile(Humanoid)`, Remove Tracks, Bone Renamer, Rest Fixer, `motion_scale`) | Existe desde 4.0 (artículo oficial) | Existe (verificado 4.7) | `SkeletonProfileHumanoid`: 56 bones, 4 groups, read-only, `root_bone "Root"`, `scale_base_bone "Hips"`. `Overwrite Axis` = "la opción más importante" (oficial). |
| Bone Pose **incluye** Bone Rest | Sí (rediseño 4.0) | Sí (verificado 4.7) | En 3.x era relativo al rest. Afecta cálculos a mano (IK custom, overrides) y retargeting. |
| `Skeleton3D.animate_physical_bones` | Existía | **Deprecada en 4.7 (verificado en la prop description)** | Workflow recomendado: `PhysicalBoneSimulator3D` hijo + su `.active`. Métodos `physical_bones_start/stop_simulation` siguen en la clase. |
| `Skeleton3D` props/métodos (`motion_scale 1.0`, `show_rest_only`, `modifier_callback_mode_process IDLE`, `get/set_bone_pose`, `get_bone_global_pose` = relativo al **skeleton**, señales `pose_updated`/`skeleton_updated`, `NOTIFICATION_UPDATE_SKELETON=50` deferred) | — (no re-verificado en 4.0) | Existentes (verificado en 4.7) | `pose_updated` NO detecta modificadores (nota oficial) → leer en `skeleton_updated`/`modification_processed`. |
| `AnimationNodeBlendSpace1D/2D` (antes `BlendSpace1D/2D` en 3.x) | — (renombrado en 4.0) | Existentes (verificado en 4.7) | En 4.x son `AnimationNode*` (`add_blend_point`). `SyncMode` tiene **4 valores** en 4.7 (`NONE/INDEPENDENT/CYCLIC_MUTABLE/CYCLIC_CONSTANT`); la prop `sync` es legacy (= `INDEPENDENT`). `cyclic_length > 0` para `CYCLIC_CONSTANT`. |
| `AnimationMixer` (base de AnimationPlayer/Tree; `deterministic`, `callback_mode_process`, root motion, señales) | 4.2 la introduce | Existe (verificado 4.7) | `AnimationPlayer` default `callback_mode_discrete = Recessive`; `AnimationTree` = `Force Continuous` (artículo oficial 4.0→4.3). |

## Migración Godot 3.x → 4.x (relacionado con esta biblioteca)

| 3.x | 4.x | Nota |
|---|---|---|
| `KinematicBody3D` | `CharacterBody3D` | Renombrado. |
| `move_and_slide(safe_margin)` | `move_and_slide()` + prop `safe_margin` | El argumento desapareció. |
| `floor_friction`, `wall_min_slide_speed` (props) | No existen | Ver fila de arriba. |
| `Input.MOUSE_MODE_CONFINED/CONFINED_HIDDEN` con otros nombres | `MOUSE_MODE_VISIBLE/HIDDEN/CAPTURED/CONFINED/CONFINED_HIDDEN` | Renombrados. |
| `Spatial` / `MeshInstance` / `AnimationNode*` viejas | `Node3D` / `MeshInstance3D` / `AnimationNode*` (misma familia, base `AnimationMixer` desde 4.x) | Ver artículo oficial de migración de animaciones. |
| `RayCast3D.cast_to` / `ShapeCast3D.cast_to` | `target_position` | Renombrado (verificado 4.7). |
| `PhysicsMaterial3D` / `PhysicsMaterial2D` (clases por dimensión, con `friction_combine_mode`/`bounce_combine_mode`) | `PhysicsMaterial` (clase única; combinación vía `rough`/`absorbent`) | La página `class_physicsmaterial3d.html` da 404 en 4.7 (verificado). |
| `RigidBody3D.friction` / `bounce` (props directas del body en 3.x) | `PhysicsMaterial` en `physics_material_override` | En 4.7 el material se aplica por body (verificado 4.7). |
| `PhysicsDirectSpaceState3D.test_motion` | `cast_motion` | Devuelve `[safe, unsafe]` (verificado 4.7). |
| `IKConstraint3D` / `InverseKinematics3D` (3.x) | Familia `SkeletonModifier3D` (solvers desde 4.6) | Removida en 4.0, volvió en 4.6 (artículo oficial). |
| `BlendSpace1D` / `BlendSpace2D` (recursos 3.x) | `AnimationNodeBlendSpace1D/2D` | Renombrado en 4.0; `add_blend_point`. |
| `AnimationNodeBlendSpace*.sync` (bool) | `sync_mode` (4 valores en 4.7) | `sync = true` ≙ `INDEPENDENT` (verificado 4.7). |

## Cómo mantener esta matriz

1. Al crear una skill, registrar aquí cualquier diferencia de versión encontrada durante la verificación (spec §46 paso 3-6).
2. Cuando cambie la rama `stable` (p. ej. 4.8), re-verificar las filas marcadas "confianza alta" y las skills *Verified*, y actualizar la columna de referencia.
3. Diferencia sin evidencia → marcar **"no verificado"** y no propagar a las skills.
