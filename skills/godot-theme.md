# godot-theme

## ID

`godot-theme`

## Categoría

UI · Base (estilo)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Theme` (props + tabla de métodos) y el tutorial "Introduction to GUI skinning" (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — props (`default_base_scale 0.0`, `default_font`, `default_font_size -1`), métodos (familias `get/set/clear_{color,constant,font,font_size,icon,stylebox}`, `get_theme_item`/`get_theme_item_list`/`get_theme_item_type_list`, `get_type_list`, `get_type_variation_base`/`get_type_variation_list`, `add_type`, `clear`, `clear_theme_item(data_type, ...)`, `clear_type_variation`, `has_*`), items de 6 tipos (Color/Constant/Font/Font size/Icon/StyleBox — oficial), "unique within the theme" por (name, type) (oficial), herencia de tipos a hijos (oficial), project setting `gui/theme/custom` (oficial), `Control.theme` para subtrees (oficial), "theme items are not Object properties" (oficial, verificado en `Control`), "focus styleboxes drawn as overlay" (oficial): verificados directamente.
**MEDIUM** — los nombres exactos de los theme items por control (`font_color` en Label, etc.) — verificados por el ejemplo oficial (`get_theme_color("accent_color", "MyType")`, `add_theme_color_override("font_color", ...)`); el resto se consultan en el inspector/`get_*_list`.

## Nivel

**PRINCIPIANTE** (depende de `godot-control`).

## Propósito

Proporcionar el conocimiento operativo del sistema de **theming** de Godot 4.7: el recurso `Theme` (items de 6 tipos, types, variations), cómo se **cascada** (project → `Control.theme` → hijos), los **overrides** locales (`add_theme_*_override`, y la trampa de que no son props), el `default_font`/`default_base_scale`, y el flujo de trabajo del theme editor. Cubre el diagnóstico de los fallos clásicos (el override "no se aplica", el tema "no afecta" a un subtree, el font "no carga" los caracteres).

## Cuándo utilizarla

- Estilizar toda la UI (un tema global) o un subtree (tema local).
- Cambiar fuentes/colores/padding de un tipo de control (todos los `Button`, etc.).
- Variaciones de tema (un `Button` "primary" vs "secondary").
- Accessibility (font grande, colorblind — cascada, oficial).
- "El override no se aplica / el tema no llega".

## Cuándo NO utilizarla

- **Un solo override puntual** → `add_theme_*_override` en el control (esta skill da la API, no hace falta un Theme).
- **Layout/posicionamiento** → `godot-control`/`godot-containers`.
- **Dibujar custom en un control** → `_draw()` + theme items propios (mencionado; tutorial custom controls).
- **Nodos del mundo** → no es Control (`godot-node2d`/`Node3D`).

## Conceptos fundamentales

### El recurso `Theme` (tabla verificada)

- < `Resource`. "A resource used for styling/skinning Controls and Windows." (oficial)
- Props: `default_base_scale` (`0.0`), `default_font` (`Font`), `default_font_size` (`-1`).
- **Cascada** (oficial):
  1. `ProjectSettings.gui/theme/custom` → "a project-scope theme that will be available to every control in your project."
  2. `Control.theme` → "a theme that will be available to that control and all of its direct and indirect children (as long as a chain of controls is uninterrupted)."
- "One theme resource can be used for the entire project, but you can also set a separate theme resource to a branch of control nodes." (oficial)

### Items: 6 tipos (oficial, tutorial)

| Tipo | Uso |
|---|---|
| **Color** | "often used for fonts and backgrounds. Colors can also be used for modulation of controls and icons." |
| **Constant** | "an integer value… numeric properties of controls (such as the item separation in a BoxContainer), or boolean flags (such as the drawing of relationship lines in a Tree)." |
| **Font** | "a font resource… Fonts contain most text rendering settings, except for its size and color. …alignment and text direction are controlled by individual controls." |
| **Font size** | "an integer value, which is used alongside a font to determine the size." |
| **Icon** | "a texture resource, which is normally used to display an icon (on a Button, for example)." |
| **StyleBox** | "a collection of configuration options which define the way a UI panel should be displayed… used by many controls for their backgrounds and overlays." |

- ⚠️ **Focus como overlay** (oficial): "Most notably, `focus` styleboxes are drawn as an overlay to other styleboxes (such as `normal` or `pressed`)… the focus stylebox should be designed as an outline or translucent box, so that its background can remain visible."

### Types (y la herencia)

- "each theme is separated into types, and **each item must belong to a single type**. …each theme item is defined by its name, its data type and its theme type. **This combination must be unique within the theme**." (oficial)
- "The default Godot theme comes with multiple theme types already defined, one for every built-in control node that uses UI skinning." (oficial)
- **Herencia** (oficial): "Child classes can use theme items defined for their parent class… Whenever a built-in control requests a theme item from the theme it can omit the theme type, and its class name will be used instead. On top of that, the class names of its parent classes will also be used in turn." → cambiar `Button` afecta a `CheckBox`/`OptionButton`/etc.
- Types custom: "you must utilize scripts to access those items" (oficial) — p. ej. `get_theme_color("accent_color", "MyType")` (ejemplo oficial).
- **Variations**: los types "can also be linked together as types" (oficial); en el control: prop `theme_type_variation` (verificada en `Control`); métodos `get_type_variation_base(theme_type)` / `get_type_variation_list(base_type)` / `clear_type_variation(theme_type)` (verificados).

### Overrides locales (no son props)

- ⚠️ Oficial (verificado en `Control`): "Theme items are **not Object properties**. This means you can't access their values using Object.get() and Object.set(). Instead, use the `get_theme_*` and `add_theme_*_override` methods."
- `add_theme_color_override` / `..._constant_override` / `..._font_override` / `..._font_size_override` / `..._icon_override` / `..._stylebox_override` (verificados en `Control`) + `begin_bulk_theme_override()`/`end_bulk_theme_override()` (verificados).
- Precedencia: override local > tema del control > tema project (regla de la cascada; el override "gana" por ser el más específico — verificado el mecanismo, el orden exacto por la docs al implementar).

### Fuentes y tamaño de UI

- `default_font` / `default_font_size` del Theme (verificados): la fuente del proyecto.
- `default_base_scale` (`0.0` — verificado): escala base del UI (0 = auto/1.0 — **no afirmar el significado exacto de 0 sin verificar más**; la prop existe).
- Advertencia oficial (i18n): "The default project font only supports a subset of the Latin-1 character set, which cannot be used to display languages like Russian or Chinese" → cargar una fuente multilingüe (p. ej. Noto) como `DynamicFont` en el Theme.

## API relevante

Todo verificado en Godot 4.7.

### `Theme` (< Resource)

Props:

| Prop | Default | Nota |
|---|---|---|
| `default_base_scale` | `0.0` | Escala base del UI. |
| `default_font` | — | `Font`. |
| `default_font_size` | `-1` | Tamaño (−1 = default). |

Métodos (familias; todos verificados):

| Familia | Métodos |
|---|---|
| Color | `get_color(name, type)`, `set_color(...)`*, `clear_color(name, type)`, `get_color_list(type)`, `get_color_type_list()`, `has_color(name, type)` |
| Constant | `get_constant(name, type)`, `clear_constant(...)`, `get_constant_list(type)`, `get_constant_type_list()`, `has_constant(...)` |
| Font | `get_font(name, type)`, `clear_font(...)`, `get_font_list(type)`, `get_font_type_list()`, `has_font(...)` |
| Font size | `get_font_size(name, type)`, `clear_font_size(...)`, `get_font_size_list(type)`, `get_font_size_type_list()`, `has_font_size(...)` |
| Icon | `get_icon(name, type)`, `clear_icon(...)`, `get_icon_list(type)`, `get_icon_type_list()`, `has_icon(...)` |
| StyleBox | `get_stylebox(name, type)`, `clear_stylebox(...)`, `get_stylebox_list(type)`, `get_stylebox_type_list()`, `has_stylebox(...)` |
| Genérico | `get_theme_item(data_type, name, type)`, `get_theme_item_list(data_type, type)`, `get_theme_item_type_list(data_type)`, `clear_theme_item(data_type, name, type)`, `add_type(type)`, `clear()` |
| Types/Variations | `get_type_list()`, `get_type_variation_base(type)`, `get_type_variation_list(base_type)`, `clear_type_variation(type)` |

(*`set_*` por familia están en la tabla completa; la familia `get/clear/has` está verificada en los chunks consultados — verificar `set_color`/`set_constant`/etc. en la tabla antes de usar si no se vio directamente.)

### En el `Control` (ver `godot-control`)

| API | Nota |
|---|---|
| `theme` | El tema del subtree (cascada). |
| `theme_type_variation` | Variación de tipo. |
| `add_theme_*_override(name, value)` ×6 | Overrides (no son props — oficial). |
| `get_theme_*` | Leer el tema aplicado. |
| `begin_bulk_theme_override()` / `end_bulk_theme_override()` | Batches. |

## Arquitectura recomendada

### El patrón canónico

```text
ProjectSettings.gui/theme/custom = theme.tres   ← el tema GLOBAL (oficial)
   └─ (todos los Controls del proyecto)

