# SKILLS_GRAPH

> Matriz de dependencias entre skills (spec §49). **Lote 7** — 2026-09-15.
> `(P)` = skill pendiente de creación.

## Grafo (Lote 5)

```text
                        ┌────────────────────────────────────────────┐
                        │      godot-third-person-character          │
                        │  (compuesta, Verified 4.7, Lote 0)         │
                        └────────────────────────────────────────────┘
   ──────────────┬───────────────┬──────────────┬───────────────┬──────────────────┬─────────────────────┬──────────────┐
   ▼              ▼               ▼              ▼               ▼        ▼          ▼                    ▼
godot-      godot-          godot-       godot-       godot-    godot-   godot-      godot-
charac-     character-      input        animation-   camera3d  shader- rendering-   lod
terbody3d   controller                        tree             spatial  performance ◄┘
   ▲              │  │            ▲              │   ▲
   │              │  │            │              │   │
   │              │  └────────────┼──────────────┘   │
   │              │               │ (camera3d ← input│ (lod mide con
   │              │               │  orbit/zoom;     │  performance)
   │              │               │  camera3d ← body:│
   │              │               │  rid exclusión)  │
   │              ▼               ▼                   │
   │         (controller ← animationtree: consume     │
   │          landed/state)                           │
   └──────────────┘◄───────────────────────────────────┘
 (controller ← input: acciones;  controller ← terbody3d: move_and_slide)

 CLUSTER FÍSICA (Lote 4, todas Verified 4.7):

   godot-physics  ◄──────────────────────────────────────────────┐
    ▲    ▲    ▲    ▲                                              │
    │    │    │    │      (monitores de física: los dos miden)    │
    │    │    │    │                                               │
    │    │    │    └──────── godot-raycast3d  ─────┐               │
    │    │    │              (queries: layers/masks, _physics_process)
    │    │    └───────────── godot-area3d ─────────┤
    │    │                (detección/influencia: layers/masks, ticks)
    │    └──────────────── godot-physics-materials ─┤
    │                 (override en Static/Rigid: damping elástico)
    ├─── godot-characterbody3d  (contexto de ticks/shapes/layers)
    └─── godot-platformer-2d (2D: misma física, CharacterBody2D)
                                  godot-rendering-performance ◄─────────────────────────────┘
 (pair count / FPS se diagnostican en ambas skills)

 CLUSTER 2D (Lote 5, todas Verified 4.7):

   godot-node2d  (raíz 2D: transform, local/global, Y abajo)
    ▲       ▲        ▲
    │       │        │
    │       │        └──── godot-camera2d  (la cámara es un Node2D;
    │       │                  get_screen_center_position ≠ global_position)
    │       │
    │       └─────────── godot-tilemap  (TileMapLayer es Node2D;
    │                    local_to_map/map_to_local; ← godot-physics
    │                    (physics layer del TileSet) + godot-physics-materials)
    │
    └──────────────── godot-platformer-2d  (CharacterBody2D es Node2D;
                     ← godot-physics (ticks/_physics_process)
                     ← godot-tilemap (el level)
                     ← godot-camera2d (follow; process_callback PHYSICS)
                     ← godot-character-controller (feel: coyote/buffer)
                     ← godot-physics-materials (fricción de superficies)

 CLUSTER ANIMACIÓN (Lote 6, todas Verified 4.7):

   godot-animationtree  (Lote 3: StateMachine, OneShot, BlendTree wiring)
    ▲
    │ (los BlendSpaces se cablean en el BlendTree: "output", parameters/...)
    │
   godot-blendspace  (BlendSpace1D/2D: blend_position, sync modes, triángulos)
    │
    │ (los clips animan bones)
    ▼
   godot-skeleton3d  (bones, pose vs rest, signals, physical bones)
    ▲                          ▲
    │ (modificadores hijos;     │ (misma familia; motion_scale)
    │  post-AnimationMixer)     │
   godot-ik  ────────────────── godot-retargeting
   (TwoBone/Chain/FABRIK/CCD/   (BoneMap import + RetargetModifier3D runtime)
    Jacobian/Spline + constraints)
   └─ godot-ik ← godot-character-controller (el target suele ser el player)
   └─ godot-ik ← godot-raycast3d (proyectar targets al suelo)
   └─ godot-retargeting ← godot-character-controller (motion_scale/root motion)

 CLUSTER UI (Lote 7, todas Verified 4.7):

   godot-control  (raíz UI: anchors/offsets, sizing, _gui_input, focus,
                  theme overrides, pivot_offset — el otro hijo de CanvasItem)
    ▲              ▲                  ▲
    │              │                  │
    │              │                  └── godot-localization
    │              │                      (auto_translate, translation_context,
    │              │                       layout_direction, tr/tr_n, RTL)
    │              │
    │              └──────── godot-containers  (los hijos ceden su
    │                                  posicionamiento; sizing options;
    │                                  13 types; nesting = editor de Godot)
    │
    └────────────── godot-theme  (Theme resource, cascada, items,
                                  overrides NO son Object properties)
   └─ godot-control ← godot-camera2d (el HUD en CanvasLayer no se mueve
      con la cámara) + godot-node2d (comparten CanvasItem: z_index/visible)
```

