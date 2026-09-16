# godot-localization

## ID

`godot-localization`

## Categoría

UI · Base (i18n)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Translation` (tabla completa) y el tutorial "Internationalizing games" (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — `Translation` (props `locale "en"`, `plural_rules_override ""`; métodos `add_message(src, xlated, context=&"")`, `add_plural_message(src, PackedStringArray, context)`, `get_message`, `get_plural_message(src, src_plural, n, context)`, `get_message_count/list`, `get_translated_message_list`, `erase_message`, virtuales `_get_message`/`_get_plural_message`; "unlike gettext, an empty context means not using any context" — oficial), flujo completo del tutorial (key→text auto en Button/Label, `tr()`, `tr_n()`, placeholders `%s` vs `.format()` con **named recommended**, contexts, `TranslationServer`, `OS.get_locale_language()` + Fallback, fuente Latin-1 + DynamicFont fallback, resource remaps, pseudolocalization, **RTL mirroring** (lista oficial), break iterator data): verificados directamente.
**MEDIUM** — el nombre exacto del setting de pseudolocalization en Project Settings (el tutorial dice "in the advanced Project Settings"; la ruta exacta no se cita sin verificar).
**No duplica**: el `layout_direction`/`auto_translate` como props de Control están anclados en `godot-control`; aquí es el dominio i18n.

## Nivel

**INTERMEDIATE** (depende de `godot-control`; lo usan todos los textos de la UI).

## Propósito

Proporcionar el conocimiento operativo de la **localización** de Godot 4.7: el recurso `Translation` (mensajes, contexts, plurales), `tr()`/`tr_n()`, placeholders (posicionales vs **named**), la auto-traducción de controles (y cómo desactivarla), `TranslationServer` (cambiar idioma en runtime), locale vs language, fuentes (Latin-1 → DynamicFont), resource remaps por locale, **RTL** (mirroring automático y sus excepciones), pseudolocalization y break iterator data en exports.

## Cuándo utilizarla

- Traducir la UI (Button/Label/tooltip).
- Plurales correctos en varios idiomas (`tr_n`).
- Strings con placeholders ("%s picked up the %s").
- Cambiar de idioma en runtime (menú de language).
- Soporte RTL (árabe/hebreo) y mirroring de UI.
- "El texto no se traduce / el font no muestra los caracteres / el plural está mal".

## Cuándo NO utilizarla

- **UI sin idiomas adicionales** → no hace falta nada (el texto se muestra tal cual).
- **El layout de la UI** → `godot-control`/`godot-containers` (la i18n hace que re-flquee; el layout no es i18n).
- **El estilo** → `godot-theme` (las fuentes se configuran ahí; la i18n elige *cuál* fuente).
- **Audio/voice** → Lote 11 pendiente (los remaps de resource cubren voces, verificado el mecanismo).

## Conceptos fundamentales

### `Translation` (recurso; tabla verificada)

- "A language translation that maps a collection of strings to their individual translations" + "convenience methods for pluralization" (oficial).
- Un **message** = context + string sin traducir. "Unlike gettext, using an **empty context string** in Godot means not using any context." (oficial)
- Props: `locale` (default `"en"`), `plural_rules_override` (default `""`).
- Hija: `OptimizedTranslation` (verificada en Inherited By).
- APIs (verificadas): `add_message`, `add_plural_message`, `erase_message`, `get_message`, `get_plural_message`, `get_message_count`, `get_message_list`, `get_translated_message_list`, virtuales `_get_message`/`_get_plural_message` (para subclases custom).

### Keys vs texto plano (la decisión de pipeline)

- **Keys** (`LIKE_THIS`): estables, el traductor ve el contexto; el control traduce automáticamente si su texto coincide con una key (oficial).
- **Texto plano** (inglés como source): "you may run into ambiguities" → usar **context** (oficial): `tr("Close", "Actions")` vs `tr("Close", "Distance")` (ejemplo oficial).
- La auto-traducción de controles: "Some controls, such as Button and Label, will automatically fetch a translation if their text matches a translation key." (oficial)
- **Desactivar** la auto-traducción por nodo: "set the Auto Translate > Mode to Disabled in the inspector" (oficial) — p. ej. un Label con el nombre del jugador (prop `auto_translate` en `Control`, verificado en `godot-control`).

### `tr()` y `tr_n()`

- `tr("LEVEL_5_NAME")` — "just look up the text in the translations and convert it if found" (oficial).
- **Plurales** (oficial): "Pluralization is meant to be used with **positive (or zero) integer numbers only**." (negativos/floats: singular/plural no aplican claramente).
  ```gdscript
  var num_apples = 5
  label.text = tr_n("There is %d apple", "There are %d apples", num_apples) % num_apples
  # con context:
  label.text = tr_n("%d job", "%d jobs", num_jobs, "Task Manager") % num_jobs
  ```
  (Los rules los maneja el locale destino automáticamente — oficial.)

### Placeholders

- Posicionales (`%s`): "The placeholder's locations can be changed, but not their order. This will probably not suffice for some target languages." (oficial)
- **Named con `String.format()`** (recomendado — oficial): "allows translators to choose the order in which placeholders appear" + "gives more context for translators":
  ```gdscript
  message.text = tr("{character} picked up the {weapon}").format({character = "Ogre", weapon = "Sword"})
  ```

### Idioma por defecto y runtime

- Oficial: "It is recommended to default to the user's preferred language which can be obtained via `OS.get_locale_language()`. If your game is not available in that language, it will fall back to the **Fallback** in Project > Project Settings > General > Internationalization > Locale, or to `en` if empty."
- **Siempre** dejar cambiar el idioma en game (oficial: "letting players change the language in game is recommended").
- `TranslationServer` (oficial): "Translations can be added or removed during runtime; the current language can also be changed at runtime." (ejemplo oficial: `TranslationServer.set_locale(language)`).
- **Translations en el proyecto**: Project > Project Settings > **Localization > Translations** (dialog para add/remove project-wide — oficial).

### Locale vs language

- Oficial: "A locale is commonly a combination of a language with a region or country… e.g. `en`, `en_GB`, `en_US`." (spelling, words, units/currency difieren).
- "Indie games generally only need to care about language" (oficial).

### Fuentes (la trampa clásica)

- Oficial: "The default project font only supports a subset of the **Latin-1** character set, which cannot be used to display languages like **Russian or Chinese**."
- Recurso: Noto Fonts (oficial); "load the TTF file into a DynamicFont resource… associate a new Theme resource to your root Control node and define the DynamicFont as the Default Font in the theme." (oficial → ver `godot-theme`)
- **Remaps no soportan DynamicFont**: "The resource remapping system isn't supported for DynamicFonts. To use different fonts depending on the language's script, use the **DynamicFont fallback system** instead… works regardless of the current language, making it ideal for things like multiplayer chat." (oficial)

### Resource remaps (assets por locale)

- "instruct Godot to use alternate versions of assets (resources) depending on the current language… localized images such as in-game billboards or localized voices" (oficial) → la pestaña **Remaps** del resource (oficial).

### RTL: mirroring automático (lista oficial)

- Oficial: "Support for bidirectional writing systems and UI mirroring is transparent, you don't usually need to change anything." Para RTL, Godot hace automáticamente:
  - "Mirrors left/right anchors and margins."
  - "Swaps left and right text alignment."
  - "Mirrors horizontal order of the child controls in the containers, and items in Tree/ItemList controls."
  - "Uses mirrored order of the internal control elements (e.g., OptionButton dropdown button, CheckBox/CheckButton alignment, List column order, TreeItem icons…)."
  - "Coordinate system is **not** mirrored."
  - "Non-UI nodes (sprites, etc.) are **not** affected."
- Overrides (props de `Control`, verificadas en `godot-control`): `text_direction` ("auto" = Unicode Bidi Algorithm), `language` (override del locale del proyecto), `layout_direction` (override del mirroring), `structured_text_bidi_override` + `_structured_text_parser`.

### Pseudolocalization (QA)

- Oficial: "enable **pseudolocalization** in the advanced Project Settings. This will replace all your localizable strings with longer versions… e.g. `Hello world, this is %s!` becomes `[Ĥéłłô ŵôŕłd́, ŧh̀íš íš %s!]`."
- Beneficios (oficial): encontrar strings no-localizables, detectar UI que no aguanta strings largos, (parcial) chequeo de font.
- "Placeholders are kept as-is" (oficial).

### Break iterator data (exports)

- Oficial: "Some languages are written without spaces… Godot includes **ICU** rule and dictionary-based break iterator data, but this data is **not included in exported projects by default**." → activarlo en Project Settings (General) para esos idiomas.

## API relevante

Todo verificado en Godot 4.7.

### `Translation` (< Resource)

| Elemento | Detalle |
|---|---|
| `locale` | `"en"`. |
| `plural_rules_override` | `""` (override de las rules del locale). |
| `add_message(src_message, xlated_message, context = &"")` | Agregar. |
| `add_plural_message(src_message, xlated_messages: PackedStringArray, context = &"")` | Agregar plural. |
| `erase_message(src_message, context = &"")` | Quitar. |
| `get_message(src_message, context = &"")` | Consultar. |
| `get_plural_message(src_message, src_plural_message, n, context = &"")` | Consultar plural (elige por `n`). |
| `get_message_count()` / `get_message_list()` / `get_translated_message_list()` | Metadatos. |
| `_get_message(...)` / `_get_plural_message(...)` | Virtuales (subclases). |

### `TranslationServer` (singleton; oficial)

- `set_locale(language)` (ejemplo oficial), add/remove de translations en runtime (oficial).
- (La tabla completa del singleton no se citó en esta revisión — verificar `class_translationserver.html` al usar más allá de `set_locale`.)

### En `Control` (ver `godot-control`)

| Prop | Nota |
|---|---|
| `auto_translate` | Auto-traducir el texto del control (default on). |
| `translation_context` | Contexto. |
| `layout_direction` | Override de LTR/RTL. |
| `localize_numeral_system` | `true` (default, verificado). |
| `tooltip_text` + `tooltip_auto_translate_mode` | Tooltips traducidos. |

## Arquitectura recomendada

### Pipeline canónico

```text
1. Strings del juego → keys (LIKE_THIS) o texto plano con context
2. Importar translations (CSV/PO — tutorial "Importing translations")
3. Project Settings → Localization → Translations (add project-wide)
4. Project Settings → General → Internationalization → Locale → Fallback = "en"
5. Font multilingüe (DynamicFont) en el Theme global (godot-theme)
6. Runtime: TranslationServer.set_locale(user_choice)
```

1. **Keys para UI estable; texto plano + context** para prose corta (oficial los dos caminos).
2. **Named placeholders** (`.format`) por defecto (oficial: orden libre para el traductor).
3. **Un `DynamicFont`** con coverage amplia en el Theme (no por control — cascada).
4. **Pseudolocalization ON en QA** (probar el UI con strings largos — oficial).

## Implementación mínima

**Menu de language + string traducido** (todo API verificada):

```gdscript
# settings.gd
func set_language(lang: String) -> void:
	TranslationServer.set_locale(lang)   # oficial (ejemplo del tutorial)
	# El UI se actualiza: los Controls con auto_translate re-traducen su texto

