# src/Layers/xrRender/light.cpp

> One light source: its shape in space, the bounding volume the visibility database indexes it by, the transform that turns a unit primitive into its volume, and the split of a shadowing point light into six cone lights.

**Needs** — [`light.h`](light.h.md) · [`light_gi.h`](light_gi.h.md) · [`light_smapvis.h`](light_smapvis.h.md) · [`xrCDB/ISpatial.h`](../../xrCDB/ISpatial.h.md) · [`Light_Package.h`](Light_Package.h.md) · [`Shader.h`](Shader.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`light.h`](light.h.md)
**Tier floor** — T1: the light's state is a densely packed record that is copied, indexed and iterated once per frame per light, and its shadow-map visibility is one such record per render context.

## Purpose

A light is the unit of work in the deferred renderer: every frame the visibility database yields the lights touching the view, and each becomes one or more draws of a volume into the accumulation buffer. This file owns everything about a light that is *geometry* — where it is, how big a sphere encloses it, what transform maps a unit sphere or unit cone onto it — plus the setters that keep the spatial index in step with those.

It does not own how a light is shaded, how its shadow map is rendered, or how it is accumulated; those live in the deferred path. What it does own, and what a rebuilder must get exactly right, is the **enclosing sphere per light type** and the **six-way split of a shadowing point light**, because both decide how much of the frame's work happens.

## State

```text
ENUM LightType { point, spot, omni_part, reflected }

RECORD Light
  type                 : LightType
  is_static            : bool      # authored into the level rather than spawned
  is_active            : bool      # registered with the spatial database
  casts_shadow         : bool
  is_volumetric        : bool
  is_hud               : bool      # belongs to the first-person weapon layer
  position             : vector3
  direction            : vector3   # normalised
  right                : vector3   # optional roll reference; zero means "pick one"
  range                : real
  virtual_size         : real      # the emitter's own radius, for soft shadows
  cone                 : real      # full angle in radians; spot and omni_part only
  colour               : colour

  occlusion_volume     : VisData   # sphere + box handed to the occlusion tester
  last_render_frame    : int

  volumetric_quality   : real
  volumetric_intensity : real
  volumetric_distance  : real

  # --- deferred path only -------------------------------------------------
  falloff              : real      # chosen so intensity reaches zero exactly at range
  attenuation_constant : real
  attenuation_linear   : real
  attenuation_quadratic: real

  omni_parts           : list<Light> of exactly 6, or empty
  indirect             : list<IndirectBounce>
  indirect_photon_count: int

  shadow_visibility    : list<ShadowCasterVis> one per render context

  spot_material        : Material
  point_material       : Material
  volumetric_material  : Material
  # plus one of each per multisample count, when multisampling is on

  transform_frame      : int       # the frame `transform` was last rebuilt
  transform            : matrix4

  visibility :
    frame_to_test      : int       # the frame the next occlusion test is due
    query_id           : int
    query_order         : int
    visible            : bool
    pending            : bool
    shadow_map_slot    : int

  projection :  ONE OF
    directional : per sun cascade { combined : matrix4, min/max x and y in texels,
                                    has_translucent_casters : bool }
    point       : { world, view, projection, combined : matrix4 }
    spot        : { view, projection, combined : matrix4, map_size, map_x, map_y,
                    has_translucent_casters : bool }
```

Invariants:

- `is_active` is exactly "registered with the spatial database". Every transition goes through the activation setter; nothing else may register or unregister, because the database's bucket for this light must be updated before the enclosing sphere moves and after it lands.
- The enclosing sphere in the spatial record is *derived*: any change to position, range, direction or cone must recompute it and re-file the light. This is the invariant the whole file exists to maintain.
- `cone` is a full angle in radians and is capped: no light may exceed a hundred and twenty degrees. Beyond that a cone is better expressed as a point light, and the shadow projection degenerates.
- `omni_parts` is either empty or exactly six, and the six are owned by this light — created lazily the first time a shadowing point light is exported, destroyed with it.
- `transform_frame` makes the transform computation idempotent within a frame: several passes ask for the same light's volume transform, and it is rebuilt at most once.

## Construction and defaults

A new light is a *point* light, inactive, unshadowed, non-volumetric, with an eight-metre range, a sixty-degree cone, white colour, a virtual size of a tenth of a metre, and volumetric quality, intensity and distance all at one. Its position is a sentinel far below the world.

