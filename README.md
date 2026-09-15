# ARENASKILLS

**Godot Engine Development Knowledge System** — biblioteca de skills técnicas para un agente de programación especializado en desarrollo de videojuegos con **Godot 4.x** (fuente de verdad: `docs.godotengine.org/en/stable/`, rama **4.7**).

## Qué es

Una colección de skills pequeñas, independientes, reutilizables y combinables (una skill = una capacidad) que permiten a un agente de código: consultar antes de implementar, elegir APIs, diseñar arquitectura, escribir GDScript 4.x, diagnosticar errores, optimizar midiendo y detectar anti-patrones.

## Estructura

```text
ARENASKILLS/
├── README.md                    ← este archivo
├── GODOT_SKILLS_INDEX.md        ← índice global (categorías, IDs, niveles, estados)
├── AGENT_ROUTING.md             ← cómo decide el agente qué skills cargar
├── SKILLS_GRAPH.md              ← matriz/grafo de dependencias entre skills
├── GODOT_VERSION_MATRIX.md      ← diferencias de API verificadas entre versiones
└── skills/
    ├── godot-third-person-character.md   ← Lote 0: skill compuesta (personaje 3D en 3ª persona)
    ├── godot-characterbody3d.md          ← Lote 3: base — cuerpo físico kinemático
    ├── godot-character-controller.md     ← Lote 3: base — patrón de control/feel de movimiento
    ├── godot-input.md                    ← Lote 3: base — Input Map, polling, ratón, gamepad
    ├── godot-animationtree.md            ← Lote 3: base — estados, blend, one-shots, root motion
    ├── godot-camera3d.md                 ← Lote 3: base — cámara 3D, spring arm, proyección
    ├── godot-shader-spatial.md           ← Lote 3: base — shaders 3D, uniforms, sRGB
    ├── godot-rendering-performance.md    ← Lote 3: base — medir/diagnosticar/optimizar rendering
    ├── godot-lod.md                      ← Lote 3: base — LOD, visibility ranges, impostor
    ├── godot-physics.md                  ← Lote 4: base — tipos de cuerpo, layers, ticks, troubleshooting
    ├── godot-raycast3d.md                ← Lote 4: base — RayCast3D, ShapeCast3D, intersect_ray
    ├── godot-area3d.md                   ← Lote 4: base — detección/influencia, triggers, overrides
    └── godot-physics-materials.md        ← Lote 4: base — PhysicsMaterial (fricción/rebote)
```

## Estado actual

**LOTE 4** (2026-09-15) — infraestructura + skill compuesta + 12 skills base:

- ✅ `godot-third-person-character` (compuesta, Verified 4.7): integra las bases.
- ✅ 8 skills base (Lote 3, Verified 4.7): `godot-characterbody3d`, `godot-character-controller`, `godot-input`, `godot-animationtree`, `godot-camera3d`, `godot-shader-spatial`, `godot-rendering-performance`, `godot-lod`.
- ✅ 4 skills base (Lote 4, Verified 4.7): `godot-physics`, `godot-raycast3d`, `godot-area3d`, `godot-physics-materials` — cierran las referencias pendientes de `godot-physics` en las skills de Lote 0/3.
- ⏳ Resto de lotes (1, 2, 5–20) pendientes — ver plan en `GODOT_SKILLS_INDEX.md`.

## Disciplina de la biblioteca

- APIs **solo** verificadas contra la documentación oficial 4.7; lo dudoso se marca "API no verificada"; **nunca inventar APIs**.
- Cada skill: formato obligatorio (ID, categoría, versión, propósito, cuándo/no usar, API, arquitectura, implementaciones, errores, anti-patrones, performance, debugging, compatibilidad, dependencias, referencias).
- Confianza explícita (HIGH/MEDIUM/LOW) y nivel (BEGINNER→EXPERT) por skill.
- Rendimiento: **MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN**.
- Generación incremental por lotes; cada lote reporta lo creado, lo verificado y lo pendiente.

## Cómo usarla (flujo del agente)

`USER REQUEST → ANALYZE → SEARCH (GODOT_SKILLS_INDEX.md) → LOAD (skills + AGENT_ROUTING.md) → CHECK DEPENDENCIES (SKILLS_GRAPH.md) → CONSULT DOCS IF NEEDED → PLAN → IMPLEMENT → TEST → DEBUG → OPTIMIZE ONLY IF NECESSARY → VERIFY`
