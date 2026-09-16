# godot-skeleton3d

## ID

`godot-skeleton3d`

## Categoría

Animation · Base (esqueleto)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Skeleton3D` (tabla completa: props/métodos/señales/enums) (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — toda la tabla de `Skeleton3D` 4.7: props (`animate_physical_bones true` — **deprecada** —, `modifier_callback_mode_process 1=IDLE`, `motion_scale 1.0`, `show_rest_only false`), ~45 métodos (bones, pose, rest, overrides, meta, physical bones, skin, version), señales (`bone_enabled_changed`, `bone_list_changed`, `pose_updated`, `rest_updated`, `show_rest_only_changed`, `skeleton_updated`), enum `ModifierCallbackModeProcess` (PHYSICS=0/IDLE=1/MANUAL=2), constant `NOTIFICATION_UPDATE_SKELETON = 50`, "global pose" = relativo al skeleton (no world) — verificados directamente en la docs 4.7.
**MEDIUM** — valores de uso (qué bone habilitar/deshabilitar, cuándo `force_update`).
**Diferencia 3.x→4.x (verificada)**: Bone Pose **incluye** Bone Rest en 4.x (en 3.x era relativo).

## Nivel

**INTERMEDIATE** (depende de `godot-animationtree`; la usan `godot-ik`, `godot-retargeting`).

## Propósito

Proporcionar el conocimiento operativo de `Skeleton3D`: la jerarquía de bones (add/find/parent/name), la distinción **pose vs rest** (y el cambio 3.x→4.x: el pose **incluye** el rest), "global pose" (relativo al skeleton, **no** world), overrides de pose, meta de bones, señales de update (y la nota de `pose_updated` vs `skeleton_updated`), physical bones (prop deprecada + `PhysicalBoneSimulator3D`), `motion_scale` (del retarget), y el ciclo de update del skeleton (modificadores post-`AnimationMixer`). Es la skill "qué es el esqueleto y cómo se consulta/mutúa" — no IK, no retarget, no blending.

## Cuándo utilizarla

- Consultar/consultar bones (índices, nombres, padres, poses, rests).
- Mover un bone a mano (pose/override) — p. ej. "el brazo sale del cuerpo al morir".
- Leer el pose **después** de los modificadores (señal correcta).
- Physical bones (ragdoll) — qué API existe en 4.7.
- Debug de "el skeleton no se actualiza / la piel no sigue".

## Cuándo NO utilizarla

- **IK / constraints / twist** → `godot-ik` (los modificadores son hijos del skeleton, pero el know-how es de esa skill).
- **Retargeting** → `godot-retargeting` (BoneMap/SkeletonProfile/`RetargetModifier3D`).
- **Blending / state machine** → `godot-animationtree`/`godot-blendspace` (los clips animan bones, pero la mezcla no es del skeleton).
- **2D** → `Skeleton3D` es 3D; en 2D: `Bone2D` (existe en 4.7, verificado en el índice) — diferente API.
- **Física de huesos en detalle** (`PhysicalBoneSimulator3D`, `SpringBoneSimulator3D`) → skills pendientes (Lote 6 ext.); aquí solo el anclaje.

## Conceptos fundamentales

### Hueso, rest y pose (el modelo de datos)

- **`Skeleton3D`** = "a node containing a bone hierarchy, used to create a 3D skeletal animation" (oficial).
- **Rest** = "The overall transform of a bone with respect to the skeleton is determined by bone pose. **Bone rest defines the initial transform of the bone pose**" (oficial). Rest = el "T-pose" importado (o el default pose del asset).
- **Pose** = el transform actual del bone.
- ⚠️ **3.x vs 4.x (oficial, verificado)**: "In Godot 3, Bone Pose is relative to Bone Rest, but in Godot 4, **it includes Bone Rest**." → todo cálculo a mano (IK custom, overrides) debe partir de eso.
- **`Bone Pose == Bone Rest`** ⇒ el skeleton está en el default pose (oficial).

### "Global pose" ≠ world

- Oficial: "**Note that 'global pose' below refers to the overall transform of the bone with respect to the skeleton, so it is not the actual global/world transform of the bone**."
- Métodos: `get_bone_global_pose(bone_idx)` (+ `_no_override`, `_override`), `get_bone_global_rest(bone_idx)`, `set_bone_global_pose(bone_idx, pose)`.
- Para el **mundo**: componer con el transform del nodo (el skeleton es un `Node3D`).

### El ciclo de update (y cuándo leer)

- Si hay `AnimationMixer` (AnimationPlayer/Tree), la animación corre primero; **los modificadores (`SkeletonModifier3D`) corren después** (oficial, ver `godot-ik`).
- Señales (verificadas):
  - `pose_updated()` — "Emitted when the pose is updated. **Note: During the update process, this signal is not fired, so modification by SkeletonModifier3D is not detected**."
  - `skeleton_updated()` — "Emitted when the final pose has been calculated **will be applied to the skin** in the update process. This means that all SkeletonModifier3D processing is complete. In order to detect the completion of the processing of each SkeletonModifier3D, use SkeletonModifier3D.modification_processed."
  - → **Para leer el pose final (con IK/retarget aplicados): `skeleton_updated()` (o `modification_processed()` del modificador específico, ver `godot-ik`).**
- `modifier_callback_mode_process` (default `1` = `MODIFIER_CALLBACK_MODE_PROCESS_IDLE`): cuándo procesan los modificadores (`PHYSICS=0`/`IDLE=1`/`MANUAL=2` — verificado); `advance(delta)` para manual.

### Physical bones (ragdoll)

- La prop `animate_physical_bones` (default `true`) está **deprecada** (verificado en la prop description 4.7): "may be changed or removed in future versions. If you follow the recommended workflow and explicitly have **PhysicalBoneSimulator3D** as a child of Skeleton3D, you can control whether it is affected by raycasting… by its SkeletonModifier3D.active…"
- Métodos de la clase (verificados): `physical_bones_start_simulation(bones = [])`, `physical_bones_stop_simulation()`, `physical_bones_add/remove_collision_exception(rid)`.
- → Workflow recomendado: un `PhysicalBoneSimulator3D` explícito como hijo (skill pendiente); los métodos del `Skeleton3D` siguen funcionando.

### `motion_scale` (del retarget)

- Prop `motion_scale` (default `1.0` — verificada): la altura de `scale_base_bone` (Hips en Humanoid) que se importa con **Normalize Position Tracks** (oficial, ver `godot-retargeting`); los Position tracks normalizados se **multiplican** por ese valor en playback.
- No confundir con un "scale del skeleton": es un factor de normalización de posición de animación.

## API relevante

Todo verificado en Godot 4.7.

### Props (4)

| Prop | Default | Nota |
|---|---|---|
| `animate_physical_bones` | `true` | **Deprecada** (verificado) → `PhysicalBoneSimulator3D.active`. |
| `modifier_callback_mode_process` | `1` (`MODIFIER_CALLBACK_MODE_PROCESS_IDLE`) | `PHYSICS=0` / `IDLE=1` / `MANUAL=2`. |
| `motion_scale` | `1.0` | Normalización de posición (retarget). |
| `show_rest_only` | `false` | Mostrar solo el rest (debug). |

### Métodos (tabla completa, ~45)

**Bones:** `add_bone(name) → int`, `clear_bones()`, `find_bone(name) → int`, `get_bone_count()`, `get_bone_name(idx)`, `set_bone_name(idx, name)`, `get_bone_parent(idx)`, `set_bone_parent(idx, parent_idx)`, `get_bone_children(idx)`, `get_parentless_bones()`, `unparent_bone_and_rest(idx)`, `get_concatenated_bone_names()`, `set_bone_enabled(idx, enabled=true)`, `is_bone_enabled(idx)`.

**Pose:** `get_bone_pose(idx)`, `set_bone_pose(idx, pose)`, `get_bone_pose_position/rotation/scale(idx)`, `set_bone_pose_position/rotation/scale(idx, ...)`, `get_bone_global_pose(idx)` (+ `_no_override`/`_override`), `set_bone_global_pose(idx, pose)`, `set_bone_global_pose_override(idx, pose, amount, persistent=false)`, `clear_bones_global_pose_override()`, `reset_bone_pose(idx)`, `reset_bone_poses()`, `force_update_all_bone_transforms()`, `force_update_bone_child_transform(bone_idx)`.

**Rest:** `get_bone_rest(idx)`, `set_bone_rest(idx, rest)`, `localize_rests()`.

**Meta:** `get_bone_meta(idx, key)`, `set_bone_meta(idx, key, value)`, `get_bone_meta_list(idx)`, `has_bone_meta(idx, key)`.

**Skin / update:** `create_skin_from_rest_transforms() → Skin`, `register_skin(skin) → SkinReference`, `get_version()`, `advance(delta)`.

**Physical bones:** `physical_bones_start_simulation(bones = [])`, `physical_bones_stop_simulation()`, `physical_bones_add_collision_exception(rid)`, `physical_bones_remove_collision_exception(rid)`.

### Señales (6)

| Señal | Cuándo |
|---|---|
| `bone_enabled_changed(bone_idx)` | `set_bone_enabled` (usar `is_bone_enabled` para el nuevo valor). |
| `bone_list_changed()` | `add_bone`/`set_bone_parent`/`unparent_bone_and_rest`/`clear_bones`. |
| `pose_updated()` | Pose actualizado (**no** durante el update → no detecta modificadores, nota oficial). |
| `rest_updated()` | Rest actualizado. |
| `show_rest_only_changed()` | La prop cambió. |
| `skeleton_updated()` | Pose final calculado, **antes** de aplicar a la skin (todos los modificadores completos — oficial). |

### Enum / Constante

| Elemento | Valores |
|---|---|
| `ModifierCallbackModeProcess` | `PHYSICS=0`, `IDLE=1` (default), `MANUAL=2`. |
| `NOTIFICATION_UPDATE_SKELETON` | `50` — "called only once per frame in a deferred process" (oficial). |

## Arquitectura recomendada

### Escena estándar de un character

```text
Character (Node3D / CharacterBody3D — godot-characterbody3d)
├─ AnimationTree (godot-animationtree)          ← la animación
├─ Skeleton3D                                    ← LOS huesos
│  ├─ MeshInstance3D (con Skin)                  ← la piel (skin reference)
│  ├─ RetargetModifier3D (si aplica — godot-retargeting)
│  ├─ TwoBoneIK3D / LookAtModifier3D (— godot-ik)
│  └─ PhysicalBoneSimulator3D (ragdoll — pendiente)
└─ Camera3D (— godot-camera3d)
```

1. **Un `Skeleton3D` por personaje** (los modificadores son hijos suyos — regla de `godot-ik`).
2. **La piel (`MeshInstance3D` con `Skin`)** referencia el skeleton; `register_skin`/`create_skin_from_rest_transforms` para skins dinámicas (verificado).
3. **El orden de los modificadores** en la lista define el pipeline (retarget → IK → ajustes).
4. **`modifier_callback_mode_process`**: `IDLE` (default) para visual; `PHYSICS` si los modificadores alimentan física/logica de tick (ver `godot-physics`).

## Implementación mínima

**Mover un bone a mano (death: el brazo cae):**

```gdscript
# skeleton.gd — Skeleton3D (API verificado 4.7)
extends Skeleton3D