# ui.gd
func _ready() -> void:
	# Opción A: texto = key (auto-translate del Label)
	title.text = "MAIN_SCREEN_GREETING1"
	# Opción B: tr() explícito
	status.text = tr("GAME_STATUS_%d" % status_index)   # ej. oficial
```

## Implementación recomendada

### 1. Plurales (el correcto para todo idioma)

```gdscript
# NO: "x1" vs "x2+" hardcodeado (oficial: no válido en todos los idiomas)
label.text = tr_n("There is %d apple", "There are %d apples", n) % n
# con context:
label.text = tr_n("%d job", "%d jobs", n, "Task Manager") % n
# solo positivos/zero (oficial)
```

### 2. Placeholders named (orden libre)

```gdscript
# Oficial (recomendado):
log.text = tr("{character} picked up the {weapon}").format({character = "Ogre", weapon = "Sword"})
# (el posicionale %s no deja reordenar en todos los idiomas — oficial)
```

### 3. Auto-translate off (nombre de jugador)

```text
Label (nombre del jugador):
  Auto Translate > Mode = Disabled   (oficial: inspector)
# o por código:
label.auto_translate = false         (prop verificada en Control)
```

### 4. Resource remaps (billboard/voz por locale)

```text
En el resource (p. ej. la textura del billboard):
  pestaña Remaps → alternativa por locale (oficial)
