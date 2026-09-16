# godot-ik

## ID

`godot-ik`

## Categoría

Animation · Base (esqueleto)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: `SkeletonModifier3D`, `IKModifier3D`, `TwoBoneIK3D`, `ChainIK3D`, `IterateIK3D`, `CCDIK3D`, `FABRIK3D`, `BoneConstraint3D`, `LookAtModifier3D` y el artículo oficial "Inverse Kinematics Returns to Godot 4.6" (docs.godotengine.org/en/stable/ + godotengine.org).

## Confidence

**HIGH** — jerarquía completa (`SkeletonModifier3D` → `IKModifier3D` → {`TwoBoneIK3D`, `ChainIK3D` → {`SplineIK3D`, `IterateIK3D` → {`FABRIK3D`, `CCDIK3D`, `JacobianIK3D`}}}), props de base (`active true`, `influence 1.0`), props de `IKModifier3D` (`mutable_bone_axes true`), props de `IterateIK3D` (`max_iterations 4`, `angular_delta_limit 0.034906585`, `min_distance 0.001`, `deterministic false`), enums (`BoneAxis` 0-5, `BoneDirection` 0-6, `SecondaryDirection` 0-7, `RotationAxis` 0-4), timeline 4.4/4.5/4.6 (oficial), "IK removed in 4.0, back in 4.6" (oficial), modificadores corren **después** del `AnimationMixer` (oficial): verificados directamente.
**MEDIUM** — valores de feel (pole direction, influence, iteraciones, ángulos de límite).
**NO existe en 4.7**: `IKConstraint3D`/`InverseKinematics3D` (páginas 404 — verificado; son de 3.x). No usar código 3.x.

## Nivel

**INTERMEDIATE** (depende de `godot-skeleton3d`).

## Propósito

Proporcionar el conocimiento operativo del sistema IK **nativo** de Godot 4.7 (los nodos `SkeletonModifier3D`): solvers (`TwoBoneIK3D`, `ChainIK3D`, `FABRIK3D`, `CCDIK3D`, `JacobianIK3D`, `SplineIK3D`), constraints (`BoneConstraint3D` y sus hijos, `LookAtModifier3D`), twist/velocidad (`BoneTwistDisperser3D`, `LimitAngularVelocityModifier3D`), el concepto de **IK determinista** (oficial), y el patrón de uso: el modificador como **hijo del `Skeleton3D`**, con un target `Node3D`, ejecutándose post-animación. Incluye el diagnóstico de los fallos clásicos (el bone "se quiebra", el target "no lo alcanza", flip de articulación).

## Cuándo utilizarla

- Mano/pie/brazo que siguen un objetivo (coger objetos, pisar terreno).
- Cabeza que mira al player / torre que apunta.
- 2-bone IK (brazo/pierna) con pole.
- Cadenas largas (cola, antena) con FABRIK/CCD/Jacobian.
- "Mejorar" el resultado de animaciones (twist disperso, límites de velocidad).

## Cuándo NO utilizarla

- **Godot 3.x** (`IKConstraint3D`/`InverseKinematics3D`) → **no existen en 4.7** (verificado 404); migrar a esta API.
- **2D** → no hay IK nativa 2D (los solvers son `*3D`); para 2D: `Bone2D` (existe en 4.7, verificado en el índice) + lógica propia, o add-on.
- **Retargeting entre modelos** → `godot-retargeting` (`RetargetModifier3D` es de esa familia, no un solver IK).
- **Física de huesos** (ragdoll, spring bones) → `PhysicalBoneSimulator3D`/`SpringBoneSimulator3D` (mencionados; skills pendientes) — no confundir con IK.
- **El blending de animaciones** → `godot-animationtree`/`godot-blendspace` (el IK se aplica **después** de la mezcla).

## Conceptos fundamentales

### El sistema: modificadores de skeleton, no "IK mágica"

