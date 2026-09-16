# godot-input

## ID

`godot-input`

## Categoría

Input · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la class reference oficial 4.7 (`docs.godotengine.org/en/stable/`) y el XML de clases de la rama `4.7-stable` (`Input.xml`, `InputMap.xml`, `InputEventMouseMotion.xml`).

## Confidence

**HIGH** — API verificada directamente en la docs 4.7.
**"No existe (verificado)"**: señales de acciones del singleton `Input` (no existen en 4.7).

## Nivel

**BASE** (sin prerrequisitos).

## Propósito

Proporcionar al agente el conocimiento operativo completo del sistema de input de Godot 4.x: el Input Map (configuración de acciones), el singleton `Input` (polling de estado), eventos por callback (`_unhandled_input`), ratón (modos, `screen_relative`, sensibilidad), gamepads (sticks, ejes, conexión) y la arquitectura que hace el código independiente del dispositivo.

## Cuándo utilizarla

- Configurar mapeos de acciones (teclado, ratón, gamepad, touch) para un juego.
- Leer estado de input: "¿está pulsado?", "¿acabo de pulsar?", "¿qué dirección del stick?".
- Manejar ratón capturado/orbitación/clicks sobre objetos 3D.
- Soporte de gamepad (ejes, botones, detección de conexión).
- Depurar "el input no llega / llega dos veces / tiene lag".

## Cuándo NO utilizarla

- **Input de UI** (botones, drag&drop en la interfaz) → el sistema de Control consume el input antes que `_unhandled_input` (el flujo es `_input` → UI → `_unhandled_input`).
- **Gestos complejos de touch multi-objetivo** (pinch, rotación) → `TouchScreenDelegate`/gestos propios (skill pendiente `godot-input-touch`).
- **Input de red** (replicar input de otros jugadores) → la capa de multiplayer envía *acciones*, no eventos (skill pendiente `godot-recipe-multiplayer-lobby`).

## Conceptos fundamentales

### Tres niveles, tres herramientas

| Nivel | Herramienta | Cuándo |
|---|---|---|
| **Configuración** | Input Map (`Project Settings → Input Map`), runtime: `InputMap` | Definir qué dispositivos disparan cada **acción** (`move_forward`, `jump`…). Se hace una vez; el código nunca ve dispositivos. |
| **Polling (estado)** | `Input.is_action_*`, `Input.get_vector`, `Input.get_axis`, `Input.get_joy_axis` | En `_process`/`_physics_process`: "ahora mismo, ¿qué pasa?". El 95% del input de juego. |
| **Eventos (callback)** | `_input(event)` / `_unhandled_input(event)` | Cosas **instantáneas** o que dependen del evento concreto: click en un punto, tecla exacta, ratón capturado, zoom con rueda. |

**Regla de arquitectura**: el código de gameplay **solo usa nombres de acción** (`"jump"`), nunca teclas/botones. Cambiar de dispositivo (o añadir un mando) no toca el código.

### Deadzone

- Los sticks tienen ruido cerca de 0. El **deadzone del proyecto** (Project Settings → Input Devices → Joypads) filtra el centro.
- `Input.get_vector(..., deadzone = -1)` usa el deadzone del proyecto; `deadzone = 0.2` lo define en la llamada.
- En el Input Map cada **evento** de eje también lleva su propio deadzone por defecto (`add_action(action, deadzone=0.2)`).

### Ratón: dos mundos

- **Modos** (`Input.mouse_mode`): `MOUSE_MODE_VISIBLE(0)` / `HIDDEN(1)` / `CAPTURED(2)` / `CONFINED(3)` / `CONFINED_HIDDEN(4)`.
- **`relative` vs `screen_relative`** (verificado, crucial):
  - `relative` / `velocity`: **escalados por el content scale factor** → la sensibilidad cambia con la resolución/escala.
  - `screen_relative` / `screen_velocity`: **sin escalar** → resolución-independiente. **El tutorial oficial de spring arm recomienda `screen_relative` para apuntar con `MOUSE_MODE_CAPTURED`.**
  - En `MOUSE_MODE_CAPTURED`, `velocity` y `screen_velocity` son (0,0): usar `screen_relative`/`relative`.
  - Puede emitirse `InputEventMouseMotion` **sin movimiento** → comprobar `screen_relative.is_zero_approx()` antes de rotar.
  - Por defecto **máximo 1 evento de motion por frame** (los deltas se acumulan); `Input.use_accumulated_input = false` para recibirlas todas (cuesta más).

