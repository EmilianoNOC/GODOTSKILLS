# godot-physics-materials

## ID

`godot-physics-materials`

## Categoría

Física · Base (materiales)

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: página de clase `PhysicsMaterial` (tabla completa: solo 4 propiedades), páginas de `RigidBody3D`/`StaticBody3D` (propiedad `physics_material_override`) y `CollisionShape3D` (sin propiedad de material), y tutorial `physics_introduction.html` (docs.godotengine.org/en/stable/).

## Confidence

**HIGH** — clase `PhysicsMaterial` (props `friction`, `bounce`, `rough`, `absorbent` + defaults), dónde se aplica (`physics_material_override` en `RigidBody3D`/`StaticBody3D`), reglas de combinación (`rough`: mínimo / el rough / máximo; `absorbent`: resta rebote), la nota oficial de que `bounce = 1.0` aún pierde energía con damping (y la receta de body perfectamente elástico), y que NO hay material por shape en 4.7: verificados directamente en la docs 4.7.
**MEDIUM** — valores numéricos "sensatos" (0.2 hielo, 0.9 trampolín…) — no son reglas oficiales; son puntos de partida para iterar.
**"No existe en la tabla oficial 4.7 (verificado)"**: la clase `PhysicsMaterial3D` (la página `class_physicsmaterial3d.html` da 404 en 4.7; la clase es `PhysicsMaterial`), `friction_combine_mode`/`bounce_combine_mode` (no hay enums de combinación: la combinación la rigen `rough`/`absorbent`), `CollisionShape3D.physics_material` (la tabla 4.7 solo lista `shape`, `disabled`, `debug_color`, `debug_fill`).

## Nivel

**BASE** (depende de `godot-physics` para el contexto de bodies y damping).

## Propósito

Proporcionar el conocimiento operativo de los materiales de física en Godot 4.7: la clase `PhysicsMaterial` (fricción, rebote, rough, absorbente), cómo se aplica a los bodies (`physics_material_override` en `StaticBody3D` y `RigidBody3D`), cómo se combinan dos materiales al chocar (las reglas de `rough`/`absorbent`), y las recetas típicas: hielo, trampolín, suelo alto-fricción, body elástico perfecto. Es la skill que cierra la última base pendiente de la familia física.

## Cuándo utilizarla

- Hacer que una superficie sea resbaladiza (hielo) o "pegajosa" (fricción alta).
- Rebote: pelota, trampolín, "bounciness" de objetos.
- Entender por qué "dos objetos con fricción 1 no se frenan igual" (la combinación).
- Receta de cuerpo perfectamente elástico (sin pérdida de energía).

## Cuándo NO utilizarla

- **Fricción de un personaje en slide** → `CharacterBody3D` no usa `PhysicsMaterial` para su slide (el slide lo programa tú; ver `godot-characterbody3d`). El material afecta a `Static`/`Rigid`.
- **Respuesta de colisión custom** (rebote con ángulo, fricción que depende del juego) → código en `_integrate_forces`/señales de contacto (`godot-physics`), no material.
- **Materiales de RENDER** (`StandardMaterial3D`, roughness metálica PBR) → son otra cosa (render, no física; ver `godot-shader-spatial`/`godot-standardmaterial3d` pendiente).
- **2D** → `PhysicsMaterial2D` (skill pendiente; la clase base es la misma `PhysicsMaterial`).

## Conceptos fundamentales

### Una sola clase: `PhysicsMaterial`

En 4.7 la clase es **`PhysicsMaterial`** (hereda `Resource`; una sola para 2D y 3D — la página de `PhysicsMaterial3D` no existe). Solo **4 propiedades** (tabla completa verificada):

| Propiedad | Default | Rango / significado |
|---|---|---|
| `friction` | `1.0` | `0.0` = sin fricción … `1.0` = fricción máxima (la docs: "from 0 (frictionless) to 1 (maximum friction)"). |
| `bounce` | `0.0` | `0.0` = sin rebote … `1.0` = rebote total (la docs: "from 0 (no bounce) to 1 (full bounciness)"). |
| `rough` | `false` | Flag que cambia la combinación de fricción (ver abajo). |
| `absorbent` | `false` | Flag que resta el rebote del otro (ver abajo). |

**Sin enums de combinación** (`friction_combine_mode`/`bounce_combine_mode` no existen en 4.7): el comportamiento de "cómo se mezclan dos materiales al chocar" lo definen exactamente `rough` y `absorbent`.