- **`SkeletonModifier3D`** (base, Node3D): "retrieves a target Skeleton3D by having a Skeleton3D parent" (oficial). Es decir: **el modificador debe ser hijo (o descendiente) del `Skeleton3D`** que modifica.
- Orden de ejecución (oficial): "If there is an AnimationMixer, a modification **always performs after playback process** of the AnimationMixer." → la animación corre primero, el IK después.
- Props de base (verificadas): `active` (default `true`), `influence` (default `1.0`) — "used by Skeleton3D to blend, so the SkeletonModifier3D should **always apply only 100% of the result without interpolation**" (nota oficial).
- Señal `modification_processed()` (verificada): "to get the modified bone pose, use `get_bone_pose()`/`get_bone_global_pose()` **at the moment this signal is fired**".
- Para modificadores custom: sobrescribir `_process_modification_with_delta(delta)` (el `_process_modification()` está **deprecado**, verificado); "must not apply influence" (lo aplica el `Skeleton3D`).

### La jerarquía de IK (verificada, oficial 4.4→4.6)

```text
SkeletonModifier3D (4.4)
├─ IKModifier3D (4.6)
│  ├─ TwoBoneIK3D (4.6)          # 2 huesos + pole
│  └─ ChainIK3D (4.6)            # cadena N huesos (genera joints raíz→final)
│     ├─ SplineIK3D (4.6)
│     └─ IterateIK3D (4.6)       # repite rotaciones pequeñas
│        ├─ FABRIK3D (4.6)       # position-based
│        ├─ CCDIK3D (4.6)        # rotation-based (CCD)
│        └─ JacobianIK3D (4.6)
├─ BoneConstraint3D (4.5)        # un bone sigue a otro/nodo
│  ├─ AimModifier3D (4.5)
│  ├─ CopyTransformModifier3D (4.5)
│  └─ ConvertTransformModifier3D (4.5)
├─ LookAtModifier3D (4.4)        # 1-bone: mirar a un target
├─ BoneTwistDisperser3D (4.6)    # reparte twist a ancestros
├─ LimitAngularVelocityModifier3D (4.6)  # limita velocidad angular
├─ RetargetModifier3D (4.4)      # → godot-retargeting
├─ SpringBoneSimulator3D (4.5)   # (skill pendiente)
└─ PhysicalBoneSimulator3D       # (skill pendiente)
```

### El patrón de uso: target `Node3D`

- Cada setting de IK (índice `settings/<index>/...`) apunta a un **target `Node3D`** (`get_target_node(index)` en `IterateIK3D`; `get_pole_node(index)`/`get_target_node(index)` en `TwoBoneIK3D` — verificado).
- El target puede ser cualquier nodo (`Node3D`, `Marker3D`, `BoneAttachment3D`); el solver mueve los huesos para que el **end bone** llegue al target.
- **Settings múltiples**: `set_setting_count(n)` (verificado en `IKModifier3D`); los métodos toman `index` ("specifies which setting list entry to return, e.g. `settings/<index>/root_bone_name`" — nota oficial).

### Solvers: cuál elegir (descripciones oficiales 4.7)

| Solver | Tipo | Descripción oficial | Mejor para |
|---|---|---|---|
| `TwoBoneIK3D` | rotation, 2 huesos + **pole** | "Rotation based intersection of two circles inverse kinematics solver… deterministic results by constructing a plane from each joint and pole target." | Brazos/piernas (2 huesos). Siempre determinista. |
| `ChainIK3D` | base de cadena | "automatically generates a joint list from the bones between the root bone and the end bone." | (base; no usar directo — usar sus hijos) |
| `FABRIK3D` | position-based | "precise and accurate tracking of targets. Ideal for **simple chains without limitations**." ⚠️ "When the target is close to the root, it tends to produce **zig-zag** patterns." | Cadenas simples, tracking preciso. |
| `CCDIK3D` | rotation-based (CCD) | "fast and effective tracking even with large joint rotations. Especially suitable for **chains with limitations**, providing smoother and more stable target tracking compared to FABRIK3D." ⚠️ "When the target is close to the root, it can cause **joint flips and oscillations**." | Cadenas con límites de joints. |
| `JacobianIK3D` | (IterateIK) | (clase en la jerarquía verificada; detalle: ver docs) | Cadena con solución jacobiana. |
| `SplineIK3D` | spline | (clase en la jerarquía verificada) | Curvas suaves (cola). Siempre determinista. |

