# godot-control

## ID

`godot-control`

## Categoría

UI · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Control` (tabla de props + descripción de input/focus/theme) (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — props verificadas (anchors `0.0` ×4, offsets `0.0` ×4, `clip_contents false`, `custom_minimum_size (0,0)`, `custom_maximum_size (-1,-1)`, `focus_mode 0`, `grow_horizontal/vertical 1`, `layout_direction 0`, `mouse_filter 0`, `mouse_default_cursor_shape 0`, `position (0,0)`, `size (0,0)`, `size_flags_horizontal/vertical 1`, `size_flags_stretch_ratio 1.0`, `pivot_offset (0,0)`, `scale (1,1)`, `rotation(_degrees)`, `theme`, `theme_type_variation`, `tooltip_text`, `translation_context`, `accessibility_*`, `auto_translate`, `localize_numeral_system true`, `shortcut_context`, `propagate_maximum_size false`, `offset_transform_*`, `mouse_force_pass_scroll_events true`, `focus_neighbor_*`, `focus_next/previous`), virtuales (`_gui_input`, `_get_minimum_size`, `_has_point`, `_get_tooltip`, `_get_cursor_shape`, `_get_drag_data`/`_can_drop_data`/`_drop_data`), métodos (`accept_event`, `add_theme_*_override` ×6, `begin/end_bulk_theme_override`), reglas de focus/mouse_filter/theme (oficiales): verificados directamente.
**MEDIUM** — los valores numéricos de los enums donde solo se verificó el default (`SizeFlags.FILL=1`, `MouseFilter.STOP=0`/`PASS=1` vía defaults; el resto de valores son estándar de la familia 4.x).
**Contraste verificado**: `Control` **SÍ** tiene `pivot_offset` en 4.7 (a diferencia de `Node2D`, que no lo tiene — verificado en Lote 5).

## Nivel

**PRINCIPIANTE** (es la raíz del cluster UI; la usan `godot-containers`, `godot-theme`, `godot-localization`).

## Propósito

Proporcionar el conocimiento operativo de `Control` (la base de toda la UI en Godot 4.7): el sistema de **anchors + offsets** (y cómo se auto-actualizan), el sizing (size flags, min/max), el input GUI (`_gui_input`, `mouse_filter`, `accept_event`), el **focus** (reglas oficiales), los **theme overrides** (y la trampa de que los theme items NO son Object properties), el pivot, y la arquitectura HUD (CanvasLayer). Es la skill de "cómo se posiciona, redimensiona, recibe input y se estiliza un control".

## Cuándo utilizarla

- Posicionar/ajustar controles a diferentes resoluciones (anchors/offsets).
- Hacer un HUD (que no se mueva con el mundo).
- Recibir input de UI (`_gui_input`, clicks, drag & drop).
- Focus de teclado/gamepad en menús.
- Estilizar UN control (overrides) vs muchos (`godot-theme`).

## Cuándo NO utilizarla

- **Layouts complejos** (filas, columnas, grids, scroll) → `godot-containers` (el container controla el posicionamiento de los hijos — oficial).
- **Estilizar toda la UI / un tema** → `godot-theme`.
- **Texto traducido / i18n** → `godot-localization` (esta skill solo ancla `auto_translate`/`translation_context`).
- **Nodos del mundo (2D/3D)** → `godot-node2d` / `Node3D` (el Control es para UI, hereda `CanvasItem` como Node2D — verificado).
- **Dibujar custom en UI** → `Control._draw()` (mencionado; la docs de custom drawing 2D aplica).

## Conceptos fundamentales

### Anchors + offsets: el posicionamiento

- Oficial: "Control features a bounding rectangle that defines its extents, **an anchor position relative to its parent control or the current viewport, and offsets relative to the anchor**. The offsets update automatically when the node, any of its parents, or the screen size change."
- Props (verificados, defaults 0.0): `anchor_left/top/right/bottom` (fracción 0..1 de la base) + `offset_left/top/right/bottom` (px relativos al anchor).
- La "base" es el parent control (o el viewport si no hay parent control).
- **El layout se re-calcula solo** (resize de ventana/parent): no "pegar" la posición en `_process`.
- `pivot_offset` (default `(0,0)` — verificado; `Control` SÍ lo tiene, `Node2D` no): el pivote de `rotation`/`scale` (y `pivot_offset_ratio`).
- `size_flags_*` (default `1` = FILL) + `size_flags_stretch_ratio` (1.0): cómo el control pide espacio dentro de un container (ver `godot-containers`).

### El input GUI (no es el input normal)

- Oficial: para UI, "it makes more sense to override the virtual method `_gui_input()`, which filters out unrelated input events, such as by checking z-order, `mouse_filter`, focus, or if the event was inside of the control's bounding box."
- **`accept_event()`**: "so no other node receives the event. Once you accept an input, it becomes handled so `Node._unhandled_input()` will not process it."
- **`mouse_filter`** (default `0` = STOP; `1` = PASS; verificado el default y el valor 1 vía Container):
  - STOP: consume el evento (default de controles activos).
  - PASS: lo consume si lo "toca" pero lo deja pasar si no.
  - IGNORE: "to tell a Control node to ignore mouse or touch events. You'll need it if you place an icon on top of un button" (oficial, ícono sobre botón).
- `mouse_default_cursor_shape` (default `0` = ARROW): cursor por defecto del control.
- Drag & drop: virtuales `_get_drag_data`/`_can_drop_data`/`_drop_data` (verificados).

### Focus (reglas oficiales)

- "Only one Control node can be in focus. Only the node in focus will receive events."
- `grab_focus()` para tomar; se pierde "when another node grabs it, or if you hide the node in focus".
- **Visual**: "Focus will not be represented visually if gained via mouse/touch input, only appearing with keyboard/gamepad input (for accessibility), or via `grab_focus()`."
- `focus_mode` (default `0` = NONE): si el control puede recibir focus.
- `focus_next`/`focus_previous`/`focus_neighbor_top/left/bottom/right` (verificados): navegación manual del focus (gamepad/teclado).
- `focus_behavior_recursive` (verificado): si el focus es recursivo a hijos.
- **CanvasLayer no está en esta tabla** (es un nodo aparte para HUD — ver *Arquitectura*).

### Theme (anclaje; el dominio es `godot-theme`)

- Oficial: "Theme resources change the control's appearance. The `theme` of a Control node affects all of its direct and indirect children (as long as a chain of controls is uninterrupted)."
- Overrides: `add_theme_color_override`, `add_theme_constant_override`, `add_theme_font_override`, `add_theme_font_size_override`, `add_theme_icon_override`, `add_theme_stylebox_override` (verificados) + `begin_bulk_theme_override()`/`end_bulk_theme_override()` (verificados, para batches).
- ⚠️ **Trampa oficial**: "Theme items are not Object properties. This means you can't access their values using Object.get() and Object.set(). Instead, use the `get_theme_*` and `add_theme_*_override` methods."
- `theme_type_variation` (verificado): usar una variación de tema en un control (ver `godot-theme`).

### i18n (anclaje; el dominio es `godot-localization`)

- `auto_translate` (verificado): el control traduce su texto automáticamente si coincide con una key.
- `translation_context` (verificado): contexto de traducción.
- `layout_direction` (verificado, default `0` = INHERIT): LTR/RTL (el mirroring RTL es automático — oficial en `godot-localization`).
- `localize_numeral_system` (verificado, default `true`).

## API relevante

Todo verificado en Godot 4.7 (tabla de `Control`).

### Props (principales; defaults verificados)

| Prop | Default | Nota |
|---|---|---|
| `anchor_left/top/right/bottom` | `0.0` | Fracción de la base (0..1). |
| `offset_left/top/right/bottom` | `0.0` | px relativos al anchor. |
| `position` / `size` | `(0,0)` / `(0,0)` | Rect resultante (el layout lo recalcula). |
| `custom_minimum_size` | `(0,0)` | Tamaño mínimo. |
| `custom_maximum_size` | `(-1,-1)` | Máximo (-1 = sin tope). |
| `size_flags_horizontal/vertical` | `1` (FILL) | Flags de sizing en container. |
| `size_flags_stretch_ratio` | `1.0` | Proporción entre expanders. |
| `grow_horizontal/vertical` | `1` (BEGIN) | Hacia dónde crece. |
| `pivot_offset` / `pivot_offset_ratio` | `(0,0)` | Pivote de rotación/escala (Control SÍ lo tiene). |
| `rotation` / `rotation_degrees` / `scale` | `0.0` / — / `(1,1)` | Transform del control. |
| `clip_contents` | `false` | Clipar hijos al rect. |
| `mouse_filter` | `0` (STOP) | STOP/PASS/IGNORE. |
| `mouse_default_cursor_shape` | `0` (ARROW) | Cursor. |
| `focus_mode` | `0` (NONE) | ¿Recibe focus? |
| `focus_next/previous/neighbor_*` | `NodePath("")` | Navegación manual. |
| `theme` / `theme_type_variation` | — / `&""` | Tema del subtree / variación. |
| `tooltip_text` | `""` | Tooltip. |
| `shortcut_context` | — | Contexto de shortcuts. |
| `auto_translate` / `translation_context` | — / `&""` | i18n. |
| `layout_direction` | `0` (INHERIT) | LTR/RTL. |
| `accessibility_*` | varios | Nombre/descripción/relaciones (a11y). |
| `offset_transform_*` | varios | Transform offset (nuevo en 4.x; verificado la tabla). |
| `propagate_maximum_size` | `false` | (Container lo sobreescribe a `true` — verificado). |

### Virtuales (override)

| Método | Uso |
|---|---|
| `_gui_input(event: InputEvent)` | Input del control (filtrado por z/mouse_filter/focus/rect). |
| `_get_minimum_size()` / `_get_maximum_size()` | Sizing custom. |
| `_has_point(point)` | Hit test custom. |
| `_get_tooltip(at_position)` / `_make_custom_tooltip(for_text)` | Tooltips. |
| `_get_cursor_shape(at_position)` | Cursor por posición. |
| `_get_drag_data(at_position)` / `_can_drop_data(at_position, data)` / `_drop_data(at_position, data)` | Drag & drop. |

### Métodos clave

| Método | Nota |
|---|---|
| `accept_event()` | "el evento queda handled" (oficial). |
| `grab_focus()` / `release_focus()` | Focus (oficial: solo uno a la vez). |
| `add_theme_color_override(name, color)` / `..._constant_override` / `..._font_override` / `..._font_size_override` / `..._icon_override` / `..._stylebox_override` | Overrides de tema (no son props — oficial). |
| `get_theme_*` (color/constant/font/font_size/icon/stylebox) | Leer el tema aplicado. |
| `begin_bulk_theme_override()` / `end_bulk_theme_override()` | Batches de overrides. |
| `find_next_valid_focus(...)` | Navegación de focus. |

## Arquitectura recomendada

### El HUD (CanvasLayer + Control raíz)

```text
Game (Node2D/Node3D)
├─ Level (el mundo)
└─ CanvasLayer                          ← NO se mueve con el mundo (HUD)
   └─ UI (Control)                       ← full rect (anchors 0..1)
      ├─ HealthBar (HBoxContainer)       ← godot-containers
      ├─ Minimap (TextureRect)
      └─ Menus (VBoxContainer)
