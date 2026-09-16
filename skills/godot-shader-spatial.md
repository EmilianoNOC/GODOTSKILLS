# godot-shader-spatial

## ID

`godot-shader-spatial`

## Categoría

Shaders · Base

## Versión Godot

**4.7 (stable)** — verificada el 2026-09-15 contra la documentación oficial 4.7: `tutorials/shaders/shader_reference/spatial_shader.html`, `shading_language.html`, `shader_functions.html` (docs.godotengine.org/en/stable/, rama 4.7).

## Confidence

**HIGH** — render modes, built-ins (globales/vertex/fragment/light), hints de uniform, uniform limits, funciones de textura, instance/global uniforms: verificados directamente en la docs 4.7.
**MEDIUM**: recomendaciones de tuning visual (intensidades de efectos).
**"No existe en 4.x (verificado)"**: sección *Spatial functions* (`get_spatial_lighting`, `sample_skybox_radiance/irradiance`) — **no existe en 4.7** (funciones de 3.x, eliminadas).

## Nivel

**BASE** (sin prerrequisitos; para 2D ver futura `godot-shader-canvasitem`).

## Propósito

Proporcionar el conocimiento operativo completo de los shaders **espaciales** (`shader_type spatial`) en Godot 4.x: estructura (vertex/fragment/light), render modes, built-ins por función, uniforms (con hints, global y per-instance), funciones de textura, y las trampas de rendimiento/version que rompen shaders (sRGB, transparent pipeline, `discard`, `textureSize`).

## Cuándo utilizarla

- Efectos visuales 3D: highlight, outline, distorsión, disolución, vertex animation, brillo.
- Leer la pantalla en 3D (`hint_screen_texture`) para efectos screen-space en objetos.
- Uniforms por instancia (N enemigos, 1 material) o globales (viento en todo el mundo).
- Depurar "el shader no compila / se ve lavado / no se ve la textura".

## Cuándo NO utilizarla

- **Efectos 2D** (UI, sprites) → `shader_type canvas_item` (skill pendiente).
- **Materiales PBR estándar sin código** → `StandardMaterial3D` (inspector) es más simple que un `ShaderMaterial` (mínima complejidad primero).
- **Sistemas de partículas** → `shader_type particles` (skill pendiente).
- **Sky/fog** → `shader_type sky` / `fog` (skills pendientes).

## Conceptos fundamentales

### Las tres funciones (todas opcionales)

| Función | Corre | Qué tocar |
|---|---|---|
| `vertex()` | Por vértice | `VERTEX` (model space), `NORMAL`, `TANGENT`, `BITANGENT`, `UV`, `COLOR`, `POINT_SIZE`. Si no se escribe, pasan sin cambios al fragment. |
| `fragment()` | Por píxel | Los **props del material**: `ALBEDO`, `METALLIC`, `ROUGHNESS`, `SPECULAR`, `EMISSION`, `ALPHA`, `NORMAL_MAP`, `RIM`, `SSS_STRENGTH`, `AO`, `TRANSMISSION`… El motor hace el shading final con lo que escribas. |
| `light()` | Por píxel **por cada luz** | `DIFFUSE_LIGHT`, `SPECULAR_LIGHT` (usar `+=` para sumar luces). Opcional; si no existe, el motor ilumina con los props de fragment. |

Si no escribes `light()`, **no escribes una iluminación desde cero**: mezclas el PBR estándar del motor. Escribir `light()` es para modelos de luz custom (Lambert propio, toon…).

### Render modes (verificados 4.7)

Los más usados (tabla completa en *API relevante*):

| Render mode | Efecto |
|---|---|
| `unshaded` | Solo albedo, sin lighting (más rápido; ignora `light()` y sombras). |
| `cull_disabled` | Doble cara (foliage, hojas). |
| `cull_front` | Solo caras frontales (interior de cuevas/burbujas). |
| `blend_add` / `blend_sub` / `blend_mul` / `blend_premul_alpha` | Modos de blending (partículas, glow). |
| `depth_draw_never` / `depth_draw_always` / `depth_draw_opaque` | Control de escritura de depth. |
| `depth_test_inverted` | Para efectos stencil (docs: "useful for stencil effects"). |
| `shadows_disabled` | No recibe sombras (sigue proyectándolas). |
| `ambient_light_disabled` | Sin contribución de ambiente/radiance. |
| `vertex_lighting` | Iluminación por vértice (móvil; **no ejecuta `light()`** — warning oficial). |
| `alpha_to_coverage` / `alpha_to_coverage_and_one` | Antialiasing de alpha (foliage). |
| `fog_disabled` | No recibe niebla (útil con `blend_add`, partículas). |
| `wireframe` | Depurar geometría (en Compatibility: `RenderingServer.set_debug_generate_wireframes(true)` **antes** de cargar el mesh). |
| `skip_vertex_transform` / `world_vertex_coords` | Controlar la transformación de vértice a mano. |
| `diffuse_*` (burley default, lambert, lambert_wrap, toon) / `specular_*` (schlick_ggx default, toon, disabled) | Modelos de shading. |
| `sss_mode_skin` | Subsurface modo piel. |

**Stencil**: `read`/`write`/`write_if_depth_fail` + `compare_*` — **experimental** (aviso oficial); solo se puede **leer en el pass transparente** (en opaque falla).

### sRGB y `source_color` (la trampa nº1 de color)