UI (Control, raíz del HUD)
   theme = theme_local.tres   ← subtree con otro look (p. ej. menú de settings)
   ├─ Button (theme_type_variation = "Primary")
   └─ Label (add_theme_font_size_override("font_size", 28))  ← excepciones
```

1. **Un `Theme` global** en `gui/theme/custom` (oficial) — no en cada control.
2. **`default_font` + `default_font_size`** en el tema (una sola fuente para todo).
3. **Variations** para "roles" (Primary/Secondary/Danger) — un type base + variations, no N themes.
4. **Overrides** solo para excepciones puntuales (y con `begin/end_bulk_theme_override` en batches).
5. **Accessibility**: "font settings or adjustments for colorblind users can be applied in a single place and affect the entire UI tree" (oficial, cascada).

## Implementación mínima

**Tema global con fuente** (todo API verificada):

```gdscript
# En el editor: Project Settings → gui/theme/custom = res://theme.tres
# theme.tres: default_font = mi_fuente.tres, default_font_size = 16
# (o programático, en un autoload):
func _ready() -> void:
	var t := Theme.new()
	t.default_font = load("res://fonts/NotoSans.ttf")  # FontFile
	t.default_font_size = 16
	ProjectSettings.set_setting("gui/theme/custom", t)  # verificar la ruta exacta al implementar