func drop_arm() -> void:
	var idx := find_bone("LeftArm")
	if idx == -1:
		return
	# El pose en 4.x INCLUYE el rest (oficial): partir del rest y rotar
	var rest := get_bone_rest(idx)
	var pose := rest * Transform3D(Basis.from_euler(Vector3(0, 0, deg_to_rad(90.0))))
	set_bone_pose(idx, pose)
	# o un override que la animación pueda "pisar" con blend:
	# set_bone_global_pose_override(idx, pose, 1.0, true)
```

## Implementación recomendada

### 1. Leer el pose final (con IK/retarget) — la señal correcta

```gdscript
# NO pose_updated() (no detecta modificadores — nota oficial)
skeleton_updated.connect(_on_final_pose)

func _on_final_pose() -> void:
	var head := find_bone("Head")
	var world := to_global(get_bone_global_pose(head).origin)  # "global pose" es relativo al SKELETON (oficial)
	# world = la posición real (componer con el nodo)
```

### 2. Meta de bones (data-driven)

```gdscript
# Marcar bones "sensibles" (ej: no retargetear, o para debug)
set_bone_meta(find_bone("LeftHand"), "no_retarget", true)
# Consultar:
if has_bone_meta(idx, "no_retarget") and get_bone_meta(idx, "no_retarget"):
	...
