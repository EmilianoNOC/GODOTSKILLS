# godot-retargeting

## ID

`godot-retargeting`

## Categoría

Animation · Base (assets)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: tutorial "Retargeting 3D Skeletons", páginas de clase `BoneMap`, `SkeletonProfile`, `SkeletonProfileHumanoid`, `RetargetModifier3D` (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — el flujo completo de retargeting **de import** (`bone_map` en el importer, `SkeletonProfileHumanoid` con auto-mapping, opciones Remove Tracks/Bone Renamer/Rest Fixer — todas verificadas en el tutorial oficial 4.7), `BoneMap` (prop `profile`, métodos `find_profile_bone_name`/`get_skeleton_bone_name`/`set_skeleton_bone_name`, señales), `SkeletonProfile` (props `bone_size`/`group_size`/`root_bone`/`scale_base_bone`, métodos), `SkeletonProfileHumanoid` (56 bones, 4 groups, `root_bone "Root"`, `scale_base_bone "Hips"`, read-only), `RetargetModifier3D` (runtime; props `enable=7`, `profile`, `use_global_pose=false`, flags `TransformFlag`), "Bone Pose en 4.x **incluye** Bone Rest" (nota oficial): verificados directamente.
**MEDIUM** — qué opciones activar por caso de uso (el tutorial da recomendaciones, no reglas absolutas).
**Nota de versión**: el retargeting de import existe desde 4.0 (artículo oficial 4.0→4.3); `RetargetModifier3D` (runtime) aparece en **4.4** (artículo oficial IK 4.6).

## Nivel

**INTERMEDIATE** (depende de `godot-skeleton3d`).

## Propósito

Proporcionar el conocimiento operativo para **compartir animaciones entre skeletons distintos** en Godot 4.7: el flujo oficial de import (BoneMap + SkeletonProfile + opciones del importer), el perfil humanoid (56 bones, T-pose, +Z), `motion_scale` (normalización de posición por altura de Hips), y el retargeting **de runtime** con `RetargetModifier3D` (hijo del skeleton, flags de transform, `use_global_pose`). Cubre el diagnóstico de los fallos clásicos (animación "deslocada", cuerpo "deformado", accesorios sin animar, slips entre modelos de distinta altura).

## Cuándo utilizarla

- Usar animaciones de un modelo (p. ej. Mixamo/Blender) en varios personajes.
- Estandarizar el pipeline de characters humanoides.
- Retargeting de runtime (un esqueleto sigue el pose de otro).
- Depurar "la animación no calza / el cuerpo se deforma / el modelo se desliza".

## Cuándo NO utilizarla

- **Un solo modelo** (sin compartir animaciones) → no se necesita; el retargeting es para **compartir**.
- **Godot 3.x** (`bone_map`/`SkeletonProfile` son de 4.x) → en 3.x no hay esta API.
- **Retarget que debe PRESERVAR el Bone Rest original** → el tutorial oficial recomienda el **Realtime Retarget Module** (add-on, GitHub de TokageItLab) — el core 4.7 **sobrescribe** el rest con "Overwrite Axis" (verificado en el tutorial).
- **IK / constraints** → `godot-ik` (misma familia de modificadores; el retarget no es un solver IK).
- **2D** → el retargeting de skeleton es 3D (`Skeleton3D`); en 2D el "share" es por nombre de `Bone2D` (verificado en el índice 4.7).

## Conceptos fundamentales

### Por qué no basta con el mismo nombre de bone

Oficial (4.7): las tracks son Position/Rotation/Scale con NodePaths a bones; "you can't share animations between multiple Skeletons **just by using the same bone names**", porque "bones that share a name can still have different Transform values" (el Bone Rest). "To share animations in Godot, it is necessary to **match Bone Rests as well as Bone Names**."

- **Diferencia 3.x vs 4.x (oficial)**: en Godot 3, Bone Pose era **relativo** a Bone Rest; en Godot 4, el Bone Pose **incluye** Bone Rest ("See animation-data-redesign-40 article").
- El rest depende del DCC: glTF de Blender trae "Edit Bone Orientation" como rest; glTF de Maya puede no tener rest rotation.

### El flujo de import (oficial, section "Retarget" del importer)

1. En el **advanced scene import**, seleccionar el `Skeleton3D` → aparece la sección **Retarget** con la prop **`bone_map`**.
2. Crear un `BoneMap` + un `SkeletonProfile`. Para humanoides: **`SkeletonProfileHumanoid`** (preset; "all parameters are read-only", verificado).
3. **Auto-mapping**: "will be performed when the SkeletonProfile is set" (oficial); usa **pattern matching de nombres** → "recommend to use common English names for bones" (oficial). Errores (missing/duplicate/incorrect parent-child) se marcan en magenta/rojo y "does not block the import process, but warns that animations may not be shared correctly".
4. Si el auto-mapping no alcanza → mapear bones a mano.
5. Exportar un `SkeletonProfile` custom: seleccionando el `Skeleton3D` y usando el menú **Skeleton3D** en la toolbar del viewport 3D (oficial).

### `BoneMap` (recurso; tabla completa verificada)

| Elemento | Detalle |
|---|---|
| Prop `profile` | El `SkeletonProfile` destino; "Key names in the BoneMap are synchronized with it." |
| `find_profile_bone_name(skeleton_bone_name)` | "In the retargeting process, the returned bone name is the bone name of the **target** skeleton." (vacío si no mapeado) |
| `get_skeleton_bone_name(profile_bone_name)` | "returned bone name is the bone name of the **source** skeleton." |
| `set_skeleton_bone_name(profile_bone_name, skeleton_bone_name)` | "the setting bone name is the bone name of the **source** skeleton." |
| Señales | `bone_map_updated()`, `profile_updated()` (verificadas) |

Semántica: el `BoneMap` usa **nombres del perfil como keys** y **nombres reales del skeleton como values** ("a dictionary that uses a list of bone names in SkeletonProfile as key names").

### `SkeletonProfile` / `SkeletonProfileHumanoid`

- Base (verificado): props `bone_size 0`, `group_size 0`, `root_bone &""`, `scale_base_bone &""`; métodos `find_bone`, `get_bone_name/parent/tail`, `get_group(_name)`, `get_reference_pose(bone_idx)`, `get_tail_direction`, `get_handle_offset`, `get_texture(group_idx)`, `is_required`, `set_bone_name/parent/tail...`. "used in EditorScenePostImport".
- **Humanoid** (verificado): `bone_size 56`, `group_size 4` (groups: `"Body"`, `"Face"`, `"LeftHand"`, `"RightHand"`), `root_bone "Root"`, `scale_base_bone "Hips"`; **read-only** ("exists for standardization, so all parameters are read-only").
- Estructura oficial: `Root → Hips → (LeftUpperLeg→LeftLowerLeg→LeftFoot→LeftToes, Right...), Spine→Chest→UpperChest→(Neck→Head→(Jaw,LeftEye,RightEye), LeftShoulder→LeftUpperArm→LeftLowerArm→LeftHand→(5 dedos), Right...)`.

### Reglas del reference pose (Rest Fixer; oficial)

- "The humanoid is **T-pose**".
- "facing **+Z** in the Right-Handed Y-UP Coordinate System".
- "should not have a Transform as Node".
- "+Y axis from the parent joint to the child joint".
- "+X rotation bends the joint like a muscle contracting".

### Las opciones del importer (verificadas, tutorial 4.7)

**Remove Tracks** (recomendado si se importa como `AnimationLibrary` **compartida**; si se importa como escena, "should be disabled in some cases" — p. ej. accesorios animados):
- **Except Bone Transform**: "Removes any tracks except the bone Transform track from the animations."
- **Unimportant Positions**: "Removes Position tracks other than `root_bone` and `scale_base_bone` defined in SkeletonProfile" (en Humanoid: "Root" y "Hips"). Nota oficial: si se deshabilita, "the animation may change the body shape unpredictably" (en 4.x el transform incluye rest).
- **Unmapped Bones**: "Removes unmapped bone Transform tracks from the animations."

**Bone Renamer**:
- **Rename Bones**: "Rename the mapped bones." (estandariza nombres → comparte más fácil)
- **Unique Node**: "Makes Skeleton a unique node with the name specified in the `skeleton_name`. This allows the animation track paths to be **unified independent of the scene hierarchy**."

**Rest Fixer**:
- **Apply Node Transform**: fixea modelos sin "Apply Transform" (p. ej. glTF de Blender sin aplicar). ⚠️ "If the imported scene contains objects other than Skeletons, this option may have a negative effect."
- **Normalize Position Tracks**: "normalizes the Position track values based on the `scale_base_bone` height. The `scale_base_bone` height is stored in the Skeleton as the **`motion_scale`**, and the normalized Position track values is **multiplied by that value on playback**." Con Humanoid: `scale_base_bone` = "Hips". Si se deshabilita: "the Skeleton's `motion_scale` is always imported as `1.0`."
- **Overwrite Axis**: "Unifies the models' Bone Rests by overwriting it to match the reference poses defined in the SkeletonProfile." ⚠️ Oficial: "**This is the most important option for sharing animations in Godot 4**, but be aware that this option can produce horrible results if the original Bone Rest set externally is important. If you want to share animations with keeping the original Bone Rest, consider to use the Realtime Retarget Module."
- **Fix Silhouette**: "Attempts to make the model's silhouette match that of the reference poses (such as T-Pose)." Con Humanoid: no necesario para T-pose; **sí para A-pose** ("the fixed foot results may be bad depending on the heel height" → `filter` array para excluir bones; `base_height_adjustment` para rodillas/pies doblados).

### Retargeting de runtime: `RetargetModifier3D` (4.4+)

- Hijo del `Skeleton3D` destino (misma regla que `godot-ik`: "retrieves ... by having a Skeleton3D parent").
- "transfers parent skeleton poses (or global poses) to child skeletons in model space with different rests" (oficial).
- "This modifier **rewrites the pose of the child skeleton directly in the parent skeleton's update process**… it overwrites the mapped bone pose set in the normal process on the target skeleton. If you want to set the target skeleton bone pose after retargeting, add a SkeletonModifier3D child to the target skeleton."
- Props (verificadas): `enable` (BitField, default **7** = ALL), `profile` (`SkeletonProfile` — "for retargeting bones with names matching the bone list"), `use_global_pose` (default `false`).
- `TransformFlag` (verificado): `POSITION=1`, `ROTATION=2`, `SCALE=4`, `ALL=7`.
- Métodos (verificados): `is/set_position_enabled`, `is/set_rotation_enabled`, `is/set_scale_enabled`.
- ⚠️ Nota oficial: con `use_global_pose = true`, un bone **unmapeado** "can cause visual problems because the global pose is applied ignoring the parent bone's pose if it has mapped bone children."

## API relevante

Todo verificado en Godot 4.7 (páginas de clase + tutorial oficial).

### `BoneMap` (< Resource)

| Elemento | Detalle |
|---|---|
| `profile` | `SkeletonProfile` (keys sincronizadas con él). |
| `find_profile_bone_name(skeleton_bone_name)` | → nombre del perfil (target). |
| `get_skeleton_bone_name(profile_bone_name)` | → nombre del source. |
| `set_skeleton_bone_name(profile_bone_name, skeleton_bone_name)` | mapear (source). |
| Señales | `bone_map_updated()`, `profile_updated()`. |

### `SkeletonProfile` (< Resource)

| Elemento | Detalle |
|---|---|
| `bone_size` / `group_size` | `0` / `0` (base). |
| `root_bone` / `scale_base_bone` | `&""` / `&""` (base); `"Root"` / `"Hips"` (humanoid). |
| `find_bone(bone_name)` / `get_bone_name(bone_idx)` / `get_bone_parent(bone_idx)` / `get_bone_tail(bone_idx)` | Consultas. |
| `get_reference_pose(bone_idx)` | `Transform3D` del reference pose. |
| `get_group(_name)` / `get_handle_offset(bone_idx)` / `get_tail_direction(bone_idx)` / `get_texture(group_idx)` / `is_required(bone_idx)` | Consultas de grupo/handles. |
| `set_bone_name/parent/tail(...)` | Edición (perfiles custom). |

### `RetargetModifier3D` (hijo de `SkeletonModifier3D`)

| Elemento | Detalle |
|---|---|
| `enable` | BitField, default `7` (POSITION\|ROTATION\|SCALE). |
| `profile` | `SkeletonProfile` de mapeo. |
| `use_global_pose` | `false`. |
| `is/set_{position,rotation,scale}_enabled` | Toggle por componente. |

## Arquitectura recomendada

### Pipeline de import (el patrón canónico)

```text
glTF (Blender/Mixamo) 
  → Advanced Import:
     Skeleton3D:
       Retarget:
         bone_map.profile = SkeletonProfileHumanoid
         (auto-mapping inglés → corregir a mano)
         Remove Tracks: Except Bone Transform ✓, Unimportant Positions ✓, Unmapped Bones ✓
         Bone Renamer: Rename Bones ✓, Unique Node ✓ (skeleton_name = "Skeleton")
         Rest Fixer: Apply Node Transform ✓ (si aplica), Overwrite Axis ✓,
                     Fix Silhouette ✓ (solo A-pose), Normalize Position Tracks ✓
  → AnimationLibrary compartida (un solo set de animaciones para todos los models)
```

1. **Un perfil por familia** (humanoid para humans; exportar custom para bestias/robots con el menú Skeleton3D del viewport — oficial).
2. **Nombres de bone en inglés estándar** (auto-mapping por patrón — oficial).
3. **Importar como `AnimationLibrary`** (compartida) y no como escena, para que las opciones de "Remove Tracks" apliquen (recomendación oficial).
4. **`Unique Node`** para que los paths de track no dependan de la jerarquía de escena (oficial).

### Runtime (un skeleton sigue a otro)

```text
TargetCharacter (Node3D)
└─ Skeleton3D (destino)
   └─ RetargetModifier3D          # hijo; profile = SkeletonProfileHumanoid
      # enable = TRANSFORM_FLAG_ROTATION (7 si también posición)
      # (el source es el skeleton padre-ancestro de la escena; el modifier
      #  lo resuelve por la jerarquía — ver docs al configurar)
   └─ (opcional) otro SkeletonModifier3D DESPUÉS para ajustar el pose (oficial)
```

## Implementación mínima

**Compartir walk/run entre 3 humanoides** (todo en el importer; cero código):

```text
1. Modelo A: importar con Retarget (Humanoid, Overwrite Axis ✓, Normalize ✓,
   Unique Node ✓) → AnimationLibrary "Humanoids".
2. Modelos B y C: importar con el MISMO profile y opciones → sus bones se
   renuean a los del perfil; la AnimationLibrary "Humanoids" les calza.
3. En cada escena: AnimationPlayer/AnimationTree con la librería compartida.
```

## Implementación recomendada

### 1. Modelo A-pose (necesita Fix Silhouette)

```text
Rest Fixer:
  Fix Silhouette: ON
  filter: ["LeftFoot", "RightFoot"]   (si los pies quedan mal por el talón — of.)
  base_height_adjustment: 0.05        (si tiene rodillas/pies doblados — of.)
```

### 2. Modelos de distinta altura (sin "slip")

```text
Normalize Position Tracks: ON
  → motion_scale = altura de Hips (oficial); los Position tracks se
    normalizan y se multiplican por motion_scale en playback (oficial)
```

### 3. Retarget de runtime con solo rotaciones

```gdscript
# Configurar el RetargetModifier3D (hijo del Skeleton3D destino):
retarget.profile = SkeletonProfileHumanoid.new()
retarget.enable = RetargetModifier3D.TRANSFORM_FLAG_ROTATION   # = 2 (of.)
# o por métodos (verificados):
retarget.set_position_enabled(false)
retarget.set_rotation_enabled(true)
retarget.set_scale_enabled(false)
```

### 4. Ajustar el pose después del retarget

```text
# Oficial: "add a SkeletonModifier3D child to the target skeleton"
Skeleton3D (destino)
├─ RetargetModifier3D      (1º)
└─ BoneTwistDisperser3D    (2º — ajusta el twist después; ver godot-ik)
```

### 5. Perfil custom (bestia/robot)

```text
1. Seleccionar el Skeleton3D del modelo "maestro"
2. Menú Skeleton3D (toolbar del viewport 3D) → exportar SkeletonProfile (of.)
3. Usarlo como profile en el BoneMap de todos los de la familia
```

## Ejemplo práctico

**Juego**: 5 personajes humanoides (3 varones, 2 mujeres) con 40 animaciones de Mixamo.

1. **Import**: todos con `SkeletonProfileHumanoid`, auto-mapping inglés, `Rename Bones` + `Unique Node` (skeleton_name "Skeleton"), `Overwrite Axis` + `Normalize Position Tracks` ON, `Except Bone Transform` + `Unimportant Positions` ON.
2. **Resultado**: 40 animaciones en una `AnimationLibrary` única; cada personaje reproduce las mismas con su cuerpo (el `motion_scale` por Hips evita el "slip" entre alturas).
3. **Accesorios**: los importados **como escena** (con sus propias animaciones) NO activan "Remove Tracks" (oficial: "these should be disabled in some cases" — ej. espada que se mueve).
4. **A-pose**: el modelo 4 es A-pose → `Fix Silhouette` ON con `filter` de pies.
5. **Debug** "un personaje parece que resbala al correr" → `Normalize Position Tracks` desactivado (motion_scale 1.0) o la altura de Hips distinta.
6. **Debug** "cuerpo se deforma" → `Unimportant Positions` desactivado (oficial: "may change the body shape unpredictably").

## Integración

- **`godot-skeleton3d`**: el skeleton destino (bones, rest, `motion_scale` — verificado la prop).
- **`godot-ik`**: misma familia de modificadores; el `RetargetModifier3D` corre en el update del skeleton y **puede** ser ajustado por otros modificadores después (oficial).
- **`godot-animationtree`**: los clips retargetizados se mezclan normalmente (el retarget no toca el grafo).
- **`godot-character-controller`**: el `motion_scale` afecta el movimiento por animación (root motion).
- **Add-on (no core)**: **Realtime Retarget Module** (GitHub TokageItLab) — recomendado por el tutorial oficial si hay que **preservar el Bone Rest original**.

## Errores frecuentes

1. **Esperar que "mismo nombre de bone" sea suficiente** → no: hay que **matchear Bone Rests y Bone Names** (oficial); el retarget existe para eso.
2. **Bone Pose relativo (mentalidad 3.x)** → en 4.x el Bone Pose **incluye** Bone Rest (oficial); los cálculos a mano deben usar eso.
3. **`Overwrite Axis` desactivado "por precaución"** → es "the most important option for sharing animations in Godot 4" (oficial); sin ella el rest no se unifica.
4. **`Overwrite Axis` cuando el rest original importa** → "can produce horrible results" (oficial); usar el Realtime Retarget Module (recomendación oficial).
5. **Nombres de bone exóticos** → el auto-mapping es por **pattern matching** (oficial); usar nombres en inglés comunes.
6. **Importar la librería compartida como escena** → las opciones de "Remove Tracks" no deben ir en escenas con accesorios animados (oficial).
7. **`motion_scale` esperando que sea 1.0** → se importa = altura de `scale_base_bone` (Hips) si `Normalize Position Tracks` está ON (oficial).
8. **A-pose sin `Fix Silhouette`** → "should be enabled for A-pose models" (oficial); cuidar pies (`filter`, `base_height_adjustment`).
9. **`use_global_pose = true` con bones unmapeados** → "can cause visual problems… ignoring the parent bone's pose if it has mapped bone children" (nota oficial).
10. **Querer ajustar el pose destino **después** del retarget sin otro modificador** → "add a SkeletonModifier3D child to the target skeleton" (oficial); el retarget sobrescribe el pose normal.
11. **`enable = 7` cuando solo querés rotaciones** → los flags son por componente (verificado); `TRANSFORM_FLAG_ROTATION = 2`.
12. **Retargeting en 2D / 3.x** → la API es 4.x y 3D (`Skeleton3D`); en 3.x/2D no existe (verificado en el índice 4.7).

## Anti-patrones

- **Un `BoneMap` distinto por animación** → un perfil + un map por **familia de modelos**; el map es del skeleton, no del clip.
- **Re-importar con flags distintos "para probar"** → el retarget es determinista por configuración; documentar la receta de import (spec §44).
- **Hand-waving del "slip" con `motion_scale` a mano** → la opción del importer lo hace (oficial); no hardcodear.
- **`Fix Silhouette` en T-pose "por si acaso"** → "does not need to be enabled for T-pose models" (oficial); solo A-pose.
- **Correr el retarget runtime con `enable = ALL` cuando la posición ya calza** → desactivar componentes que no usás (flags verificados).
- **Ignorar las advertencias magenta/rojo del auto-mapping** → "warns that animations may not be shared correctly" (oficial); corregir el mapeo.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El retarget de import es coste CERO en runtime** (es un paso del pipeline; los clips ya están escritos contra el perfil).
- **El retarget de runtime (`RetargetModifier3D`) sí corre por frame**: "rewrites the pose of the child skeleton directly in the parent skeleton's update process" (oficial).
- **Qué medir**: `Performance.TIME_PROCESS`/`TIME_PHYSICS_PROCESS` (familia verificada 4.7) con el `RetargetModifier3D` activo vs `active = false` (prop de `SkeletonModifier3D`, oficial); contar bones mapeados (más bones = más escritura de pose).
- **Optimizaciones (contra un cuello medido)**:
  1. `enable` solo con los componentes necesarios (rotación sin posición/escala).
  2. Menos bones mapeados en el perfil (un perfil por familia, no "todo").
  3. Preferir el retarget **de import** (0 runtime) cuando el modelo es fijo; el runtime es para skeletons que cambian en vivo.
- **No**: "el retarget runtime es gratis" — reescribe poses en cada update del skeleton (oficial).

## Debugging

### "La animación no calza (el cuerpo está torcido)"

```text
1. ¿Overwrite Axis ON? (oficial: la opción más importante)
2. ¿El modelo es T-pose y +Z? (reglas del reference pose)
3. ¿A-pose sin Fix Silhouette?
4. ¿Auto-mapping con advertencias magenta/rojo? (mapear a mano)
```

### "El modelo se deforma al animar"

```text
1. Unimportant Positions: ¿ON? (oficial: sin ella "may change the body shape unpredictably")
2. ¿El bone map mapeó bones incorrectos (duplicates)?
3. ¿Scale tracks en las animaciones? (Except Bone Transform las quita)
```

### "Un personaje resbala al correr (slip)"

```text
1. Normalize Position Tracks: ¿ON? (oficial: motion_scale por altura de Hips)
2. ¿Las alturas de Hips son muy distintas entre modelos?
3. ¿El floor/velocidad de la animación se corresponde? (ajustar speed scale)
```

### "Los accesorios no se animan (o sí y no deben)"

```text
1. Import como AnimationLibrary compartida → Remove Tracks activas → los
   accesorios (outros bones) se pierden (oficial: deshabilitar en escenas)
2. ¿Necesitás que el espada se mueva? → importar la escena del character
   sin "Remove Tracks" para ese asset
```

### "El retarget runtime se ve mal / pisa el pose"

```text
1. use_global_pose: ¿necesario? (oficial: visual problems con unmapeados)
2. enable: ¿los componentes correctos? (flags)
3. ¿Otro modificador corre después y ajusta? (oficial: agregar SkeletonModifier3D)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tutorial + 4 páginas de clase (lista en *Referencias*).
- **4.0 → 4.7**: el retargeting de import existe desde **4.0** (artículo oficial "Migrating Animations from Godot 4.0 to 4.3": "the reset behavior is also convenient with retargeting which was introduced in 4.0"); las tablas consultadas son las de 4.7.
- **`RetargetModifier3D`**: disponible desde **4.4** (artículo oficial IK 4.6); en 4.7 verificado.
- **3.x → 4.x**: no hay equivalente en 3.x (verificado: la API es 4.x); en 3.x el "share" era por nombre de bone (y no calzaba por rest).
- **Bone Pose**: 3.x relativo a rest; 4.x incluye rest (oficial, verificado).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-skeleton3d` | El skeleton (rest, `motion_scale`) |
| `godot-animationtree` | Los clips retargetizados se mezclan |
| `godot-ik` | Misma familia de modificadores (post-retarget adjustments) |
| `godot-character-controller` | Root motion y `motion_scale` |

## Skills relacionadas

- `godot-recipe-mixamo-pipeline` — pendiente (germen en *Ejemplo práctico*)
- `godot-error-retarget-deform` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Retargeting 3D Skeletons (tutorial completo: Bone Map, Remove Tracks, Bone Renamer, Rest Fixer, reglas T-pose/+Z, `motion_scale`, `filter`, `base_height_adjustment`, Realtime Retarget Module): https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/retargeting_3d_skeletons.html
- BoneMap (prop `profile`, 3 métodos, señales, semántica source/target): https://docs.godotengine.org/en/stable/classes/class_bonemap.html
- SkeletonProfile (props, métodos, "used in EditorScenePostImport"): https://docs.godotengine.org/en/stable/classes/class_skeletonprofile.html
- SkeletonProfileHumanoid (56 bones, 4 groups, read-only, `root_bone "Root"`, `scale_base_bone "Hips"`, estructura): https://docs.godotengine.org/en/stable/classes/class_skeletonprofilehumanoid.html
- RetargetModifier3D (runtime: `enable` flags, `profile`, `use_global_pose`, "rewrites the pose… in the parent skeleton's update process"): https://docs.godotengine.org/en/stable/classes/class_retargetmodifier3d.html
- Migrating Animations from Godot 4.0 to 4.3 (retargeting desde 4.0, `deterministic` blending): https://godotengine.org/article/migrating-animations-from-godot-4-0-to-4-3/