- **`IterateIK3D`** (base de FABRIK/CCD/Jacobian): props verificadas — `max_iterations 4`, `angular_delta_limit 0.034906585` (≈2°), `min_distance 0.001`, `deterministic false`, `setting_count 0`; límites por joint (`get/set_joint_limitation(index, joint)`, resource `JointLimitation3D`, `get_joint_rotation_axis` → `RotationAxis`).

### IK determinista vs no (concepto oficial 4.6)

- **Deterministic** = "the result does not depend on the state of the previous frame" (oficial).
- `TwoBoneIK3D` y `SplineIK3D`: **siempre** deterministas (oficial).
- `IterateIK3D` (y por lo tanto FABRIK/CCD/Jacobian): depende de `deterministic` (default `false`):
  - `false`: la iteración **lleva estado** del frame anterior → con pocas iteraciones el end bone llega "eventualmente" (más suave, no reproducible).
  - `true`: sin estado → "if the number of iterations per frame is small, the end bone may **never reach its goal**" (oficial); ideal para **red** ("only the coordinates of the IK target are shared to synchronize the model's pose" — oficial).
- "deterministic IK cannot avoid causing rotation with large angular velocities by its design. LimitAngularVelocityModifier3D is useful for smoothing this out" (oficial).

### Constraints y "mejoras" (diseño oficial 4.6)

- Filosofía oficial: los solvers son **mínimos**; el "ajuste estético" son modificadores separados que se encadenan:
  - `BoneConstraint3D` (4.5; en 4.6 puede referirse a un **`Node3D`**, no solo a un bone — oficial): un bone (`apply_bone`) sigue el transform de un reference (bone o nodo). Métodos verificados: `set_amount(index, amount)`, `set_apply_bone(_name)`, `set_reference_bone(_name)`, `set_reference_node`, `set_reference_type` (enum `ReferenceType`), `set_setting_count`, `clear_setting`.
  - `LookAtModifier3D` (4.4): rota un bone a mirar un target (props verificadas: `bone`/`bone_name`, `origin_from`, `origin_safe_margin 0.1`, `forward_axis` default `4` = +Z, `primary_rotation_axis` default `1` = Y, `primary/secondary_limit_angle`, `duration`/`ease_type`, `relative`). Nota oficial: con múltiples LookAt, **el del parent va arriba del del hijo** en la lista.
  - `BoneTwistDisperser3D` (4.6): "provides a simple way" a aplicar rotación de descendientes a ancestros (twist disperso) — reemplaza el setup manual complejo con `ConvertTransformModifier3D`.
  - `LimitAngularVelocityModifier3D` (4.6): limita velocidad angular (suaviza el IK determinista).
- Ejemplo oficial "magnet IK" (del viejo `SkeletonIK3D`) = **FABRIK determinista 2-pass** + `LimitAngularVelocityModifier3D` (oficial).

## API relevante

Todo verificado en Godot 4.7 (páginas de clase + artículo oficial).

### `SkeletonModifier3D` (base)

| Elemento | Detalle |
|---|---|
| Prop `active` | `true`. |
| Prop `influence` | `1.0` — el `Skeleton3D` blend; aplicar **solo 100%** en el código (oficial). |
| Señal `modification_processed()` | Leer poses **en ese instante** (oficial). |
| Virtual `_process_modification_with_delta(delta)` | Para modificadores custom (el `_process_modification()` está deprecado). |
| Virtual `_skeleton_changed(old, new)`, `_validate_bone_names()` | Custom. |
| `get_skeleton()` | El `Skeleton3D` padre (oficial). |
| Enum `BoneAxis` | `PLUS_X=0, MINUS_X=1, PLUS_Y=2, MINUS_Y=3, PLUS_Z=4, MINUS_Z=5`. |
| Enum `BoneDirection` | `PLUS_X=0 … MINUS_Z=5, FROM_PARENT=6`. |
| Enum `SecondaryDirection` | `NONE=0, PLUS_X=1 … MINUS_Z=6, CUSTOM=7`. |
| Enum `RotationAxis` | `X=0, Y=1, Z=2, ALL=3, CUSTOM=4`. |