```

### 3. Habilitar/deshabilitar bones (cortar coste)

```gdscript
set_bone_enabled(find_bone("RightHand"), false)   # of: la señal bone_enabled_changed
is_bone_enabled(idx)                              # consultar
```

### 4. Forzar update (tras mutar el pose a mano)

```gdscript
# Si cambiaste el pose/rest por código y la piel no sigue:
force_update_all_bone_transforms()                # of: verificado
# o por bone:
force_update_bone_child_transform(idx)
```

### 5. Physical bones (ragdoll simple)

```gdscript
# La prop animate_physical_bones está DEPRECADA (verificado) — usar:
# 1) un PhysicalBoneSimulator3D como hijo del Skeleton3D (recomendado, oficial)
#    y controlar su .active (ver godot-ik / skill pendiente)
# 2) o los métodos de la clase (verificados):
physical_bones_start_simulation(["LeftArm", "RightArm"])
physical_bones_stop_simulation()
physical_bones_add_collision_exception(player_body.get_rid())
```

## Ejemplo práctico

**Juego**: TPS con ragdoll al morir + head look.

1. **Skeleton** con `modifier_callback_mode_process = IDLE` (default, visual).
2. **Head look**: `LookAtModifier3D` (hijo) → leer la posición del head en `skeleton_updated()` para el hitbox de la cabeza (`godot-ik` da el modificador; esta skill da la señal correcta).
3. **Muerte**: `physical_bones_start_simulation()` (o `PhysicalBoneSimulator3D.active = true`) + `set_bone_pose` del brazo (receta 1).
4. **Data-driven**: `set_bone_meta` en los bones de attachment (manos) para que el sistema de items los encuentre sin hardcodear índices.
5. **Debug** "la piel queda en el último frame" → `force_update_all_bone_transforms()`; verificar `show_rest_only` para aislar rest vs pose.

## Integración

- **`godot-animationtree`**: los clips animan estos bones (la mezcla es previa al skeleton final).
- **`godot-ik`**: los modificadores son **hijos** de este nodo y corren post-`AnimationMixer` (oficial).
- **`godot-retargeting`**: `RetargetModifier3D` + `motion_scale` (prop verificada aquí).
- **`godot-character-controller`**: el `CharacterBody3D` es el nodo; el skeleton va adentro.
- **`godot-physics`**: `physical_bones_*` toman RIDs de colisión (verificado).
- **`godot-performance` (Lote 14 pendiente)**: el coste de N bones × modificadores (ver § Performance).

## Errores frecuentes

1. **Mentalidad 3.x (pose relativo al rest)** → en 4.x el Bone Pose **incluye** el rest (oficial); los cálculos a mano se rompen.
2. **Leer `get_bone_global_pose` y tratarlo como world** → es **relativo al skeleton** (nota oficial); componer con `to_global`.
3. **Usar `pose_updated()` para leer el pose final** → "During the update process, this signal is not fired, so modification by SkeletonModifier3D is not detected" (nota oficial) → `skeleton_updated()` (o `modification_processed()` del modificador).
4. **`animate_physical_bones = true` en código nuevo** → **deprecada** (verificado) → `PhysicalBoneSimulator3D` + `.active`.
5. **`NOTIFICATION_UPDATE_SKELETON` esperando un callback síncrono** → "called only once per frame in a **deferred** process" (oficial).
6. **Mutar el pose a mano y "nada pasa"** → el update es deferred; `force_update_all_bone_transforms()` (o esperar al siguiente frame).
7. **`find_bone` con índice en vez de nombre** → `find_bone(name) → int` (oficial); el inverso es `get_bone_name(idx)`.
8. **`motion_scale` como "scale del character"** → es un factor de normalización de **posición de animación** del retarget (oficial); no escala el nodo.
9. **`set_bone_enabled` esperando efecto inmediato en el skin** → verificar con `is_bone_enabled` en la señal `bone_enabled_changed` (oficial).
10. **Crear bones a mano para "completar" un skeleton importado** → el rest viene del asset; `add_bone`/`set_bone_rest` es para skeletons procedurales (verificado el método), no para parchar imports (usar retarget, `godot-retargeting`).

## Anti-patrones

- **Hardcodear índices de bone** (`skeleton.get_bone_pose(7)`) → `find_bone("Head")` (verificado); los índices cambian por asset.
- **Copiar el pose en `_process` para "guardarlo"** → usar la señal (`skeleton_updated`) — no polling.
- **`show_rest_only = true` en runtime "por si acaso"** → es una prop de **debug** (nombre oficial); en runtime es `false`.
- **Un `Skeleton3D` por parte del cuerpo** (cabeza separada, torso separado) → un skeleton por character; las piezas van como bones/skins (el pipeline de retarget asume un skeleton).
- **Re-implementar el IK "sobre" el skeleton a mano** cuando existe `godot-ik` → los modificadores nativos (4.6+) resuelven; el custom va en `_process_modification_with_delta` (ver `godot-ik`).
- **Fisicar bones con `RigidBody3D` hijos** → physical bones (`PhysicalBoneSimulator3D`) es la vía (verificado la API); no enganchar rígidos a bones a mano.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Coste real**: N bones × (skin update + modificadores activos) por frame. El skeleton es caro cuando **muchos bones + muchos modificadores + skin con muchos vértices por bone**.
- **Qué medir**: `Performance.TIME_PROCESS`/`TIME_PHYSICS_PROCESS` (familia verificada 4.7) con el character completo vs sin modificadores (`active = false` — oficial, ver `godot-ik`); contar `get_bone_count()`.
- **Optimizaciones (contra un cuello medido)**:
  1. `set_bone_enabled(idx, false)` en bones inanimados (verificado) — cortan el trabajo.
  2. Menos modificadores activos (`active = false`, oficial) — cada uno corre post-animación.
  3. `modifier_callback_mode_process = MANUAL` + `advance(delta)` para controlar cuándo se procesa (verificado) — p. ej. solo cuando se ve.
  4. Menos bones en el skin (binds por vértice) — decisión de asset, no de código.
- **No**: "el skeleton es barato" — 100+ bones con 5 modificadores y un skin denso no lo es.

## Debugging

### "La piel no sigue a los bones / queda congelada"

```text
1. ¿El MeshInstance3D tiene la Skin correcta (skeleton reference)?
2. force_update_all_bone_transforms() (verificado) tras mutar a mano
3. modifier_callback_mode_process: ¿MANUAL sin advance()? (verificado)
4. show_rest_only: ¿true "olvidado"? (prop de debug)
```

### "El bone que muevo a mano 'rebota' al siguiente frame"

```text
1. La animación lo está pisando (corre ANTES que tu código en _process?
   → el update del skeleton es deferred; la animación lo restaura)