```

1. **`CanvasLayer`** para todo HUD (no se afecta por la cámara 2D/3D).
2. **El Control raíz del UI** en full rect: `anchor_right = 1.0`, `anchor_bottom = 1.0` (o "Full Rect" del inspector).
3. **Anchors para el layout base; containers para el interior** (oficial: "To build flexible UIs, you'll need a mix of UI elements that inherit from Control and Container nodes").
4. **Temas en la raíz** (`godot-theme`): un `Theme` en el Control raíz afecta todo el subtree (oficial).

## Implementación mínima

**HUD base que responde a resize** (anchors verificados):

```gdscript
# En el editor: UI (Control) con anchors full rect.
# Un panel en la esquina superior derecha:
#   anchor_left = 1.0, anchor_top = 0.0, anchor_right = 1.0, anchor_bottom = 0.0
#   offset_left = -300, offset_top = 8, offset_right = -8, offset_bottom = 8
# (el layout se recalcula solo al redimensionar — oficial)
```

**Click en un control** (patrón oficial):

```gdscript
extends Control

func _gui_input(event: InputEvent) -> void:
	if event is InputEventMouseButton and event.pressed and event.button_index == MOUSE_BUTTON_LEFT:
		do_thing()
		accept_event()          # el evento queda "handled" (oficial)