### Gamepad: ejes no son eventos

Los sticks son **pollos** (`Input.get_joy_axis(device, JOY_AXIS_...)`), no eventos: se leen en `_process`/`_physics_process`, no en `_unhandled_input`. La **única señal** del singleton `Input` es `joy_connection_changed(device_id, connected)` (verificado 4.7: no existen `action_just_pressed`/`action_released` como señales).

## API relevante

Todo verificado en Godot 4.7.

### Singleton `Input` — polling

| API | Devuelve | Nota |
|---|---|---|
| `is_action_pressed(action)` | `bool` | Estado continuo (pulsado ahora). |
| `is_action_just_pressed(action)` | `bool` | Un solo "frame" (en `_physics_process`, un paso de física). Eventos. |
| `is_action_just_released(action)` | `bool` | Idem, al soltar. |
| `is_action_from_point(pos, action)` | `bool` | Si la acción viene del punto (UI/touch). |
| `get_action_strength(action)` | `float` 0..1 | Fuerza (análogo: sticks/triggers). |
| `get_axis(neg_action, pos_action)` | `float` | `strength(pos) - strength(neg)`. |
| `get_vector(neg_x, pos_x, neg_y, pos_y, deadzone = -1.0)` | `Vector2` | El caballo de batalla: WASD + stick izquierdo → un vector. `deadzone = -1` → el del proyecto. |
| `is_mouse_button_pressed(button)` | `bool` | `MOUSE_BUTTON_LEFT(1)` etc. |
| `get_joy_axis(device, axis)` | `float` -1..1 | Sticks/ejes: `JOY_AXIS_LEFT_X(0)`, `JOY_AXIS_LEFT_Y(1)`, `JOY_AXIS_RIGHT_X(2)`, `JOY_AXIS_RIGHT_Y(3)`, `JOY_AXIS_TRIGGER_LEFT(4)`, `JOY_AXIS_TRIGGER_RIGHT(5)`. |
| `get_joy_button_state(button)` | `bool` | `JOY_BUTTON_A(0)`… (números estándar de gamepad). |
| `is_joy_connection_available(device)` | `bool` | — |
| `get_joypad_connection(device)` | `enum` | `JOYPAD_CONNECTION_NONE/CONNECTED`. |
| `mouse_mode` | `enum` | `MOUSE_MODE_VISIBLE(0)`, `HIDDEN(1)`, `CAPTURED(2)`, `CONFINED(3)`, `CONFINED_HIDDEN(4)`. |
| `use_accumulated_input` | `bool` | `true` (default): 1 motion/frame (acumulado); `false`: todos los deltas. |
| `warp_mouse(x, y)` | — | Teletransportar el cursor (solo con modo visible/confined). |
| `set_custom_mouse_cursor(image, shape, hotspot)` | — | Cursor propio (shapes: `MOUSE_CURSOR_SHAPE_ARROW/POINTING_HAND/...`). |
| `action_press(action)` / `action_release(action)` | — | Simular acciones (tests, cinemáticas). No emite eventos de `_input`. |
| `event_from_string` / `event_to_string` | | Serializar eventos (debug, guardado de perfiles). |
| `flush_buffered_events()` | — | Vaciar la cola de eventos (p. ej. al volver de un menú). |

**Señales del singleton `Input` (verificado 4.7): SOLO `joy_connection_changed(device_id: int, connected: bool)`.** No existen señales por acción.

### `InputMap` (runtime)

| API | Nota |
|---|---|
| `add_action(action: StringName, deadzone: float = 0.2)` | Crear acción en runtime. |
| `action_add_event(action, event)` | Añadir un `InputEventKey`/`Button`/`Joypad`/`Screen` a la acción. |
| `action_erase_event(action, event)` / `action_erase_events(action)` | Quitar (un / todos). |
| `action_get_events(action)` | `Array[InputEvent]`. |
| `action_has_event(action, event)` | — |
| `action_get_deadzone(action)` / `action_set_deadzone(action, d)` | — |
| `event_is_action(event, action, exact_match = false)` | ¿Este evento dispara la acción? (UI: saber qué acción hizo el click). |
| `get_action_description(action)` | String legible ("W, ↑") — para HUD de atajos. |
| `get_actions()` | Todas las acciones. |
| `has_action(action)` | Guard antes de `is_action_*`. |
| `erase_action(action)` | — |
| `load_from_project_settings()` | Recargar los mapeos del proyecto. |
| Señal: `project_settings_loaded` | — |