2. Usar set_bone_global_pose_override (verificado) para un override
   que el blend respete (amount, persistent)
3. ¿Otro modificador lo pisa después? (orden en la lista — godot-ik)
```

### "El 'global pose' no calza con el mundo"

```text
1. Es relativo al SKELETON (nota oficial) → to_global(...)
2. ¿El nodo del skeleton tiene transform? (componer)
3. root_motion: ¿la animación mueve el root track? (godot-animationtree)
```

### "El ragdoll no arranca / no se para"

```text
1. animate_physical_bones está deprecada → PhysicalBoneSimulator3D + .active
   (verificado en la prop description)
2. physical_bones_start_simulation(bones) con los nombres correctos (of.)
3. Colisiones: physical_bones_add_collision_exception(rid) (verificado)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla completa (lista en *Referencias*).
- **3.x → 4.x (verificado, oficial)**: Bone Pose **incluye** Bone Rest (3.x: relativo); el diseño de datos cambió ("animation-data-redesign-40").
- **4.0 → 4.7**: la tabla consultada es la de 4.7; la prop `animate_physical_bones` aparece **deprecada** en 4.7 (el `PhysicalBoneSimulator3D` es el workflow recomendado — oficial). No afirmar cuándo exacto se deprecó sin verificar.
- **`Skeleton3D` en 4.x**: la clase existe desde 4.0 (reemplaza `Spatial`+skeleton de 3.x); el sistema de modificadores (`SkeletonModifier3D`) llegó en 4.4 (artículo oficial IK 4.6).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-animationtree` | Los clips animan los bones |
| `godot-ik` | Modificadores hijos (post-mix) |
| `godot-retargeting` | Retarget + `motion_scale` |
| `godot-character-controller` | El cuerpo del character |
| `godot-physics` | RIDs de colisión (physical bones) |

## Skills relacionadas

- `godot-recipe-bone-death-pose` — pendiente (germen en *Implementación mínima*)
- `godot-recipe-ragdoll-on-death` — pendiente (germen #5)
- `godot-error-skin-not-following` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Skeleton3D (props, ~45 métodos, 6 señales, enum `ModifierCallbackModeProcess`, `NOTIFICATION_UPDATE_SKELETON=50`, "global pose" relativo al skeleton, `animate_physical_bones` deprecada, `motion_scale`): https://docs.godotengine.org/en/stable/classes/class_skeleton3d.html
- Retargeting 3D Skeletons (Bone Pose incluye rest en 4.x; `motion_scale`; reference poses): https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/retargeting_3d_skeletons.html
- Inverse Kinematics Returns to Godot 4.6 (los modificadores corren post-`AnimationMixer`; `SkeletonModifier3D` desde 4.4): https://godotengine.org/article/inverse-kinematics-returns-to-godot-4-6/
