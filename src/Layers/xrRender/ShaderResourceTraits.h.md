# src/Layers/xrRender/ShaderResourceTraits.h

> The per-stage compilation table: for each kind of shader program, what its source file is called, which profile it compiles against, which entry point, how the device is asked to create it — plus the one generic create-or-reuse routine that drives all six from that table.

**Needs** — [`ResourceManager.h`](ResourceManager.h.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`ResourceManager.h`](ResourceManager.h.md) · [`dx11ResourceManager_Resources.cpp`](../xrRenderDX11/dx11ResourceManager_Resources.cpp.md) · [`glResourceManager_Resources.cpp`](../xrRenderGL/glResourceManager_Resources.cpp.md) · [`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md) · [`r4_shaders.cpp`](../xrRenderPC_R4/r4_shaders.cpp.md)
**Tier floor** — T1: it hands raw compiled blobs to a driver and reflects named constants back out of them.

## Purpose

Six shader stages differ in exactly four ways — file extension, compilation profile, device creation call, and which half of the constant table their constants belong to — and are otherwise identical: look the name up, on a miss read the source, pick a target, compile, create, register. This file states the four differences once per stage as a table, and writes the shared routine once.

It is a header with no implementation file because the table is consulted by type, resolved where it is used. That is incidental. What is load-bearing is the *table itself*: a rebuild needs exactly these four facts per stage, and the shipped shader sources depend on the values.

## State

`Stateless.` It defines a table read at compile time and operates on the registry's per-stage maps.

```text
TABLE per shader stage
  stage     file ext   profile                       constant destination
  vertex    ".vs"      the device's geometry profile  vertex
  pixel     ".ps"      the device's raster profile    pixel
  geometry  ".gs"      by device feature level:       geometry
                       one of three generations
  hull      ".hs"      the newest generation only     hull
  domain    ".ds"      the newest generation only     domain
  compute   ".cs"      the newest generation, or      compute
                       one step below on older parts
```

**Invariants**

- **The extensions are frozen.** The shipped shader sources are named by stem plus one of these six suffixes, and the stem is what a material script names. A rebuild that renames them cannot load the shipped shaders.
- The constant destination is not cosmetic: a named constant discovered while reflecting a compiled program is recorded against the stage it belongs to, because the same name may be bound to a different register in the vertex and pixel programs of one pass and both bindings must survive in the merged table.
- The older-generation profile names (`vs_1_1` through `ps_2_0`) are selected **by scanning the source text for an entry-point name**, not by a declaration. This is the compatibility hinge of the whole shader system and is described below.

## `GetCompilationTarget` — the entry-point convention

**Contract** — given the source text of a program, choose the profile and the entry-point name to compile against. Pure string inspection; no I/O.

```text
FUNCTION select_target(stage, source_text) -> (profile, entry_point)
  entry_point = "main"
  FOR EACH legacy profile of this stage, oldest first
    # vertex: 1_1 then 2_0 ; pixel: 1_1, 1_2, 1_3, 1_4, then 2_0
    IF the source text contains the name "main_<stage>_<profile>"
      RETURN (that profile, that name)
  RETURN (the device's best profile for this stage, "main")
```

**Notes**

- **This is how one shipped shader source serves several hardware generations.** A source file may define `main`, and also `main_ps_1_1`, `main_ps_1_4` and so on. Whichever legacy entry point is present *first in the search order* wins — the search is ordered oldest-to-newest, so a file defining both `main_ps_1_1` and `main_ps_2_0` compiles as the oldest. That is the opposite of what "best available" would do, and it is correct: those entry points exist only in shaders written for the oldest renderer, where the oldest profile is the target.
- The match is a plain substring search over the whole file, comments included. A source that merely *mentions* one of these names in a comment compiles against that profile. No shipped file does, but a rebuild should match a declaration rather than a substring, and must keep the naming convention itself.
- On the newer backends the legacy branch is bypassed entirely and the device's own profile always wins: those backends cannot compile the old profiles at all, and the old shaders are not used by them. A rebuild targeting one graphics API only can delete the legacy search — but not the naming convention, because it must still find `main`.
- The geometry and compute profiles are chosen from the device's reported feature level, mapping three generations of hardware to three profile names. The vertex and pixel profiles come from the device's own reported best, which the device layer computes. The tessellation stages have exactly one profile because the hardware that has them has only one.

## `CreateShader` — create or reuse, for any stage

**Contract** — given a resource name (and optionally a different file name), return the compiled program for it, compiling on a miss. Registers the result. Blocks on file I/O and compilation. Exits the process with a user-facing message if compilation fails and no fallback applies — a shader that will not compile means the device cannot run the game.

```text
FUNCTION create_shader(stage, name, file_name, compile_flags) -> Program
  IF the stage's registry map already holds name, RETURN it

  program = new empty program, registered under name
  IF name is the literal "null"
    leave its handle empty and RETURN                 # a pass may legitimately bind no program

  stem = (file_name, or name) truncated at the first "("
  path = <renderer's shader folder> + stem + <stage extension>, resolved under the shader root
  source = read path
  IF absent and the missing-shader fallback is enabled
    source = read "stub_default" + <stage extension>    # and log the substitution

  (profile, entry) = select_target(stage, source)
  add the row-major matrix packing flag, plus optimization or debug flags by build
  compile source with (entry, profile, flags) into program, reflecting its constants
  IF compilation failed and the fallback has not been tried, retry with the stub
  FAIL WITH "your device does not meet the requirements" if it still failed
  RETURN program
```

**Invariants and notes**

- **The name is the cache key; the file name is not.** A program's name carries its macro set, conventionally as a parenthesized suffix, and the file is found by truncating at the parenthesis. Two programs compiled from one source with different macros are two entries under two names sharing one file. A rebuild must keep the key and the path distinct; merging them silently shares the wrong compilation.
- **The literal name `null` yields a program with no handle.** Binding it clears the stage. This is a frozen name used by the material scripts.
- **The shader folder is chosen by the renderer generation** — each backend reports its own subfolder, and the shipped data has one folder per generation. The path is then resolved against the shader root of the virtual filesystem, so a mod can override a single shader by dropping a file in.
- **Row-major matrix packing is forced, in every build.** The engine's own matrices are row-major and the constant setters upload them as-is; a compiler defaulting to column-major would transpose every matrix the shaders read. This single flag is the difference between a correct frame and a garbled one, and it is the least obvious line in the file.
- The fallback to a neutral stub is enabled by a command-line switch in the shipping build and unconditionally in development builds. It converts "this shader is missing" from a crash into a visibly wrong but running frame, which is what lets an incomplete shader set be debugged at all. It is attempted twice at most — once for a missing file, once for a failed compile — and the second attempt is reached by re-entering the first, which is the one place in the file where the control flow is doing something a rebuild should simply write as a loop with a bound.
- Optimization is forced to the highest level in release builds and replaced by debug information otherwise. This is the only place the engine states a preference about shader optimization.

## `DestroyShader`

**Contract** — remove a program from its stage's registry map. Returns whether it was found; reports a miss, because a registered resource failing to find itself means the name was changed after registration. Does not release the device handle — the program record's own destruction does that.

## `GetShaderMap`

**Contract** — selects the registry map for a stage. Pure dispatch; the six one-line bodies are the mechanism by which the shared routine becomes six routines. Nothing decides here.

## The OpenGL compilation helpers

**Contract** — compile a source string into a program object for a given stage, or restore one from a cached binary blob; link several stage programs into a pipeline, or into a single monolithic program when the device lacks separable programs; report compilation failures with the driver's log and the source text.

**Notes**

- **The fragment-output bindings are frozen names.** Before linking, output locations 0, 1 and 2 are bound to the names `SV_Target`, `SV_Target0`, `SV_Target1` and `SV_Target2` — with both `SV_Target` and `SV_Target0` mapped to location 0. Those are the *other* graphics API's output semantics, and binding them by name is how one set of shipped shader sources compiles under both. A rebuild that translates the shipped shaders must reproduce this mapping or the deferred pass writes to the wrong targets.
- The separable-program path exists so that a vertex and a pixel program can be mixed without relinking the pair. When the device does not support it, the same programs are linked monolithically per combination — correct, but it multiplies link time by the number of combinations. Both paths must exist because the minimum device the engine claims to support does not guarantee the feature.
- The binary-blob path is what makes the compiled-shader cache work on this backend: a program that was linked once is stored as a driver blob and restored directly. The blob is driver-specific and must be invalidated when the driver changes — the cache key is the responsibility of the resource manager, not of this file.
