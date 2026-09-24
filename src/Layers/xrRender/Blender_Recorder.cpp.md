# src/Layers/xrRender/Blender_Recorder.cpp

> The material compiler: the scratch context a blender emits passes into, and the rules that turn "$base0" plus a detail-texture convention into a concrete, deduplicated, immutable pass list.

**Needs** — [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`Blender.h`](Blender.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`Shader.h`](Shader.h.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`tss.h`](tss.h.md) · [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`Blender_Recorder.h`](Blender_Recorder.h.md); callers name that, not this file.
**Tier floor** — T1: the state it accumulates is a flat table of device-state tokens whose numeric values are the graphics API's own, and it hands the resource manager byte-comparable keys built from them.

## Purpose

A blender describes *a kind of surface*; a material instance supplies *this surface's* textures, matrices and constants. This file is the object that joins the two. It is a **recorder**: the blender calls a sequence of "begin a pass / set the depth mode / bind this sampler to that texture / end the pass" methods on it, and what accumulates is a pass list. When the blender is done, the recorder hands every accumulated piece to the resource manager, which returns a *shared* immutable copy — so two materials that compile to identical state end up pointing at one state object, one texture list, one constant table.

Two compilers live in one class, because the two renderer generations record differently and share the front half:

- **the fixed-function recorder** (`PassBegin` … `StageBegin`/`StageEnd` … `PassEnd`) builds a pass out of numbered *texture stages*, each with a colour and alpha combine operation. This is the oldest renderer's path.
- **the programmable recorder** (`r_Pass` … `r_Sampler` … `r_End`, in [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md)) builds a pass out of named GPU programs, and resolves texture bindings *by looking the sampler's name up in the compiled programs' reflection data*.

Both end by calling the shared constant-binding step in [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md).

## State

```text
RECORD CompileContext            # lives only for the duration of one material's compilation

  # --- supplied by the caller, before compile ---
  textures    : list<text>       # the material instance's texture names, positional
  constants   : list<text>       # its constant names, positional
  matrices    : list<text>       # its matrix (UV animation) names, positional
  blender     : Blender          # the template being compiled
  target      : ShaderElement    # where the finished passes go
  element     : int              # which element of the material is being built (see below)
  want_detail : bool             # caller's request; may be denied

  # --- decided during compile ---
  detail_texture : optional<text>        # resolved detail texture, or absent
  detail_scaler  : optional<ConstantSetup> # its tiling parameters, as a per-frame binder
  detail_diffuse : bool                  # the detail layer modulates colour
  detail_bump    : bool                  # the detail layer perturbs the normal
  use_steep_parallax : bool

  # --- accumulator for the pass currently being recorded ---
  state_recorder  : StateSimulator   # flat (token, value) table of device state
  pass_textures   : list<(stage:int, texture)>
  pass_matrices   : list<matrix>
  pass_constants  : list<constant>
  constant_table  : ConstantTable    # merged reflection data of this pass's programs
  stage           : int              # fixed-function recorder's cursor
  destination     : Pass             # the pass under construction
```

**Invariants**

- The accumulators are cleared at the *start* of every pass, not the end. A blender that emits a second pass inherits nothing from the first except the target list.
- `stage` is only meaningful on the fixed-function path. On the programmable path the same field is reused to carry the last resolved sampler index, which is a genuine overload of one variable for two purposes and a rebuild should split it.
- A sampler index of "invalid" is a normal, expected answer, not an error: it means the compiled programs for this pass do not reference that name, so the binding is silently dropped. This is how one blender emits the same binding calls for several renderer generations and lets the shader source decide which land.

## Positional names — `$base0` … `$base7`, `$null`

**Contract** — wherever a texture, matrix or constant is named, the name may be a literal asset name *or* a positional reference into the material instance's own list. `"$baseN"` means "the N-th name the material instance supplied"; `"$null"` means none.

```text
FUNCTION resolve_positional(name) -> optional<int>
  IF name == "$null"   THEN RETURN none
  IF name matches "$base" followed by a digit 0..7 THEN RETURN that digit
  RETURN none          # any other name is a literal asset name
```

**Invariants** — a positional reference beyond the end of the instance's list is fatal, not clamped. The material library and the level's material assignment were authored together; a mismatch means the data is wrong and rendering it would produce silently wrong art.

**Notes** — The cap of eight is not enforced anywhere but the parser; it exists because no shipped material names more than a handful. The scheme is what makes a blender *a template*: the template says "sample the base texture, modulate by the lightmap", and the instance says which two files those are.

## `compile(target_element)` — the entry point

**Contract** — prepares the detail-texture and parallax decisions, then hands control to the blender's own `compile`, which calls back into this object to record passes. Mutates `target_element` by appending passes. Fatal on a positional reference the instance cannot satisfy. Not reentrant, not thread-safe: one of these exists per compilation and compilation is serialized.