### De la compuesta hacia las bases (Lote 0, ya materializadas)

| Skill | Depende de | Tipo | Justificación |
|---|---|---|---|
| `godot-third-person-character` | `godot-characterbody3d` | obligatoria | Cuerpo físico y `move_and_slide` |
| `godot-third-person-character` | `godot-character-controller` | obligatoria | Lógica de velocidad, salto, tuning |
| `godot-third-person-character` | `godot-input` | obligatoria | Acciones, `get_vector`, captura de ratón |
| `godot-third-person-character` | `godot-animationtree` | obligatoria | Locomoción y eventos de animación |
| `godot-third-person-character` | `godot-camera3d` | obligatoria | Cámara orbital + `SpringArm3D` |
| `godot-third-person-character` | `godot-shader-spatial` | opcional | Aspecto/destacado del personaje |
| `godot-third-person-character` | `godot-rendering-performance` | opcional | Medición y optimización |
| `godot-third-person-character` | `godot-lod` | opcional | LOD del personaje lejano (multiplayer) y del mundo |
| `godot-third-person-character` | `godot-physics` / `godot-physics-materials` / `godot-raycast3d` / `godot-area3d` | opcional | Capas, materiales, raycasts y detección del mundo (Lote 4) |

### Entre skills base (Lote 3)

| Skill | Depende de | Tipo | Justificación |
|---|---|---|---|
| `godot-character-controller` | `godot-characterbody3d` | obligatoria | Escribe `velocity` y lee el estado post-`move_and_slide` |
| `godot-character-controller` | `godot-input` | obligatoria | Acciones (`get_vector`, `just_pressed`, `get_joy_axis`) |
| `godot-character-controller` | `godot-camera3d` | opcional | Inyectar el yaw de la cámara (TPS) |
| `godot-character-controller` | `godot-animationtree` | opcional | Consumidor de `landed()`/`wall_slide_changed()` (sentido inverso: la animación consume al controlador) |
| `godot-camera3d` | `godot-input` | obligatoria | `_unhandled_input` (motion/rueda) + `get_joy_axis` (stick derecho) |
| `godot-camera3d` | `godot-characterbody3d` | opcional | `get_rid()` para excluir al jugador del cast del brazo |
| `godot-animationtree` | `godot-characterbody3d` | opcional | `callback_mode_process = PHYSICS` alinea anim y física |
| `godot-lod` | `godot-rendering-performance` | obligatoria | Medir los ahorros (regla §60) |
| `godot-rendering-performance` | `godot-lod` | opcional | Las palancas de distancia (visibility ranges, mesh LOD) |
| `godot-rendering-performance` | `godot-shader-spatial` | opcional | El coste de fragment/overdraw se aplica en el shader |
| `godot-characterbody3d` | `godot-physics` | base compartida | Capas, shapes, tipos de cuerpo (Lote 4) |
| `godot-characterbody3d` | `godot-physics-materials` | base compartida | Fricción/rebote de superficies (Lote 4) |
| `godot-shader-spatial` | `godot-rendering-performance` | opcional | El coste de fragment se mide con ella |

### Cluster física (Lote 4)