**Notes** — The sentinel position exists for one reason: a debug build warns when a light is activated while still at it, catching the common bug of registering a light before placing it. A rebuild can replace this with an explicit "placed" flag; what must survive is the check, because an unplaced active light is invisible and silent and costs a full volume draw every frame.

Each of the per-context shadow-visibility records is stamped with its own context index at construction, because each later reaches back into *its* render context to mark visuals — see [`light_smapvis.cpp`](light_smapvis.cpp.md).

## Teardown

Destroys the six cone children if they exist, deactivates (which unregisters from the spatial database), and then — this is the part that is easy to miss — **scrubs itself out of the renderer's list of last frame's lights**. That list is a raw carry-over from the previous frame used to decide which lights kept their shadow-map slots; a destroyed light left in it would be dereferenced next frame. A rebuild with a different shadow-slot scheme must still guarantee that destroying a light removes every reference the frame pipeline holds.

## `set_active`

**Contract** — Register or unregister with the spatial database. Idempotent: setting the current value does nothing. Registration files the light and then immediately recomputes its enclosing sphere; deactivation recomputes the sphere *first* and then unregisters, so the database never holds a stale bucket for a light that is about to leave.

## `set_position`, `set_range`, `set_rotation`, `set_cone`

**Contract** — Change one aspect of the light's placement and re-file it in the spatial database — but only when the change is large enough to matter.

The thresholds are the decision here, and each is different:

- **Position** — refiled when it moves more than a fixed small epsilon. Any real motion refiles.
- **Range** — refiled only when it changes by more than **ten percent of the current range** (with a small absolute floor). Range is animated continuously by flickering lights and by the weather system; refiling on every flicker would dominate the database's cost, and a ten-percent error in an enclosing sphere is harmless because the sphere is conservative anyway.
- **Rotation** — the direction and roll vectors are normalised, and the light is refiled only when the direction actually turned. Pure roll changes the volume's orientation but not its enclosing sphere, so they cost nothing.
- **Cone** — refiled on any change, after asserting the hundred-and-twenty-degree cap. The assertion's message names the likely cause: an angle passed in degrees where radians were expected.

## `spatial_move` — the enclosing sphere

**Contract** — Recompute the sphere the spatial database indexes this light by, re-file it, regenerate the indirect bounce set if the light is active, and invalidate every render context's shadow-caster visibility cache. This is the single choke point through which every placement change passes.

The sphere per type:

```text
FUNCTION enclosing_sphere() -> sphere
  CASE type OF
    point, reflected:
      # trivial: the light reaches `range` in every direction
      RETURN sphere(position, range)

    spot:
      # the minimal sphere enclosing a cone of half-angle `cone/2` and slant
      # length `range`. Two cases, and which one applies is decided by whether
      # the cone is wider than a right angle.
      IF cone >= quarter_turn
        # obtuse: the enclosing sphere is centred on the cone's base and its
        # radius is the base's own radius
        centre = position + direction * range
        radius = range * tan(cone / 2)
      ELSE
        # acute: the sphere passes through the apex and the whole base rim;
        # its centre lies on the axis at the circumradius
        radius = range / (2 * cos(cone / 2)^2)
        centre = position + direction * radius
      RETURN sphere(centre, radius)

    omni_part:
      # one face of a cube map: a right-angled cone of slant length `range`.
      # The tight enclosing sphere is centred at range/sqrt(2) along the axis
      # with that same radius — which is what the acute case above reduces to
      # at exactly a right angle, written out directly because this type is
      # created six at a time and the cost matters.
      radius = range * (1 / sqrt(2))
      RETURN sphere(position + direction * radius, radius)
```

**Invariants** — The sphere must *contain* the lit volume; a too-small sphere culls a light that should be visible, which shows as a light popping off at a sector boundary. The spot formulae are exact, not conservative, so a rebuild must reproduce both branches — using the obtuse formula for an acute cone yields a sphere that is too small.

Invalidating the shadow-caster caches is not optional: those caches remember which visuals were found to cast no shadow from this light, and moving the light makes that finding worthless.

## `occlusion_volume`

**Contract** — Publish the light's enclosing sphere and its axis-aligned box to the occlusion tester, deriving the box from the sphere on each call. The box is the sphere's bounding cube, not a tight box around the cone — the occlusion test wants a cheap conservative volume and a tight cone box would be neither.

## `spatial_sector_point`