- Godot renderiza en **espacio de color lineal**. Las texturas de color vienen en **sRGB**.
- **`source_color`** en el uniform de la textura dice "es sRGB, conviértela". **Obligatorio en Forward+ y Mobile** (y en `canvas_item` con HDR 2D); opcional en Compatibility. **Recomendado siempre** (funciona al cambiar de renderer).
- Sin `source_color`: la textura se ve **lavada** (síntoma clásico).
- Normal/roughness/metallic/height: **NO** llevan `source_color` (son datos, no color).

### Transparent pipeline (el coste de escribir `ALPHA`)

**Si escribes `ALPHA` en cualquier branch, el material pasa al pipeline transparente** (docs 4.7). Consecuencias oficiales:

- Ordenado back-to-front (painter's algorithm) → **sorting issues** posibles (ver *3D rendering limitations*).
- **No proyecta sombras** y **no aparece** en `hint_screen_texture`/`hint_depth_texture` (no entra en screen-space reflections/refraction).
- SDFGI: solo reflejos rough visibles (no sharp).
- → **Evitar escribir `ALPHA` si no es necesario** (p. ej. `ALPHA = 1.0` constante NO fuerza el pipeline; no escribir nada es mejor aún).

## API relevante

Todo verificado en Godot 4.7.

### Built-ins globales (disponibles en todas las funciones)

| Built-in | Tipo | Nota |
|---|---|---|
| `TIME` | `in float` | Segundos desde el arranque; **repite cada 3600 s** (configurable con `rendering/limits/time/time_rollover_secs`); afectado por `Engine.time_scale`, NO por pausa. Para tiempo no rolleo: uniform global propio actualizado por frame. |
| `PI` / `TAU` / `E` | `in float` | 3.141592 / 6.283185 / 2.718281. |
| `OUTPUT_IS_SRGB` | `in bool` | `true` en Compatibility; `false` en Forward+/Mobile. |
| `CLIP_SPACE_FAR` | `in float` | 0.0 (Forward+/Mobile) / -1.0 (Compatibility). |
| `IS_MULTIVIEW` | `in bool` | XR estéreo. |
| `IN_SHADOW_PASS` | `in bool` | ¿Se está renderizando en la shadow map? (renderizar distinto en sombras). |

### Built-ins de vertex

| Built-in | Tipo | Nota |
|---|---|---|
| `VERTEX` | `inout vec3` | Posición, **model space** (world con `world_vertex_coords`). |
| `NORMAL` / `TANGENT` / `BITANGENT` | `inout vec3` | Model space. |
| `POSITION` | `out vec4` | Si se escribe (en cualquier branch), **sobrescribe la posición final en clip space** (el motor no proyecta; tú eres responsable). |
| `UV` / `UV2` | `inout vec2` | Pasan al fragment si no se modifican. |
| `COLOR` | `inout vec4` | Color de vértice. **Limitado a 0..1 con 8 bits/canal** (256 niveles); para más precisión usar `CUSTOM0`–`CUSTOM3`. |
| `POINT_SIZE` | `inout float` | Tamaños de puntos. |
| `MODELVIEW_MATRIX` | `inout mat4` | Model→view (preferir sobre MODEL×VIEW por float precision). |
| `MODELVIEW_NORMAL_MATRIX` | `inout mat3` | — |
| `MODEL_MATRIX` | `in mat4` | Model→world. |
| `MODEL_NORMAL_MATRIX` | `in mat3` | — |
| `PROJECTION_MATRIX` | `inout mat4` | View→clip. |
| `VIEW_MATRIX` / `INV_VIEW_MATRIX` | `in mat4` | World↔view. |
| `INV_PROJECTION_MATRIX` | `in mat4` | Clip→view. |
| `NODE_POSITION_WORLD` / `NODE_POSITION_VIEW` | `in vec3` | Posición del nodo. |
| `CAMERA_POSITION_WORLD` / `CAMERA_DIRECTION_WORLD` | `in vec3` | — |
| `CAMERA_VISIBLE_LAYERS` | `in uint` | Capas de culling de la cámara del pass. |
| `INSTANCE_ID` | `in int` | ID de instancia (instancing). |
| `INSTANCE_CUSTOM` | `in vec4` | Datos custom por instancia (partículas: x=rotación rad, z=frame, y/w=frac de vida). |
| `VIEW_INDEX` / `VIEW_MONO_LEFT` / `VIEW_RIGHT` / `EYE_OFFSET` | — | Multiview/XR. |
| `VIEWPORT_SIZE` | `in vec2` | Píxeles. |
| `VERTEX_ID` | `in int` | Índice de vértice en el buffer. |
| `BONE_INDICES` | `in uvec4` | Índices de hueso (skin; lectura). |
| `ROUGHNESS` | `out float` | Solo para vertex lighting. |

### Built-ins de fragment

| Built-in | Tipo | Nota |
|---|---|---|
| `VIEWPORT_SIZE` | `in vec2` | — |
| `FRAGCOORD` | `in vec4` | Píxel en pantalla (0,0 = arriba-izq; 1,1 = abajo-der); `z` = profundidad. |
| `FRONT_FACING` | `in bool` | Caras frontales. |
| `VERTEX` | `in vec3` | Posición del fragment en **view space** (interpolada). |
| `LIGHT_VERTEX` | `inout vec3` | Versión escribible de `VERTEX` para luz/sombras (no mueve el fragment). |
| `NORMAL` / `TANGENT` / `BINORMAL` | `inout vec3` | View space (desde `vertex()`). |
| `UV` / `UV2` | `in vec2` | Desde `vertex()`. |
| `COLOR` | `in vec4` | Desde `vertex()`. |
| `VIEW` | `in vec3` | Vector fragment→cámara (view space), normalizado. |
| `MODEL_MATRIX` / `MODEL_NORMAL_MATRIX` / `VIEW_MATRIX` / `INV_VIEW_MATRIX` / `PROJECTION_MATRIX` / `INV_PROJECTION_MATRIX` | `in` | Matrices (mismo que vertex). |
| `NODE_POSITION_WORLD` / `NODE_POSITION_VIEW` / `CAMERA_POSITION_WORLD` / `CAMERA_DIRECTION_WORLD` | `in vec3` | — |
| `CAMERA_VISIBLE_LAYERS` | `in uint` | — |
| `VIEW_INDEX` / `VIEW_MONO_LEFT` / `VIEW_RIGHT` / `EYE_OFFSET` | — | XR. |
| `SCREEN_UV` | `in vec2` | UV de pantalla del píxel (0,0 = arriba-izq). |
| **`SCREEN_TEXTURE` / `DEPTH_TEXTURE`** | — | **Eliminados en Godot 4** → usar `sampler2D` con `hint_screen_texture` / `hint_depth_texture`. |
| `DEPTH` | `out float` | Depth custom [0,1]; si escribes en un branch, debes escribirlo en **todos**. |
| `NORMAL_MAP` | `out vec3` | Normal en **tangent space** (el canal blue se reconstruye: compatible con RGTC). |
| `NORMAL_MAP_DEPTH` | `out float` | Default 1.0. |
| `BENT_NORMAL_MAP` | `out vec3` | Bent normals (occlusión especular; requiere autoría). |
| `ALBEDO` | `out vec3` | Base color (default blanco). |
| `ALPHA` | `out float` | [0,1]; **escribirla (en cualquier branch) → pipeline transparente**. |
| `ALPHA_SCISSOR_THRESHOLD` | `out float` | Descartar alphas bajos (alpha test; evita el pipeline transparente completo). |
| `ALPHA_HASH_SCALE` | `out float` | Alpha hash (dithering), default 1.0. |
| `ALPHA_ANTIALIASING_EDGE` | `out float` | Umbral de `alpha_to_coverage` (default 0; < `ALPHA_SCISSOR_THRESHOLD`). |
| `ALPHA_TEXTURE_COORDINATE` | `out vec2` | UVs para alpha-to-coverage (`UV * tamaño_textura`). |
| `PREMUL_ALPHA_FACTOR` | `out float` | Solo con `blend_premul_alpha` (materiales shaded). |
| `METALLIC` / `SPECULAR` / `ROUGHNESS` | `out float` | [0,1]; `SPECULAR` default 0.5 (0.0 = sin reflejos; "not physically accurate to change"). |
| `RIM` / `RIM_TINT` | `out float` | Rim lighting (tamaño depende de `ROUGHNESS`; tint 0=blanco→1=albedo). |
| `CLEARCOAT` / `CLEARCOAT_GLOSS` | `out float` | Clearcoat. |
| `ANISOTROPY` / `ANISOTROPY_FLOW` | `out float` / `out vec2` | Anisotropía (flowmaps). |
| `SSS_STRENGTH` / `SSS_TRANSMITTANCE_COLOR` / `SSS_TRANSMITTANCE_DEPTH` / `SSS_TRANSMITTANCE_BOOST` | | Subsurface scattering. |
| `BACKLIGHT` | `inout vec3` | Backlighting (aproximación barata de SSS). |
| `AO` / `AO_LIGHT_AFFECT` | `out float` | AO pre-bakeada (0..1); cuánta afecta a luz directa. |
| `EMISSION` | `out vec3` | **HDR** (puede superar 1,1,1). |
| `FOG` | `out vec4` | Mezclar color final con `FOG.rgb` por `FOG.a`. |
| `RADIANCE` / `IRRADIANCE` | `out vec4` | Mezclar env. map con el propio (reflejos/luz de ambiente custom). |

### Built-ins de light (en `light()`)

| Built-in | Tipo | Nota |
|---|---|---|
| `NORMAL` | `in vec3` | View space. |
| `SCREEN_UV` / `UV` / `UV2` | `in vec2` | — |
| `VIEW` | `in vec3` | Vector de vista. |
| `LIGHT` | `in vec3` | Vector hacia la luz (view space). |
| `LIGHT_COLOR` | `in vec3` | Color × energy × **PI** (el PI porque el PBR divide por PI). |
| `SPECULAR_AMOUNT` | `in float` | Omni/Spot: `2.0 × light_specular`; Directional: `1.0`. |
| `LIGHT_IS_DIRECTIONAL` | `in bool` | — |
| `LIGHT_IS_AREA` | `in bool` | (usado en el ejemplo oficial de light). |
| `ATTENUATION` | `in float` | Atenuación por distancia/sombra. |
| `ALBEDO` / `METALLIC` / `ROUGHNESS` / `BACKLIGHT` | `in` | Los props que escribiste en `fragment()`. |
| `DIFFUSE_LIGHT` / `SPECULAR_LIGHT` | `out vec3` | **Usar `+=`** para que las luces se sumen. |
| `ALPHA` | `out float` | Como en fragment (transparent pipeline). |
| `VIEWPORT_SIZE` / `FRAGCOORD` / matrices | `in` | Disponibles también. |
| `LIGHT_AREA_DIFFUSE_MULTIPLIER` / `LIGHT_AREA_SPECULAR_MULTIPLIER` | `in` | Solo luces de área (ejemplo oficial). |

**Warning oficial**: `light()` **no se ejecuta** si `render_mode vertex_lighting` o si está activo *Rendering > Quality > Shading > Force Vertex Shading* (default en móvil).

Ejemplo oficial de `light()` (Lambertian):

```glsl
void light() {
	if (LIGHT_IS_AREA) {
		DIFFUSE_LIGHT += LIGHT_AREA_DIFFUSE_MULTIPLIER * ATTENUATION * LIGHT_COLOR;
		SPECULAR_LIGHT += LIGHT_AREA_SPECULAR_MULTIPLIER * ATTENUATION * LIGHT_COLOR * SPECULAR_AMOUNT;
	} else {
		DIFFUSE_LIGHT += clamp(dot(NORMAL, LIGHT), 0.0, 1.0) * ATTENUATION * LIGHT_COLOR / PI;
	}
}
```

### Uniforms y hints (tabla completa verificada 4.7)

| Tipo | Hint | Descripción |
|---|---|---|
| vec3, vec4 | `source_color` | Color (sRGB). |
| int | `hint_enum("A", "B")` | Dropdown en el inspector. Con colon: `hint_enum("Slow:30", "Fast:200")` → valores explícitos. `set_shader_parameter` usa el **int**. |
| int, float | `hint_range(min, max[, step])` | Rango limitado. |
| sampler2D | `source_color` | Albedo/color (sRGB). |
| sampler2D | `hint_normal` | Normal map. |
| sampler2D | `hint_default_white` / `hint_default_black` / `hint_default_transparent` | Defaults. |
| sampler2D | `hint_anisotropy` | Flowmap. |
| sampler2D | `hint_roughness[_r,_g,_b,_a,_normal,_gray]` | Roughness limiter (anti aliasing especular). |
| sampler2D | `filter[_nearest,_linear][_mipmap][_anisotropic]` | Filtrado. |
| sampler2D | `repeat[_enable,_disable]` | Repeat. |
| sampler2D | `hint_screen_texture` | **La textura de pantalla** (post en 3D). |
| sampler2D | `hint_depth_texture` | **La textura de depth**. |
| sampler2D | `hint_normal_roughness_texture` | Normal+roughness (solo Forward+). |

Reglas:

- El **valor por defecto va DESPUÉS del hint**: `uniform vec4 c : source_color = vec4(1.0);`
- `const` **no** admite hints (y es más barato que uniform: no es editable en runtime por material).
- **Límites de uniforms** (verificado): desktop **65536 bytes** (4096 `vec4`); móvil **16384 bytes** (1024 `vec4`). `vec2/vec3` se rellenan a `vec4`; escalares no; `bool` → `int`. Arrays = tamaño total. → Para datos grandes, **textura** (el contenido de la textura no cuenta, solo el sampler).

### Uniforms globales (verificados 4.7)

- Se crean en **Project Settings → Shader Globals** (nombre + tipo). Deben existir **cuando se guarda el shader** o no compila (el default en el shader se ignora).
- Uso: `global uniform vec4 my_color;` (disponible en TODOS los shaders del proyecto, todos los tipos).
- Runtime: `RenderingServer.global_shader_parameter_set("my_color", Color(...))` — **sin coste** (no sincroniza CPU/GPU).
- Añadir/quitar en runtime: `global_shader_parameter_add(name, RenderingServer.GLOBAL_VAR_TYPE_COLOR, value)` / `global_shader_parameter_remove(name)` — tiene coste (no es lo mismo que set).
- **Warning oficial**: `global_shader_parameter_get(name)` tiene **gran penalización** (sincroniza el render thread). No leer en bucle; si hay que leer, guardar el valor en un autoload al setear.

### Uniforms por instancia (verificados 4.7)

- `instance uniform vec4 my_color : source_color = vec4(1.0, 0.5, 0.0, 1.0);` — el valor se edita **en el `GeometryInstance3D`** (no en el material).
- Runtime: `geometry_instance.set_instance_shader_parameter("my_color", Color(...))`.
- Restricciones oficiales: **sin texturas ni arrays** (workaround: array como uniform normal + índice como instance uniform; en Compatibility/GLSL<4.0 usar `switch` para indexar el array).
- **Máximo práctico: 16 instance uniforms por shader.**
- Mesh con varios materiales: ganan los parámetros del **primer** material.
- Caso de uso: N árboles/enemigos con material compartido y un color/highlight propio (sin duplicar materiales → sin duplicar draw calls por material).

### Funciones de textura (verificadas 4.7, `shader_functions.html`)

| Función | Nota |
|---|---|
| `texture(s, uv[, bias])` | Muestreo estándar (`gsampler2D`/`samplerCube`…). |
| `textureLod(s, uv, lod)` | LOD explícito (deriva = 0). |
| `textureGrad(s, uv, dPdx, dPdy)` | Gradientes explícitos (mipmaps a mano). |
| `textureProj(s, uv[, bias])` / `textureProjLod` / `textureProjGrad` | Con proyección (divide por el último componente). |
| `texelFetch(s, ivec, lod)` | Texel exacto (sin filtrado/LOD). |
| `textureGather(s, uv[, comps])` | 4 texels (barrido de 2×2). |
| `textureSize(s, lod)` | **Performance: evitar** (leída completa de textura; "always performs a full texture read" — pasar el tamaño por uniform si es constante). |
| `textureQueryLod(s, uv)` / `textureQueryLevels(s)` | LOD que se usaría / nº de niveles. |

### Derivadas (verificadas 4.7)

`dFdx`, `dFdy`, `fwidth` (+ `Coarse`/`Fine`: solo fragment; `Coarse/Fine` **no disponibles en Compatibility**). Warning oficial: derivadas de orden superior (`dFdx(dFdx(x))`) = undefined.

### Random/noise en shader

- Funciones de ruido builtin (`noise_*`, `cellular_*`, `perlin_*`, `value_*`) y `rand*`: **API no verificada** en esta revisión (la sección no aparece en `shader_functions.html` 4.7); verificar en la rama actual antes de usar o usar hash propio (abajo).
- Hash clásico (determinista, sin ruido builtin):

```glsl
float hash(vec2 p) {
	return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
}
```

### `discard`

- Descarta el fragment (nada se escribe).
- **Coste oficial**: impide que el **depth prepass** sea efectivo en esa superficie (y el vértice se renderiza igual). Usar solo donde no se pueda evitar (mejor `ALPHA_SCISSOR_THRESHOLD` cuando el corte es por alpha).

### Lo que NO existe en 4.7 (verificado)

| API 3.x | Estado en 4.7 | Alternativa |
|---|---|---|
| `get_spatial_lighting()` | **No existe** (sección *Spatial functions* eliminada) | Escribir `light()` propio. |
| `sample_skybox_radiance()` / `sample_skybox_irradiance()` | **No existen** | `RADIANCE`/`IRRADIANCE` (out vec4) para mezclar env. map. |
| Built-ins `SCREEN_TEXTURE` / `DEPTH_TEXTURE` | **Eliminados en Godot 4** | `uniform sampler2D x : hint_screen_texture;` |

## Arquitectura recomendada

### Estructura de un shader (plantilla)

```glsl
shader_type spatial;
render_mode cull_disabled;              // solo si es necesario (doble cara)

// --- Constants (baratas, no editables) ---
const vec3 TINT = vec3(1.0, 0.9, 0.8);

// --- Uniforms por material ---
uniform sampler2D albedo : source_color, hint_default_black;
uniform vec4 highlight_color : source_color = vec4(1.0, 0.85, 0.2, 1.0);

// --- Uniform por instancia (edible en cada GeometryInstance3D) ---
instance uniform float highlight_amount : hint_range(0.0, 1.0) = 0.0;

// --- Uniform global (Project Settings > Shader Globals) ---
// global uniform vec4 wind_color;

void vertex() {
	// Solo si hay vertex animation / desplazamiento.
	// VERTEX, NORMAL, TANGENT en model space.
}

void fragment() {
	ALBEDO = texture(albedo, UV).rgb;
	// Escribe solo los props que necesitas: el motor optimiza lo que no tocas.
}

// void light() { ... }   // solo para modelos de luz custom
```

Decisiones:

1. **Escribir solo lo que se usa**: "if you don't write to them, Godot will optimize away the corresponding functionality" (docs).
2. **`const` para lo fijo, `uniform` para lo editable, `instance uniform` para lo por-nodo, `global uniform` para lo por-mundo** (de más barato a más "global").
3. **`source_color` en TODAS las texturas de color** (recomendación oficial; evita el bug de "lavado" al cambiar de renderer).
4. **Evitar `ALPHA` si no es transparente**: fuerza el pipeline transparente (sin sombras, sin screen-space reflections).
5. **Alpha test con `ALPHA_SCISSOR_THRESHOLD`** (foliage) en vez de `ALPHA` real cuando el corte es duro.

## Implementación mínima

**Highlight de selección** (el shader de la compuesta Lote-0, verificado 4.7):

```glsl
shader_type spatial;

instance uniform float highlight_amount : hint_range(0.0, 1.0) = 0.0;
uniform vec4 highlight_color : source_color = vec4(1.0, 0.85, 0.2, 1.0);

void fragment() {
	vec4 base = texture(albedo_texture(), UV);  // ver nota abajo
	ALBEDO = mix(base.rgb, highlight_color.rgb, highlight_amount);
}
```

> **Nota**: el built-in `TEXTURE` (sampler del albedo del material) existe en 4.x como acceso al albedo estándar; la forma explícita y robusta es declarar `uniform sampler2D albedo : source_color;` y asignar la textura en el material. (Verificar el nombre exacto del built-in `TEXTURE` en la versión antes de usarlo — **API no verificada** en esta revisión; la vía `uniform` siempre funciona.)

```gdscript
# Desde GDScript (GeometryInstance3D, verificado 4.7):
$MeshInstance3D.set_instance_shader_parameter("highlight_amount", 0.6)
# o por material (ShaderMaterial):
$MeshInstance3D.get_surface_override_material(0).set_shader_parameter("highlight_amount", 0.6)
```

## Implementación recomendada

### 1. Disolución (dissolve) con borde de fuego

```glsl
shader_type spatial;

uniform sampler2D dissolve_noise : hint_default_black;
uniform float dissolve_progress : hint_range(0.0, 1.0) = 0.0;
uniform vec3 edge_color : source_color = vec3(1.0, 0.5, 0.1);
uniform float edge_width : hint_range(0.0, 0.2) = 0.04;

void fragment() {
	vec4 base = texture(TEXTURE, UV);          // albedo del material
	float n = texture(dissolve_noise, UV * 3.0).r;
	float d = n - dissolve_progress;
	if (d < 0.0) {
		discard;                                 // el trozo desaparece
	}
	float edge = smoothstep(0.0, edge_width, d);
	ALBEDO = mix(base.rgb, edge_color, 1.0 - edge);
	EMISSION = edge_color * (1.0 - edge) * 2.0; // borde brillante (HDR)
	ALPHA = base.a;
}
```

> `discard` desactiva el depth prepass de la superficie (coste oficial); si el disolve es frecuente y caro, considerar `ALPHA_SCISSOR_THRESHOLD` con un corte suave.

### 2. Efecto screen-space en 3D (resplandor que "lee" la pantalla)

```glsl
shader_type spatial;
render_mode blend_add;

uniform float glow_strength : hint_range(0.0, 4.0) = 1.0;

void fragment() {
	vec4 screen = texture(TEXTURE_SCREEN, SCREEN_UV);  // si usas el built-in de pantalla
	ALBEDO = screen.rgb * glow_strength;
	ALPHA = 1.0;
}
```

> Patrón robusto (sin depender de built-ins de pantalla): `uniform sampler2D screen_tex : hint_screen_texture;` + asignar la textura de pantalla desde el script (`Viewport.get_texture()` → `set_texture`). **Verificar el built-in `TEXTURE_SCREEN` en la versión** (API no verificada en esta revisión); la vía `hint_screen_texture` está verificada.

### 3. Viento en foliage (uniform global)

```glsl
shader_type spatial;

global uniform vec4 wind_params;   // xyz = fuerza/dirección, w = frecuencia (en Shader Globals)

void vertex() {
	float sway = sin(TIME * wind_params.w + VERTEX.x * 2.0 + VERTEX.z * 1.3);
	VERTEX += NORMAL * sway * wind_params.x * VERTEX.y;   // más alto, más se mueve
}
```

```gdscript
# Actualizar el viento (sin coste, verificado):
RenderingServer.global_shader_parameter_set("wind_params", Vector4(0.4, 0.0, 0.2, 1.5))
# NUNCA en bucle: RenderingServer.global_shader_parameter_get("wind_params") → gran penalización (sync).
# Guardar el valor en un autoload al setear si hay que leerlo.
```

### 4. Vertex animation procedural (sin clips)

```glsl
shader_type spatial;
uniform float wave_amplitude : hint_range(0.0, 1.0) = 0.2;

void vertex() {
	VERTEX.y += sin(VERTEX.x * 4.0 + TIME * 2.0) * wave_amplitude;
	# Recalcular normal aproximada (o aceptar la normal "lisa"):
	# (para correctness, desplazar la normal con la derivada)
}
```

### 5. Por instancia: N enemigos, 1 material, highlight propio

```gdscript
# Material: ShaderMaterial compartido por todos los enemigos.
# Shader: instance uniform float highlight_amount (ver *Implementación mínima*).
for enemy in enemies:
	enemy.mesh.set_instance_shader_parameter("highlight_amount", 0.6)
```

## Ejemplo práctico

**Juego**: arena 3D con 40 enemigos + mundo con niebla.

1. **Highlight**: material compartido `enemy_material` (ShaderMaterial con `instance uniform highlight_amount`); al pasar el cursor, `set_instance_shader_parameter` en el enemigo → 1 draw call menos que con materiales duplicados (verificado: el instancing automático de Forward+ agrupa mismo mesh+material).
2. **Disolución**: al morir, `set_shader_parameter("dissolve_progress", 1.0)` animado por `Tween`; el borde de `EMISSION` es HDR (visión bloom).
3. **Viento**: `global uniform wind_params` seteado por un `WindSystem` autoload (1 llamada por frame, sin coste); el foliage (grass/trees) lo consume en `vertex()`.
4. **Niebla en partículas**: `render_mode fog_disabled` en las partículas de `blend_add` (no las tiñe la niebla; docs: "useful for blend_add materials like particles").
5. **Debug**: `wireframe` render mode (o `set_debug_generate_wireframes` en Compatibility) para ver la geometría del disolve.

## Integración

- **Materiales**: `StandardMaterial3D` (PBR sin código) vs `ShaderMaterial` (custom): el mismo `GeometryInstance3D` acepta ambos; `material_override` (reemplazar) / `material_overlay` (encimar).
- **Código**: `set_shader_parameter` (por material) / `set_instance_shader_parameter` (por nodo) / `RenderingServer.global_shader_parameter_set` (por mundo).
- **Personaje**: `godot-shader-spatial` para el aspecto; `godot-characterbody3d`/`controller` no dependen del shader (el shader no mueve al body: solo `VERTEX` visual; si el body debe moverse, es root motion — `godot-animationtree`).
- **Rendimiento**: `godot-rendering-performance` (fragment cost = píxeles; overdraw de transparencia).

## Errores frecuentes

1. **Textura de color sin `source_color`** → se ve **lavada** (sRGB no convertida). Obligatorio en Forward+/Mobile (verificado). Solución: `uniform sampler2D albedo : source_color;`.
2. **Usar `SCREEN_TEXTURE`/`DEPTH_TEXTURE` (3.x)** → **eliminados en Godot 4** (verificado). → `hint_screen_texture`/`hint_depth_texture`.
3. **`get_spatial_lighting()` (3.x)** → no existe en 4.7 (sección *Spatial functions* eliminada; verificado). → `light()` propio.
4. **Escribir `ALPHA` "por si acaso"** → el material pasa al **pipeline transparente** (sin sombras, sin screen-space reflections, sorting issues). No escribir `ALPHA` si es opaco.
5. **`light()` con `=` en vez de `+=`** → la última luz pisa a las anteriores. El ejemplo oficial usa `+=`.
6. **`light()` esperando que corra con `vertex_lighting`** → **no se ejecuta** (warning oficial; default en móvil con Force Vertex Shading).
7. **`textureSize()` en el fragment** → "always performs a full texture read" (coste; evitar; pasar por uniform).
8. **`COLOR` de vértice para datos de alta precisión** → limitado a 8 bits/canal (256 niveles, verificado) → `CUSTOM0`–`CUSTOM3` o textura.
9. **Lerps de shader por `TIME` para algo que debe ser determinista** → `TIME` **rolloa cada 3600 s** y respeta `time_scale` (verificado): para "tiempo desde el arranque sin rolleo", uniform global actualizado por frame.
10. **`discard` "para ahorrar" en una superficie grande** → desactiva el depth prepass (coste oficial; a veces es más lento que renderizar). Medir (regla de rendimiento).
11. **Esperar que `global uniform` exista si no está en Shader Globals** → el shader **no compila** si no existe en Project Settings al guardarlo.
12. **Leer `global_shader_parameter_get` en bucle** → gran penalización de sync (warning oficial) → guardar en autoload al setear.
13. **Más de 16 `instance uniform`** → límite práctico (verificado) → agrupar en menos (vec4) o pasar por material normal.
14. **`instance uniform` con textura/array** → no soportado (verificado) → array como uniform normal + índice instance.
15. **Shader que compila en el editor pero falla en Compatibility** → `Coarse/Fine` de derivadas no disponibles en Compatibility (verificado); `hint_normal_roughness_texture` solo Forward+.
16. **`POSITION` escrito en un branch y no en otros** → "if written to on any branch… you are responsible for ensuring that it always has an acceptable value" (verificado): si escribes `POSITION`, escríbelo en todos los paths.

## Anti-patrones

- **`StandardMaterial3D` duplicado por enemigo** (duplicar el material para un color) → 1 material compartido + `instance uniform` (o `material_overlay`). Cada material nuevo = draw calls/state changes nuevos (ver `godot-rendering-performance`).
- **Un shader de 400 líneas "hace todo"** (highlight + disolución + viento + damage flash) → separar por efecto (materiales por capa con `material_overlay`, o props conmutables); shaders pequeños = compilación rápida + reutilizables.
- **Cálculo caro por fragment "porque es simple"** (ej. `textureGather` ×5 por píxel) → el fragment cost es **por píxel**: un cálculo ×1080p×1920p. Medir el FPS con/without (regla de rendimiento).
- **Uniforms gigantes (miles de floats)** → límite 4096 `vec4` desktop / 1024 móvil (verificado); para datos grandes, textura (el contenido no cuenta, solo el sampler).
- **Polling de `global_shader_parameter_get`** (arriba, #12).
- **Animar uniforms por `Tween` en 60 Hz para un efecto que el shader puede hacer solo con `TIME`** → si el efecto es temporal y continuo, hazlo en el shader (`TIME`); los Tweens de uniforms son para transiciones controladas por juego (disolve, highlight in/out).
- **Depender de `TEXTURE`/`TEXTURE_SCREEN` sin verificar la versión** → las vías explícitas (`uniform sampler2D : source_color/hint_screen_texture`) son las robustes (ver *Implementación recomendada*).

## Performance

**Regla (spec §60): MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN.**

- **Fragment cost = por píxel en pantalla** (el coste dominante de un shader 3D). Un fragment "medio" en un personaje a pantalla completa se paga en millones de píxeles. El vertex cost es bajo en desktop (ver `godot-rendering-performance`).
- **Overdraw**: transparencias/`blend_add`/outline se suman (se renderizan todas, sin Z-buffer). Doble el overdraw ≈ doble el fragment cost.
- **Materiales**: pocos materiales distintos = menos state changes (el renderer agrupa por material/shader; `StandardMaterial3D` reutiliza shaders con la misma configuración — verificado en `gpu_optimization.html`).
- **`textureSize()`**: lectura completa (arriba #7).
- **`discard`**: mata el depth prepass (arriba #10).
- **Mobile**: `vertex_lighting` / `Force Vertex Shading` (arriba #6); evitar precision innecesaria (`lowp/mediump` en fragment; ver shading language: "often useful in the fragment processor").
- **Medición**: `Performance.get_monitor(Performance.TIME_FPS)` y `RENDER_TOTAL_DRAW_CALLS_IN_FRAME` antes/después del cambio de shader (ver `godot-rendering-performance`). El truco del "ventana pequeña" (V-Sync off, comparar FPS con ventana grande vs pequeña) detecta fill-rate limit (verificado en `gpu_optimization.html`).

## Debugging

### "El shader no compila"

```text
1. ¿El global uniform existe en Shader Globals? (error #11)
2. ¿Escribes un built-in que no existe en esa función? (p. ej. SCREEN_UV en vertex()
   no existe; VERTEX en light() tampoco)
3. ¿POSITION escrito en un branch y no en otros? (error #16)
4. ¿Compatibility usando Coarse/Fine de derivadas? (error #15)
5. Leer el mensaje exacto del editor (línea + columna): el editor marca la línea.
```

### "Se ve lavado / los colores no son los del asset"

```text
1. ¿Falta source_color?  (error #1, el más común)
2. ¿El asset se importó como sRGB?  (Texture2D import settings: "sRGB" on para color)
3. ¿OUTPUT_IS_SRGB? (Compatibility=true): si el shader asume linear y el renderer
   es sRGB, compensar (o no, según el diseño).
```

### "El material es transparente / no proyecta sombras"

```text
1. ¿Estás escribiendo ALPHA en algún branch? (error #4) → quitar (o usar
   ALPHA_SCISSOR_THRESHOLD para alpha test).
2. ¿depth_draw_never? (no escribe depth → no recorta a lo que hay detrás).
```

### "Las luces no suman / solo se ve una"

```text
1. ¿light() con = en vez de +=? (error #5)
2. ¿light() no se ejecuta? (vertex_lighting / Force Vertex Shading — error #6)
3. ¿SPECULAR_AMOUNT / LIGHT_IS_DIRECTIONAL usados bien? (ver built-ins de light)
```

### "El viento/noise 'salta' cada hora"

```text
1. TIME rola cada 3600 s (verificado) → si el efecto debe ser continuo en el tiempo,
   usar uniform global propio (Time.get_ticks_msec() / 1000.0) en vez de TIME.
```

## Compatibilidad

- **Verificado: 4.7 (stable)** — render modes, built-ins, hints, funciones, instance/global uniforms, límites: verificados en la docs stable (4.7).
- **4.0 ↔ 4.7**: el lenguaje es estable en los cimientos; las diferencias (p. ej. `hint_normal_roughness_texture` Forward+ only, `Coarse/Fine` no en Compatibility) se marcan arriba por feature.
- **3 → 4 (trampas de port, verificadas)**:
  - `SCREEN_TEXTURE`/`DEPTH_TEXTURE` eliminados → `hint_screen_texture`/`hint_depth_texture`.
  - `get_spatial_lighting`/`sample_skybox_*` eliminados → `light()` / `RADIANCE`/`IRRADIANCE`.
  - El built-in `TEXTURE` (albedo) y `TEXTURE_SCREEN`: **verificar nombre exacto por versión** (API no verificada en esta revisión) — la vía `uniform` es la robusta.
  - Lenguaje: GLSL ES 3.0-ish (4.x) vs 1.0 (3.x): sintaxis moderna (arrays, structs, swizzles, `in/out/inout`).

## Dependencias

| Skill | Relación |
|---|---|
| (ninguna base) | Skill atómica de aspecto |
| `godot-rendering-performance` | Coste de fragment/overdraw/materiales |
| `godot-third-person-character` | Compuesta que usa este shader (highlight) (Lote 0) |
| `godot-shader-canvasitem` (pendiente) | El equivalente 2D |
| `godot-standardmaterial3d` (pendiente) | El PBR sin código (alternativa a shader para la mayoría) |

## Skills relacionadas

- `godot-recipe-dissolve-effect` — receta completa (pendiente; germen en *Implementación recomendada* #1)
- `godot-recipe-screen-space-effect-3d` — pendiente
- `godot-error-shader-not-compiling` — pendiente
- `godot-error-texture-washed-out` — pendiente (el `source_color` es la causa nº1)

## Referencias oficiales

Verificadas en la rama `stable` (4.7) el 2026-09-15:

- Spatial shaders (render modes, stencil, built-in globales/vertex/fragment/light): https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/spatial_shader.html
- Shading language (uniforms, hints, source_color, global uniforms, per-instance uniforms, uniform limits, set_shader_parameter, varyings, interpolation, discard): https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/shading_language.html
- Built-in shader functions (texture/textureLod/textureGrad/textureProj/texelFetch/textureGather/textureSize/derivadas): https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/shader_functions.html
- ShaderMaterial (set_shader_parameter): https://docs.godotengine.org/en/stable/classes/class_shadermaterial.html
- GeometryInstance3D (set_instance_shader_parameter, material_override/overlay, transparency): https://docs.godotengine.org/en/stable/classes/class_geometryinstance3d.html
- RenderingServer (global_shader_parameter_set/add/remove/get): https://docs.godotengine.org/en/stable/classes/class_renderingserver.html
- 3D rendering limitations (transparency sorting): https://docs.godotengine.org/en/stable/tutorials/3d/3d_rendering_limitations.html
- Tutorial "Using Shaders": https://docs.godotengine.org/en/stable/tutorials/shaders/using_shaders.html
- GPU optimization (materiales, fill rate, textura): https://docs.godotengine.org/en/stable/tutorials/performance/gpu_optimization.html
