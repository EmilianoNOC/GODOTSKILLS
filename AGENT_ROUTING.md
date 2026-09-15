# AGENT_ROUTING

> Guía de enrutamiento para agentes LLM: qué skills cargar para cada tipo de petición, y qué NO usar.
> **Estado: LOTE 5** — 2026-09-15. API referenciada: Godot **4.7 (stable)**.

## Cómo funciona

1. El agente identifica el **subsistema** de la petición (3D characters, animation, input, cámara 3D, shaders 3D, performance, física, 2D, mapas…).
2. Carga la **skill base** correspondiente; si la petición es de alto nivel ("hazme un personaje"), carga la **compuesta**, que orquesta las bases.
3. Carga las **opcionales** solo si la petición toca ese subsistema (no cargar todo).
4. Si la API no aparece verificada en la skill, **no se inventa**: se marca "API no verificada" y se busca en la docs oficial (rama actual).

## Tabla de enrutamiento

### Petición de alto nivel

| Si la petición dice… | Cargar |
|---|---|
| "hazme un personaje 3D de acción / tercera persona" | `godot-third-person-character` (compuesta) + las 5 obligatorias que lista |
| "hazme un platformer 2D" | `godot-platformer-2d` + `godot-tilemap` + `godot-node2d` (+ `godot-camera2d` para el follow) |
| "hazme un top-down 2D" | `godot-node2d` + `godot-physics` (`MOTION_MODE_FLOATING`) + `godot-tilemap` (+ `godot-area2d` pendiente) |

### Por subsistema (skills base, todas Verified 4.7)

| Subsistema | Petición típica | Skill |
|---|---|---|
| Cuerpo 3D | "move_and_slide no funciona", "el personaje atraviesa", "floor_snap" | `godot-characterbody3d` |
| Feel de control | "coyote time", "air control", "wall slide", "saltar alto con mantención" | `godot-character-controller` |
| Input | "mapear teclas", "gamepad", "capturar ratón", "remapeo en runtime" | `godot-input` |
| Animation | "AnimationTree", "estado de locomoción", "one-shot", "root motion" | `godot-animationtree` |
| Cámara 3D | "cámara orbital", "spring arm", "FOV", "transición entre cámaras" | `godot-camera3d` |
| Shaders 3D | "escribir shader", "render mode", "uniform", "sRGB", "per-instance" | `godot-shader-spatial` |
| Performance | "pocos FPS", "diagnosticar", "VRAM", "draw calls" | `godot-rendering-performance` |
| LOD | "LOD", "impostor", "visibility range" | `godot-lod` |
| Física (base) | "capas de colisión", "RigidBody vs Static", "tunneling", "la física no es determinista" | `godot-physics` |
| Queries 3D | "raycast", "picking con mouse", "shapecast", "cast_from" | `godot-raycast3d` |
| Detección 3D | "pickup", "trigger", "zona de daño", "gravedad local" | `godot-area3d` |
| Materiales físicos | "piso resbaladizo", "rebotar", "fricción" | `godot-physics-materials` |
| **2D (base)** | "Node2D", "transform 2D", "local vs global", "rotar 2D" | `godot-node2d` |
| **Mapas 2D** | "tilemap", "mapa por tiles", "terreno", "isométrico", "datos por tile" | `godot-tilemap` |
| **Juego 2D** | "platformer", "saltos", "plataforma una-vía", "plataformas móviles" | `godot-platformer-2d` |
| **Cámara 2D** | "follow del player", "límites de cámara", "zoom 2D", "lookahead" | `godot-camera2d` |

### Cargas opcionales (solo si la petición lo toca)

| Si además dice… | Cargar también |
|---|---|
| "el personaje debe ver bien / destacarse" | `godot-shader-spatial` |
| "el personaje lejano en un mundo grande / multiplayer" | `godot-lod` |
| "que el mundo tenga colisiones / pickups / zonas" | `godot-physics` + `godot-area3d` (+ `godot-raycast3d` para line-sight/picking) |
| "pisos resbaladizos / reboteros" | `godot-physics-materials` |
| "el level es por tiles" | `godot-tilemap` (+ `godot-physics` para las physics layers del TileSet) |
| "la cámara siga al player 2D sin jitter" | `godot-camera2d` (`process_callback = PHYSICS`) + `godot-platformer-2d` |
| "quiero que salte/siga con buen feel" | `godot-character-controller` (el feel, 2D y 3D) |

### Reglas de NO-carga (evitar sobrecarga)