### `IKModifier3D` (base de solvers)

| Elemento | Detalle |
|---|---|
| Prop `mutable_bone_axes` | `true` — `true`: lee ejes del pose cada frame; `false`: los cachea del rest ("increases performance slightly, but position changes in the bone pose made before processing this IKModifier3D are ignored" — oficial). |
| `clear_settings()` / `get_setting_count()` / `set_setting_count(count)` / `reset()` | Settings (verificado). |

### `TwoBoneIK3D` (los métodos toman `index`)

`get/set_root_bone(_name)`, `get/set_middle_bone(_name)`, `get/set_end_bone(_name)`, `get/set_pole_node`, `get_pole_direction`/`get_pole_direction_vector`, `get/set_target_node`, `get/set_end_bone_length`, `get/set_end_bone_direction`, `set_extend_end_bone`, `is_end_bone_extended`, `is_using_virtual_end` (verificados). Nota oficial: "If there are more than one bone between each set bone, their rotations are ignored, and the straight line connecting the root-middle and middle-end joints are treated as **virtual bones**."

### `ChainIK3D` / `IterateIK3D`

- `ChainIK3D`: `get/set_root_bone(_name)`, `get/set_end_bone(_name)`, `get_joint_bone(_name)(index, joint)`, `get_joint_count(index)`, `set_extend_end_bone` (verificados).
- `IterateIK3D`: props `max_iterations 4`, `angular_delta_limit 0.034906585`, `min_distance 0.001`, `deterministic false`, `setting_count 0`; `get/set_target_node(index)`; `get/set_joint_limitation(index, joint)` (`JointLimitation3D`); `get_joint_rotation_axis(index, joint)` (verificados).

## Arquitectura recomendada

### Escena: el modificador ES hijo del Skeleton3D

```text
Character (Node3D)
├─ AnimationTree / AnimationPlayer   (la animación: godot-animationtree)
├─ Skeleton3D                          (los huesos)
│  ├─ TwoBoneIK3D                       (HIJO del skeleton — requisito, oficial)
│  │  # settings/0: root_bone, middle_bone, end_bone, target_node, pole
│  ├─ LookAtModifier3D                  (cabeza → mira al player)
│  └─ BoneTwistDisperser3D              (reparte twist del brazo)
├─ ArmTarget (Marker3D)                 (el objetivo; lo mueve la lógica del juego)
└─ HandTarget (BoneAttachment3D)        (ojo: hijo de un bone del skeleton)
```

1. **Hijo del `Skeleton3D`** siempre ("retrieves a target Skeleton3D by having a Skeleton3D parent" — oficial).
2. **El target vive afuera** (o en `BoneAttachment3D`) y el juego lo mueve; el solver hace el resto.
3. **Orden en la lista** importa: retarget/animación primero (los mezcladores corren post-`AnimationMixer`), luego los ajustes (twist, limit velocity).
4. **Un solver por función** (brazo = `TwoBoneIK3D`, cola = `FABRIK3D`, cabeza = `LookAtModifier3D`); no un `ChainIK3D` genérico para todo.

## Implementación mínima

**Brazo 2-bone que sigue un target** (editor-first; código solo para mover el target):

```text
1. Skeleton3D → añadir hijo TwoBoneIK3D (Add Node)
2. settings/0: root_bone="UpperArm", middle_bone="LowerArm", end_bone="Hand"
3. target_node → $ArmTarget (Marker3D)
4. pole: pole_node → $PoleTarget (o pole direction vector)
```

```gdscript
# En el game loop (el target se mueve; el solver hace el IK):
func _physics_process(delta: float) -> void:
	arm_target.global_position = grab_point   # la mano va al punto a agarrar
# El TwoBoneIK3D (hijo del Skeleton3D) resuelve solo, post-animación (oficial)
```

## Implementación recomendada

