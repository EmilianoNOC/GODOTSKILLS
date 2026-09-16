# godot-containers

## ID

`godot-containers`

## Categoría

UI · Base (layout)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Container` (tabla completa: props/señales/constantes/métodos + Inherited By) y el tutorial "Using Containers" (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — `Container` base (props: `accessibility_region false`, `mouse_filter 1` = PASS **override**, `propagate_maximum_size true` **override**; señales `pre_sort_children`/`sort_children`; constantes `NOTIFICATION_PRE_SORT_CHILDREN = 50` / `NOTIFICATION_SORT_CHILDREN = 51`; métodos `fit_child_in_rect`, `queue_sort`, virtuales `_get_allowed_size_flags_horizontal/vertical`), Inherited By completa (13 types), "los hijos dan su posicionamiento al container" (oficial), sizing options Fill/Expand/Shrink/Stretch Ratio (oficial, tutorial), "la UI del editor de Godot es enteramente containers" (oficial): verificados directamente.
**MEDIUM** — props específicas de cada container (HBox `theme constant separation`, Grid `columns`, etc.) — no se verificaron tabla por tabla en esta revisión; verificar la clase específica antes de citar su default.
**No duplica**: el anchor/offset del Control raíz es `godot-control`; aquí es el layout **dentro** del container.

## Nivel

**PRINCIPIANTE** (depende de `godot-control`).

## Propósito

Proporcionar el conocimiento operativo de los `Container` de Godot 4.7: la regla de oro ("los hijos dan su posicionamiento al container"), las **sizing options** (Fill/Expand/Shrink/Stretch Ratio), la lista completa de types (los 13 de la Inherited By), cuándo usar cuál (el mapa del tutorial oficial), el nesting (la fuerza real), y el diagnóstico de los fallos clásicos (el hijo "no respeta" su posición, el layout "colapsa", el expand "no reparte").

## Cuándo utilizarla

- Layouts que deben reflow (filas, columnas, grids, scroll, tabs, splits).
- UI "OS-like" (RPGs, chats, tycoons, tools — oficial).
- Margenes/padding por tema (MarginContainer).
- "El layout no se ve como en el editor" (sizing options).

## Cuándo NO utilizarla

- **UI simple de 3-4 controles con anchors** → `godot-control` (anchors/offsets alcanzan; los containers son para layouts complejos — oficial).
- **El anchor del control raíz / HUD** → `godot-control` (CanvasLayer + full rect).
- **El estilo (colores, fuentes, estilo de tabs)** → `godot-theme` (los containers usan theme constants/styleboxes).
- **Nodos del mundo** → `godot-node2d` (los containers son Control).

## Conceptos fundamentales

### La regla de oro: los hijos ceden su posicionamiento

Oficial (tutorial 4.7): "When a Container-derived node is used, **all children Control nodes give up their own positioning ability**. This means the Container will control their positioning and **any attempt to manually alter these nodes will be either ignored or invalidated the next time their parent is resized**."

→ No setear `position`/`size` a mano en hijos de containers; el layout lo hace el container.
- "The real strength of containers is that they can be **nested** (as nodes), allowing the creation of very complex layouts that resize effortlessly." (oficial)
- "the Godot editor user interface is **entirely done using them**" (oficial) — el estándar de facto.

### Sizing options (por hijo, por eje)

Oficial (tutorial 4.7), independientes por eje vertical/horizontal (no todos los containers usan todas):

| Opción | Efecto oficial |
|---|---|
| **Fill** | "Ensures the control fills the designated area within the container." (on por default). |
| **Expand** | "Attempts to use as much space as possible in the parent container (in each axis). Controls that don't expand will be pushed away by those that do." |
| **Shrink Begin** | "When expanding, try to remain at the left or top of the expanded area." |
| **Shrink Center** | "…remain at the center of the expanded area." |
| **Shrink End** | "…remain at the right or bottom of the expanded area." |
| **Stretch Ratio** | "The ratio of how much expanded controls take up the available space in relation to each other. A control with '2' will take up twice as much available space as one with '1'." |

- Mapeo a props de `Control` (verificados en `godot-control`): `size_flags_horizontal/vertical` (BitField de `SizeFlags`) + `size_flags_stretch_ratio`.
- "Experimenting with these flags and different containers is recommended" (oficial).

### La familia (Inherited By verificados, 13)

| Type | Uso (tutorial oficial) |
|---|---|
| `HBoxContainer` / `VBoxContainer` | Filas/columnas; "In the opposite of the designated direction, it just expands the children." (oficial). Usan Stretch Ratio con Expand. |
| `GridContainer` | Grid ("amount of columns must be specified" — oficial); usa expand horizontal y vertical. |
| `MarginContainer` | "Child controls are expanded towards the bounds… Padding will be added on the margins depending on the theme configuration." (oficial) — los márgenes son **tema** (constant), no props. |
| `TabContainer` | "several child controls stacked on top of each other, with only the current one visible" (oficial); títulos por defecto = nombre del nodo; API para override. |
| `HSplitContainer` / `VSplitContainer` | "Arranges child controls vertically or horizontally and creates **grabbers** between them" (oficial) (el usuario puede redimensionar el split). |
| `CenterContainer` | Centra un hijo. |
| `AspectRatioContainer` | Mantiene aspect ratio. |
| `FlowContainer` (H/V) | Flow (wrapping) de hijos. |
| `FoldableContainer` | Plegable (collapse/expand). |
| `PanelContainer` | Panel con theme (StyleBox) + un hijo. |
| `ScrollContainer` | Scroll de contenido (válido en un eje). |
| `SubViewportContainer` | Muestra un `SubViewport` (minimaps, split screen). |
| `EditorProperty` / `GraphElement` | Editor (fuera de scope de juego). |

### Container base: props y señales (verificadas)

- `mouse_filter` = `1` (PASS) — **override** de `Control` (default STOP): el container no traga los clicks (los hijos sí).
- `propagate_maximum_size` = `true` — **override** de `Control` (default false).
- `accessibility_region` = `false`: "this container is marked as a region for accessibility… Screen readers can navigate between regions using landmark navigation" (oficial).
- Señales: `pre_sort_children()` ("going to be sorted") y `sort_children()` ("sorting the children is needed").
- Constantes: `NOTIFICATION_PRE_SORT_CHILDREN = 50` ("just before children are going to be sorted"), `NOTIFICATION_SORT_CHILDREN = 51` ("it must be obeyed immediately").
- Métodos: `queue_sort()` (re-sorteo), `fit_child_in_rect(child, rect)` (custom); virtuales `_get_allowed_size_flags_horizontal/vertical()` (para containers custom: "limits the options available to the user in the Inspector dock" — oficial).

## API relevante

Todo verificado en Godot 4.7.

### `Container` (< Control)

| Elemento | Detalle |
|---|---|
| `mouse_filter` | `1` (PASS) — override. |
| `propagate_maximum_size` | `true` — override. |
| `accessibility_region` | `false` (region para screen readers — oficial). |
| Señal `pre_sort_children()` | Antes de sortear. |
| Señal `sort_children()` | Cuando hace falta sortear. |
| `NOTIFICATION_PRE_SORT_CHILDREN` | `50`. |
| `NOTIFICATION_SORT_CHILDREN` | `51` ("must be obeyed immediately"). |
| `queue_sort()` | Re-sorteo deferred. |
| `fit_child_in_rect(child, rect)` | API de layout custom. |
| `_get_allowed_size_flags_horizontal/vertical()` | Para containers custom (Inspector). |

### Sizing options → props (ver `godot-control`)

| Opción (Inspector) | Prop |
|---|---|
| Fill | `size_flags_*` con `FILL` (default `1`) |
| Expand | `size_flags_*` con `EXPAND` |
| Shrink Begin/Center/End | `size_flags_*` con `SHRINK_BEGIN`/`SHRINK_CENTER`/`SHRINK_END` |
| Stretch Ratio | `size_flags_stretch_ratio` (default `1.0`) |

## Arquitectura recomendada

### El patrón canónico: CanvasLayer → raíz full rect → containers

```text
CanvasLayer
└─ UI (Control, full rect)            ← godot-control (anchors)
   └─ VBoxContainer                    ← el layout vertical principal
      ├─ HBoxContainer (HUD top)
      │  ├─ HealthBar (PanelContainer)
      │  └─ Minimap (TextureRect)
      ├─ CenterContainer (game screen)
      └─ HBoxContainer (HUD bottom)
         ├─ ItemIcon (HBox, Flow)
         └─ GoldLabel