```text
FUNCTION compile(target_element)
  target       := target_element
  state_recorder.reset()

  # 1. Which texture counts as "the base"? Everything below is keyed off it.
  base := none
  IF want_detail AND blender.can_be_detailed() THEN
      base := resolve_named_texture(blender.base_texture_name)
      # the detail layer is a property of the BASE TEXTURE, not of the material:
      # the texture-description database maps a base texture to its detail partner.
      IF NOT texture_descriptions.get_detail(base) -> (detail_texture, detail_scaler) THEN
          want_detail := false
  ELSE
      IF blender.can_use_steep_parallax() THEN
          base := resolve_named_texture(blender.base_texture_name)
      want_detail := false

  # 2. The oldest renderer can be told to skip detail entirely, as a quality setting.
  IF renderer_is_forward AND option_no_detail_textures THEN want_detail := false

  # 3. How does the detail layer apply — as colour, as normal, or both?
  detail_diffuse := false ; detail_bump := false
  IF want_detail THEN
      (detail_diffuse, detail_bump) := texture_descriptions.get_usage(base)
      IF NOT (advanced_pixel_pipeline AND option_detail_bump) THEN
          # the hardware or the user said no to detail bump: fold it into
          # diffuse rather than dropping the layer, so the surface still
          # gains its high-frequency variation
          detail_diffuse := detail_diffuse OR detail_bump
          detail_bump    := false

  use_steep_parallax := texture_descriptions.uses_steep_parallax(base)
                        AND blender.can_use_steep_parallax()

  blender.compile(self)
```

**Notes** — Step 1 is the heart of the **detail-texture convention** the shipped art depends on. A detail texture is never named by a material; it is named by an entry in the texture-description database keyed on the *base* texture. That indirection is why the base texture has to be resolved before anything else happens, and why a blender that cannot say which of its textures is the base cannot be detailed at all.

Step 3's fold-down is the decision worth copying: when the detail layer's bump half cannot be used, the engine does not discard it — it promotes it to a diffuse detail. Dropping it instead makes affected surfaces visibly flat next to unaffected ones.

## `set_params(priority, strict_back_front)`

**Contract** — writes the two sort knobs into the element being built. Asserts that strict back-to-front ordering is only requested at priority 2 or 3 (see [`Blender.cpp`](Blender.cpp.md) for why). In the authoring tools the violation is logged and the flag cleared instead of aborting, because an artist should get a warning, not a crash.

## Fixed-function pass recording

### `PassBegin` / `PassEnd`

**Contract** — bracket one pass. `PassBegin` clears every accumulator and writes a known default pipeline state, so a blender only has to state what differs from the default. `PassEnd` closes the last texture stage, interns every accumulated piece through the resource manager, and appends the resulting pass to the element.

```text
FUNCTION pass_begin()
  clear state_recorder, pass_textures, pass_matrices, pass_constants,
        constant_table ; stage := 0
  set_depth(test = true, write = true)
  set_blend(enabled = false, src = one, dst = zero, alpha_test = false, ref = 0)
  set_light_fog(light = false, fog = false)

FUNCTION pass_end()
  # a stage with no combine operation terminates the fixed-function chain
  state_recorder.set_stage(stage, colour_op, disable)
  state_recorder.set_stage(stage, alpha_op,  disable)

  # a pass always names both programs, even on the fixed-function path,
  # where "null" is a real, loadable, do-nothing program
  IF destination.vertex_program is none THEN destination.vertex_program := program("null")
  IF destination.pixel_program  is none THEN destination.pixel_program  := program("null")

  bind_standard_constants()              # see Blender_Recorder_StandartBinding

  destination.state     := resources.intern_state(state_recorder.table)
  destination.constants := resources.intern_constant_table(constant_table)
  destination.textures  := resources.intern_texture_list(pass_textures)
  destination.matrices  := resources.intern_matrix_list(pass_matrices)
  destination.consts    := resources.intern_constant_list(pass_constants)
  target.passes.append(resources.intern_pass(destination))
```

**Notes** — Every `intern_*` call is a *deduplicating* lookup: the resource manager compares the candidate against everything already made and returns the existing one on a match. That is what keeps the cost of thousands of material instances proportional to the number of *distinct* pipeline configurations, and it is also what makes the draw-stream's sort key able to compare state by pointer identity rather than by content. A rebuild that skips the deduplication will still render correctly and will sort much worse.

The "null" program is a real asset, not a sentinel. Keeping every pass's program slots non-empty removes a branch from the innermost part of the draw loop; the cost is one no-op program per backend.

### `PassSET_ZB(test, write, invert)`

**Contract** — sets the depth comparison and depth write. **Depth write is forced off on any pass after the first**: a multi-pass material has already laid down its depth in pass 0, and writing again from a pass that is blending would fight itself. The inverted comparison exists for passes drawn against a reversed depth buffer.