```

## Implementación recomendada

### 1. Ícono sobre un botón (sin robar el click)

```text
Button
└─ TextureRect (icono)
   mouse_filter = MOUSE_FILTER_IGNORE   (oficial: "if you place an icon on top of a button")
```

### 2. Menú con focus de gamepad

```text
1. focus_mode = ALL (en los controles navegables)
2. focus_next / focus_previous (o focus_neighbor_*) para el orden
3. grab_focus() al abrir el menú
# El focus visual solo aparece con keyboard/gamepad o grab_focus (oficial)
```

### 3. Estilizar UN control sin tocar el tema

```gdscript
# Los theme items NO son Object properties (oficial): no usar get/set
label.add_theme_color_override("font_color", Color.MAGENTA)   # verificado
label.add_theme_font_size_override("font_size", 24)
# Batches:
label.begin_bulk_theme_override()
# ... varios add_theme_*_override ...
label.end_bulk_theme_override()
```

### 4. Rotar/escalar un control alrededor de su centro

```gdscript
pivot_offset = size * 0.5     # pivot al centro (Control SÍ tiene pivot_offset — verificado)
# o pivot_offset_ratio = Vector2(0.5, 0.5)
rotation_degrees = 45.0
```

### 5. Drag & drop

```gdscript
func _get_drag_data(at_position: Vector2) -> Variant:
	# iniciar drag: devolver el data (y un ícono de preview)
	return item