```

1. **Un container por "zona"**; nesting libre (oficial: la fuerza real).
2. **`MarginContainer`** para padding (márgenes por **tema** — oficial; ver `godot-theme`).
3. **Expand + Stretch Ratio** para el reparto de espacio (el "game screen" expande; el HUD no).
4. **`ScrollContainer`** para contenido variable (inventarios largos) — con un eje de scroll válido.
5. **`PanelContainer`** para paneles con estilo (StyleBox del tema).

## Implementación mínima

**Barra de inventario horizontal con scroll** (todo containers; cero `position` a mano):

```text
HBoxContainer (inventario)
└─ ScrollContainer (horizontal)
   └─ HBoxContainer (slots)
      ├─ PanelContainer (slot 1)
      ├─ PanelContainer (slot 2)
      └─ ...
# Los slots: sin Expand (toman su size mínimo); el ScrollContainer hace el scroll
```

## Implementación recomendada

### 1. Reparto de espacio con Stretch Ratio

```text
HBoxContainer
├─ PanelA:   Expand, Stretch Ratio 1   →  1/3 del ancho
├─ PanelB:   Expand, Stretch Ratio 2   →  2/3 del ancho  (oficial: "2 → twice as much")
└─ PanelC:   (sin Expand)              →  su size mínimo
```

### 2. Margen por tema (no por prop)

```text
MarginContainer (arriba del layout)
# Los márgenes son constantes del TEMA (oficial):
# en el inspector: Theme Overrides → Constants → margin_left/right/top/bottom
# o por código (ver godot-theme):
margin_container.add_theme_constant_override("margin_left", 16)
```

### 3. Tabs para sub-menus

```text
TabContainer
├─ Character (nodo)   → título "Character"
├─ Inventory (nodo)   → título "Inventory"
└─ Map (nodo)         → título "Map"
# Títulos = nombre del nodo por defecto (oficial); override por la API de TabContainer
```

### 4. Split ajustable por el usuario

```text
HSplitContainer (o VSplit)
├─ PanelIzq
└─ PanelDer
# El grabber entre los dos permite redimensionar (oficial)
```

### 5. Minimapa (SubViewport)

```text
SubViewportContainer (en el HUD)
└─ SubViewport
   └─ Camera2D (zoom 0.2, godot-camera2d)