### `PassSET_ablend_mode` / `PassSET_ablend_aref` / `PassSET_Blend`

**Contract** — sets colour blending and alpha testing together, since a blender almost always decides both at once.

```text
FUNCTION set_blend_mode(enabled, src, dst)
  # "enabled with src=one dst=zero" is exactly "disabled" and costs a
  # blend unit; normalize it so two materials that mean the same thing
  # intern to the same state object
  IF enabled AND src == one AND dst == zero THEN enabled := false
  write enabled, src, dst
  # alpha's blend factors are always the colour ones: the engine never
  # separates them, so the two are kept in lockstep rather than exposed
  write alpha_src := src, alpha_dst := dst
```

**Invariants** — the alpha reference is clamped to 0..255. The named shorthands (`_BLEND`, `_SET`, `_ADD`, `_MUL`, `_MUL2X`) are the five combinations the shipped materials actually use, and they are worth keeping as named modes rather than raw factor pairs: `MUL2X` in particular (destination × source, doubled) is the lightmap convention this engine's art is authored against.

### Texture stages — `StageBegin` / `StageEnd` / `StageSET_*` / `Stage_Texture` / `Stage_Matrix` / `Stage_Constant`

**Contract** — record one fixed-function texture stage: its address mode, its colour and alpha combine, and its texture/matrix/constant triple. `StageEnd` advances the cursor; stages are implicitly numbered in emission order.

`Stage_Matrix` is the one with a real decision in it. A stage's matrix is not just a UV transform — *which* coordinates it transforms is decided by the matrix's own mode:

```text
FUNCTION stage_matrix(name, uv_channel)
  m := resolve(name)                      # positional or literal
  SELECT m.mode
    programmable      -> transform 3 components, source = camera-space POSITION
    uv_animation      -> transform 2 components, source = the model's uv_channel
    cubic_reflection  -> transform 3 components, source = camera-space REFLECTION vector
    spherical_reflect -> transform 2 components, source = camera-space NORMAL
    none / absent     -> no transform,           source = the model's uv_channel
```

**Notes** — This is environment mapping expressed as a texture-coordinate source rather than as a shader: the "matrix" a material names may mean "these UVs scroll" or it may mean "generate UVs from the reflection vector". A rebuild on a programmable-only API turns each mode into a different coordinate expression in generated shader source; the *set* of modes is what must survive, because the shipped materials name matrices that rely on each one.

`StageTemplate_LMAP0` is the canonical lightmap stage — clamp addressing, select the texture, read UV channel 1 — factored out because every lightmapped template emits exactly it.

## Sampler setup by name — `SetupSampler` and `SampledImage`

**Contract** — `SampledImage(sampler_name, image_name, texture)` binds one texture to one named sampler in the pass's programs, and configures that sampler's filtering from a small table keyed on the sampler's *name*. Returns the resolved sampler index, or "invalid" if the programs do not use that name.

```text
FUNCTION setup_sampler(index, sampler_name)
  # defaults
  address := wrap ; min := linear ; mip := linear ; mag := linear

  SELECT sampler_name
    "smp_nofilter"  -> address := clamp ; min := point ; mip := none ; mag := point
    "smp_rtlinear"  -> address := clamp ;                mip := none
    "smp_material"  -> address := clamp ;                mip := none
                       # the material lookup is a 3D table: only the third axis wraps
                       set third-axis address := wrap
    "s_base"        -> min := anisotropic ; mag := anisotropic
    "s_detail"      -> min := anisotropic ; mag := anisotropic

  apply address, (min, mip, mag)
  IF index < 4 THEN disable projective division
```

```text
FUNCTION sampled_image(sampler_name, image_name, texture) -> int
  # Some devices expose one object per (sampler, texture) pair and some
  # expose them separately. The lookup key differs; the result does not.
  lookup_name  := combined_samplers ? image_name : sampler_name
  index        := constant_table.find(lookup_name, kind = sampler)
  IF index valid THEN setup_sampler(index, sampler_name)

  texture_index := combined_samplers ? index
                                     : constant_table.find(image_name, kind = texture)
  IF texture_index valid AND texture name non-empty THEN
      pass_textures.append(texture_index, resources.load_texture(normalize(texture)))
  RETURN index
```

**Invariants** — the texture name is put through the engine's texture-name normalization (lowercase, separators, extension stripped) before it reaches the loader, because material data names textures with the case and separators the art tools happened to use.

**Notes** — Filtering by sampler *name* rather than by explicit declaration is how anisotropy gets onto exactly the two samplers that want it — the base colour and the detail layer — without every one of fifty blenders having to say so. It is a convention encoded in the shader source's identifier choices, which means the identifiers `s_base`, `s_detail`, `smp_nofilter`, `smp_rtlinear` and `smp_material` are **part of the frozen shader-data contract**, not internal names. A rebuild that renames them must rename them in the shipped shader sources too.
