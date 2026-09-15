# SKILLS_GRAPH

> Matriz de dependencias entre skills (spec §49). **Lote 4** — 2026-09-15.
> `(P)` = skill pendiente de creación.

## Grafo (Lote 4)

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
    └─── godot-characterbody3d  (contexto de ticks/shapes/layers)
                                  │
        godot-rendering-performance ◄─────────────────────────────┘
        (pair count / FPS se diagnostican en ambas skills)
```

## Aristas (qué necesita de qué)

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

### Hacia pendientes (referenciados desde las bases)

| Skill | Depende de (P) | Tipo |
|---|---|---|
| `godot-input` | `godot-input-touch` (Lote 20) | extensión |
| `godot-camera3d` | `godot-cinematic-camera` (Lote 0 ext.) / `godot-xr` (Lote 20) | extensión |
| `godot-shader-spatial` | `godot-shader-canvasitem` (Lote 9) / `godot-standardmaterial3d` (Lote 8) | alternativas |
| `godot-animationtree` | `godot-root-motion` (Lote 6) / `godot-recipe-locomotion-blend` (Lote 6) | extensión |
| `godot-physics` | `godot-vehiclebody3d` (Lote 4 ext.) | extensión |
| `godot-physics` | `godot-softbody3d` (fuera de scope declarado) | — |
| `godot-raycast3d` | `godot-raycast2d` (Lote 5) | espejo |
| `godot-area3d` | `godot-area2d` (Lote 5) / `godot-audio-buses` (Lote 11) | espejo / puente audio |
| `godot-physics-materials` | `godot-physics-materials-2d` (Lote 5) | espejo |

## Reglas del grafo

1. Las skills **base** no dependen de compuestas (evita ciclos).
2. Una compuesta lista como *obligatorias* solo las bases sin las que el sistema no funciona; el resto son *opcionales* (aumentan calidad/escala).
3. Al crear una skill base pendiente, actualizar: (a) este grafo, (b) `GODOT_SKILLS_INDEX.md` (estado → Verified), (c) la sección *Dependencias* de las compuestas que la usan.
4. Si dos skills comparten contenido, la base lo posee y las compuestas lo referencian (spec §54 — evitar duplicación).
5. Las aristas **opcionales** se cargan solo cuando la petición toca ese subsistema (el router de `AGENT_ROUTING.md` decide).
6. `godot-physics` es raíz del cluster: el contenido compartido (layers/masks, ticks, debug shapes) se escribe UNA vez ahí y las skills del cluster lo referencian (no lo duplican).