- **No cargar `godot-shader-spatial`** si la petición es solo de movimiento/cámara (no agrega valor).
- **No cargar `godot-lod`** si el mundo es pequeño/cerrado (LOD no aplica).
- **No cargar `godot-physics`** si la petición es solo de UI o solo de animación de personaje ya construido (el cuerpo ya existe).
- **No cargar `godot-raycast3d`/`godot-area3d`** si no hay world interaction en la petición.
- **No cargar `godot-tilemap`** si el level 2D es por nodos/mesh (sin tiles).
- **No cargar `godot-camera2d`** para una escena 3D (y viceversa con `godot-camera3d`).

## Ejemplos de enrutamiento

1. **"Quiero un personaje 3D que corra, salte y tenga cámara orbital"**
   → `godot-third-person-character` (compuesta) → arrastra a `characterbody3d`, `character-controller`, `input`, `animationtree`, `camera3d`. Opcionales: `shader-spatial` (visual) + `lod` (mundo grande). No cargar: `rendering-performance` salvo que haya un problema de FPS reportado.

2. **"El personaje atraviesa la pared cuando corre rápido"**
   → `godot-physics` (tunneling: ticks, CCD) + `godot-characterbody3d` (safe_margin, `up_direction`). No cargar: shaders, LOD, input.

3. **"Hacer un platformer 2D con plataformas que cruzás por abajo"**
   → `godot-platformer-2d` (una-vía, `move_and_slide`) + `godot-tilemap` (el level) + `godot-node2d` (base). Opcional: `godot-camera2d` (follow). No cargar: skills 3D.

4. **"La cámara 2D tiembla al seguir al player"**
   → `godot-camera2d` (process_callback PHYSICS, smoothing) + `godot-platformer-2d` (body físico). No cargar: nada 3D.

5. **"El juego va a 15 FPS en el level grande"**
   → `godot-rendering-performance` (flujo de diagnóstico, §60) + `godot-lod` (si hay mundo grande) + `godot-physics` (si son cuerpos: pair count). No cargar: input, animation.

6. **"Piso de hielo / trampolín que lanza al personaje"**
   → `godot-physics-materials` (materiales, override) + `godot-physics` (damping elástico). No cargar: shaders.

## Si la skill no existe aún (lotes 1, 2, 6–20)

| Tema | Estado | Qué hacer mientras |
|---|---|---|
| Editor, project settings, lifecycle, señales (Lote 1) | Pendiente | Usar la docs oficial 4.7; no afirmar APIs sin verificar |
| Sintaxis GDScript, clases, await, @tool (Lote 2) | Pendiente | Usar la docs oficial 4.7 |
| BlendSpace, IK, retargeting, Skeleton3D (Lote 6) | Pendiente | `godot-animationtree` cubre lo básico (AnimationNode, callbacks) |
| UI/containers/theme (Lote 7) | Pendiente | Docs oficial 4.7 |
| Renderers, MSAA, sombras, culling (Lote 8) | Pendiente | `godot-rendering-performance` cubre el diagnóstico (no la config) |
| Shaders canvasitem/post-process (Lote 9) | Pendiente | `godot-shader-spatial` (3D) es el patrón de shader; el 2D difiere |
| Navigation2D/3D, IA (Lote 10) | Pendiente | `godot-tilemap` cubre los navigation layers (pintar), no la navegación |
| Audio (Lote 11) | Pendiente | Docs oficial 4.7 |
| Networking (Lote 12) | Pendiente | Docs oficial 4.7 (MultiplayerAPI, RPC) |
| Save/load, inventarios (Lote 13) | Pendiente | Docs oficial 4.7 |
| Pooling, threading, profiling (Lote 14) | Pendiente | `godot-rendering-performance` (medir) + `godot-physics` (ticks) |
| Export (Lote 15) | Pendiente | Docs oficial 4.7 |
| Editor plugins (Lote 16) | Pendiente | Docs oficial 4.7 |
| Anti-patrones (Lote 17) | Pendiente | Cada skill tiene su § Anti-patrones + § Errores frecuentes |
| Recetas (Lote 18) | Pendiente | Los gérmenes están en las skills base (ver § Skills relacionadas) |
| 3D avanzado (Lote 19) | Pendiente | `godot-lod` + `godot-rendering-performance` como base |
| Procedural/testing/mobile/web/VR (Lote 20) | Pendiente | Docs oficial 4.7 |

## Regla de oro (spec §19)

**APIs no inventadas.** Si una API no aparece verificada en la skill (o la skill la marca "API no verificada"), el agente **no la usa** como si existiera: se verifica en la docs oficial de la rama actual antes, o se declara "API no verificada" en el código entregado. Lista consolidada: `GODOT_SKILLS_INDEX.md` → *APIs "no verificadas" declaradas*.

## Actualización

Al crear una skill (lotes pendientes):
1. Añadir su fila en las tablas de enrutamiento.
2. Revisar la sección "Si la skill no existe aún" y quitar la fila correspondiente.
3. Ajustar las reglas de NO-carga si la nueva skill solapa.
4. Bump el estado del header a "LOTE N".