| Skill | Depende de | Tipo | Justificación |
|---|---|---|---|
| `godot-physics` | (ninguna) | raíz del cluster | Fundacional: tipos de cuerpo, layers/masks, ticks, troubleshooting |
| `godot-raycast3d` | `godot-physics` | obligatoria | Layers/masks, regla de `_physics_process` (space locked), debug shapes |
| `godot-area3d` | `godot-physics` | obligatoria | Layers/masks, ciclo de ticks, `disable_mode` |
| `godot-physics-materials` | `godot-physics` | obligatoria | `physics_material_override` en Static/Rigid; receta del damping elástico |
| `godot-area3d` | `godot-raycast3d` | opcional (cruce) | El overlap instantáneo que Area no da (`ShapeCast3D` target 0 + `force_shapecast_update`) |
| `godot-raycast3d` | `godot-characterbody3d` | opcional (cruce) | Consumidor principal (ray de piso/escalera/muro) |
| `godot-rendering-performance` | `godot-physics` | opcional (cruce) | Diagnóstico conjunto del FPS: monitores de física (20/21) + los de render |

### Cluster 2D (Lote 5)

| Skill | Depende de | Tipo | Justificación |
|---|---|---|---|
| `godot-node2d` | (ninguna) | raíz 2D | Transform 2D, local/global, Y abajo, `to_global/to_local` |
| `godot-tilemap` | `godot-node2d` | obligatoria | `TileMapLayer` es `Node2D`; `local_to_map`/`map_to_local` |
| `godot-tilemap` | `godot-physics` | obligatoria | Physics layers del `TileSet` (collision_layer/mask/priority) |
| `godot-tilemap` | `godot-physics-materials` | opcional | `PhysicsMaterial` por physics layer del TileSet |
| `godot-platformer-2d` | `godot-node2d` | obligatoria | `CharacterBody2D` es `Node2D` (Y abajo) |
| `godot-platformer-2d` | `godot-physics` | obligatoria | `_physics_process` (space locked), layers/masks, troubleshooting |
| `godot-platformer-2d` | `godot-tilemap` | obligatoria | El level (physics de tiles) |
| `godot-platformer-2d` | `godot-character-controller` | opcional | El feel (coyote/buffer/air control) sobre la base física |
| `godot-platformer-2d` | `godot-camera2d` | opcional (cruce) | Follow del player (`process_callback = PHYSICS`) |
| `godot-platformer-2d` | `godot-physics-materials` | opcional | Fricción/rebote de superficies 2D |
| `godot-camera2d` | `godot-node2d` | obligatoria | La cámara es `Node2D` (canvas, transform) |

### Cluster UI (Lote 7)

| Skill | Depende de | Tipo | Justificación |
|---|---|---|---|
| `godot-control` | (ninguna) | raíz UI | Anchors/offsets, sizing, input GUI, focus, overrides |
| `godot-containers` | `godot-control` | obligatoria | Los hijos son `Control`; sizing options = props de `Control` |
| `godot-theme` | `godot-control` | obligatoria | Overrides y `theme_type_variation` viven en `Control` |
| `godot-localization` | `godot-control` | obligatoria | `auto_translate`/`translation_context`/`layout_direction` son props de `Control` |
| `godot-control` | `godot-camera2d` | opcional (cruce) | El HUD en `CanvasLayer` no se mueve con la cámara |
| `godot-control` | `godot-node2d` | opcional (cruce) | El otro hijo de `CanvasItem` (comparten `z_index`/`visible` — oficial) |
| `godot-containers` | `godot-theme` | obligatoria | Los márgenes/padding son constantes del tema (oficial) |
| `godot-localization` | `godot-theme` | obligatoria | La fuente multilingüe (DynamicFont) vive en el Theme (oficial) |
| `godot-localization` | `godot-containers` | opcional (cruce) | El reflow con strings largos |

### Cluster animación (Lote 6)

