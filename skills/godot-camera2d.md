# godot-camera2d

## ID

`godot-camera2d`

## Categoría

2D · Base (cámara)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `Camera2D` (tabla completa de props/métodos/enums) (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — props (`position_smoothing_*`, `rotation_smoothing_*`, `drag_*`, `limit_*` + `limit_smoothed`, `zoom`, `anchor_mode`, `process_callback`, `ignore_rotation`, `enabled`, `offset`), métodos (`make_current`, `reset_smoothing`, `align`, `force_update_scroll`, `get_screen_center_position`, `get_target_position`, `set/get_limit(Side)`, `set/get_drag_margin(Side)`), enums (`AnchorMode`, `Camera2DProcessCallback`), "una cámara activa por viewport", `global_position` ≠ posición real de pantalla (smoothing/limits): verificados directamente en la docs 4.7.
**MEDIUM** — valores de feel (speed de smoothing, magnitudes de drag margin, zoom ranges).
**Diferencia 2D↔3D (verificado)**: `Camera2D` **sí** tiene smoothing incorporado; `Camera3D` **no** (verificado en Lote 3) — en 3D el smoothing se programa a mano.

## Nivel

**BASE** (depende de `godot-node2d`; la usan `godot-platformer-2d` y todo juego 2D).

## Propósito

Proporcionar el conocimiento operativo de `Camera2D`: seguir a un objetivo, smoothing de posición/rotación, límites del mundo, zoom, drag margins (offset de mira), anchor mode, y la regla de "una cámara activa por viewport". Cubre los patrones típicos: follow con suavizado, límites del level, zoom dinámico (táctico), lookahead, y el diagnóstico de los fallos clásicos (jitter, "no llega a la orilla", "la cámara rota con el nodo").

## Cuándo utilizarla

- Seguir al player (follow + smoothing).
- Limitar la cámara al mundo/level (`limit_*`).
- Zoom dinámico (juegos tácticos, acción).
- Offset de mira (drag margins) en TPS 2D/shooters.
- Transiciones entre cámaras (`make_current`).

## Cuándo NO utilizarla

- **Cámara 3D** → `godot-camera3d` (`Camera3D` + `SpringArm3D`; sin smoothing nativo — verificado).
- **UI/HUD** → `CanvasLayer` (no se mueve con el mundo; skill pendiente).
- **Shake/efectos de pantalla** → se hacen moviendo el canvas (`Viewport.canvas_transform`, vía oficial documentada) o scripts propios; `Camera2D` no tiene "shake" nativo.
- **Cámara para captura/mini-map** → `SubViewport` + `Camera2D` dentro (el `custom_viewport` de `Camera2D` lo permite — verificado la prop).

## Conceptos fundamentales

### Una cámara activa por viewport

- `Camera2D` "forces the screen (current layer) to scroll following this node" (oficial). Se registra en el `Viewport` más cercano (o el global).
- **Solo una cámara puede ser activa por viewport** (oficial). Cambiar de cámara: `make_current()` (o `enabled=false` en la otra).
- `is_current()` para saber quién manda; `custom_viewport` (prop verificada) para apuntar a un `Viewport` específico (mini-maps, split screen).

### La trampa de oro: `global_position` ≠ posición de pantalla

Oficial (4.7): el `global_position` del nodo `Camera2D` **no** representa la posición real de la pantalla, "which may differ due to applied smoothing or limits":

- Posición real de la pantalla: **`get_screen_center_position()`** (verificado).
- Rotación real de la pantalla: **`get_screen_rotation()`** (verificado) — la `global_rotation` del nodo puede diferir por el smoothing de rotación.
- `get_target_position()`: la posición hacia la que la cámara va (verificado).
→ Para cualquier cálculo "¿dónde está mirando la pantalla?" usar `get_screen_center_position()`, no `global_position`.

### Smoothing: nativo en 2D (a diferencia de 3D)

- `position_smoothing_enabled` (default `false`) + `position_smoothing_speed` (default `5.0`).
- `rotation_smoothing_enabled` (default `false`) + `rotation_smoothing_speed` (default `5.0`).
- `reset_smoothing()`: reinicia el estado del smoothing (p. ej. tras un teleporte, para que no "recorra" la distancia).
- `align()`: alinea la cámara (snap) (verificado el método; el caso de uso típico es tras teletransportes).
- Contraste 3D: `Camera3D` **no** tiene smoothing (verificado Lote 3) → en 2D es gratis, en 3D se programa.

### Limits: el mundo tiene bordes

- `limit_enabled` (default `true`) + `limit_left` (-10000000), `limit_right` (10000000), `limit_top` (-10000000), `limit_bottom` (10000000) (defaults verificados: ±10M ≈ "sin límite").
- `limit_smoothed` (default `false`): los límites se aplican con smoothing (evita el "corte" al llegar al borde).
- `get_limit(margin: Side)` / `set_limit(margin, limit)` (verificados) — `Side` es el enum global (LEFT/TOP/RIGHT/BOTTOM).
- Editor: `editor_draw_limits` (default `false`) dibuja los límites en el editor.

### Zoom

- `zoom` (default `Vector2(1,1)`) — escala de la cámara (2 = 2× zoom in).
- No hay "zoom smoothing" nativo: se anima con `Tween`/lerp en el script (patrón, MEDIUM).

### Drag margins (lookahead)

- `drag_horizontal_enabled` / `drag_vertical_enabled` (defaults `false`) + `drag_left_margin`/`drag_right_margin`/`drag_top_margin`/`drag_bottom_margin` (defaults `0.2` — fracción de la pantalla, verificado) + `drag_horizontal_offset`/`drag_vertical_offset` (defaults `0.0`).
- Efecto: cuando el player supera el margen, la cámara se "arrastra" para dejarlo del lado opuesto (lookahead en shooters 2D / TPS).
- `set/get_drag_margin(margin, …)` (verificados).
- `anchor_mode`: `DRAG_CENTER (1)` (default) = "takes into account vertical/horizontal offsets and the screen size"; `FIXED_TOP_LEFT (0)` = "top-left corner is always at the origin" (descriciones oficiales).
- `ignore_rotation` (default `true`): la cámara no rota con la rotación del nodo (para que el drag margin funcione en ejes fijos).

### `process_callback`: cuándo actualiza la cámara

- `process_callback` (default `CAMERA2D_PROCESS_IDLE (1)`): la cámara actualiza en el frame de **process**.
- `CAMERA2D_PROCESS_PHYSICS (0)`: actualiza en el **physics** frame.
- Regla: si la cámara sigue a un body físico, `PHYSICS` elimina el jitter de "un tick" (el follow se alinea al paso de física). MEDIUM: elegir por medir (el jitter visible es el síntoma).

## API relevante

Todo verificado en Godot 4.7.

### Props (tabla completa)

| Prop | Default | Nota |
|---|---|---|
| `enabled` | `true` | Cámara activa/inactiva. |
| `zoom` | `Vector2(1, 1)` | Zoom (2 = 2×). |
| `offset` | `Vector2(0, 0)` | Offset fijo de la cámara. |
| `anchor_mode` | `DRAG_CENTER (1)` | `FIXED_TOP_LEFT (0)`. |
| `ignore_rotation` | `true` | No rota con el nodo. |
| `position_smoothing_enabled` | `false` | Suavizado de posición. |
| `position_smoothing_speed` | `5.0` | Velocidad del suavizado. |
| `rotation_smoothing_enabled` | `false` | Suavizado de rotación. |
| `rotation_smoothing_speed` | `5.0` | — |
| `limit_enabled` | `true` | Límites activos. |
| `limit_left/top/right/bottom` | `-10000000` / `-10000000` / `10000000` / `10000000` | Bordes del mundo. |
| `limit_smoothed` | `false` | Límites con smoothing. |
| `drag_horizontal_enabled` / `drag_vertical_enabled` | `false` / `false` | Lookahead. |
| `drag_left/right/top/bottom_margin` | `0.2` | Fracción de pantalla. |
| `drag_horizontal_offset` / `drag_vertical_offset` | `0.0` | Offset extra. |
| `process_callback` | `CAMERA2D_PROCESS_IDLE (1)` | `PHYSICS (0)` / `IDLE (1)`. |
| `custom_viewport` | — | Viewport destino (mini-map). |
| `editor_draw_screen` / `editor_draw_limits` / `editor_draw_drag_margin` | `true` / `false` / `false` | Solo editor. |

### Métodos (tabla completa)

| Método | Nota |
|---|---|
| `make_current()` | Hacerla la cámara activa del viewport. |
| `is_current()` | ¿Es la activa? |
| `reset_smoothing()` | Resetear el estado del smoothing (post-teleporte). |
| `align()` | Alinear/snap la cámara. |
| `force_update_scroll()` | Forzar update del scroll. |
| `get_screen_center_position()` | **Posición real de la pantalla** (no `global_position`). |
| `get_screen_rotation()` | Rotación real de la pantalla. |
| `get_target_position()` | Hacia dónde va la cámara. |
| `set_limit(margin: Side, limit: int)` / `get_limit(margin)` | Límites por lado. |
| `set_drag_margin(margin: Side, drag_margin: float)` / `get_drag_margin(margin)` | Margen de drag por lado. |

### Enums

| Enum | Valores |
|---|---|
| `AnchorMode` | `FIXED_TOP_LEFT (0)`, `DRAG_CENTER (1)` (default). |
| `Camera2DProcessCallback` | `PHYSICS (0)`, `IDLE (1)` (default). |

## Arquitectura recomendada

### La cámara hija del player (el patrón por defecto)

```text
Player (CharacterBody2D)
└─ Camera2D
   # smoothing on, limits al tamaño del level
```

1. **Hija del player** = follow "gratis" (la cámara sigue el `global_position` del padre). Si el follow debe ser de otro nodo (p. ej. una cámara que sigue un objetivo dinámico), la cámara se mueve a mano + `make_current()`.
2. **`process_callback = PHYSICS`** cuando sigue a un body físico (elimina el jitter de un tick).
3. **Límites = tamaño del level** (no ±10M): `limit_left = -32` (con margen), etc.
4. **`reset_smoothing()` + `align()` tras teletransportes** (checkpoint, muerte): sin eso, la cámara "corre" desde la posición vieja.

## Implementación mínima

**Follow con smoothing + límites del level** (todo API verificada):

```gdscript
# camera.gd — Camera2D hija del player
extends Camera2D

@export var world_w := 3200
@export var world_h := 1800

func _ready() -> void:
	position_smoothing_enabled = true
	position_smoothing_speed = 8.0
	process_callback = Camera2D.CAMERA2D_PROCESS_PHYSICS  # sigue a un body físico
	limit_enabled = true
	limit_left = 0
	limit_top = 0
	limit_right = world_w
	limit_bottom = world_h
	limit_smoothed = true
```

## Implementación recomendada

### 1. Teleporte sin "carrera" de cámara

```gdscript
# Al cambiar de level / respawn:
func teleport_player(to: Vector2) -> void:
	player.global_position = to
	cam.reset_smoothing()      # no suavizar la distancia del teleporte (verificado)
	cam.align()                # snap (verificado)
```

### 2. Zoom dinámico (táctico/acción)

```gdscript
# Mover el zoom con Tween (no hay zoom smoothing nativo — MEDIUM: patrón):
func zoom_to(z: Vector2, t := 0.4) -> void:
	var tw := create_tween()
	tw.tween_property(cam, "zoom", z, t)
# Usos: "modo planificación" zoom (2,2) → "modo acción" (1,1);
# o zoom out con más enemigos en pantalla.
```

### 3. Lookahead (drag margins) en un shooter 2D

```text
Camera2D:
  drag_horizontal_enabled = true
  drag_right_margin = 0.35     # el player se queda a la izquierda cuando corre a la derecha
  drag_left_margin  = 0.35
  anchor_mode = DRAG_CENTER    (default)
  ignore_rotation = true       (default)
```

### 4. Mini-map (cámara en un SubViewport)

```gdscript
# El SubViewport tiene su propia Camera2D (custom_viewport apunta ahí):
mini_cam.custom_viewport = $MiniMapViewport
mini_cam.zoom = Vector2(0.15, 0.15)
mini_cam.position_smoothing_enabled = false   # el mini-map no suaviza
# make_current() NO es necesario: cada viewport tiene su propia activa (oficial:
# "only one camera can be active per viewport")
```

### 5. Sacar la "posición real" para lógica de juego

```gdscript
# ¿Qué parte del mundo está en el centro de la pantalla AHORA (con smoothing)?
var screen_center: Vector2 = cam.get_screen_center_position()   # verificado
# NO: cam.global_position (puede diferir por smoothing/limits — oficial)
# ¿Está el enemigo en pantalla?
var on_screen := get_canvas_transform().affine_inverse() * enemy.global_position
```

## Ejemplo práctico

**Juego**: acción 2D con levels de 4000×2000, checkpoint y modo táctico.

1. **Cámara hija del player**, `position_smoothing_speed = 8`, `process_callback = PHYSICS` (body físico → sin jitter), límites al level con `limit_smoothed = true`.
2. **Checkpoint/respawn**: `reset_smoothing() + align()` (receta #1).
3. **Modo táctico** (botón): `zoom_to(Vector2(1.6, 1.6))` (receta #2); volver al combate: `zoom_to(Vector2(1,1))`.
4. **Lookahead**: drag margins horizontales 0.35 en los sections de shooter (receta #3).
5. **Mini-map** en la esquina: `SubViewport` + `Camera2D` con `zoom (0.1,0.1)` (receta #4).
6. **Debug** "la cámara no llega a la orilla": los `limit_*` en coordenadas **world** (no pantalla) — imprimir `get_screen_center_position()` vs los límites.

## Integración

- **`godot-node2d`**: la cámara es un `Node2D` (su transform se compone con los padres — cuidado con padres escalados).
- **`godot-platformer-2d`**: el player que sigue (body físico → `process_callback = PHYSICS`).
- **`godot-camera3d`**: la contraparte 3D (sin smoothing nativo — verificado; `SpringArm3D`).
- **`godot-rendering-performance`**: la cámara no es un cuello de render en 2D; el coste está en lo que muestra (fill rate 2D).
- **`godot-area2d` (pendiente)**: "en pantalla" se resuelve con `VisibleOnScreenNotifier2D` (existe en 4.7, verificado en la lista de `Node2D`) o `get_screen_center_position()` + zoom.

## Errores frecuentes

1. **Usar `global_position` de la cámara como "centro de la pantalla"** → puede diferir por smoothing/limits (oficial) → `get_screen_center_position()`.
2. **Jitter de cámara siguiendo a un body** → `process_callback = CAMERA2D_PROCESS_IDLE` (default) sigue en el frame de process, el body se mueve en physics → `PHYSICS (0)`.
3. **La cámara "corre" tras un teleporte** → `reset_smoothing()` (y `align()`) — el smoothing suaviza la distancia del salto.
4. **`ignore_rotation = false` con drag margins** → el drag se aplica en ejes rotados (el lookahead se rompe); el default `true` existe para esto.
5. **Límites en coordenadas de pantalla** → son **world** (imprimir `get_screen_center_position()` para calibrar).
6. **Dos `make_current()` compitiendo** (cámara del level + cámara del player) → una sola activa por viewport (oficial); desactivar la otra (`enabled=false`).
7. **Zoom esperando smoothing nativo** → no existe (MEDIUM: animar con `Tween`/lerp).
8. **`limit_smoothed = false` en un level grande** → "corte" al llegar al borde; activarlo para bordes suaves.
9. **Cámara en un `CanvasLayer`** → el `CanvasLayer` no se mueve con el mundo (HUD); la cámara debe estar en el mundo.
10. **Esperar "shake" nativo** → no hay prop de shake (verificado: la tabla no lo tiene); mover el canvas o el `offset` con un ruido (patrón custom).

## Anti-patrones

- **Cámara root del level (no hija del player)** cuando el follow es simple → hija del player = follow gratis; la cámara a mano es para objetivos dinámicos/switching.
- **`global_position` en lógica de juego** (spawns, "en pantalla") → `get_screen_center_position()`/conversión de canvas.
- **Smoothing speed muy bajo "para que se sienta suave"** (0.5) → la cámara arrastra; iterar el speed (5 default) midiendo el feel.
- **Re-hacer el smoothing a mano** cuando `position_smoothing_enabled` ya lo hace → la API nativa existe (2D); en 3D sí hay que programarlo.
- **`process_callback = PHYSICS` "por si acaso"** en una cámara que sigue a un nodo no físico → `IDLE` (default) es el correcto; cambiarlo cuando hay jitter de body.
- **Zoom gigante (4,4) en un level abierto** → fill rate 2D se dispara (lo que se muestra se renderiza); medir.

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **La cámara no es un cuello de CPU** (un nodo barato). El coste real:
  - **Lo que muestra** (fill rate 2D — ver `godot-rendering-performance`; el truco de la ventana pequeña detecta fill-rate limit).
  - **Smoothing** es un lerp por frame: despreciable.
  - **`get_screen_center_position()`** por frame en bucles grandes: barata, pero no ponerla en código por píxel.
- **Qué medir**: FPS con/without la cámara (zoom alto en un level denso); si el cuello es "lo que muestra", el remedio es render (culling, `visible` por distancia), no la cámara.
- **No**: "la cámara pesa" sin medir — el fill rate 2D es lo que se mide.

## Debugging

### "La cámara tiembla / hace jitter"

```text
1. ¿Sigue a un body físico con process_callback = IDLE? → PHYSICS (error #2)
2. ¿Smoothing speed muy bajo + objetivo que se mueve en physics? (mismatch de ticks)
3. ¿El padre de la cámara se mueve en _process y el objetivo en _physics_process?
   (el follow debe estar en el mismo "ritmo" que el objetivo)
```

### "No llega a la orilla del level"

```text
1. ¿limit_* en coords world? (imprimir get_screen_center_position() en el borde)
2. ¿La cámara es hija de un nodo con transform? (los límites son en el canvas)
3. ¿zoom > 1? (con zoom, la "pantalla" abarca menos mundo; el límite se
   comporta según la zoom — calibrar imprimiendo)
```

### "El lookahead no funciona / se aplica al revés"

```text
1. drag_*_enabled (default false) — ¿lo activaste?
2. anchor_mode (DRAG_CENTER default) + ignore_rotation (true default)
3. Los drag margins son fracciones de la pantalla (0.2 default) — imprimir
   get_drag_margin(Side.LEFT) para verificar
```

### "Tras el respawn la cámara viaja en línea recta"

```text
1. reset_smoothing() (el smoothing suaviza la distancia del teleporte)
2. align() para el snap
3. ¿La cámara es la activa? (is_current())
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla completa de `Camera2D` (lista en *Referencias*).
- **3.x → 4.x (estable)**: la API consultada (smoothing, drag, limits, zoom, `make_current`, `process_callback`) es la misma familia que 3.x; en 4.7 los defaults verificados son los de la tabla.
- **2D ↔ 3D (verificado)**: smoothing nativo en `Camera2D` **sí**; en `Camera3D` **no** (verificado en Lote 3) — no portar código de smoothing 3D a 2D (ya es nativo).
- **4.0 ↔ 4.7**: la tabla consultada es estable para el uso cubierto.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-node2d` | Base (transform, canvas) |
| `godot-platformer-2d` | El objetivo que sigue (body físico) |
| `godot-camera3d` | La contraparte 3D (sin smoothing nativo) |
| `godot-rendering-performance` | Lo que la cámara muestra (fill rate 2D) |
| `godot-control` (pendiente) | El HUD que NO se mueve con la cámara |

## Skills relacionadas

- `godot-recipe-camera-lookahead-2d` — pendiente (germen #3)
- `godot-recipe-dynamic-zoom-2d` — pendiente (germen #2)
- `godot-error-camera-jitter-2d` — pendiente (germen en *Debugging*)

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- Camera2D (props, métodos, enums, una cámara por viewport, `global_position` ≠ pantalla, `get_screen_center_position`, smoothing, limits, drag, `process_callback`): https://docs.godotengine.org/en/stable/classes/class_camera2d.html
- Node2D (la cámara hereda de Node2D; `VisibleOnScreenNotifier2D` en Inherited By): https://docs.godotengine.org/en/stable/classes/class_node2d.html