### Dónde se aplica (solo en bodies, no en shapes)

- `StaticBody3D.physics_material_override` (verificado) y `RigidBody3D.physics_material_override` (verificado).
- **No hay material por shape en 3D 4.7**: `CollisionShape3D` no tiene `physics_material` (verificado). Un body con varios shapes usa el material del body para todos.
- El tipo de la propiedad es `PhysicsMaterial` (verificado en ambas páginas de clase).
- "Override": si asignas material, **se usa en vez de cualquier otro material** (p. ej. uno heredado — texto oficial). En la práctica: el material del body es el que manda para ese body.

### Combinación al chocar (las reglas que la docs documenta)

Al chocar dos objetos, el motor decide la fricción/rebote efectiva con estas reglas:

**Fricción → lo rige `rough`** (texto oficial de `rough`):

| rough A | rough B | Fricción que se usa |
|---|---|---|
| `false` | `false` | El **mínimo** de la fricción de ambos. |
| `true`  | `false` | La fricción del marcado **rough** (la de A). |
| `true`  | `true`  | El **máximo** de la fricción de ambos. |

**Rebote → lo rige `absorbent`** (texto oficial de `absorbent`):

- Si un objeto es `absorbent = true`, **resta** su rebote al del objeto que choca (en vez de sumarlo).
- Intuición: una pelota (bounce 0.8) contra un suelo `absorbent` con `bounce 0.5` → rebote efectivo ≈ 0.3 (el suelo "absorbe" 0.5).

### `bounce = 1.0` NO es energía infinita (nota oficial)

La docs lo dice explícito: **aun con `bounce = 1.0` se pierde energía con el tiempo por el damping lineal y angular**. Receta oficial de un body que **conserva toda la energía**:

```text
bounce = 1.0
linear_damp_mode = REPLACE  (si aplica)  y  linear_damp = 0.0
angular_damp_mode = REPLACE (si aplica)  y  angular_damp = 0.0
```

(Es decir: el material pone el rebote, y el body se encarga de no disipar con damping.)

## API relevante

Todo verificado en Godot 4.7.

### `PhysicsMaterial` (hereda `Resource`)

```gdscript
# Crear y configurar (código):
var mat := PhysicsMaterial.new()
mat.friction = 0.3    # 0..1
mat.bounce = 0.7      # 0..1
mat.rough = false
mat.absorbent = false
```

| Propiedad | Default | Nota |
|---|---|---|
| `friction` | `1.0` | 0 = sin fricción, 1 = máxima. |
| `bounce` | `0.0` | 0 = sin rebote, 1 = rebote total (aun así disipa por damping — ver arriba). |
| `rough` | `false` | Cambia la combinación de fricción (tabla de reglas). |
| `absorbent` | `false` | Resta el rebote al otro. |

Getters/setters: `set_friction`/`get_friction`, `set_bounce`/`get_bounce`, `set_rough`/`is_rough`, `set_absorbent`/`is_absorbent` (verificados).

### Dónde se asienta

| Propiedad del body | Tipo | Nota |
|---|---|---|
| `StaticBody3D.physics_material_override` | `PhysicsMaterial` | Verificado. |
| `RigidBody3D.physics_material_override` | `PhysicsMaterial` | Verificado. |
| `CollisionShape3D.*` | — | **Sin** `physics_material` (verificado). |

```gdscript
# Aplicar a un body (código):
$StaticBody3D.physics_material_override = mat
# O desasignar (volver al default del motor):
$StaticBody3D.physics_material_override = null
```

## Arquitectura recomendada

### Reciclado de materiales (Resource)

`PhysicsMaterial` es un `Resource`: **crear uno por superficie-tipo** y reutilizarlo (o guardarlo como `.tres`). No es un material por instancia a menos que el comportamiento difiera.

```gdscript
# materials.gd — factory (o .tres editados en el editor)
static func hielo() -> PhysicsMaterial:
	var m := PhysicsMaterial.new()
	m.friction = 0.05   # resbaladizo (MEDIUM: punto de partida, no oficial)
	return m

static func trampolin() -> PhysicsMaterial:
	var m := PhysicsMaterial.new()
	m.bounce = 0.9      # elástico (MEDIUM)
	return m

static func concreto() -> PhysicsMaterial:
	var m := PhysicsMaterial.new()
	m.friction = 0.9    # agarre alto (MEDIUM)
	m.rough = true      # que GANE su fricción al chocar
	return m
```

