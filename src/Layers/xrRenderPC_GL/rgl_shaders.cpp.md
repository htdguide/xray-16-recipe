# src/Layers/xrRenderPC_GL/rgl_shaders.cpp

> The shader pipeline: the macro set that turns console settings into a program variant, the include resolver, the cache key, and the on-disk compiled-binary cache.

**Needs** — [`../xrRenderGL/glHW.h`](../xrRenderGL/glHW.h.md) · [`../xrRenderGL/glResourceManager_Resources.cpp`](../xrRenderGL/glResourceManager_Resources.cpp.md) · [`../xrRender/ShaderResourceTraits.h`](../xrRender/ShaderResourceTraits.h.md) · [`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it produces a driver-specific compiled binary, keys a cache on the driver's own identity strings, and checksums the result.

## Purpose

This is the single most consequential file in the chapter, because it is where the recipe's largest stated compatibility hazard is actually resolved. §5 of [SYSTEM-REQUIREMENTS](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) says shader *source* ships with the game, in a dialect belonging to another API, and that a port must either translate it at load time or ship replacements.

**This backend ships replacements.** There is no translator. Alongside the high-level sources for the Direct3D renderers, the game data carries a parallel tree of sources in this API's own shading language, written to *look* like the originals: a shared header redefines the foreign language's type names, intrinsics, swizzle helpers and semantic names so that a shader body reads almost identically in both trees. The port cost was paid once, by hand, and it is carried in the data rather than in the code.

Two consequences a rebuilder must absorb:

- **A third API needs a third tree.** Nothing here generalizes. A rebuild targeting Vulkan or Metal either writes another parallel tree, or writes the translator this project decided not to write.
- **The semantic-name defines in that shared header are a contract with the engine's vertex-layout table.** They bind each semantic to a numeric attribute location, and those numbers must match [`glBufferUtils.cpp`](../xrRenderGL/glBufferUtils.cpp.md)'s table exactly.

Everything else in the file serves one goal: never compile a program twice. Compiling the shipped shader set from source is measured in tens of seconds; loading it back as driver binaries is not.

## State

```text
RECORD ShaderVariantKey          # what makes two compiles different
  base_name      : text          # the shader source's own name
  stage          : text          # two characters: vertex, pixel, geometry, ...
  option_digits  : text          # one digit per option, in a FIXED order

RECORD CacheEntry                # the on-disk record, in this exact order
  adapter_name       : text      # the driver's renderer string
  api_version        : text
  shading_version    : text
  binary_format      : int (32-bit)   # the driver's own binary format tag
  source_tree_crc    : int (32-bit)   # over the WHOLE shader directory
  binary_crc         : int (32-bit)   # over everything that follows
  binary             : bytes

# Invariant: the binary checksum is the LAST field before the payload, because
# it is computed over "the rest of the file". Adding a field after it silently
# breaks every existing cache entry rather than invalidating it.
# Invariant: an entry is used only when adapter, both version strings, and the
# source-tree checksum all match. Any mismatch recompiles.
```

## `compile_shader(name, source, entry_point, stage, flags, result)`

**Contract** — produces a compiled program for one stage of one shader, either from the on-disk binary cache or by compiling the source. Parses the result's constant table for pixel-stage programs. Blocks. Writes the cache on a successful fresh compile. Returns success or failure; a failure aborts material compilation upstream.

```text
FUNCTION compile_shader(name, source, entry_point, stage, flags, result) -> bool
  options := []      # lines prepended to the source
  digits  := []      # the variant key, one digit per option

  # --- preamble: the language version and the separable-program extension
  options += "#version 410"
  options += "enable the separable-program extension"
  options += debug build ? "optimization off" : "optimization on"
  digits  += debug build ? 0 : 1
  options += a comment naming the shader and stage     # for readable error logs

  # --- the macro set, in a FIXED order that defines the variant key
  emit("SMAP_size", shadow map size)                   # always, with its digits
  emit_if(fp16 filtering)          emit_if(fp16 blending)
  emit_if(hardware shadow maps)    emit_if(hardware shadow PCF)
  emit_if(four-tap depth fetch)    emit_if(shadow jitter)
  emit_if(raster model >= 3 -> branching allowed)
  emit_if(vertex texture fetch available)
  emit_if(translucent shadows)     emit_if(motion blur)
  emit_if(sun filtering)           emit_if(static sun lighting)
  emit_if(forced gloss, with its value)  emit_if(forced skinning colour)
  emit_if(ambient-occlusion blur)
  IF horizon-based occlusion is off and hemisphere occlusion is on THEN
      emit("HDAO"); digits += 1, 0, 0
  ELSE
      digits += 0, then the horizon-based flag
      emit_if(horizon-based occlusion)  emit_if(vectorized occlusion)
  emit_if(packed occlusion data, as 1 or 2 for full or half precision)
  emit_if(skinning mode, as one of six mutually exclusive macros)
  emit_if(advanced post-processing AND soft water)
  IF advanced post-processing AND screen-space reflections THEN
      emit("SSR_QUALITY", level); emit_if(half-depth reflections)
                                  emit_if(reflection jitter)
  emit_if(advanced post-processing AND soft particles)
  emit_if(advanced post-processing AND depth of field)
  emit_if(advanced post-processing AND sun shafts, with quality)
  emit_if(advanced post-processing AND ambient occlusion, with quality)
  emit_if(advanced post-processing AND sun quality)
  emit_if(advanced post-processing AND steep parallax)
  emit_if(geometry-buffer packing)
  emit_if(shader model 4.1 features)        # suppressed on one platform; see Notes
  emit_if(min/max shadow maps)
  emit_if(first-game compatibility mode)
  IF multi-sampling THEN
      emit("USE_MSAA"); emit("MSAA_SAMPLES", count); emit("ISAMPLE", index)
      emit_if(multi-sample optimization)
      emit exactly one of three alpha-to-coverage strategies, digits for all three
  ELSE
      digits += seven zeros                 # the same width, all off

  # --- cache lookup
  key       := "gl/" + name + "." + stage's first two letters + "/" + digits
  full_path := "$app_data_root$/shaders_cache_oxr/" + key
  tree_crc  := checksum over the ENTIRE shader source directory

  program := 0
  IF the device can retrieve program binaries AND a cache file exists THEN
      entry := read(full_path)
      IF entry.adapter, entry.api_version, entry.shading_version all match
         AND entry.source_tree_crc == tree_crc
         AND checksum(entry.binary) == entry.binary_crc THEN
          program := load_binary(entry.binary, entry.binary_format)

  # --- compile on a miss
  IF program == 0 THEN
      sources := resolve_includes(source)
      program := compile(options ++ sources, stage)
      IF program != 0 AND the device can retrieve program binaries THEN
          binary, format := retrieve(program)
          write(full_path, adapter, versions, format, tree_crc,
                checksum(binary), binary)

  IF program == 0 THEN RETURN failure
  result.handle := program
  IF stage is the pixel stage THEN result.constants.parse(program, stage)
  RETURN success
```

**Invariants** — **the digit sequence is positional and every branch must emit the same number of digits.** Look at the multi-sampling block: the "off" arm emits seven zeros to match the seven the "on" arm emits. A branch that emits a different count shifts every following digit and makes two genuinely different variants share a cache filename, which produces a program compiled for someone else's settings — a bug that survives a restart and looks like corrupted data. This is the one discipline a rebuild must not relax; the honest fix is to key the cache on a hash of the emitted macro text instead of on a hand-maintained digit string.

**Invariants** — the cache is invalidated by *four* independent things: a different driver (adapter string), a different API version, a different shading-language version, and any change anywhere in the shader source tree. The last is a whole-directory checksum rather than a per-file one, which is coarse — editing one shader recompiles all of them — and correct, because the include graph is not tracked.

**Invariants** — only the *pixel* stage's constants are parsed here. The other stages' tables are built by the same call for their own programs; the guard exists because the reflection destination differs per stage and the pixel stage is the one whose samplers must be numbered from zero (see [`glr_constants.cpp`](../xrRenderGL/glr_constants.cpp.md)).

**Notes** — the entry point and compile flags the interface passes are accepted and ignored. This API compiles a whole translation unit with a fixed entry point name, so the engine's convention of naming an entry function per profile does not apply; the replacement shader tree simply uses the fixed name.

**Notes** — the shader-model-4.1 macro is suppressed on one platform. The reason is specific and instructive: that platform's driver claims support for the language version but requires a compile-time-constant offset on a gather instruction, which the shader uses variably. The engine could not detect this at run time, so it is disabled by platform. A rebuild will meet the same class of problem — a driver that advertises a feature it does not fully implement — and should expect to carry a small table of such exceptions.

**Notes** — the binary cache is only used when the device can *both* retrieve program binaries and use separable programs. The monolithic path is uncached and is marked in the original as wanting the same treatment. On a device without separable programs the game therefore recompiles its whole shader set at every level load.

## `resolve_includes(source)`

**Contract** — flattens a shader's include graph into an ordered list of source fragments, without concatenating them. Recursive. Each included file is read whole, terminated, and spliced in at the point of its directive by *splitting the including text in two* and recording the two halves around the inclusion.

```text
FUNCTION resolve_includes(file) -> list of fragments
  data := file's bytes, copied into a writable buffer, newline- and null-terminated
  record data as a fragment

  str := data
  WHILE str contains an include directive
    str := the directive's position
    fn  := the quoted filename after it
    terminate the buffer AT the directive    # ends the preceding fragment
    terminate fn at its closing quote

    path := the backend's shader directory + fn, resolved against the data root,
            with separators normalized
    included := open(path); FAIL IF absent
    resolve_includes(included)               # depth-first: the include's own
                                             # includes are spliced here
    close(included)

    record the text after the closing quote as the next fragment
```

**Invariants** — the fragments must be handed to the compiler *in order and as a list*, because the splitting leaves them as interior pointers into buffers that are freed together at the end. This is the whole reason the compile entry point takes an array of sources rather than one string: assembling one string would mean another copy of every shader.

**Invariants** — the option lines are prepended as the first fragments, before any source. This is what makes them act as macro definitions, and it is also why the version directive has to be the very first option — the language requires it before anything else.

**Notes** — the include scanner is textual and does not understand comments, so a commented-out include is still followed. The original marks this. It is not a problem in the shipped tree and would be in an edited one.

**Notes** — path separators are normalized toward one platform's convention inside a resolved path. The shipped shader sources write their includes with the other convention. The virtual filesystem is case- and separator-tolerant by design ([§4](../../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)), so this is belt and braces.

## `add_shader_option(name, value)`

**Contract** — appends a macro definition to a renderer-level option string that is prepended to *every* shader in addition to the per-compile set above. Used by the renderer's own configuration to inject settings that do not vary per shader.

## The option and name accumulators

**Contract** — two small fixed-capacity accumulators, one for macro lines and one for the digit string. They are fixed-capacity because they are built on the stack during a compile and their maxima are known: one hundred and twenty-eight option lines, and a path-length digit string.

**Notes** — the fixed capacities are unchecked. Adding enough new options overruns them silently. A rebuild should use growable storage; the only reason not to is that this runs inside a level load where allocation churn is visible, and that is not a good enough reason.