### Eventos (callback)

| API | Nota |
|---|---|
| `_input(event: InputEvent)` | Antes que la UI. Devolver `get_viewport().set_input_as_handled()` para consumir. |
| `_unhandled_input(event)` | **Tras** la UI: lo estándar para gameplay (ratón/teclado cuando la UI no lo ha consumido). |
| `InputEventKey` | `keycode`, `pressed`, `echo`, `physical_keycode` (layout-independiente), `location`. |
| `InputEventMouseButton` | `button_index` (`MOUSE_BUTTON_LEFT=1`, `RIGHT=2`, `MIDDLE=3`, `WHEEL_UP=4`, `WHEEL_DOWN=5`…), `position`, `pressed`. |
| `InputEventMouseMotion` | `relative` (escala con content scale), `screen_relative` (**sin escalar** — recomendado en captured), `velocity`, `screen_velocity`, `position`. |
| `InputEventJoypadMotion` | `axis` + `axis_value` (si se prefiere por eventos en vez de polling). |
| `InputEventJoypadButton` | `button_index` + `pressed`. |
| `InputEventScreenTouch` / `InputEventScreenDrag` | Touch (position, index, pressed). |

## Arquitectura recomendada

### Input Map (Project Settings → Input Map)

| Acción | Teclado | Gamepad | Lectura en código |
|---|---|---|---|
| `move_left` | A / ← | stick izq. X− | `get_vector` |
| `move_right` | D / → | stick izq. X+ | `get_vector` |
| `move_forward` | W / ↑ | stick izq. Y− | `get_vector` |
| `move_back` | S / ↓ | stick izq. Y+ | `get_vector` |
| `jump` | Espacio | `JOY_BUTTON_A` (o B) | `is_action_just_pressed` |
| `sprint` | Shift izq. | `JOY_BUTTON_X`/LB | `is_action_pressed` |
| `interact` | E | `JOY_BUTTON_B` | `is_action_just_pressed` |
| (cámara) | — | stick der. | `get_joy_axis` en `_process` |
| (ratón) | botones/ruela | — | `_unhandled_input` |

Decisiones:

1. **Una acción = un verbo de juego**, no un control. `jump` (no `space_key`), `interact` (no `e_key`).
2. **Los 4 ejes de movimiento como 4 acciones** (`get_vector` las combina); el stick derecho de cámara NO es una acción de movimiento (se pollea directamente por eje).
3. **Deadzone del proyecto** para sticks (0.5 típico); no hardcodear deadzones en el código.
4. **`physical_keycode`** si el juego depende de la posición de la tecla (layout-independiente) en vez del `keycode` (carácter según layout).

### Patrón de lectura (dónde va cada cosa)

```gdscript
func _physics_process(delta: float) -> void:
	# Movimiento continuo: get_vector (polling)
	var v := Input.get_vector("move_left", "move_right", "move_forward", "move_back")
	# Saltos: just_pressed (evento acotado)
	if Input.is_action_just_pressed("jump"): ...

func _process(delta: float) -> void:
	# Sticks de cámara: polling de ejes (no son eventos)
	var stick := Vector2(Input.get_joy_axis(0, JOY_AXIS_RIGHT_X),
		Input.get_joy_axis(0, JOY_AXIS_RIGHT_Y))

func _unhandled_input(event: InputEvent) -> void:
	# Instantáneos: click, tecla exacta, rueda, motion del ratón capturado
	if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
		var mm := event as InputEventMouseMotion
		if not mm.screen_relative.is_zero_approx():   # puede venir sin movimiento
			_rotate(mm.screen_relative * sensitivity)
```

## Implementación mínima

```gdscript
# input_debug.gd — comprobar que el input llega (4.7, APIs verificadas)
extends Node

func _ready() -> void:
	# ¿El gamepad está conectado? (única señal del singleton)
	Input.joy_connection_changed.connect(func(dev, conn):
		print("Gamepad ", dev, ": ", "conectado" if conn else "desconectado"))

func _process(_delta: float) -> void:
	print("move: ", Input.get_vector("move_left", "move_right", "move_forward", "move_back"),
		" | jump pressed: ", Input.is_action_pressed("jump"),
		" | joy stickR: ", Vector2(Input.get_joy_axis(0, JOY_AXIS_RIGHT_X),
			Input.get_joy_axis(0, JOY_AXIS_RIGHT_Y)))
```