### Elegir `rough` con criterio

- **Superficie que define el agarre** (suelo de concreto, arena, gripe): `rough = true` → su fricción manda aunque lo que toque sea "liso".
- **Objeto que resbala** (hielo, metal pulido, pelota): `rough = false` + fricción baja → el mínimo gana y resbala.
- **Dos "rough" chocan** → el máximo (el más agarrosos de los dos).

### Elegir `absorbent` con criterio

- **Colchón / suelo que amortigua** (rebaja el rebote de lo que cae): `bounce` alto + `absorbent = true`.
- **Superficie puramente elástica** (no absorbe): `bounce` alto + `absorbent = false`.

## Implementación mínima

**Suelo de hielo y trampolín** (los dos casos canónicos):

```text
Floor_Ice (StaticBody3D)
└─ CollisionShape3D (BoxShape3D)
   # physics_material_override → PhysicsMaterial { friction = 0.05, bounce = 0.0 }

Trampoline (StaticBody3D)
└─ CollisionShape3D (BoxShape3D)
   # physics_material_override → PhysicsMaterial { friction = 0.1, bounce = 0.9 }
```

```gdscript
# setup.gd (o en el editor asignando los .tres)
const ICE := preload("res://materials/ice.tres")
const TRAMP := preload("res://materials/trampoline.tres")

func _ready() -> void:
	$Floor_Ice.physics_material_override = ICE
	$Trampoline.physics_material_override = TRAMP
```

## Implementación recomendada

### 1. Pelota que rebot "para siempre" (body elástico perfecto)

```gdscript
# ball.gd — RigidBody3D
@ready
func _ready() -> void:
	var m := PhysicsMaterial.new()
	m.bounce = 1.0
	physics_material_override = m
	# La receta oficial de "sin pérdida de energía" (verificado):
	linear_damp_mode = RigidBody3D.DAMP_MODE_REPLACE
	linear_damp = 0.0
	angular_damp_mode = RigidBody3D.DAMP_MODE_REPLACE
	angular_damp = 0.0
```

> Solo con `bounce = 1.0` la pelota **acaba** perdiendo altura (damping — nota oficial). Los 4 valores juntos son los que la mantienen (receta de la docs).

### 2. Suelo que "agarra" a todo (fricción dominante)

```gdscript
# ground.gd — StaticBody3D
var m := PhysicsMaterial.new()
m.friction = 1.0
m.rough = true      # su fricción GANA contra lo que toque (verificado)
physics_material_override = m
```

### 3. Colchón amortiguador (absorbe el rebote)

```gdscript
# mat.gd — StaticBody3D
var m := PhysicsMaterial.new()
m.bounce = 0.6
m.absorbent = true  # RESTA 0.6 al rebote de lo que cae (verificado)
physics_material_override = m
```

### 4. Hielo (resbaladizo) + caja que se desliza

```gdscript
# ice.gd — StaticBody3D
var m := PhysicsMaterial.new()
m.friction = 0.03
m.rough = false     # el mínimo gana → resbala (verificado)
physics_material_override = m
```

Caja (`RigidBody3D`) con `rough = false` + fricción media: al chocar con el hielo, el **mínimo** (el hielo) manda → se desliza.

### 5. Comprobar en runtime el material de un body

```gdscript
func print_material_of(b: PhysicsBody3D) -> void:
	var m: PhysicsMaterial = (b as StaticBody3D).physics_material_override \
		if b is StaticBody3D else (b as RigidBody3D).physics_material_override
	if m:
		print("friction=", m.friction, " bounce=", m.bounce,
			  " rough=", m.rough, " absorbent=", m.absorbent)
	else:
		print("sin material (defaults del motor)")
```

## Ejemplo práctico

**Juego**: nivel de plataformas 3D con hielo, trampolines y un campo de minas de pinchos.