### 1. Mano que "pisotea" terreno (2-bone + pole dinámico)

```text
TwoBoneIK3D settings/0:
  root_bone="Thigh", middle_bone="Calf", end_bone="Foot"
  target_node → $FootTarget   (proyectado al suelo por raycast — godot-raycast3d)
  pole → pole_node en la rodilla (delante)
```

### 2. Cabeza que mira (LookAt con límites)

```text
LookAtModifier3D (hijo del Skeleton3D):
  bone_name="Head"
  origin_from: BONE (neck)        # enum OriginFrom (verificado la prop)
  primary_rotation_axis: Y
  primary_limit_angle: 45.0       # no que le gire 360°
  primary_damp_threshold: 5.0     # banda muerta (no micro-rotaciones)
  target: el player (Node3D)
```

### 3. IK determinista para red

```text
FABRIK3D (IterateIK3D):
  deterministic = true            # of: reproducible sin state
  max_iterations: 10              # of: "pocas iteraciones → puede no llegar"
+ LimitAngularVelocityModifier3D  # of: suaviza las rotaciones grandes
# En red: solo se sincronizan las coords del target (oficial)
```

### 4. "Magnet IK" (patrón oficial del artículo 4.6)

```text
FABRIK3D #1 (deterministic, 1 pass)
FABRIK3D #2 (deterministic, 1 pass)   # 2 passes
LimitAngularVelocityModifier3D        # suavizado (oficial)
# = emula el "magnet" del viejo SkeletonIK3D (oficial)
```

### 5. Twist disperso (brazo)

```text
BoneTwistDisperser3D (hijo del Skeleton3D, DESPUÉS del IK):
  # reparte el twist del resultado IK a los ancestros (oficial:
  # "simple way to achieve" lo que ConvertTransformModifier3D hacía a mano)
```

## Ejemplo práctico

**Juego**: acción con grappling y cabeza que mira.

1. **Brazo** (`TwoBoneIK3D`): target = `BoneAttachment3D` de la mano (el rope lo mueve); pole = nodo del codo.
2. **Cabeza** (`LookAtModifier3D`): mira al objetivo de mira; límites ±45° en X.
3. **Antena** (`FABRIK3D`): cadena simple sin límites (ideal, oficial); `deterministic = false` (no hay red en la antena).
4. **Red** (si el grappling se sincroniza): FABRIK `deterministic = true` + `max_iterations` alto + `LimitAngularVelocity` (patrón oficial).
5. **Debug** "el codo se quiebra cuando el target está cerca": nota oficial de `TwoBoneIK3D` (virtual bones) → verificar que root/middle/end están en cadena consecutiva; y para CCD/FABRIK: "target close to root → zig-zag/flips" (oficial).

## Integración

- **`godot-skeleton3d`**: el skeleton que se modifica (poses, meta, señales `pose_updated`/`skeleton_updated`).
- **`godot-animationtree`**: la animación corre **antes** (los modificadores post-`AnimationMixer`, oficial).
- **`godot-retargeting`**: `RetargetModifier3D` es de la misma familia (hijo del skeleton); el orden entre retarget e IK lo da la lista.
- **`godot-character-controller`**: el target suele ser el `CharacterBody3D` del jugador (posiciones).
- **`godot-raycast3d`**: proyectar targets al suelo (pies) — verificado en Lote 4.
- **`godot-performance` (Lote 14 pendiente)**: el coste de iterar (ver § Performance).

## Errores frecuentes