## Implementación recomendada

### 1. Gestor de ratón capturado (patrón reutilizable)

```gdscript
# mouse_capture.gd — toggle de captura con ratón, cursor, y sensibilidad.
extends Node

signal captured_changed(captured: bool)

@export var sensitivity := 0.0035

func _ready() -> void:
	# Acción "toggle_capture" (F1 o similar) en el Input Map.

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventKey:
		var k := event as InputEventKey
		if k.pressed and k.keycode == KEY_ESCAPE:
			set_captured(false)
	elif event is InputEventMouseButton:
		var mb := event as InputEventMouseButton
		if mb.pressed and mb.button_index == MOUSE_BUTTON_RIGHT:
			set_captured(true)

func set_captured(captured: bool) -> void:
	Input.mouse_mode = Input.MOUSE_MODE_CAPTURED if captured else Input.MOUSE_MODE_VISIBLE
	captured_changed.emit(captured)

func get_captured() -> bool:
	return Input.mouse_mode == Input.MOUSE_MODE_CAPTURED
```

### 2. Input re-mapeable en runtime (perfil de controles)

```gdscript
# Remapear "jump" a la tecla K en runtime (API InputMap verificada):
var e := InputEventKey.new()
e.physical_keycode = KEY_K
InputMap.action_erase_events("jump")      # quitar viejos
InputMap.action_add_event("jump", e)
# Guardar/perfil: InputMap.event_to_string(e) / InputMap.event_from_string(s)
# Recargar mapeos del proyecto: InputMap.load_from_project_settings()
```

### 3. HUD de atajos (qué tecla hace qué, sin hardcodear)

```gdscript
func _build_shortcut_labels() -> void:
	# "Salta: Espacio" generado desde el Input Map (multidispositivo, sin hardcode):
	$HUD/JumpLabel.text = "Salta: " + InputMap.get_action_description("jump")
```

### 4. Eventos para UI → acción (¿qué pulsó el click?)

```gdscript
# En un control: saber si el click fue una acción del Input Map:
func _gui_input(event: InputEvent) -> void:
	if event is InputEventMouseButton and (event as InputEventMouseButton).pressed:
		for action in InputMap.get_actions():
			if InputMap.event_is_action(event, action):
				print("La acción ", action, " disparó este click")
```

## Ejemplo práctico

**Juego**: TPS desktop + console.

1. **Input Map** como la tabla de arriba (WASD + flechas + stick izq. en las 4 acciones; `jump` = Espacio + A + B).
2. **Movimiento**: `get_vector` con `deadzone = -1` → teclado (1.0) y stick (0..1 con deadzone del proyecto) dan el mismo vector.
3. **Cámara**: `_unhandled_input` con `InputEventMouseMotion` capturado → `screen_relative * 0.0035` (resolución-independiente, recomendado por el tutorial oficial de spring arm). Rueda = zoom. Escape = soltar (señal `captured_changed` → UI muestra cursor).
4. **Gamepad**: en `_process`, stick derecho polleado; si `length() > 0.15` → orbitar. `joy_connection_changed` → UI "mando conectado, pulsa A".
5. **Remape**: en opciones, el usuario cambia `sprint` a Ctrl → `action_erase_events` + `action_add_event` + `event_to_string` guardado en `user://controls.cfg`.

## Integración

- **Controladores** (`godot-character-controller`): consume acciones (`move_*`, `jump`, `sprint`) vía `get_vector`/`is_action_*`.
- **Cámara** (`godot-camera3d`): `_unhandled_input` (motion/rueda) + `get_joy_axis` (stick).
- **UI**: el Input Map alimenta `get_action_description` (atajos); la UI consume el input **antes** de `_unhandled_input` (flujo estándar).
- **Audio**: las acciones también sirven como trigger de sonidos (el mismo `"jump"` dispara anim, sonido y física — una sola fuente de verdad).

## Errores frecuentes

