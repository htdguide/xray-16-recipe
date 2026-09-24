# src/Layers/xrRender/TextureDescrManager.cpp

> The texture-description database: the side table that says which detail texture, bump map, surface material and parallax setting belong to a base texture — the convention the shipped art is authored against.

**Needs** — [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`ETextureParams.h`](ETextureParams.h.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`r_constants.h`](r_constants.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [Seam: Threads](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — reached through its declarations in [`TextureDescrManager.h`](TextureDescrManager.h.md); callers name that, not this file.
**Tier floor** — T1: the `.thm` half reads a chunked binary record written by the art tools, and the constant binder it hands out is written straight into a shader constant register every frame.

## Purpose

Nothing in a material says "this wall also gets a close-up gravel detail layer, and its normals come from `wall_bump`". That is said *about the texture*, in a database loaded once at startup and consulted by the material compiler while it builds a pass (see [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md), step 1 of `compile`). This file is that database.

The indirection matters: it is what lets one material template — "lightmap times base" — produce a detailed surface for one texture and a flat one for another, without the level's material assignment knowing anything about detail textures. **A rebuild that resolves detail textures from the material instead of from the base texture will render the shipped levels wrong.**

Two sources feed it, and a texture may be described by either:

- **`textures.ltx`** — one text file per root, with all associations on one line each. This is the form the released games ship.
- **`*.thm`** — one small chunked binary file per texture, written by the art tools. This is the authoring form; it carries strictly more (the surface material, the parallax flag) and it wins, because it is loaded second.

## State

```text
RECORD TextureAssociation          # "this base texture has a detail layer"
  detail_name   : text             # the detail texture's asset name
  diffuse_detail: bool             # the layer modulates colour
  bump_detail   : bool             # the layer perturbs the normal

RECORD TextureSpecification        # "this is what kind of surface it is"
  bump_name          : text        # the normal/height map's asset name, or empty
  material           : real        # surface-material index plus a sub-index weight
  use_steep_parallax : bool

RECORD TextureDescription
  association   : optional<TextureAssociation>
  specification : optional<TextureSpecification>

RECORD Database
  by_texture : map<text, TextureDescription>
  scalers    : map<text, DetailScaler>     # one per texture that has a detail layer
```

**Invariants**

- The key is the texture's asset name **with its extension removed** — `.tga`, `.thm`, `.dds`, `.bmp` and `.ogm` are all stripped, case-insensitively, and nothing else is. Lookups elsewhere in the renderer pass names in that same extensionless form, so a rebuild that keys on the full filename will match nothing.
- The two halves of a description are independent. A texture may have an association and no specification, or the reverse; a query for the missing half returns that half's default without failing.
- `scalers` is keyed by the *base* texture, not by the detail texture, because two base textures may share a detail texture at different tiling densities.
- A scaler object **outlives the database's entries**: unloading a level clears `by_texture` but deliberately leaves `scalers` alive, because compiled material passes hold references to the scaler as their per-frame constant binder. The scalers die only when the renderer does.
- When a texture is described twice (the same name in the level's file and the global one), the later load replaces the earlier description **but reuses the existing scaler object**, mutating its scale in place. That is not an optimization: replacing the object would leave already-compiled passes bound to the stale one.

## `Database.load()`

**Contract** — fills the database from four sources, in this order, and this order is the precedence rule. Blocks; reads many files; parallelizes within each source. Called once when the renderer comes up and again after a level changes.

```text
FUNCTION load()
  load_ltx(root = game_textures)   # global associations
  load_ltx(root = level)           # this level may override any of them
  load_thm(root = game_textures)   # authoring records win over the ltx form
  load_thm(root = level)
```

**Notes** — The `.thm` pass is last because it is the richer record: only it carries the surface material and the parallax flag, so letting a terse `.ltx` line overwrite it would silently drop those. A diagnostic switch makes the whole load single-threaded and logs every file, because when a `.thm` is malformed the useful information is *which one*, and a parallel load scrambles that.

## `load_ltx(root)` — the text form

**Contract** — reads `textures.ltx` under the given root if it exists, and consumes two of its sections. Missing file is normal and silent. A malformed line is fatal, not skipped.

```text
FUNCTION load_ltx(root)
  file := root + "textures.ltx"
  IF NOT exists(file) THEN RETURN
  ini := parse_config(file)

  FOR EACH (texture, value) IN ini.section("association") IN PARALLEL
      # value looks like:  detail\detail_grnd_asphalt, 0.25, usage[diffuse]
      (detail_name, scale) := parse(value, "<text up to a comma>, <real>")
      IF parse failed THEN FAIL WITH "bad texture association"

      entry := by_texture[texture]          # created if absent, under a lock
      entry.association := new TextureAssociation(detail_name)

      scaler := scalers[texture]
      IF scaler exists THEN scaler.scale := scale ELSE scalers[texture] := new DetailScaler(scale)

      # the usage tag is matched as a SUBSTRING of the whole value, not parsed
      IF value contains "usage[diffuse_or_bump]" THEN set both flags
      ELSE IF value contains "usage[bump]"       THEN bump_detail   := true
      ELSE IF value contains "usage[diffuse]"    THEN diffuse_detail := true
      # no usage tag at all leaves BOTH flags false: the detail texture is
      # named but never applied. This is how a texture is given a detail
      # partner for the authoring tools without the game using it.

  FOR EACH (texture, value) IN ini.section("specification") IN PARALLEL
      # value looks like:  bump_mode[use:wall_bump], material[1.0]
      (mode, material) := parse(value, "bump_mode[<text>], material[<real>]")
      IF parse failed THEN FAIL WITH "bad texture specification"
      entry := by_texture[texture]
      entry.specification := new TextureSpecification(material)
      IF mode starts with "use:" THEN entry.specification.bump_name := rest of mode
```

**Invariants** — the association parse takes the detail name as *everything up to the first comma*, so a detail texture name may not contain one. Both sections are parsed leniently in their tail: extra comma-separated fields after the ones named are ignored, which is exactly how the `usage[...]` tag rides along on an association line.

**Notes** — The text form has **no way to express steep parallax and no way to express a surface material other than through `material[...]`**; the parallax flag is not even defaulted here, so a texture described only by `.ltx` inherits whatever the record was constructed with. A rebuild should default it to false explicitly.

The known defect, stated plainly because the shipped data exercises it: the `usage[diffuse_or_bump]` branch combines the two usage flags with a bitwise-or **of their ordinal positions** (0 and 1) rather than of their bit masks, so it sets only the bump flag. In the running engine, `usage[diffuse_or_bump]` therefore means exactly `usage[bump]`. Whether to reproduce this is a judgement call: the shipped art was tuned against the engine's behaviour, not against the tag's name, so reproducing the *behaviour* is the safer reading.

## `load_thm(root)` — the authoring form

**Contract** — reads every `*.thm` under the given root. Each file is one texture's full authoring record. A file that cannot be opened, or that lacks the expected chunk, is fatal — a missing `.thm` is an installation problem worth stopping for, most often a case-sensitivity mismatch on a filesystem the data was not authored on.

```text
FUNCTION load_thm(root)
  FOR EACH file IN list(root, "*.thm") IN PARALLEL
      key    := strip_extension(file.name)
      params := read_texture_params(file)      # chunked record; see ETextureParams

      # only these three kinds describe a surface; the rest (cube maps, terrain
      # masks, UI images) carry no detail or material information
      IF params.type NOT IN { image, terrain, normal_map } THEN CONTINUE

      IF params.detail_name is non-empty
         AND params.flags has (diffuse_detail OR bump_detail) THEN
          entry.association := new TextureAssociation(params.detail_name)
          entry.association.diffuse_detail := params.flags has diffuse_detail
          entry.association.bump_detail    := params.flags has bump_detail
          reuse-or-create scalers[key] with scale = params.detail_scale

      entry.specification := new TextureSpecification()
      # the surface material is an INTEGER index plus a fractional weight,
      # collapsed into one real: the whole part selects the material and the
      # fraction blends toward the next one. The shader samples a 3D lookup
      # table with this value on the third axis.
      entry.specification.material := real(params.material) + params.material_weight
      entry.specification.use_steep_parallax := false
      IF params.bump_mode == use          THEN bump_name := params.bump_name
      IF params.bump_mode == use_parallax THEN bump_name := params.bump_name
                                               use_steep_parallax := true
```

**Notes** — Unlike the text form, a `.thm` always replaces the specification, even when it carries nothing interesting; that is what makes the load order a precedence rule rather than a merge. The association, by contrast, is only replaced when the record actually names a detail texture, so a `.thm` with no detail layer does not erase an association the `.ltx` supplied.

## `get_detail(texture) -> optional<(detail_name, scaler)>`

**Contract** — the query the material compiler makes before anything else. Answers only when the texture has an association; the scaler may be absent even then (a description loaded without one), and the caller must tolerate that.

**Notes** — The scaler returned is not a value, it is a **binder**: an object the compiled pass keeps and calls once per frame to write four numbers into a shader constant — the tiling scale on three axes and `1 / detail_range` on the fourth. The detail range is a single engine-wide number, 50 metres, and it is the distance at which the detail layer has faded completely out. Passing its reciprocal rather than the range itself is what lets the shader fade with a multiply instead of a divide; the constant is the *only* place the range appears, so changing it is a one-line change with a global effect.

## `get_usage(texture) -> (diffuse, bump)`

**Contract** — reports how the detail layer applies. **It does not write its outputs when the texture is undescribed** — the caller's values survive. That is load-bearing: the compiler initializes both to false and relies on this to mean "no detail".

## `get_bump_name(texture) -> text`

**Contract** — the normal-map asset paired with this base texture, or the empty string. The empty string is the normal answer for an unbumped surface and is not an error.

**Notes** — The pairing by name is one half of the engine's bump-map convention; the other half is a *second* texture derived by appending a suffix to the bump name (see the blenders, which sample `<bump>` and `<bump>#`). The suffixed variant carries the height/parallax channels. Both files ship; a rebuild must keep the suffix spelling exactly.

## `get_material(texture) -> real`

**Contract** — the surface-material coordinate, defaulting to **1.0** when the texture is undescribed. The default is not zero: zero is a real material in the lookup table, and defaulting to it would silently reclassify every undescribed surface.

## `uses_steep_parallax(texture) -> bool`

**Contract** — whether this base texture's bump map carries a usable height channel. False when undescribed. Only consulted for templates that answered yes to the parallax capability query.

## `unload()` and teardown

**Contract** — `unload` drops every description and leaves the scalers. Teardown drops the scalers. The asymmetry is the ownership rule stated in **State**: a compiled pass outlives a level's description table but not the renderer.