# NO para DynamicFont → usar el DynamicFont fallback system (oficial)
```

### 5. RTL (árabe/hebreo)

```text
1. Nada a configurar: el mirroring es automático (oficial, lista completa)
2. Si un control debe forzar LTR: layout_direction = LTR (prop verificada)
3. text_direction = "auto" (Unicode Bidi) para mixed text (oficial)
4. NO espejar la coordinate system ni los sprites (oficial: no se hacen)
```

### 6. QA con pseudolocalization

```text
Project Settings (advanced) → pseudolocalization ON (oficial)
→ revisar: UI que no aguanta strings largos, strings no-localizables,
  font coverage (parcial — oficial: no es test efectivo para CJK/RTL)
```

## Ejemplo práctico

**Juego**: aventura con ES/EN/PT-BR/JA/AR.

1. **Strings**: keys en la UI (`MAIN_MENU_PLAY`); prose con contexto (`tr("Close", "Actions")`).
2. **Import**: CSV por locale (tutorial "Importing translations" — oficial).
3. **Font**: `DynamicFont` (Noto) en el Theme global; fallback system para chat multilingüe (oficial).
4. **Plurales**: `tr_n` en contadores (items, oro, jobs).
5. **Placeholders**: `.format` named en todo el log ("{a} atacó a {b}").
6. **AR**: RTL automático (mirroring de anchors/containers — oficial); `layout_direction` LTR en el minimapa si hiciera falta.
7. **QA**: pseudolocalization ON un ciclo de QA completo (oficial).
8. **Export**: activar break iterator data (ICU) por el japonés (oficial: no viene por default en exports).

## Integración

- **`godot-control`**: las props (`auto_translate`, `translation_context`, `layout_direction`, `localize_numeral_system`).
- **`godot-theme`**: la fuente multilingüe (DynamicFont) vive en el Theme (oficial).
- **`godot-containers`**: el reflow con strings largos (el layout debe aguantar — oficial: "make sure to read the tutorial on Size and anchors").
- **`godot-input`**: el cambio de language es un input (menú) — Lote 3.
- **Audio (Lote 11 pendiente)**: voces por locale vía resource remaps (mecanismo verificado).

## Errores frecuentes

1. **`%s` posicionales "porque es más corto"** → "will probably not suffice for some target languages" (oficial); usar `.format` named.
2. **Plurales con `if n > 1`** → "not valid in all languages" (oficial); `tr_n` con las rules del locale.
3. **Esperar que el default font muestre CJK/cirílico** → Latin-1 subset (oficial) → DynamicFont (Noto).
4. **`auto_translate` en un Label de nombre de jugador** → "you most likely don't want the player's name to be translated" (oficial) → Auto Translate Mode = Disabled.
5. **Remap de DynamicFont** → "isn't supported" (oficial) → DynamicFont fallback system.
6. **Esperar que el RTL espeje la coordinate system** → "Coordinate system is not mirrored" (oficial); ni los sprites.
7. **`tr("Close")` sin context para "cerca/cerrar"** → ambigüedad (oficial) → `tr("Close", "Actions")` / `tr("Close", "Distance")`.
8. **Cargar la translation en código sin registrarla** → debe estar en Project Settings → Localization → Translations (oficial) o en el `TranslationServer` en runtime.
9. **Fallback vacío esperando "auto"** → "it will fall back to the Fallback… or to `en` if empty" (oficial); setear Fallback explícito.
10. **Exportar al japonés sin break iterator data** → "not included in exported projects by default" (oficial); activar en Project Settings.

## Anti-patrones

- **Hardcodear strings en scripts sin `tr()`** → "no-localizable" (los detecta la pseudolocalization — oficial); todo texto visible pasa por `tr()` o auto-translate.
- **Interpolación "a mano" de plurales** ("1 item" / "2 items") → `tr_n` (oficial: rules por locale).
- **`String.format` con posicionales `{} {}`** → named (oficial: orden libre + contexto).
- **Un `Translation` por string** → un `Translation` por locale (el import lo genera).
- **`translation_context` vacío "por costumbre"** → el contexto resuelve ambigüedades (oficial); el vacío es "sin contexto" (oficial).
- **Forzar LTR en toda la UI para "simplificar" el RTL** → el mirroring es automático y transparente (oficial); los overrides son por control, no globales.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El lookup de `tr()` es un dict por (context, key)**: barato; el `OptimizedTranslation` (verificada como hija) existe para optimizar (no afirmar detalles sin verificar la clase).
- **El coste real del i18n**: el **render de texto** (glyphs, shaping, Bidi) — fill rate 2D + CPU de text shaping (ver `godot-control`/`godot-rendering-performance`).
- **Qué medir**: FPS con la UI de texto completo vs oculto; si el cuello es text shaping, es del font/layout, no del lookup.
- **Optimizaciones (contra un cuello medido)**:
  1. Menos `RichTextLabel` (el shaping + parseo es más caro que `Label`).
  2. Fonts con coverage justa (menos glyphs = menos lookup de fallback).
  3. `get_message_count` para detectar translations hinchadas (metadatos verificados).
- **No**: "tr() es caro" sin medir — el lookup es dict; el shaping es lo que se mide.

## Debugging

### "El texto no se traduce"

```text
1. ¿La translation está en Project Settings → Localization → Translations? (oficial)
2. ¿El control tiene auto_translate? (el nombre del jugador → Disabled, oficial)
3. ¿La key/context coinciden? (imprimir get_message(key) — verificado)
4. ¿El locale actual? (TranslationServer; imprimir el locale)
```

### "El font no muestra los caracteres (□□□)"

```text
1. Default font = Latin-1 subset (oficial) → DynamicFont multilingüe (Noto)
2. ¿Está en el Theme (default_font)? (cascada — godot-theme)
3. Pseudolocalization para chequear coverage (parcial — oficial)
```

### "El plural está mal en algún idioma"

```text
1. ¿Usás tr_n? (el if n>1 no es válido en todos — oficial)
2. ¿n es entero positivo/zero? (oficial: floats/negativos fuera)
3. plural_rules_override (prop verificada) si el locale tiene rules especiales
```

### "La UI en RTL se ve mal / no se espeja"

```text
1. ¿El locale es RTL? (ar/he) → el mirroring es automático (oficial)
2. ¿Alguno control tiene layout_direction forzado? (override)
3. Coordinate system y sprites NO se espejan (oficial) — no "fixearlo"
```

### "El string largo rompre el layout"

```text
1. Pseudolocalization ON (oficial) para detectar antes
2. Layout: size flags / wrap de Label / containers (godot-containers)
3. "the UI can accommodate longer-than-usual strings" (oficial)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — `Translation` + tutorial (lista en *Referencias*).
- **3.x → 4.x**: `Translation`/`TranslationServer`/`tr()`/`tr_n()` son de la misma familia (3.x ya tenía i18n); los detalles de props se verifican contra la tabla 4.7 (aquí).
- **4.0 ↔ 4.7**: la tabla consultada es la de 4.7 (no afirmar diferencias menores sin verificar).

## Dependencias

| Skill | Relación |
|---|---|
| `godot-control` | Las props de i18n (obligatoria) |
| `godot-theme` | La fuente multilingüe (DynamicFont) |
| `godot-containers` | El reflow con strings largos |

## Skills relacionadas

- `godot-recipe-rtl-arabic-ui` — pendiente (germen en *Implementación recomendada* #5)
- `godot-error-font-glyphs-missing` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Translation (props, métodos, context vacío, pluralization, `OptimizedTranslation`): https://docs.godotengine.org/en/stable/classes/class_translation.html
- Internationalizing games (keys/auto-translate, `tr`/`tr_n`, placeholders named, contexts, `TranslationServer`, `OS.get_locale_language()`, Fallback, fuente Latin-1, DynamicFont fallback, remaps, pseudolocalization, RTL mirroring (lista), break iterator data): https://docs.godotengine.org/en/stable/tutorials/i18n/internationalizing_games.html
- Control (`auto_translate`, `translation_context`, `layout_direction`, `localize_numeral_system`): https://docs.godotengine.org/en/stable/classes/class_control.html