1. **Conectar `Input.action_just_pressed` / `action_released` como señales** — **no existen en 4.7** (verificado: la única señal es `joy_connection_changed`). "signal not found". → polling `is_action_just_pressed()` o `_unhandled_input`.
2. **`Input.get_vector` para saltos** → `get_vector` da el vector continuo; el salto es `is_action_just_pressed("jump")`. Con `get_vector` el jugador salta en bucle mientras empuja "arriba".
3. **Sensibilidad de cámara con `event.relative`** → escala con el content scale: la sensibilidad cambia al cambiar la resolución/escala del proyecto. → `screen_relative` (recomendado por la docs oficial para aiming capturado).
4. **Leer sticks en `_unhandled_input`** → los ejes no son eventos: el stick "se pega" al último valor recibido. → polling `get_joy_axis` en `_process`/`_physics_process`.
5. **Mover por el `keycode` sin `physical_keycode`** → en layout AZERTY, Q/W/A/S cambian de posición: el jugador con layout francés pierde el control. → `physical_keycode` si importa la posición, o mejor: acciones del Input Map (que ya lo resuelven).
6. **`is_action_just_pressed` en `_process` esperando "un solo frame"** → funciona por *frame de proceso*; en `_physics_process` funciona por *paso de física* (120 Hz de render con física a 60 Hz: dos frames por paso). Coherente si todo el gameplay está en `_physics_process`.
7. **No chequear `is_zero_approx()` en `InputEventMouseMotion`** → puede emitirse sin movimiento; con `relative == (0,0)` el lerp de cámara se "congela" un frame (jitter en captura).
8. **Deadzone inconsistente** (0.2 en el proyecto, 0.5 en el Input Map, 0.3 hardcodeado en el código) → el stick "muere" en el centro y da saltos de 0→1. → un solo sitio: el proyecto + `deadzone = -1` en `get_vector`.
9. **`Input.action_press` para "hacer saltar al jugador en una cinemática" y esperar que dispare `_input`** → no emite eventos de `_input` (lo dice la docs); dispara el estado de la acción (polling). Para cinemáticas, llamar la función de salto directamente.
10. **`warp_mouse` con el ratón capturado** → sin efecto (no hay cursor visible). Usar `CONFINED`/`VISIBLE` si se necesita teletransportar el cursor.
11. **Esperar 1 evento de motion por frame y "frenar" la cámara** → es el comportamiento por defecto (`use_accumulated_input = true`, 1 acumulado/frame): el delta ya contiene todo lo acumulado; no compensar multiplicando. Si se necesitan todos los deltas (tracking fino), `use_accumulated_input = false`.
12. **`mouse_mode = HIDDEN` esperando captura** → `HIDDEN` oculta el cursor pero el ratón sigue libre (sale de la ventana). Captura = `CAPTURED`.

## Anti-patrones

- **Teclas hardcodeadas en el código** (`if event.keycode == KEY_W`) → acciones del Input Map; el remape y el gamepad dejan de ser imposibles.
- **Un `_unhandled_input` de 300 líneas con if/elif por tipo** → separar por sistema (cámara, pausa, interact) con early-return por tipo.
- **`get_tree().root.get_child(0)` en busca del "gestor de input global"** → autoload `InputManager` (servicio) que expone `is_paused`, `set_captured`, perfiles — no un dios que hace polling.
- **Re-mapear en runtime sin guardar** (`action_add_event` sin `event_to_string`) → el remape muere al cerrar. Guardar en `ConfigFile` en `user://`.
- **Doble lectura** (polling + `_unhandled_input` para la misma acción) → la misma acción saltando dos veces (p. ej. `just_pressed` en física Y click en `_unhandled_input`). Un único punto por acción.
- **Consumir input en `_input` cuando la UI debe verlo** (pausa con Escape mientras un diálogo lo necesita) → `_unhandled_input` respeta el flujo UI; `_input` es solo para cosas que deben ir siempre (p. ej. F11 fullscreen).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- El polling de `Input` es **baratísimo** (lecturas de estado en memoria): no es un cuello de botella típico.
- Los costes reales: `use_accumulated_input = false` con ratón de 1000+ Hz (más eventos por frame), y scripts que hacen **trabajo por evento** (cada motion → raycast costoso). Mitigación: el trabajo caro por frame (1×), no por evento.
- Medición estándar: `Performance.get_monitor(Performance.TIME_PROCESS)` (s de process por frame) — si el input crece el tiempo de process, el problema es el trabajo por evento, no el input.