1. **Usar `IKConstraint3D`/`InverseKinematics3D` (3.x)** → **no existen en 4.7** (verificado 404); API nueva = `SkeletonModifier3D` family.
2. **El modificador NO es hijo del `Skeleton3D`** → no encuentra el skeleton ("by having a Skeleton3D parent" — oficial); nada pasa.
3. **Aplicar `influence` a mano en el código del solver** → "must not apply influence… Skeleton3D automatically applies influence" (oficial); el `influence` es prop del nodo.
4. **Esperar el IK en `_process` del juego** → corre en el update del skeleton, **post-`AnimationMixer`** (oficial); leer el pose en `modification_processed()`.
5. **FABRIK/CCD con el target pegado a la raíz** → zig-zag (FABRIK) / flips y oscilación (CCD) — notas oficiales; alejar el target o cambiar solver.
6. **`deterministic = true` con `max_iterations` bajo** → "the end bone may never reach its goal" (oficial); subir iteraciones (a costa de CPU).
7. **`mutable_bone_axes = false` cuando el pose cambia antes del IK** → los ejes cacheados del rest "are ignored" los cambios (nota oficial); dejar `true` (default) salvo cuello medido.
8. **Múltiples `LookAtModifier3D` con el hijo arriba del parent** → "the LookAtModifier3D assigned to the parent bone must be put **above** the one assigned to the child bone" (nota oficial).
9. **`ChainIK3D` directo** → es la **base** (auto-genera joints); usar `FABRIK3D`/`CCDIK3D`/`JacobianIK3D`/`SplineIK3D` (hijos verificados).
10. **Esperar que el IK "sustituya" la animación** → se aplica **después** de la mezcla (oficial); si la animación mueve esos huesos, el IK los sobrescribe (diseño).
11. **Settings múltiples sin entender el `index`** → cada setting es una cadena independiente (`settings/<index>/...`, nota oficial); `set_setting_count` antes de configurar.
12. **IK en 2D con estos nodos** → son `*3D`; en 2D no hay IK nativa (verificado: la familia es 3D).

## Anti-patrones

- **Un solver IK gigante para todo el cuerpo** → un solver por función (brazo/pierna/cabeza) + constraints de ajuste.
- **`influence` animado a mano por frame en el script** → es prop del nodo; el `Skeleton3D` blend (oficial).
- **Re-implementar el solver "a mano"** cuando `FABRIK3D`/`CCDIK3D` existe → el sistema es mínimo por diseño (oficial); el custom va en `_process_modification_with_delta`.
- **`mutable_bone_axes = false` "para performance" sin medir** → la nota oficial dice que ignora cambios de pose previos; solo con cuello medido (spec §60).
- **Add-on IK de 3.x/migrado** cuando el core 4.6+ lo resuelve → primero el core (verificado aquí); el add-on solo para casos que el core no cubre.
- **Pole estático fijo "una vez"** → el pole es lo que define el doblado; un `pole_node` que se mueve (o `pole_direction_vector`) controla el gesto.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Coste real**: los solvers iterativos (`IterateIK3D` y por lo tanto FABRIK/CCD/Jacobian) hacen trabajo por **iteración × joints** por frame; `TwoBoneIK3D`/`SplineIK3D` (deterministas, cierre directo) son más baratos.
- **Qué medir**: `Performance.TIME_PROCESS`/`TIME_PHYSICS_PROCESS` (familia verificada 4.7) con N modificadores vs 0; contar solvers activos (`active = false` desactiva — oficial).
- **Optimizaciones (contra un cuello medido)**:
  1. `active = false` en solvers que no aplican (el prop existe, oficial).
  2. `max_iterations` acorde al look (no "100 por si acaso"; default 4).
  3. `mutable_bone_axes = false` cuando el pose no cambia antes del solver (nota oficial: "increases performance slightly").
  4. Menos modificadores encadenados (cada uno corre post-animación).
- **No**: "el IK es barato, no medir" — 10 solvers iterativos en un esqueleto de 100 bones no lo es.

## Debugging

### "Nada pasa (el bone no se mueve)"

```text
1. ¿El modificador es HIJO del Skeleton3D? (requisito, oficial)
2. ¿active = true? (default true)
3. ¿target_node apunta a un nodo que existe?
4. ¿root/middle/end (2-bone) o root/end (cadena) están en cadena?
```

### "El end bone no llega al target"

```text
1. ¿deterministic = true con pocas iteraciones? (oficial: puede no llegar)
2. ¿El target está más allá del alcance de la cadena? (longitudes de bones)
3. angular_delta_limit (default ~2°) × max_iterations: ¿el paso permite llegar?
4. set_extend_end_bone (verificado) si el end debe extenderse
```