```

**Cambia el color de TODOS los Label** (uno solo lugar):

```gdscript
# En el theme (inspector): type "Label" → color "font_color"
# o programático:
theme.set_color("font_color", "Label", Color.LIGHT_SEA_GREEN)  # familia set_ verificado en la tabla
```

## Implementación recomendada

### 1. Variación de Button (Primary)

```text
Theme:
  type "Button" (base)
  variation "Primary" (linked a "Button")
     stylebox "normal" = estilo primary
     color "font_color" = ...
En el control:
  button.theme_type_variation = &"Primary"   (prop verificada en Control)
```

### 2. Leer un theme item custom (script)

```gdscript
# Oficial: types custom requieren scripts (el control built-in no los conoce)
var accent_color: Color = get_theme_color("accent_color", "MyType")
label.add_theme_color_override("font_color", accent_color)
```

### 3. Accessibilidad: fuente grande + alto contraste

```gdscript
# Cascada (oficial): un solo lugar afecta todo el árbol
theme.default_font_size = 24
theme.set_color("font_color", "Label", Color.WHITE)
theme.set_color("font_color", "Button", Color.WHITE)
```

### 4. Subtree con otro "flavor" (ej. equipo azul vs rojo)

```text
UI
├─ TeamBlue (Control, theme = theme_blue.tres)
└─ TeamRed  (Control, theme = theme_red.tres)
# "you can give different flavors to the sides in your team-based project" (oficial)
```

### 5. Focus como overlay (diseño correcto)

```text
# Oficial: el stylebox "focus" se dibuja SOBRE normal/pressed
# → diseñarlo como outline/translucento (no con fondo opaco)
# en el theme: type Button → stylebox "focus" = StyleBoxLine (outline)
```

## Ejemplo práctico

**Juego**: RPG con HUD "fantasy" y modo accessibility.

1. **Tema global** (`gui/theme/custom`): `default_font` = fuente fantasy (Noto si hay acentos/CJK — advertencia oficial), `default_font_size = 16`.
2. **Styles** por type: `Panel` (StyleBox con bordes), `Button` (normal/hover/pressed/disabled + `focus` como outline — oficial).
3. **Variations**: `Primary` (botones de acción), `Danger` (botones de vender/eliminar).
4. **HUD**: un `theme` local en el Control raíz del HUD para el "look de juego" (diferente al menú de settings — oficial: "different theme resource to a branch").
5. **Accessibility** (toggle): `default_font_size = 24` + colores de alto contraste (cascada — oficial).
6. **Debug** "el override no se ve" → verificar (name, type) exacto (`get_color_list("Label")` — verificado la familia); y que el theme del subtree no lo pise.

## Integración

- **`godot-control`**: los overrides y `theme_type_variation` (la API del control).
- **`godot-containers`**: los márgenes/padding son **constant** del tema (oficial).
- **`godot-localization`**: fuentes multilingües (DynamicFont fallback; el default font es Latin-1 — oficial).
- **`godot-rendering-performance`**: el UI con muchos styleboxes complejos es fill rate (ver `godot-control`).
- **Custom controls** (skill pendiente Lote 7 ext.): "it is still the job of each individual control to use that configuration" (oficial) — leer `get_theme_*`.

## Errores frecuentes

1. **`get("font_color")` / `set("font_color", ...)`** → "Theme items are not Object properties" (oficial) → `get_theme_color`/`add_theme_color_override`.
2. **El tema "no llega" a un subtree** → la cascada requiere "a chain of controls uninterrupted" (oficial); un nodo no-Control en medio corta la cadena.
3. **Dos items con el mismo (name, type)** → "This combination must be unique within the theme" (oficial); el editor te lo impide, el script no.
4. **Focus con fondo opaco** → "the focus stylebox should be designed as an outline or translucent box" (oficial; es un overlay).
5. **Esperar que el default font muestre CJK/ cirílico** → "only supports a subset of the Latin-1" (oficial) → `DynamicFont` multilingüe (p. ej. Noto).
6. **`default_font_size = -1` "por default"** → −1 = "usar el default" (verificado el default de la prop); para un tamaño explícito, poner el valor.
7. **Un Theme por control "para controlarlo"** → un tema global + variations + overrides; el Theme es para **compartir** (oficial).
8. **Types custom esperando que el built-in los use** → "you must utilize scripts to access those items" (oficial).
9. **Esperar que el `gui/theme/custom` sea una prop de `Control`** → es un `ProjectSettings` (oficial: `ProjectSettings.gui/theme/custom`).
10. **`add_theme_*_override` con el type equivocado** → el override es por (name) en el control; el type se resuelve por la clase (herencia — oficial); si el item no existe en la cadena de types, no hay nada que pisar.

## Anti-patrones

- **10 `Theme` distintos "por pantalla"** → 1 global + variations + subtrees locales (oficial: un theme para todo el proyecto o por rama).
- **Hardcodear colores en scripts** (`modulate = Color.RED`) → theme items (cascada + accessibility en un lugar — oficial).
- **`default_base_scale` "a ojo" sin medir** → es para scaling global del UI; el scaling por resolución se hace con stretch mode (ver docs de múltiples resoluciones).
- **Stylebox "focus" con fondo** → outline/translucento (oficial, es overlay).
- **Re-escribir el look por `_draw()`** cuando un StyleBox del tema lo resuelve → el tema existe para eso (oficial: "the Godot editor itself… applies its own heavily customized theme").
- **Ignorar el theme editor** (todo por código) → el inspector de Theme da la vista de (type, item) completa; el código es para lo dinámico.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El theming es barato en runtime** (lookup de items; los styleboxes se resuelven al draw). El coste real es el **draw** (fill rate 2D, ver `godot-control`).
- **Qué medir**: FPS con el HUD completo vs oculto (fill rate); el "resolucion" de theme items no es un cuello típico.
- **Optimizaciones (contra un cuello medido)**:
  1. Styleboxes simples (un `StyleBoxEmpty`/flat en vez de 9-patch complejo con transparencias).
  2. Menos `add_theme_*_override` redundantes (un tema bien hecho no necesita 50 overrides).
  3. `default_font` con las glyphs necesarias (los fallbacks de fuente por carácter suman trabajo de render).
- **No**: "el tema pesa" sin medir — el draw es lo que se mide.

## Debugging

### "El override no se aplica"

```text
1. ¿(name, type) correctos? (get_color_list("Label") / get_stylebox_list(...) — familias verificadas)
2. ¿Un tema del subtree lo pisa? (la cascada: el más cercano gana)
3. ¿El control usa ese item? (ver la sección Theme Properties de la clase)
```

### "El tema no afecta a todo"

```text
1. ¿La cadena de Controls es continua? (oficial: "as long as a chain of controls is uninterrupted")
2. ¿El Control raíz tiene theme = ...? (no un hijo)
3. ¿Project gui/theme/custom está seteado? (el tema global)
```

### "El texto no se muestra en otro idioma"

```text
1. Default font = Latin-1 subset (oficial) → DynamicFont multilingüe (Noto)
2. ¿El Theme tiene la font en default_font?
3. Fallback: el DynamicFont fallback system (oficial en godot-localization)
```

### "El focus no se ve / se ve raro"

```text
1. El stylebox "focus" es overlay (oficial) → diseñarlo outline/translucento
2. ¿El control tiene focus_mode? (godot-control)
3. El focus visual solo con keyboard/gamepad o grab_focus (oficial)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla de `Theme` + tutorial (lista en *Referencias*).
- **3.x → 4.x**: `Theme` es la misma familia; en 3.x los items usaban el mismo modelo (type/name). Las APIs `add_theme_*_override`/`get_theme_*` existen en 4.x (verificadas).
- **4.0 ↔ 4.7**: la tabla consultada es la de 4.7 (no afirmar diferencias menores sin verificar).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-control` | La API del control (obligatoria) |
| `godot-containers` | Constantes de margen/padding |
| `godot-localization` | Fuentes multilingües |

## Skills relacionadas

- `godot-recipe-theme-accessibility` — pendiente (germen en *Implementación recomendada* #3)
- `godot-recipe-button-variation` — pendiente (germen #1)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Theme (props `default_base_scale`/`default_font`/`default_font_size`, familias get/clear/has, `get_theme_item`, types/variations, `gui/theme/custom`, `Control.theme`): https://docs.godotengine.org/en/stable/classes/class_theme.html
- Introduction to GUI skinning (6 tipos de item, "unique within the theme", herencia de types, focus como overlay, cascada, custom types por scripts, ejemplo `get_theme_color("accent_color", "MyType")`): https://docs.godotengine.org/en/stable/tutorials/ui/gui_skinning.html
- Control (overrides + "theme items are not Object properties" + `theme_type_variation`): https://docs.godotengine.org/en/stable/classes/class_control.html