## Debugging

### "El input no llega / el personaje no responde"

```text
1. ¿La acción existe?          → print(InputMap.has_action("jump"))
2. ¿Está mapeada?              → print(InputMap.get_action_events("jump"))
3. ¿Alguien consume antes?     → UI con foco (un Control que come el Escape/Espacio);
   el flujo es _input → UI → _unhandled_input.
4. ¿El nodo tiene process activo?  → set_process(false) / process_mode PAUSABLE y el
   juego pausado → _unhandled_input no corre (ver set_process_mode).
5. ¿just_pressed esperando "dos frames"? → el just es UNO por frame; si hay dos
   lecturas por frame (p. ej. dos nodos), el segundo nunca lo ve.
```

### "El ratón va muy rápido / lento al cambiar de resolución"

```text
1. ¿Usas event.relative?  → screen_relative (sin escalado; recomendado por la docs)
2. ¿Content scale factor ≠ 1?  → es la causa del escalado de relative.
3. ¿Deadzone del stick de cámara? → get_joy_axis no tiene deadzone automático;
   aplicar umbral (0.15) en el código.
```

### "El gamepad no se detecta / va al revés"

```text
1. ¿joy_connection_changed conectado?  → print en _ready (patrón input_debug.gd)
2. ¿Ejes invertidos por layout?        → JOY_AXIS_RIGHT_X/Y: verificar signo con el
   stick en mano; algunos mandos reportan Y invertido.
3. ¿Device 0 vacío pero hay mando 1?   → iterar Input.get_joy_name(d) para d en 0..7.
```

### "Dos saltos con una tecla"

```text
1. ¿Dos nodos leyendo just_pressed de "jump"? (p. ej. controlador + un "quick fix")
2. ¿El Input Map tiene "jump" duplicado con la misma tecla?
3. ¿echo=true procesado? → event.echo en InputEventKey: no procesar repetición
   (tecla mantenida) como pulsación nueva.
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — `Input.xml`, `InputMap.xml`, `InputEventMouseMotion.xml` (4.7-stable) + docs stable.
- **4.0 ↔ 4.7**: sin cambios relevantes en el polling de acciones ni en `mouse_mode` (verificado en las dos ramas).
- **3 → 4**: `Input.action_press` (3.x) → `Input.action_press` (igual en 4.x pero con el cambio de `Input.action_press` a…); renombres menores de constantes de `Input` (p. ej. `Input.MOUSE_MODE_*` en vez de `Input.MOUSE_MODE_VISIBLE` legacy); los detalles de migración están en el guide oficial 3→4.

## Dependencias

| Skill | Relación |
|---|---|
| (ninguna base) | Skill atómica |
| `godot-character-controller` | Consumidor principal (acciones de movimiento) |
| `godot-camera3d` | Consumidor de `_unhandled_input` + `get_joy_axis` |
| `godot-ui` (pendiente) | Flujo `_input`/UI/`_unhandled_input` |
| `godot-recipe-multiplayer-lobby` (pendiente) | Replicar acciones, no eventos |

## Skills relacionadas

- `godot-input-touch` — gestos touch (pendiente)
- `godot-input-remapping` — perfil de controles persistente (pendiente; el germen está en *Implementación recomendada* #2)
- `godot-error-input-not-working` — árbol de diagnóstico dedicado (pendiente)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- Input (singleton, mouse_mode, get_vector, get_joy_axis, action_press, use_accumulated_input, señal joy_connection_changed): https://docs.godotengine.org/en/stable/classes/class_input.html
- InputMap (API runtime completa): https://docs.godotengine.org/en/stable/classes/class_inputmap.html
- InputEventMouseMotion (relative vs screen_relative, aviso de event sin movimiento, 1 event/frame): https://docs.godotengine.org/en/stable/classes/class_inputeventmousemotion.html
- InputEvent (clase base, echo, set_input_as_handled): https://docs.godotengine.org/en/stable/classes/class_inputevent.html
- Project Settings → Input Map (UI): https://docs.godotengine.org/en/stable/tutorials/editor/index.html (sección Input Map)
- Tutorial "Third-person camera with spring arm" (uso oficial de `screen_relative` con captura): https://docs.godotengine.org/en/stable/tutorials/3d/spring_arm.html