| Skill | Depende de | Tipo | Justificación |
|---|---|---|---|
| `godot-skeleton3d` | (ninguna del lote) | raíz del cluster | Huesos, pose vs rest, señales, physical bones |
| `godot-blendspace` | `godot-animationtree` | obligatoria | Los BlendSpaces se cablean en el `AnimationNodeBlendTree` (nodo "output", `parameters/...`) |
| `godot-blendspace` | `godot-character-controller` | opcional | El controller produce el `blend_position` (velocidad/dirección) |
| `godot-ik` | `godot-skeleton3d` | obligatoria | Los modificadores son **hijos del `Skeleton3D`** (oficial); corren post-`AnimationMixer` |
| `godot-ik` | `godot-animationtree` | base compartida | La animación corre antes; el IK ajusta después |
| `godot-ik` | `godot-character-controller` | opcional (cruce) | El target suele ser el player |
| `godot-ik` | `godot-raycast3d` | opcional (cruce) | Proyectar targets al suelo (pies) |
| `godot-retargeting` | `godot-skeleton3d` | obligatoria | El skeleton destino (rest, `motion_scale`) |
| `godot-retargeting` | `godot-ik` | base compartida | Misma familia de modificadores; ajustes post-retarget |
| `godot-retargeting` | `godot-character-controller` | opcional (cruce) | `motion_scale` y root motion |
| `godot-animationtree` | `godot-blendspace` | (cruce inverso) | La skill de Lote 3 referencia la de mezcla para el dominio profundo |

### Hacia pendientes (referenciados desde las bases)

| Skill | Depende de (P) | Tipo |
|---|---|---|
| `godot-input` | `godot-input-touch` (Lote 20) | extensión |
| `godot-camera3d` | `godot-cinematic-camera` (Lote 0 ext.) / `godot-xr` (Lote 20) | extensión |
| `godot-shader-spatial` | `godot-shader-canvasitem` (Lote 9) / `godot-standardmaterial3d` (Lote 8) | alternativas |
| `godot-animationtree` | `godot-recipe-locomotion-blend` (Lote 18) / `godot-recipe-root-motion` (Lote 18) | recetas |
| `godot-blendspace` | `godot-recipe-locomotion-blend` (Lote 18) | receta |
| `godot-ik` | `godot-physicalbones` / `godot-springbones` (Lote 6 ext.) | simuladores de la misma familia |
| `godot-skeleton3d` | `godot-physicalbones` (Lote 6 ext.) | ragdoll en detalle |
| `godot-retargeting` | `godot-recipe-mixamo-pipeline` (Lote 18) | receta |
| `godot-physics` | `godot-vehiclebody3d` (Lote 4 ext.) | extensión |
| `godot-physics` | `godot-softbody3d` (fuera de scope declarado) | — |
| `godot-raycast3d` | `godot-raycast2d` (Lote 5 ext.) | espejo |
| `godot-area3d` | `godot-area2d` (Lote 5 ext.) / `godot-audio-buses` (Lote 11) | espejo / puente audio |
| `godot-physics-materials` | `godot-physics-materials-2d` (Lote 5 ext.) | espejo |
| `godot-node2d` | `godot-canvasitem` (Lote 9) | `z_index`+drawing (el otro hijo de `CanvasItem`) |
| `godot-tilemap` | `godot-navigation2d` (Lote 10) / `godot-light2d` (Lote 11) | navigation layers / occlusion layers |
| `godot-platformer-2d` | `godot-area2d` (Lote 5 ext.) | pickups/zonas del platformer |

## Reglas del grafo

1. Las skills **base** no dependen de compuestas (evita ciclos).
2. Una compuesta lista como *obligatorias* solo las bases sin las que el sistema no funciona; el resto son *opcionales* (aumentan calidad/escala).
3. Al crear una skill base pendiente, actualizar: (a) este grafo, (b) `GODOT_SKILLS_INDEX.md` (estado → Verified), (c) la sección *Dependencias* de las compuestas que la usan.
4. Si dos skills comparten contenido, la base lo posee y las compuestas lo referencian (spec §54 — evitar duplicación).
5. Las aristas **opcionales** se cargan solo cuando la petición toca ese subsistema (el router de `AGENT_ROUTING.md` decide).
6. `godot-physics` es raíz del cluster de física (2D **y** 3D): el contenido compartido (layers/masks, ticks, troubleshooting) se escribe UNA vez ahí y las skills de ambos mundos lo referencian.
7. `godot-node2d` es raíz del cluster 2D: el contenido compartido 2D (transform, local/global, Y abajo) se escribe UNA vez ahí.