```

## Ejemplo práctico

**Juego**: RPG con inventario grid, tabs y menú de pausa.

1. **Inventario**: `GridContainer` (columns = 5) dentro de `ScrollContainer` (vertical).
2. **Menú**: `TabContainer` (Character/Inventory/Map); `MarginContainer` afuera para el padding (tema).
3. **HUD top**: `HBoxContainer` con barra de vida (`PanelContainer`) + minimapa (`SubViewportContainer` expand ratio 0.5).
4. **Reparto**: el "game screen" `CenterContainer` con Expand ratio 1; los HUD sin Expand (toman lo que necesitan).
5. **Debug** "el slot no se agranda al Expand" → el `GridContainer` reparte por celdas; si querés celdas variables, HBox con Stretch Ratio.
6. **Debug** "el scroll no hace scroll" → `ScrollContainer` con un solo eje válido y el hijo con `size_flags` correctas.

## Integración

- **`godot-control`**: la base (anchors, sizing props, input, focus).
- **`godot-theme`**: los márgenes/padding y los styleboxes de los containers (tema — oficial).
- **`godot-localization`**: el texto de los labels/botones dentro del layout (los containers reflow con strings largos — ver pseudolocalización).
- **`godot-camera2d`**: el minimapa (`SubViewportContainer` + `SubViewport` + `Camera2D`).
- **`godot-rendering-performance`**: el UI es fill rate (ver `godot-control`).

## Errores frecuentes

1. **Setear `position`/`size` a mano en hijos de containers** → "will be either ignored or invalidated the next time their parent is resized" (oficial); usar sizing options.
2. **Expand sin entender el reparto** → "Controls that don't expand will be pushed away by those that do" (oficial); el espacio entre expanders lo reparte `size_flags_stretch_ratio`.
3. **Esperar padding por props en `MarginContainer`** → "the margins are a Theme value" (oficial); son constantes del tema.
4. **Dos ejes de scroll en `ScrollContainer`** → es un solo eje; anidar dos `ScrollContainer` (uno por eje).
5. **`GridContainer` sin `columns`** → "amount of columns must be specified" (oficial).
6. **Hijos de container con `mouse_filter = STOP` "por si acaso"** → el container es PASS por default (verificado); los hijos deciden su propio `mouse_filter`.
7. **`TabContainer` esperando títulos por props** → "titles are generated from the node names by default (although they can be overridden via TabContainer API)" (oficial).
8. **Esperar que `propagate_maximum_size` del container sea false** → es `true` (override verificado; el `Control` base es `false`).
9. **Re-sortear el layout "a mano"** → `queue_sort()` (verificado) o mutar y dejar que el layout re-corra.
10. **Un `FlowContainer` para "todo"** → el flow wrapping no respeta filas/celdas; Grid para tablas, HBox/VBox para filas/colunas.

## Anti-patrones

- **Anchors manuales para un layout de 10 controles** → containers (oficial: "the Godot editor user interface is entirely done using them").
- **Un container gigante para todo el HUD** → por zonas (HBox top, Center medio, HBox bottom) — más mantenible.
- **`custom_minimum_size` en cada hijo "para que no colapse"** → las sizing options (Fill/Expand) resuelven el sizing; el min size es para contenido.
- **Hardcodear el padding en px** → `MarginContainer` + tema (oficial) → un solo lugar (`godot-theme`).
- **`ScrollContainer` en ambos ejes "por si acaso"** → un eje por container (anidar si hace falta).
- **Ignorar las sizing options en el Inspector** → "Experimenting with these flags and different containers is recommended" (oficial); el 90% de los "bug de layout" son flags.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El layout es barato** (re-calculación deferred); el coste real es **lo que se muestra** (fill rate 2D — ver `godot-control`) y los **sorts de muchos hijos**.
- **Qué medir**: FPS con el HUD completo vs oculto; si hay "resort" visible (items reordenándose), `pre_sort_children`/`sort_children` por frame es síntoma de datos que cambian por frame.
- **Optimizaciones (contra un cuello medido)**:
  1. Menos hijos en un solo container (un grid de 500 slots = 500 controles visibles; considerar virtualizar con un `ItemList` — existe en 4.7, verificado en Inherited By de `Control`).
  2. `visible = false` en tabs/panels inactivos (el `TabContainer` solo muestra el current — oficial).
  3. `clip_contents` en scroll (evita draw de lo que no se ve… verificar el comportamiento al implementar).
- **No**: "el layout pesa" sin medir — normalmente es el fill rate o el nº de controles.

## Debugging

### "El hijo no respeta su posición / se mueve solo"

```text
1. ¿Está dentro de un container? (entonces el container manda — oficial)
2. Sizing options del hijo (Fill/Expand/Shrink)
3. ¿Un container padre anidado está re-floweando?
```

### "El Expand no reparte el espacio"

```text
1. ¿El hijo tiene Expand ACTIVO en el eje del container? (HBox → horizontal)
2. size_flags_stretch_ratio (los ratios relativos — oficial)
3. ¿Otro hijo sin Expand "empuja"? (oficial: "pushed away")
```

### "El layout colapsa al cambiar de tamaño"

```text
1. custom_minimum_size del contenido (text largo)
2. clip_contents (¿el contenido se sale?)
3. ¿El container tiene tamaño 0? (el padre no le da espacio → anchors/flags del container)
```

### "El scroll no funciona"

```text
1. ¿Eje válido del ScrollContainer (un solo eje)?
2. El contenido es más grande que el viewport del scroll
3. size_flags del hijo del scroll (no Expand en el eje del scroll)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla de `Container` + tutorial (lista en *Referencias*).
- **3.x → 4.x**: `Container` es la misma familia; los 13 types de la Inherited By 4.7 incluyen `FlowContainer`, `FoldableContainer`, `AspectRatioContainer` (4.x); en 3.x faltaban algunos.
- **4.0 ↔ 4.7**: la tabla consultada es la de 4.7 (no afirmar diferencias menores sin verificar).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-control` | La base (obligatoria) |
| `godot-theme` | Márgenes/styleboxes (oficial: tema) |
| `godot-localization` | Reflow con strings largos |
| `godot-camera2d` | Minimapa (SubViewport) |

## Skills relacionadas

- `godot-recipe-inventory-grid` — pendiente (germen en *Ejemplo práctico*)
- `godot-error-container-reflow` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Container (props con overrides, señales, NOTIFICATION 50/51, `queue_sort`, `fit_child_in_rect`, Inherited By de 13 types): https://docs.godotengine.org/en/stable/classes/class_container.html
- Using Containers (regla de oro, sizing options, tipos: Box/Grid/Margin/Tab/Split, "la UI del editor es containers"): https://docs.godotengine.org/en/stable/tutorials/ui/gui_containers.html
- Control (sizing props: `size_flags_*`, `custom_minimum_size` — ver `godot-control`): https://docs.godotengine.org/en/stable/classes/class_control.html