1. **Hielo**: `StaticBody3D` + `PhysicsMaterial {friction=0.03, rough=false}`. El personaje (CharacterBody) y las cajas resbalan (el mínimo gana).
2. **Trampolín**: `bounce = 0.9` (MEDIUM) + `absorbent=false`; las `RigidBody3D` que caen rebotan alto.
3. **Suelo de agarre** (tramos con textura de goma): `friction = 1.0, rough = true` → detiene a las cajas que rodaban.
4. **Balas de boliche**: `RigidBody3D` con `bounce=0.2, rough=false` → rebotan poco, se deslizan.
5. **Debug**: imprimir `friction/bounce/rough/absorbent` del body en el punto de contacto (receta #5) para ver QUÉ material está mandando; cruzar con la tabla de `rough` (mínimo/rough/máximo).
6. **No**: no crear 200 `PhysicsMaterial` idénticos (uno por `.tres` por tipo de superficie y reutilizar).

## Integración

- **`godot-physics`**: el material vive en `physics_material_override` de Static/Rigid; el damping (`linear/angular_damp` + `DampMode`) es el compañero necesario para el body elástico perfecto (receta oficial); `gravity_scale`/`mass` son el resto del comportamiento.
- **`godot-characterbody3d`**: el personaje NO usa el material para su slide (lo programa tú); el material sí afecta a los `RigidBody3D`/`StaticBody3D` con los que el personaje interactúa.
- **`godot-area3d`**: complementario: el Area puede cambiar `linear_damp` local (que interactúa con el damping del body); el material no se aplica por area.
- **`godot-rendering-performance`** / materiales de render: no confundir `PhysicsMaterial` (física) con `StandardMaterial3D` (render).
- **`godot-vehiclebody3d` (pendiente)**: el agarre de ruedas también usa el material de la superficie.

## Errores frecuentes

1. **Crear `PhysicsMaterial3D` (no existe)** → la clase es `PhysicsMaterial` (verificado: la página de `PhysicsMaterial3D` da 404). Usar `PhysicsMaterial.new()`.
2. **Usar `friction_combine_mode`/`bounce_combine_mode`** → no existen en 4.7 (verificado). La combinación es `rough` (fricción) y `absorbent` (rebote).
3. **Poner material en `CollisionShape3D`** → no tiene `physics_material` (verificado). Va en `physics_material_override` del body.
4. **`bounce = 1.0` y "la pelota debe durar para siempre"** → pierde energía por damping (nota oficial). Faltan los 4 valores (bounce + damp 0 + REPLACE ×2).
5. **Esperar que el material afecte al slide de un `CharacterBody3D`** → no: el slide del personaje lo programa el código (`godot-characterbody3d`). El material afecta a Static/Rigid.
6. **Fricción alta "no agarra"** → probablemente el otro lado es `rough=false` y tiene fricción más baja → el **mínimo** gana (ver tabla). Poner `rough=true` en el que debe mandar.
7. **`absorbent` entendido como "suma rebote"** → hace lo contrario: **resta** (verificado). Para sumar, `absorbent=false` en ambos.
8. **Crear un `PhysicsMaterial` por instancia** (200 iguales) → es un `Resource`: uno por tipo, reutilizado (o `.tres`).
9. **Confundir con el material de render** (`roughness` PBR de `StandardMaterial3D`) → distinto sistema; el de física es `friction`/`bounce`.
10. **Asignar material a un body y que "no haga nada"** → ¿el body es Static/Rigid? (a un `Area3D`/`CharacterBody3D` no le aplica como tal); ¿reemplazaste el material correcto (`override`)? ¿el otro lado manda (rough)?

## Anti-patrones

- **Material por instancia cuando alcanza uno por tipo** (reciclado de `Resource`).
- **Afinar `bounce`/`friction` a ciegas** ("pongo 0.7") → medir el resultado (FPS del comportamiento/altura de rebote) e iterar; los valores de esta skill son MEDIUM (puntos de partida, no oficiales).
- **Usar `PhysicsMaterial` para fricción de personaje** → el slide es código (`godot-characterbody3d`); el material no controla el slide del character.
- **Depender de `bounce=1.0` solo** para "conservar energía" → incompleto (falta el damping — nota oficial).
- **Esperar que un Area aplique material** → el material es del body; el Area ajusta gravedad/damping/viento (ver `godot-area3d`).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **El material no tiene coste de CPU/GPU directo**: es un `Resource` con 4 floats/bools que el motor lee al resolver el contacto. No es un "garganta de rendimiento" por sí mismo.
- **Dónde SÍ hay coste (indirecto, medible)**:
  - Un rebote alto (`bounce` cercano a 1) mantengo cuerpos en movimiento → más ticks con pares activos → se refleja en `Performance.PHYSICS_3D_ACTIVE_OBJECTS`/`COLLISION_PAIRS` (índices 20/21, ver `godot-physics`) y en el tiempo de tick (FPS).
  - Un body "elástico perfecto" (receta #1) que nunca duerme → `can_sleep` relevante (ver `godot-physics`).
- **Qué medir**: si introduces rebote/fricción y el FPS baja, mirar el pair count y el tiempo de física (no culpes al material).
- **No hay "optimización de material" por intuición**: el material es barato; el movimiento que provoca es lo que se mide.

## Debugging

### "No resbala / no rebota como quiero"

```text
1. ¿El material está asignado al body correcto (physics_material_override)?
   (print del material, receta #5)
2. Fricción: ¿quién manda? Ver la tabla de rough (mínimo / rough / máximo).
   Si el otro lado tiene fricción más baja y rough=false, manda el otro.
3. Rebote: ¿el otro lado es absorbent? (resta — verificado)
4. ¿Es CharacterBody? (el material no controla su slide)
5. ¿bounce=1 pero se frena? → damping (receta de 4 valores, ver error #4)
```

### "El rebote es distinto en el piso A que en el B"

```text
1. Imprimir friction/bounce/rough/absorbent de AMBOS lados en el contacto.
2. Aplicar la tabla de rough y la regla de absorbent (ver Conceptos).
3. Cruzar con el damping de cada body (DAMP_MODE_COMBINE suma el del
   area/default — godot-physics).
```

### "Asgné un material y sigue el default"

```text
1. ¿Asignaste al body o a un shape? (solo body: physics_material_override)
2. ¿"override" reemplaza (texto oficial) — no se suma a otro material?
3. ¿El body tiene OTRO material que manda (rough del otro lado)?
4. ¿Reasignaste a null por error? (null = defaults del motor)
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — tabla de `PhysicsMaterial` + páginas de `RigidBody3D`/`StaticBody3D`/`CollisionShape3D` (lista en *Referencias*).
- **3.x → 4.x (trampas, verificado contra tablas 4.7)**:
  - `PhysicsMaterial3D`/`PhysicsMaterial2D` como clases separadas: la clase de la docs 4.7 es **`PhysicsMaterial`** (una sola). La página `class_physicsmaterial3d.html` no existe (404).
  - `friction_combine_mode`/`bounce_combine_mode` (enums AVERAGE/MAX/MIN/MULTIPLY de versiones anteriores): **no están en la tabla 4.7**. La combinación es `rough`/`absorbent`.
  - `CollisionShape3D.physics_material` (material por shape en 3D): **no está en la tabla 4.7** → material por body (`physics_material_override`).
- **4.0 ↔ 4.7**: la clase consultada (4 props) es estable para el uso cubierto.

## Dependencias

| Skill | Relación |
|---|---|
| `godot-physics` | Base: bodies, damping (receta elástica), `gravity_scale` |
| `godot-characterbody3d` | Aclaración: el slide no usa el material |
| `godot-area3d` | Damping local por zona (complemento) |
| `godot-vehiclebody3d` (pendiente) | Agarre de ruedas |
| `godot-physics-materials-2d` (pendiente) | Espejo 2D (misma clase base) |

## Skills relacionadas

- `godot-recipe-bouncy-surfaces` — receta (pendiente; germen en *Implementación recomendada*).
- `godot-recipe-ice-sliding` — pendiente (germen #4).
- `godot-error-physics-material-not-working` — pendiente (germen en *Debugging*).

## Referencias oficiales

Verificadas el 2026-09-15 en `stable` (4.7):

- PhysicsMaterial (las 4 props, reglas de `rough`/`absorbent`, nota de `bounce=1.0` y damping, getters/setters): https://docs.godotengine.org/en/stable/classes/class_physicsmaterial.html
- RigidBody3D (`physics_material_override`, `linear/angular_damp_mode`, `DampMode`): https://docs.godotengine.org/en/stable/classes/class_rigidbody3d.html
- StaticBody3D (`physics_material_override`): https://docs.godotengine.org/en/stable/classes/class_staticbody3d.html
- CollisionShape3D (sin `physics_material`): https://docs.godotengine.org/en/stable/classes/class_collisionshape3d.html
- Physics introduction (static/rigid usan PhysicsMaterial; fricción/rebote/absorbent/rough): https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html