**Contract** — The point used to decide which sector the light belongs to: the light's *position*, not its enclosing sphere's centre. For a spot light those differ substantially, and the position is right — a lamp bolted to a wall belongs to the room it is in, not to the room its beam reaches.

## `set_texture`

**Contract** — Build the light's materials, or with no name, release them and fall back to the untextured defaults. A named texture produces a projected-texture spot material, a volumetric material, and — when multisampling is enabled — one variant per sample count.

**Invariants** — The material names are composed from a frozen prefix and the texture's name, and the compositions must be reproduced exactly because the shipped material files are found by those names.

**Notes** — Only the *shadowed spot* light actually uses the projected texture; the other types build materials that ignore it. This is recorded as a known gap in the original. A rebuild that wants projected textures on point lights must add the cube-map path itself.

## `export_to_package`

**Contract** — Add this light to the frame's light package, in the bucket the deferred path will draw it from. This is where the six-way split happens.

```text
FUNCTION export_to_package(package)
  IF NOT casts_shadow
    CASE type OF
      point: package.unshadowed_point.append(this)
      spot : package.unshadowed_spot.append(this)
    RETURN

  CASE type OF
    spot:
      package.shadowed.append(this)     # one shadow map, one draw

    point:
      # A shadowing point light needs a shadow map in every direction. Rather
      # than a cube shadow map, the light is split into six right-angled cone
      # lights, one per cube face, each of which the rest of the pipeline can
      # treat as an ordinary shadowed spot. This is why `omni_part` exists as
      # a type at all.
      IF omni_parts is empty THEN create six child lights
      FOR EACH face IN the six axis directions
        child = omni_parts[face]
        child.type          = omni_part
        child.casts_shadow  = true
        child.position      = position
        child.direction     = face.axis
        child.right         = cross(face.up, face.axis)
        child.cone          = quarter_turn        # exactly one cube face
        child.range         = range
        child.virtual_size  = virtual_size
        child.colour        = colour
        child.sector        = sector              # inherited, see Notes
        child.materials     = this.materials      # shared, not rebuilt
        child.volumetric_*  = this.volumetric_*
        package.shadowed.append(child)
```

**Invariants**

- The six axis directions and their matching up-vectors are a frozen table: `+x`, `-x`, `+y`, `-y`, `+z`, `-z`, with `+y` up for the four horizontal faces and `∓z` up for the two vertical ones. This is the standard cube-map face convention and the same table is reused by the pixel-coverage tool ([`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md)); changing it swaps faces and rotates two of them.
- The children are created once and reused across frames. They are not re-registered with the spatial database — they are never queried, only drawn — which is why they inherit the parent's sector rather than detecting their own.
- The children share the parent's materials by reference. A rebuild must not have the children release them.

**Notes** — The sector inheritance is flagged in the original as possibly wrong. It is: a cone reaching through a portal into the next sector is drawn with the parent's sector, so the deferred path's per-sector scissoring can clip it. In practice point lights are small enough that it does not show. A rebuild that fixes it must detect the sector per child, at a cost of six ray queries per shadowing point light per placement change.

## `set_attenuation_params`

**Contract** — Set the three attenuation coefficients and the falloff together. They are set as a group because they are only meaningful together: the falloff is chosen so that the attenuation curve reaches zero exactly at `range`, and setting one without the others leaves a light that either cuts off visibly before its volume ends or is still bright at its boundary.

## `level_of_detail`

**Contract** — A factor in zero to one saying how strongly this light's shadow should contribute, from its screen-space size. Unshadowed lights always return one — they have no shadow to fade.

```text
FUNCTION level_of_detail() -> real
  IF NOT casts_shadow THEN RETURN 1
  distance_sq = camera_position.distance_squared_to(enclosing_sphere.centre)
  coverage    = shadow_fade_scale * enclosing_sphere.radius / distance_sq
  RETURN sqrt(clamp((coverage - fade_end) / (fade_start - fade_end), 0, 1))
```

**Invariants** — The coverage measure is radius over distance *squared*, not over distance. It is not an angle; it is the same screen-space-area heuristic the whole renderer sorts by, and it must match the one the scene graph uses ([`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md)) or lights and geometry will fade at inconsistent distances. The square root at the end makes the fade perceptually linear rather than quadratic.

A light whose level of detail falls to zero is dropped entirely by the frame's light collection, so this function is also the shadowing light's far cull.