### "El bone hace zig-zag / flip / oscila"

```text
1. Target cerca de la raíz: notas oficiales de FABRIK (zig-zag) y CCD
   (flips/oscilación) → cambiar solver o alejar target
2. Pole mal dirigido (2-bone) → pole_node/pole_direction_vector
3. Iterativo sin smooth: + LimitAngularVelocityModifier3D (oficial)
```

### "El resultado del IK no es el que veo en el inspector"

```text
1. Leer el pose EN modification_processed() (oficial: en ese instante)
2. ¿Otro modificador corre DESPUÉS y lo pisa? (orden en la lista)
3. ¿La animación sigue moviendo ese bone en el mismo frame? (post-mix)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tablas de la familia + artículo oficial 4.6 (lista en *Referencias*).
- **Disponibilidad por versión (oficial)**: `SkeletonModifier3D`/`LookAtModifier3D`/`RetargetModifier3D` = **4.4**; `SpringBoneSimulator3D`/`BoneConstraint3D`(+hijos) = **4.5**; `IKModifier3D` + 7 solvers, `BoneTwistDisperser3D`, `LimitAngularVelocityModifier3D`, referencia a `Node3D` en `BoneConstraint3D` = **4.6**. En 4.7 todo lo listado existe (verificado en las tablas).
- **3.x → 4.x**: `IKConstraint3D`/`InverseKinematics3D` (3.x) **no existen en 4.7** (verificado 404); el reemplazo es esta familia. El `SkeletonModifier3D` como API de base de modificadores apareció en 4.4.
- **`IKConstraint3D` en 4.0–4.3**: deprecado en 4.0, removido después (la historia oficial dice "IK was removed during the upgrade to 4.0" y volvió en 4.6 — artículo oficial).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-skeleton3d` | El skeleton modificado (obligatoria) |
| `godot-animationtree` | La animación corre antes (post-mix) |
| `godot-character-controller` | El target suele ser el player |
| `godot-raycast3d` | Proyectar targets (pies al suelo) |
| `godot-retargeting` | Misma familia de modificadores |

## Skills relacionadas

- `godot-recipe-2bone-arm-ik` — pendiente (germen en *Implementación mínima*)
- `godot-recipe-head-look-at` — pendiente (germen en *Implementación recomendada* #2)
- `godot-error-ik-flip` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Inverse Kinematics Returns to Godot 4.6 (timeline 4.4/4.5/4.6, jerarquía de 7 solvers, IK determinista, magnet IK, "removed in 4.0, back in 4.6"): https://godotengine.org/article/inverse-kinematics-returns-to-godot-4-6/
- SkeletonModifier3D (base, `active`/`influence`, `modification_processed`, enums, "post-AnimationMixer", `_process_modification_with_delta`): https://docs.godotengine.org/en/stable/classes/class_skeletonmodifier3d.html
- IKModifier3D (`mutable_bone_axes`, settings, `reset()`): https://docs.godotengine.org/en/stable/classes/class_ikmodifier3d.html
- TwoBoneIK3D (pole, virtual bones, métodos por `index`): https://docs.godotengine.org/en/stable/classes/class_twoboneik3d.html
- ChainIK3D (auto-genera joints raíz→final): https://docs.godotengine.org/en/stable/classes/class_chainik3d.html
- IterateIK3D (`max_iterations 4`, `angular_delta_limit`, `min_distance`, `deterministic`, joint limitations): https://docs.godotengine.org/en/stable/classes/class_iterateik3d.html
- FABRIK3D / CCDIK3D (descripciones + notas de zig-zag/flips): https://docs.godotengine.org/en/stable/classes/class_fabrik3d.html , https://docs.godotengine.org/en/stable/classes/class_ccdik3d.html
- BoneConstraint3D (reference bone/nodo, `amount`, settings): https://docs.godotengine.org/en/stable/classes/class_boneconstraint3d.html
- LookAtModifier3D (props, orden parent>child): https://docs.godotengine.org/en/stable/classes/class_lookatmodifier3d.html
