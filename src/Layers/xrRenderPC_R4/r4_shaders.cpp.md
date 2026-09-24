# src/Layers/xrRenderPC_R4/r4_shaders.cpp

> The shader pipeline: the macro set that turns settings into a program variant, the cache key that names that variant, the on-disk binary cache, the include resolver, and the reflection pass that builds the by-name constant table.

**Needs** — [`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md) · [`../xrRender/ShaderResourceTraits.h`](../xrRender/ShaderResourceTraits.h.md) · [`../xrRenderDX11/dx11r_constants.cpp`](../xrRenderDX11/dx11r_constants.cpp.md) · [`../xrRenderDX11/dx11ResourceManager_Resources.cpp`](../xrRenderDX11/dx11ResourceManager_Resources.cpp.md) · [`../xrRenderDX11/dx11HW.h`](../xrRenderDX11/dx11HW.h.md) · [`../../xrCore/FileCRC32.h`](../../xrCore/FileCRC32.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it hands a compiler a raw source span, checksums compiled bytes, and reads a binary reflection of the result.

## Purpose

Every material the game loads is compiled here. This is the densest file in the chapter, and it answers five questions a rebuilder filling the graphics-device seam must answer for themselves:

1. **What dialect does the shipped source speak, and do I have to translate it?** This filling does not translate. The retail game ships high-level sources in this API's own language, under the directory this backend reads directly. The [OpenGL filling](../xrRenderPC_GL/rgl_shaders.cpp.md) had to carry a hand-written parallel tree; this one does not, and that asymmetry is the single biggest cost difference between the two fillings. A third API inherits the OpenGL problem.
2. **What parameterizes a compile?** Roughly forty macros, derived from device capabilities, graphics settings, and two mutable renderer fields the caller sets first.
3. **How is a variant named?** A digit string in a frozen positional order, appended to the source name.
4. **What is cached, and what invalidates it?** The compiled binary, invalidated by a checksum over the source *and its include graph*.
5. **How does the engine find a constant afterwards?** By reflecting the compiled binary and building a name-keyed table, which is what the whole material system binds against.

## State

```text
RECORD CompileInputs              # the compile depends on all of these, not just its arguments
  external_macros : list<(name, value)>   # pushed by the material being compiled, then cleared
  skinning_mode   : int                   # -1 = none, 0..4 = a skinning variant
  sample_index    : int                   # -1 = per-pixel, 0..n = compile for one sample
  shadow_map_size : int
  options         : Options               # the renderer's resolved settings, see r2.cpp
  device_tier     : enum                  # what the driver granted

RECORD CacheEntry                 # the on-disk record, in this exact order
  old_dialect   : int (32-bit)    # was the backwards-compatibility retry needed?
  source_crc    : int (32-bit)    # over the source AND every file it includes
  binary_crc    : int (32-bit)    # over everything that follows
  binary        : bytes

# Invariant: the binary checksum is the LAST field before the payload, because it is
# computed over "the rest of the file". Appending a field after it silently corrupts
# every existing entry instead of invalidating it.
# Invariant: all three leading fields must be present, so an entry shorter than one
# of them is discarded rather than parsed.
```

**Invariants** — `external_macros`, `skinning_mode` and `sample_index` are **ambient**: the caller sets them on the renderer, asks for a compile, and clears them. Two calls with identical arguments can therefore produce different programs. This is the file's worst design property and a rebuild should pass them as arguments; what must survive is that they participate in the variant, because the cache key has to cover them (see below, where one of them does not and the caller compensates).

## `compile_shader(name, source, entry_point, target, flags, result)`

**Contract** — produces one compiled program for one stage of one shader, from the on-disk cache when it is valid and by compiling otherwise; then reflects it into a constant table and, for the vertex stage, interns its input signature. Blocks. Writes the cache on a fresh compile. Returns success or failure; failure aborts the material's compilation upstream.

```text
FUNCTION compile_shader(name, source, entry_point, target, flags, result) -> ok
  macros, digits := build_variant()            # see below — the two are built together
  key            := cache_key(name, target, digits)
  source_crc     := checksum_with_includes(source)

  IF cache file at key exists THEN
      entry := read(key)
      IF entry is long enough
         AND entry.source_crc == source_crc
         AND checksum(entry.binary) == entry.binary_crc THEN
          RETURN instantiate(target, entry.binary, entry.old_dialect, result)

  binary, errors := compile(source, macros, include_resolver, entry_point, target, flags)
  IF failed AND errors name the old-dialect diagnostic THEN
      flags := flags + accept_old_dialect
      binary, errors := compile(...)           # exactly one retry
  IF failed THEN log the compiler's own message ; RETURN failure

  write(key, old_dialect = accept_old_dialect was set,
             source_crc, checksum(binary), binary)
  RETURN instantiate(target, binary, old_dialect, result)
```

**Invariants** — **the cache is never validated against the macro set, only against the source.** Two variants of the same source are distinguished purely by the key's digit string. A digit that is missing, or a branch that emits the wrong number of digits, silently hands one variant another variant's binary — a fault that survives a restart and looks like corrupt data rather than a bug.

**Invariants** — the retry is the *only* place the older shader dialect is accepted, and whether it was needed is **stored in the cache and fed forward into reflection**. It has to be: in that dialect a texture and its sampler may legitimately carry the same name, and the constant table needs to know that before it decides whether a name collision is an error (see [`dx11r_constants.cpp`](../xrRenderDX11/dx11r_constants.cpp.md)). A rebuild that forgets to persist this flag produces caches that work on the run that wrote them and fail on the next.

**Notes** — the retry is triggered by matching the compiler's diagnostic *text* for one specific error code. The source itself marks this as unsatisfactory ("is there a better way?"). A rebuild should compile with the compatibility setting decided per shader tree, not per failure.

## The variant: macros and the name digits

Built in one pass, so that each option contributes a macro (when it is on) and a digit (always). The order is frozen — it *is* the cache key.

```text
FUNCTION build_variant() -> (macros, digits)
  macros += the external macros the material pushed   # NO digits; see the note below
  macros += ("SMAP_size", shadow map size) ; digits += that size as text

  # --- device capabilities: what the driver can actually do ---
  emit_if(16-bit float targets can be filtered)
  emit_if(16-bit float targets can be blended into)
  emit_if(depth textures can be sampled with a compare)
  emit_if(...and the comparison result filtered)
  emit_if(four-tap depth fetch available)
  emit_if(shadow jitter)
  emit_if(raster model >= 3 -> dynamic branching is affordable)
  emit_if(vertex texture fetch available)

  # --- lighting and shadow settings ---
  emit_if(translucent shadows)   emit_if(motion blur)
  emit_if(sun filtering)         emit_if(sun is baked into lightmaps)
  emit_if(forced gloss, carrying its value)
  emit_if(forced skinning colour)

  # --- ambient occlusion: three digits, one strategy ---
  IF hemisphere occlusion THEN
      macros += ("HDAO", 1) ;  digits += 1, 0, 0
  ELSE
      digits += 0, horizon-based flag, half-precision flag
      IF horizon-based THEN
          macros += ("SSAO_OPT_DATA", half-precision ? 2 : 1)
          emit_if(vectorized occlusion code)
          macros += ("USE_HBAO", 1)

  # --- skinning: six mutually exclusive macros, six digits ---
  emit_if(mode == none) ; emit_if(mode == 0) ; ... ; emit_if(mode == 4)

  # --- the advanced post-processing group: each also requires the group to be on ---
  emit_if(group AND soft water)      emit_if(group AND soft particles)
  emit_if(group AND depth of field)
  emit_if(group AND sun shafts,        carrying the quality level)
  emit_if(group AND ambient occlusion, carrying the quality level)
  emit_if(group AND sun quality,       carrying the quality level)
  emit_if(group AND steep parallax)

  # --- shape and capability tier ---
  emit_if(geometry-buffer packing)
  emit_if(the intermediate shader model's features)
  emit_if(device tier is the newest -> the newest shader model)
  emit_if(double-precision operations supported)
  emit_if(extended double-precision instructions supported)
  emit_if(the four-way sum-of-absolute-differences instruction supported)
  emit_if(min/max shadow maps)

  # --- multisampling: LAST, and deliberately so ---
  IF multisampling THEN
      emit_if(multisampling)                       # 1
      emit_if(sample count, carrying the count)    # 1
      emit_if(sample index,  carrying the index)   # 1
      emit_if(per-sample optimization)             # 1
      emit exactly one of three alpha-to-coverage strategies ; digits += all three
  ELSE
      digits += six zeros
```

**Invariants** — **`emit_if` writes its digit unconditionally and its macro only when the value is non-zero.** That rule has a consequence worth stating plainly: an option whose value is legitimately *zero* gets a digit but no macro. It bites exactly once, on the sample index, where compiling "for sample 0" emits no sample-index macro at all. The shader sources must therefore supply a default for it.

**Invariants** — **the multisampling block is last because its two arms emit different digit counts** — seven when on, six when off. That is safe only because the block is terminal and because its first digit already separates the two families: no multisampled variant can ever share a name with a non-multisampled one, so the length difference cannot cause a collision. The source carries an emphatic comment saying the block must stay at the end. A rebuild that keeps the digit-string scheme must keep that rule; the honest fix is to key the cache on a hash of the emitted macro text and delete the digit string entirely.

**Notes** — **the external macros contribute no digits.** They are the per-material defines a material description pushes before asking for a compile — tessellation method, whether a lightmap hemisphere term is present, whether detail textures are used. They are keyed into the cache through the shader *name* instead: the material builder spells the same set into a parenthesized suffix on the name it asks for. The two lists are maintained by hand, in different files, and must agree. This is the one place where losing the discipline produces a wrong binary rather than a redundant recompile.

## The cache key and where it lives

```text
FUNCTION cache_key(name, target, digits) -> path
  stage  := the first two letters of the target profile   # vs, ps, gs, cs, hs, ds
  tier   := device tier is newest ? "r4" : "r3"           # lower tiers are refused outright
  RETURN "$app_data_root$/shaders_cache_oxr/" + tier + "/" + name + "." + stage
                                              + "/" + digits
```

**Invariants** — the cache lives under the *writable application data* root, never beside the game's own files, because the game directory may be read-only and is shared between the original engine and this one.

**Notes** — the cache is namespaced by device tier *and* carries a shader-model digit inside the digit string, so the tier is recorded twice. Harmless, and it makes a cache directory readable by eye.

**Notes** — the shader **source** directory is the same for both tiers: this backend reads the tree the retail game shipped for its highest renderer. Only the cache is split. A machine that can run the newest tier and is asked for an older preset compiles the same sources with a different macro set into a different cache namespace.

## `checksum_with_includes`

**Contract** — the cache's invalidation input: a checksum over the shader source *plus every file it transitively includes*, each included file resolved against the directory of the file that included it.

**Invariants** — following the include graph is what makes editing a shared header invalidate only the shaders that actually use it. The [OpenGL filling](../xrRenderPC_GL/rgl_shaders.cpp.md) takes the coarse route — one checksum over the whole shader directory — and recompiles everything when anything changes. This one is finer and strictly better; a rebuild should copy this side.

**Notes** — the per-file checksums are **summed**, not chained, so the combined value is insensitive to the order of the includes. Two edits that swap the contents of two included files would not change it. Unreachable in practice, and a chained hash would cost nothing.

**Notes** — the include scan here is textual and independent of the resolver the compiler uses; a missing include file is a hard fault at checksum time rather than a compile error, which is a better diagnostic but means the two scanners must agree on how a path is resolved.

## `include_resolver`

**Contract** — supplies the compiler with the contents of an included file, on demand, during a compile. Looks first inside this backend's shader directory, then at the shader root; a file found in neither fails the compile. Returns a copy that the resolver itself frees when the compiler is done with it.

**Invariants** — the two-step lookup is the mechanism that lets one shared header serve every backend: backend-specific headers sit in the backend's own directory and shadow the shared ones, and anything not shadowed resolves to the shared copy. A rebuild needs the same two-level search or it must duplicate the shared headers per backend.

**Notes** — the contents are copied and explicitly terminated before being handed over, because the virtual filesystem may return a span inside a mapped archive and the compiler expects a terminated buffer it may hold onto.

## `instantiate` — from bytes to a usable program

**Contract** — takes a compiled binary and the target profile, creates the device object for the stage the profile names, and builds everything the engine needs to *use* it. Fails loudly, naming the shader, if the device rejects the binary.

```text
FUNCTION instantiate(target, binary, old_dialect, result) -> ok
  stage := first letter of target -> vertex | pixel | geometry | compute | hull | domain
  result.program := device.create_stage_program(stage, binary)
  IF creation failed THEN log the shader's name ; RETURN failure

  IF stage IS vertex THEN
      signature := extract the input signature from the binary
      result.signature := intern(signature)        # deduplicated byte-for-byte

  reflection := reflect(binary)
  IF reflection succeeded THEN
      result.constants.old_dialect := old_dialect
      result.constants.parse(reflection, stage)    # the by-name table
  ELSE
      log; the program is still usable, but nothing can be bound to it

  IF disassembly is enabled THEN
      write the compiler's disassembly to a file named after the shader
```

**Invariants** — the **input signature** is extracted only for vertex programs, and it is what geometry declarations are matched against. The engine's vertex layouts are created lazily per (declaration, signature) pair, so interning the signature by byte equality is what lets two vertex programs with identical input requirements share every layout — and what makes releasing a program able to find and destroy the layouts it caused. See [`dx11ResourceManager_Resources.cpp`](../xrRenderDX11/dx11ResourceManager_Resources.cpp.md).

**Invariants** — **reflection is how the by-name binding the whole material system rests on comes to exist.** The material names a quantity; the compiled program knows where it lives; this call joins them once, at load time, so that nothing on the per-draw path ever compares a string. A rebuild whose shader toolchain does not expose reflection must produce the same table some other way — from a side-car description emitted at compile time, for instance — because the alternative is to bind by slot, and the shipped material descriptions do not carry slots.

**Notes** — a failed reflection is not fatal. The program is created and the material proceeds with an empty constant table, which produces a shader that draws with whatever the constants happened to contain. This is a deliberate choice to keep a broken shader from killing a level load, and it is a poor one: the symptom is visual and far from the cause.

**Notes** — the disassembly dump and the debug name attached to each program exist only to make a graphics-debugger capture readable. Both are incidental; what a rebuild should keep is the *idea* that every program carries its source name into the captured frame, because without it a capture of this engine is several thousand anonymous draws.

## `add_shader_option(name, value)`

**Contract** — appends one macro to the renderer-level external list that the next compile will prepend. Paired with a clear-all on the renderer. Called by a material description immediately before it asks for a pass, and cleared immediately after.

**Notes** — see the invariant above: these macros do not reach the cache key on their own. The caller is responsible for spelling them into the shader name as well.
