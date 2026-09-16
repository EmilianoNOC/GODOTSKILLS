# godot-blendspace

## ID

`godot-blendspace`

## Categoría

Animation · Base (mezcla)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: páginas de clase `AnimationNodeBlendSpace1D` y `AnimationNodeBlendSpace2D` (props/métodos/enums completos) (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — props completas de ambos (1D: `blend_mode`, `cyclic_length`, `min_space -1.0`, `max_space 1.0`, `snap 0.1`, `sync`, `sync_mode`, `value_label "value"`; 2D: `auto_triangles true`, `min/max_space Vector2(±1,±1)`, `snap Vector2(0.1,0.1)`, `x_label/y_label`, `triangles_updated`), métodos (`add_blend_point`, `find_blend_point_by_name`, `get_blend_point_count/name/node/position`, `remove/reorder_blend_point`, `set_blend_point_*`, `add_triangle`, `get_triangle_count/point`, `remove_triangle`), enums (`BlendMode` 3 valores, `SyncMode` **4** valores), "blend lineal de los dos nodos adyacentes" (1D) / "blend lineal de los **tres** adyacentes (triángulo que contiene al punto)" (2D): verificados directamente en la docs 4.7.
**MEDIUM** — valores de diseño (posiciones de puntos, snap, duraciones de clip) — no son reglas oficiales; iterar con el resultado.
**No duplica**: el cableado de `AnimationNodeBlendTree` (nodo "output", `add_node`/`connect_node`, parámetros `parameters/...`) vive en `godot-animationtree` — esta skill da el dominio completo de los BlendSpace.

## Nivel

**INTERMEDIATE** (depende de `godot-animationtree`).

## Propósito

Proporcionar el conocimiento operativo completo de `AnimationNodeBlendSpace1D`/`2D`: la API exacta (props, enums, métodos), el significado de `BlendMode`/`SyncMode` (incl. los nuevos `CYCLIC_MUTABLE`/`CYCLIC_CONSTANT`), los modos discretos para animación 2D frame-a-frame, triángulos en 2D, y los patrones típicos: locomoción por velocidad (1D), 4/8 direcciones (2D), mezcla de capas (run + aim), y el diagnóstico de los fallos clásicos (clip "saltando", desincronización, "no llega" al extremo).

## Cuándo utilizarla

- Locomoción: walk→run con un eje (velocidad).
- Movimiento direccional: 4/8 direcciones con dos ejes.
- Animación 2D frame-a-frame (modo DISCRETE).
- Clips de distinto largo que deben "bailar juntos" (sync modes).
- Depurar "el blend salta / se desfasa / queda congelado".

## Cuándo NO utilizarla

- **El árbol completo** (StateMachine, OneShot, BlendTree wiring) → `godot-animationtree`.
- **Blend de 2 clips fijo** (ataque normal vs fuerte) → `AnimationNodeBlend2` (más simple; `godot-animationtree`).
- **Transición temporal entre estados** → `AnimationNodeStateMachine` + crossfade (`godot-animationtree`).
- **IK / retargeting / huesos** → `godot-ik` / `godot-retargeting` / `godot-skeleton3d`.

## Conceptos fundamentales

### Qué es un BlendSpace

Recurso (`AnimationRootNode`) usado dentro de un `AnimationNodeBlendTree`: "a set of AnimationRootNodes placed on a virtual axis, crossfading between the two adjacent ones" (1D, oficial) / "placed on 2D coordinates, crossfading between the **three** adjacent ones" (2D, oficial). El output es el blend lineal de los adyacentes al valor actual.

- **1D**: un eje (p. ej. velocidad 0→máx). "Outputs the linear blend of the two adjacent AnimationRootNodes."
- **2D**: un plano (p. ej. dirección X/Y). "Adjacent in this context means the three AnimationRootNodes **making up the triangle that contains the current value**."
- Los puntos se agregan con `add_blend_point(node, pos, at_index=-1, name=&"")` — el nodo debe ser un `AnimationRootNode` (típicamente `AnimationNodeAnimation`).
- Extensión del eje: `min_space`/`max_space` (defaults 1D: **-1.0/1.0**; 2D: `Vector2(-1,-1)`/`Vector2(1,1)`).

### `blend_position`: la palanca que el código controla

- El BlendSpace expone el parámetro **`blend_position`** (float en 1D, Vector2 en 2D) en el `AnimationTree` (`$Tree.get("parameters/Move/blend_position")` — patrón verificado en 4.7, ver `godot-animationtree`).
- El código **pasa el valor crudo** (velocidad, dirección) y el BlendSpace **mezcla**; no hacer lerp manual de pesos (anti-patrón, ver `godot-animationtree`).
- `snap` (default 1D 0.1 / 2D `Vector2(0.1,0.1)`): cuántos puntos se "agarran" alrededor del valor (reduce cambios de mezcla por ruido).

### `BlendMode` (verificado: 3 valores)

| Valor | Descripción oficial | Uso |
|---|---|---|
| `BLEND_MODE_INTERPOLATED (0)` | "The interpolation between animations is linear." (default) | Locomoción 3D (walk/run) |
| `BLEND_MODE_DISCRETE (1)` | "plays the animation of the node which blending position is closest to. Useful for **frame-by-frame 2D animations**." | Sprite frames 2D |
| `BLEND_MODE_DISCRETE_CARRY (2)` | "Similar to DISCRETE, but starts the new animation at the last animation's playback position." | 2D sin "reset" de loop |

### `SyncMode` (verificado: 4 valores — la prop `sync` quedó como legacy)

| Valor | Descripción oficial |
|---|---|
| `SYNC_MODE_NONE (0)` | "Inactive animations are frozen and do not advance." (default) |
| `SYNC_MODE_INDEPENDENT (1)` | "Inactive animations advance with a weight of 0. This is equivalent to the previous `sync = true` behavior." |
| `SYNC_MODE_CYCLIC_MUTABLE (2)` | "All animations are time-scaled so they stay in sync, with the cycle length dynamically computed from active blend weights. Self-normalizing: a solo animation plays at normal speed." |
| `SYNC_MODE_CYCLIC_CONSTANT (3)` | "All animations are time-scaled so they complete one cycle in `cyclic_length` seconds, keeping them in sync regardless of their individual lengths." |

- Nota oficial (4.7) para `CYCLIC_*`: si aplicás `AnimationNodeTimeSeek` al resultado con clips de distinto largo, **se rompe la sincronía**; usar `AnimationNodeAnimation.use_custom_timeline` para alinear largos.
- `cyclic_length` (default 0.0): el ciclo (s) que usan los modos `CYCLIC_CONSTANT`; "must be greater than 0 for cyclic sync to take effect".
- **Elección práctica**: clips de locomoción de distinto largo (walk 0.7s, run 0.5s) que se ven "juntos" → `CYCLIC_MUTABLE` (se auto-normaliza) o `CYCLIC_CONSTANT` con `cyclic_length` (control total).

### 2D: triángulos

- `auto_triangles` (default **true**): triangulación automática de los puntos.
- Manual: `add_triangle(x, y, z, at_index=-1)` + `remove_triangle(triangle)` + `get_triangle_count()`/`get_triangle_point(triangle, point)` (índices de vértice).
- Señal `triangles_updated()` (verificada): "emitted every time the blend space's triangles are created, removed, or when one of their vertices changes position".
- Si la triangulación automática no da lo que querés (p. ej. forma irregular), definirla a mano.

## API relevante

Todo verificado en Godot 4.7.

### `AnimationNodeBlendSpace1D` (hereda `AnimationRootNode`)

Props:

| Prop | Default | Nota |
|---|---|---|
| `blend_mode` | `BLEND_MODE_INTERPOLATED (0)` | Ver tabla. |
| `sync_mode` | `SYNC_MODE_NONE (0)` | Ver tabla. |
| `sync` | (bool legacy) | Equivale a `sync_mode = INDEPENDENT` (verificado en el texto de `INDEPENDENT`). |
| `min_space` / `max_space` | `-1.0` / `1.0` | Extremos del eje. |
| `snap` | `0.1` | Tolerancia de "agarre" al punto. |
| `cyclic_length` | `0.0` | Ciclo (s) para `CYCLIC_CONSTANT`. |
| `value_label` | `"value"` | Etiqueta del eje (inspector). |

Métodos:

| Método | Nota |
|---|---|
| `add_blend_point(node: AnimationRootNode, pos: float, at_index: int = -1, name: StringName = &"")` | Agregar clip al eje. |
| `get_blend_point_count()` / `get_blend_point_name(point)` / `get_blend_point_node(point)` / `get_blend_point_position(point)` | Consultar. |
| `set_blend_point_name(point, name)` / `set_blend_point_node(point, node)` / `set_blend_point_position(point, pos)` | Editar. |
| `find_blend_point_by_name(name)` | → índice (-1 si no existe). |
| `remove_blend_point(point)` / `reorder_blend_point(from, to)` | Gestionar. |

### `AnimationNodeBlendSpace2D` (hereda `AnimationRootNode`)

Props: idem 1D pero `min_space`/`max_space` = `Vector2`, `snap` = `Vector2(0.1, 0.1)`, y además:

| Prop | Default | Nota |
|---|---|---|
| `auto_triangles` | `true` | Triangulación automática. |
| `x_label` / `y_label` | `"x"` / `"y"` | Etiquetas de ejes. |

Métodos adicionales (verificados): `add_triangle(x, y, z, at_index=-1)`, `get_triangle_count()`, `get_triangle_point(triangle, point)`, `remove_triangle(triangle)`, señal `triangles_updated()`.

## Arquitectura recomendada

### Locomoción 1D (el patrón canónico)

```text
AnimationTree
└─ output (AnimationNodeBlendTree)
   └─ Locomotion (AnimationNodeBlendSpace1D)   [min 0, max 1 (o velocidad máx)]
      ├─ Idle   (pos 0.0)
      ├─ Walk   (pos 0.4)
      └─ Run    (pos 1.0)
```

1. Un `BlendSpace1D` por "canal" de mezcla (velocidad; dirección).
2. Los puntos van en **posiciones con sentido** (0=parado, 1=máx), no en "0 y 1 y a rezar".
3. El `blend_position` lo escribe el controller (`godot-character-controller` da el valor de velocidad; esta skill da el cómo mezclar).
4. OneShots (ataque) **por encima** de la locomoción en el grafo (`godot-animationtree`).

### Dirección 2D (4/8 direcciones)

```text
Direction (AnimationNodeBlendSpace2D)   [min (-1,-1), max (1,1)]
├─ Back      (0, -1)
├─ Left      (-1, 0)
├─ Right     (1, 0)
└─ Forward   (0, 1)
   # 8 dirs: sumar (±0.707, ±0.707); auto_triangles=true resuelve el resto
```

- Pasar la **dirección normalizada** (o con magnitud = intensidad) como `blend_position`.
- Verificar la triangulación con `triangles_updated` (o el inspector) si hay artefactos en diagonales.

## Implementación mínima

**Locomoción 1D por velocidad** (patrón oficial de `godot-animationtree`, el blend_position se escribe así en 4.7):

```gdscript
# En el personaje (AnimationTree como hijo):
var tree: AnimationTree
func _physics_process(delta: float) -> void:
	var speed := absf(velocity.x)
	tree.set("parameters/Locomotion/blend_position", clampf(speed / run_speed, 0.0, 1.0))
# Si los clips walk/run tienen distinto largo y querés que "balancen" juntos:
# tree.get("animation").sync_mode = ... NO: es prop del nodo BlendSpace:
# en el editor: Locomotion.sync_mode = SYNC_MODE_CYCLIC_MUTABLE
```

## Implementación recomendada

### 1. Locomoción con clips de largo distinto (sync)

```text
BlendSpace1D "Locomotion":
  sync_mode = SYNC_MODE_CYCLIC_MUTABLE   (clips walk 0.7s / run 0.5s)
  # o
  sync_mode = SYNC_MODE_CYCLIC_CONSTANT
  cyclic_length = 0.6                    (un ciclo de 0.6s para todos)
```

- Si necesitás `AnimationNodeTimeSeek` encima: alinear largos con `AnimationNodeAnimation.use_custom_timeline` (nota oficial) o la sync se rompe.

### 2. 8 direcciones con 2D

```gdscript
# blend_position = dirección normalizada del input (get_vector)
var dir := Input.get_vector("left", "right", "up", "down")
tree.set("parameters/Direction/blend_position", dir)  # Vector2 (-1..1 en cada eje)
# Los 8 puntos van en las 8 posiciones unitarias; auto_triangles = true (default)
```

### 3. Frame-a-frame 2D (modo DISCRETE)

```text
BlendSpace1D "Sprite" (blend_mode = BLEND_MODE_DISCRETE):
  Frames 0..N en posiciones 0.0, 0.1, 0.2, ...
# Código:
tree.set("parameters/Sprite/blend_position", frame_index * 0.1)
# DISCRETE_CARRY si no querés que el loop "reinicie" al cambiar de frame
```

### 4. Capas de mezcla (run + aim)

```text
output (BlendTree)
├─ Move (BlendSpace1D, locomoción)
└─ Aim (BlendSpace1D, por ángulo)      # capa por encima con Transition/Blend
   # o AnimationNodeAdd3 para sumar capas (ver godot-animationtree)
```

### 5. Crear el BlendSpace por código (raro, pero posible)

```gdscript
var bs := AnimationNodeBlendSpace1D.new()
bs.min_space = 0.0
bs.max_space = 1.0
bs.add_blend_point(AnimationNodeAnimation.new(), 0.0, -1, "idle")  # API verificado
# ... set_blend_point_node / set_blend_point_name ...
# Y conectarlo en el BlendTree (ver godot-animationtree: add_node/connect_node)
```

## Ejemplo práctico

**Juego**: acción 3D con 8-way run + capa de apuntado.

1. `Locomotion` (1D): idle 0.0, walk 0.4, run 1.0; `min 0`, `max 1`.
2. `Direction` (2D): 8 puntos unitarios; `auto_triangles = true`.
3. `sync_mode = CYCLIC_MUTABLE` en ambos (clips importados de distinto largo).
4. `Aim` (1D por ángulo -1..1) por encima con `AnimationNodeAdd3` (ver `godot-animationtree`).
5. **Debug** "el run no aparece hasta casi al final": el punto `run` está en 1.0 pero el `blend_position` se normaliza mal → imprimir el valor que escribís.
6. **Debug** "las diagonales se ven raras": triángulos → desactivar `auto_triangles` y definir `add_triangle` manual.

## Integración

- **`godot-animationtree`**: el cableado (BlendTree, nodo "output", `parameters/...`, StateMachine, OneShots) vive ahí; esta skill es el dominio BlendSpace.
- **`godot-character-controller`**: produce el `blend_position` (velocidad/dirección); no duplicar su lógica de movimiento aquí.
- **`godot-skeleton3d`**: los clips que se mezclan animan huesos del skeleton.
- **`godot-ik`**: el IK se aplica **después** de la mezcla (los modificadores corren post-`AnimationMixer` — verificado).
- **`godot-performance` (Lote 14 pendiente)**: el coste del blend (ver § Performance).

## Errores frecuentes

1. **Lerps manuales de pesos en vez de `blend_position`** → el BlendSpace **es** el tuning (verificado el diseño; anti-patrón documentado en `godot-animationtree`).
2. **Esperar que la prop `sync` haga "ciclo"** → `sync` es legacy (= `INDEPENDENT`, verificado en la descripción); para ciclos: `sync_mode = CYCLIC_*` + `cyclic_length`.
3. **`CYCLIC_CONSTANT` con `cyclic_length = 0`** (default) → "must be greater than 0 for cyclic sync to take effect" (nota oficial).
4. **`AnimationNodeTimeSeek` encima de clips de distinto largo con `CYCLIC_*`** → "synchronization will be broken" (nota oficial); usar `use_custom_timeline`.
5. **`min_space`/`max_space` dejados en ±1 y escribir velocidad 0..8** → el blend se satura; normalizar el `blend_position` o ajustar los extremos.
6. **Puntos muy juntos + `snap` bajo** → el blend "salta" entre pares adyacentes por ruido del input; subir `snap` (default 0.1) o limpiar el input.
7. **`BLEND_MODE_DISCRETE` en locomoción 3D** → el modo es para frame-a-frame 2D (descripción oficial); en 3D usar `INTERPOLATED`.
8. **Triangulación manual mal hecha (2D)** → blend incorrecto en diagonales; verificar con `triangles_updated`/inspector.
9. **Esperar que el BlendSpace "guarde" el blend_position** → es un parámetro del `AnimationTree` (`parameters/...`); el nodo no lo persiste (ver `godot-animationtree`).
10. **Un BlendSpace2D para "todo"** (velocidad + dirección en el mismo plano) → separar canales (1D velocidad × 2D dirección); mezclar canales distintos en un plano rompe el significado del blend.

## Anti-patrones

- **Re-implementar crossfade con `AnimationPlayer`** cuando un BlendSpace lo resuelve → el nodo existe para eso.
- **BlendTree gigante para locomoción** (40 nodos) → un BlendSpace1D/2D por canal cubre el 99% (ver `godot-animationtree`).
- **Puntos de clip sin nombre** → `add_blend_point` acepta `name`; sin nombres, el inspector y el debug son inusables.
- **`auto_triangles = false` "por si acaso"** → dejar `true` (default) y solo manual cuando hay un problema concreto (diagonal que se ve mal).
- **Hardcodear `cyclic_length` por ensayo sin medir** → elegir el sync mode por el síntoma (clips "desfasados" = sin sync; "diferente velocidad de ciclo" = `CYCLIC_CONSTANT`).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Qué mide**: el blend de N clips tiene un coste proporcional a la **cantidad de clips activos en el blend** (los adyacentes) y a la longitud de los tracks (huecos incluidos). En un BlendSpace, solo los 2-3 adyacentes participan → barato por diseño.
- **Monitorear**: `Performance.TIME_PROCESS`/`TIME_PHYSICS_PROCESS` (familia verificada en 4.7, ver `godot-rendering-performance`) con y sin el BlendSpace; si el cuello es "muchos tracks", el remedio es del clip (tracks innecesarios, "Remove Tracks" del import — ver `godot-retargeting`).
- **Optimizaciones (contra un cuello medido)**:
  1. Menos puntos en el espacio (cada punto adyacente suma mezcla).
  2. Clips sin tracks muertos (el import puede quitarlos; ver `godot-retargeting` "Except Bone Transform").
  3. `SYNC_MODE_NONE` (default) congela los inactivos → más barato que `INDEPENDENT` (que los avanza con peso 0).
- **No**: "el blend es gratis, no medir" — un BlendSpace con 30 puntos y clips de 200 tracks no lo es.

## Debugging

### "El blend salta / el clip parpadea"

```text
1. Imprimir el blend_position que escribís (¿ruido del input? ¿normalización?)
2. snap (default 0.1): subirlo para "agarrar" el punto
3. Puntos demasiado juntos en el eje (¿la distancia real justifica 2 clips?)
```

### "Los clips se ven desfasados al mezclar"

```text
1. sync_mode: ¿están congelados (NONE default) o deben "bailar juntos"?
2. CYCLIC_MUTABLE (auto) vs CYCLIC_CONSTANT (cyclic_length > 0)
3. ¿AnimationNodeTimeSeek encima con clips de distinto largo? (nota oficial)
```

### "No llega a Run / el extremo no se activa"

```text
1. max_space: ¿el blend_position llega a 1.0? (clamp + normalización)
2. El punto "run" está en 1.0 y el eje termina en 1.0 (no 1.2)
3. snap alto: puede "agarrar" al punto anterior
```

### "Las diagonales (2D) se ven mal"

```text
1. Verificar la triangulación (triangles_updated / inspector)
2. auto_triangles = false + add_triangle manual (verificado el método)
3. ¿Faltan puntos en las diagonales? (8-way requiere los 8)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tablas completas de ambos nodos (lista en *Referencias*).
- **3.x → 4.x**: en 3.x eran recursos `BlendSpace1D`/`BlendSpace2D` (con `add()`/`set_blend()`); en 4.x son `AnimationNodeBlendSpace1D`/`2D` (`add_blend_point`). Migrar scripts.
- **4.0 → 4.7 (verificado en las tablas 4.7)**: la prop `sync` sigue presente pero la descripción oficial de `SYNC_MODE_INDEPENDENT` la define como "equivalent to the previous sync = true behavior" → tratar `sync_mode` como la API. `SyncMode` tiene **4** valores en 4.7 (`CYCLIC_MUTABLE`/`CYCLIC_CONSTANT` no estaban en las versiones tempranas — no afirmar su disponibilidad pre-4.x sin verificar).
- **`AnimationNodeBlendSpace` (base) sin 1D/2D**: no existe nodo base genérico; son 1D y 2D separados.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-animationtree` | El cableado del grafo (obligatoria) |
| `godot-character-controller` | El valor de `blend_position` (velocidad/dirección) |
| `godot-skeleton3d` | Los clips animan huesos |
| `godot-ik` | IK post-mezcla |
| `godot-retargeting` | Clips compartidos entre skeletons (import) |

## Skills relacionadas

- `godot-recipe-locomotion-blend` — pendiente (germen en *Implementación recomendada* #1-2; referencia en `godot-animationtree`)
- `godot-error-blend-flicker` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- AnimationNodeBlendSpace1D (props/métodos/enums completos, BlendMode 3 valores, SyncMode 4 valores, `cyclic_length`, notas de sync): https://docs.godotengine.org/en/stable/classes/class_animationnodeblendspace1d.html
- AnimationNodeBlendSpace2D (auto_triangles, add_triangle, `triangles_updated`, "triángulo que contiene el valor"): https://docs.godotengine.org/en/stable/classes/class_animationnodeblendspace2d.html
- Using AnimationTree (tutorial listado en ambas páginas): https://docs.godotengine.org/en/stable/tutorials/animation/animation_tree.html
- AnimationNodeBlendTree (wiring, nodo "output" — ver `godot-animationtree`): https://docs.godotengine.org/en/stable/classes/class_animationnodeblendtree.html