func _can_drop_data(at_position: Vector2, data: Variant) -> bool:
	return data is Item

func _drop_data(at_position: Vector2, data: Variant) -> void:
	do_drop(data, at_position)
```

## Ejemplo práctico

**Juego**: RPG con HUD, minimap, inventario y menú de pausa.

1. **`CanvasLayer`** + Control raíz full rect (anchors 1.0/1.0).
2. **HUD** con containers (`godot-containers`): barra de vida arriba, minimap arriba-derecha (anchor 1.0/0.0 + offsets negativos), contador de oro abajo-izquierda.
3. **Inventario**: `GridContainer` con slots (TextureRect IGNORE sobre los íconos).
4. **Menú pausa**: focus gamepad (`focus_mode = ALL`, `focus_next` en orden), `grab_focus()` al abrir.
5. **Debug** "el botón no recibe el click" → hay un Control encima con `mouse_filter = STOP` → ponerle `IGNORE` (oficial).
6. **Debug** "el panel no se ajusta al resize" → anchors mal (deben ser 0..1 relativos a la base; offsets en px).

## Integración

- **`godot-containers`**: el layout del interior (los hijos dan su posicionamiento al container — oficial).
- **`godot-theme`**: el estilo (tema global + overrides locales).
- **`godot-localization`**: el texto (auto_translate, context, RTL).
- **`godot-node2d`**: el otro hijo de `CanvasItem` (comparten `z_index`/`visible` — nota oficial).
- **`godot-camera2d`**: la cámara del mundo (el HUD en `CanvasLayer` no se mueve con ella).
- **`godot-rendering-performance`**: el UI es fill rate 2D (lo que se muestra se renderiza).

## Errores frecuentes

1. **`Object.get("font_color")`** → "Theme items are not Object properties" (oficial) → `get_theme_color("font_color")`.
2. **Setear `position` a mano en runtime** → el layout lo recalcula al redimensionar (oficial); usar anchors/offsets o containers.
3. **Esperar que el focus visual aparezca con click** → "only appearing with keyboard/gamepad input, or via grab_focus()" (oficial).
4. **Dos nodos "en focus"** → "Only one Control node can be in focus" (oficial); `grab_focus` transfiere.
5. **`pivot_offset` en `Node2D`** → NO existe en `Node2D` 4.7 (verificado Lote 5); en `Control` SÍ (verificado).
6. **Ícono sobre botón que "traga" el click** → `mouse_filter = IGNORE` en el ícono (oficial).
7. **`anchor_right = 0` (default) para "full width"** → el default de todos los anchors es `0.0` (verificado); full rect = left 0, top 0, right 1, bottom 1.
8. **Esperar que `size` sea estable** → es el resultado del layout (size flags, container, anchors); no hardcodear.
9. **`clip_contents = false` "por defecto"** → si un hijo se sale del rect (scroll, marquee), activarlo.
10. **`grow_direction` equivocado en container** → el control crece hacia el lado equivocado (default BEGIN, verificado).

## Anti-patrones

- **Posicionar a mano 30 controles con `position`** → anchors + containers (oficial: el layout se recalcula solo).
- **Un `CanvasLayer` por control** → un `CanvasLayer` para el HUD; los controles van adentro.
- **Overrides de tema "por si acaso" en cada control** → un `Theme` en la raíz (`godot-theme`); los overrides son para excepciones.
- **`_unhandled_input` para clicks de UI** → `_gui_input` (oficial: filtra z-order/mouse_filter/focus/rect).
- **`custom_minimum_size` gigante "para que no se achique"** → es para contenido real; el sizing de container lo resuelve el layout (flags).
- **`visible = false` para "ocultar" vs `mouse_filter`** → `visible` también quita el input (y el focus si estaba en focus — oficial); elegir la semántica correcta.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El UI es fill rate 2D**: lo que se muestra se renderiza (overdraw de paneles, textos grandes). El layout (anchors/containers) es CPU barata.
- **Qué medir**: FPS con el HUD completo vs oculto (ver `godot-rendering-performance`; el truco de la ventana pequeña detecta fill-rate limit).
- **Optimizaciones (contra un cuello medido)**:
  1. Menos `clip_contents`/overlaps (el overdraw de 20 paneles superpuestos suma).
  2. `visible = false` en pantallas inactivas (no solo `mouse_filter`).
  3. Textos: menos `RichTextLabel` pesados; `Label` para el simple.
  4. Layout: no re-crear controles por frame (el layout es automático — oficial).
- **No**: "el UI pesa" sin medir — normalmente es el fill rate, no el layout.

## Debugging

### "El control se mueve / se achica al cambiar de resolución"

```text
1. Anchors: ¿los 4 son 0..1 correctos? (full rect = 0,0,1,1)
2. Offsets: ¿en px relativos al anchor?
3. ¿Está dentro de un container? (el container controla su posición — oficial)
```

### "No recibe el click / el click va a otro nodo"

```text
1. ¿Un Control encima con mouse_filter = STOP? (íconos → IGNORE, oficial)
2. ¿focus_mode / visible?
3. ¿El rect del control cubre el punto? (_has_point)
4. Z-order: los Controls se ordenan por la escena + z_index (CanvasItem)
```

### "El focus no se mueve con el gamepad"

```text
1. focus_mode = ALL (default NONE — verificado)
2. focus_next/focus_previous/neighbor_* (verificados)
3. ¿El control está visible? (el focus se pierde al ocultar — oficial)
```

### "El texto/panel se sale del contenedor"

```text
1. clip_contents = true (default false — verificado)
2. custom_minimum_size del contenido (¿el text es más ancho que el slot?)
3. size_flags del hijo dentro del container (godot-containers)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla de `Control` (lista en *Referencias*).
- **3.x → 4.x**: `Control` es la misma familia; `pivot_offset` existe en `Control` en 3.x y 4.7 (verificado en 4.7). Los enums `SizeFlags`/`MouseFilter`/`FocusMode` son de la familia 4.x.
- **2D ↔ Control (verificado)**: ambos heredan `CanvasItem` y comparten `z_index`/`visible` (nota oficial); NO comparten `pivot_offset` (Node2D no lo tiene en 4.7 — verificado Lote 5).
- **4.0 ↔ 4.7**: la tabla consultada es la de 4.7; `offset_transform_*` y `accessibility_*` presentes (no afirmar la versión exacta de introducción sin verificar).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-containers` | El layout del interior |
| `godot-theme` | El estilo |
| `godot-localization` | El texto |
| `godot-node2d` | El otro hijo de CanvasItem |
| `godot-camera2d` | El HUD no se mueve con la cámara (CanvasLayer) |

## Skills relacionadas

- `godot-recipe-hud-scaling` — pendiente (germen en *Arquitectura*)
- `godot-error-ui-not-resizing` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Control (props, virtuales, métodos, reglas de focus/mouse_filter/theme, "theme items are not Object properties", CanvasItem sharing): https://docs.godotengine.org/en/stable/classes/class_control.html
- Using Containers (el container controla el posicionamiento de los hijos; sizing options — ver `godot-containers`): https://docs.godotengine.org/en/stable/tutorials/ui/gui_containers.html
- Introduction to GUI skinning (theme cascading, items, types — ver `godot-theme`): https://docs.godotengine.org/en/stable/tutorials/ui/gui_skinning.html
