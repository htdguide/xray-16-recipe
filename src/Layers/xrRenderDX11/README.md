# `src/Layers/xrRenderDX11` — one filling of the graphics-device seam

Chapter 20 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order), with
[`../xrRenderPC_R4`](../xrRenderPC_R4/README.md), which registers it as a selectable
renderer.

Everything above this directory — the scene graph and visibility walk of chapter 18, the
deferred frame graph of chapter 19, the game itself — is written against a device it never
names. This directory is the *filling*: the body of that contract for Direct3D 11. Its
sibling [`../xrRenderGL`](../xrRenderGL/README.md) is the same contract filled for
OpenGL, and both are compiled against the same chapters 18 and 19.

Read this chapter as an answer sheet, not as a Direct3D manual. The question each page
answers is **what must a backend provide so that the engine above it works** — and the
answer transfers, because the engine's demands are the same whether the filling is
Vulkan, Metal, WebGPU or Direct3D 12. What does *not* transfer is confined, deliberately,
to [`CommonTypes.h`](CommonTypes.h.md): every other file in the directory speaks the
engine's own dialect, and a port to another API replaces that one translation table and
leaves the structure alone.

The seam itself is specified in
[§3 Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device); this chapter
is the proof that its capability list is achievable.

---

## What the seam actually demands

Eight obligations, in the order a rebuild will meet them. Each names the pages that
discharge it.

### 1. A device, a capability level, and a lifecycle that can lose everything

The backend must bring up a device, choose an adapter, and **negotiate the highest
capability level the machine grants rather than assume one**, because the same executable
runs on hardware a decade apart. The capability record it fills in is branched on by the
whole renderer: which shader profile to compile against, what the rasterizer can do,
whether multisampling is available and at what quality, and how many physical graphics
processors the frame must be pipelined across.

The obligation that is easy to miss and expensive to retrofit is the **device-state
machine**. Presentation can report occlusion, reset-required or device-removed. The frame
loop consults the device once per frame and decides to render, to rebuild, or to die.
Rebuilding means *every* resource that lives in device memory is recreated — which is why
every render target knows how to recreate itself from its own description, and why a
material's resolved state block must never outlive a reset.
[`dx11HW.cpp`](dx11HW.cpp.md) · [`dx11HWCaps.cpp`](dx11HWCaps.cpp.md) ·
[`dx11SH_RT.cpp`](dx11SH_RT.cpp.md)

### 2. Buffers with two lifetimes, and a vertex layout that is frozen on disk

Geometry buffers come in exactly two lifetimes: **built once and never changed** (level
geometry, model meshes) and **rewritten every frame** (particles, interface quads, decals,
debug lines). The first is assembled in host memory and uploaded in one shot as an
immutable device buffer; the host copy is then thrown away unless the caller asked to keep
reading it. The second is a host-writable device buffer written in a ring discipline with
a discard hint at wrap.

The load-bearing consequence for a rebuild: **the staging path never touches the device
until upload**, so a level loader can build geometry on a worker thread with no device
involvement at all. The seam's "what may be created off the render thread" question has
this as its main answer.

The vertex-layout description is *frozen*: the engine's model files ship a legacy
element-list form, so that form is the on-disk truth and the device's form is derived from
it at layout-creation time. A rebuild may not choose its own layout description.
[`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md) ·
[`CommonTypes.h`](CommonTypes.h.md)

### 3. A vertex layout that depends on the program, not only on the geometry

A vertex declaration alone cannot produce a layout object: the device validates the layout
against the **input signature of the bound vertex program**. So the layout is a function of
two things and is memoized per (declaration, signature) pair, created lazily on first draw
and destroyed when either side goes away. This is the single biggest structural difference
between this API generation and the one before it, and any rebuild on a modern API meets
it too.
[`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) ·
[`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md)

### 4. Textures: four kinds of content behind one name, one flat slot space

A texture name may resolve to a still image, a video stream, an animated sequence of
stills, or a target another pass rendered into — and the draw path must not branch on
which. It does not: resolution happens once at load by **choosing a binding function**,
and binding thereafter is an indirect call. The same mechanism gives loading its lazy
trigger — the initial binding function is "load me, then re-dispatch" — so a texture is
loaded the first time something draws with it, not when it is named.

Loading itself must answer: where the file is looked for, what it falls back to when
missing (a missing texture must not be fatal in a shipping game), how many top mip levels
are dropped for a texture-quality setting, and how a file the disk refuses is retried.
Block-compressed content is uploaded without decoding; the format vocabulary is frozen on
both sides and translated in one table.
[`dx11SH_Texture.cpp`](dx11SH_Texture.cpp.md) · [`dx11Texture.cpp`](dx11Texture.cpp.md) ·
[`dx11TextureUtils.cpp`](dx11TextureUtils.cpp.md) · [`dx11SH_RT.cpp`](dx11SH_RT.cpp.md)

### 5. State as deduplicated blocks

State is set as objects, not fields; identical objects are interned once at load and bound
by pointer comparison thereafter. This has a directory of its own — the ideas, the
normalization step that makes interning effective rather than merely correct, and the
ownership rule that follows from it, are in
[**`StateManager/README.md`**](StateManager/README.md). The field-level rules it rests on
— every field's default, which fields are irrelevant given which others, how two
descriptions are compared and hashed — are in
[`dx11StateUtils.cpp`](dx11StateUtils.cpp.md).

### 6. The constant binding model — and the by-name binding the material system needs

This is the obligation a rebuilder is most likely to underestimate, because it is not in
the capability list; it is in the *shape* of the contract.

The material system names the quantities it sets — `m_WVP`, `fog_color`, `s_base` — and
never learns where they live. A compiled program knows where everything lives but not what
the engine calls it. Joining the two, **once, at load**, is what makes by-name binding
free at draw time:

- A compiled program describes itself: its uniforms grouped into named blocks, and its
  bound resources in numbered slots. That self-description is walked and turned into, per
  named uniform, a record saying *which stages want it, which block it lives in, at what
  byte offset, in what shape*; and per named resource, *which flat slot it binds to*.
  Stage and slot are packed into bit fields of one integer. **No string is compared on
  the per-draw path.**
- A constant record carries a set of *stage flags*, so one `set("fog_color", …)` call fans
  out to the vertex program's copy and the pixel program's copy at their respective
  offsets, and the caller never mentions a shader stage. Setting a quantity no stage
  declares is a deliberate no-op.
- Blocks are **shared across programs by layout identity**: same block name, same member
  types, same member names in the same order. So a value written once is uploaded once,
  no matter how many materials read it. Member *names* participate in identity precisely
  because the by-name binding resolves against them.
- A block is a host-side byte image plus a device buffer, uploaded only when dirty and
  only when a draw demands it. Constant offsets are stored 16-bit, capping a block at
  64 KiB — which is the shader model's own ceiling, so nothing is lost.
- A material pass hands over a whole constant *table*; the backend distributes its blocks
  into per-stage slot arrays and rebinds only the contiguous range that changed. Then it
  runs the material's *by-name handlers* — the callbacks that compute a value at draw time
  rather than storing one.

A rebuild whose shading language exposes uniforms differently still owes all five of these
properties. The engine's material data assumes them.
[`dx11r_constants.cpp`](dx11r_constants.cpp.md) ·
[`dx11r_constants_cache.h`](dx11r_constants_cache.h.md) ·
[`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) ·
[`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md)

The matrix storage convention is part of the frozen contract, not a preference: the
shipped shader sources are written for one ordering, so the host must write matrices in
that ordering. See [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md).

### 7. Draw submission: record, then bind in one fixed order

Chapter 18 declares a *command list* — a shadow of all device state with a redundancy
filter in front of it. This chapter supplies its bodies, and three properties matter:

- **Setters record; draws bind.** A setter updates the shadow and defers the device work.
  Nothing reaches the device until a draw or dispatch demands it, and then everything is
  bound in one fixed order.
- **The redundancy filter is the whole point.** A comparison is cheaper than a device
  call, so every setter is on the hot path of every draw and must inline into its caller.
  This is why the backend's command-list implementation is a header with no separate
  implementation file — an incidental C++ fact standing for a real constraint.
- **The submission context is plural.** The device creates one immediate context plus a
  fixed pool of deferred ones, so visibility walks for the main view, for each sun cascade
  and for the rain map can record independently. A rebuild on an API without deferred
  recording must either serialize those walks or build its own command buffers.

Occlusion and timestamp queries are declared in chapter 18 against this directory's
vocabulary and compiled per backend; they do not get pages here. What the backend owes is
the query object, the two query kinds, and the rule that a query's result is read *a frame
or more later, without blocking* — the occlusion path splits a light set into "answered"
and "still pending" precisely so that no frame ever waits on a readback.
[`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) ·
[`CommonTypes.h`](CommonTypes.h.md)

### 8. Materials: the script surface, and how one name becomes many programs

A shipped material file is a small program describing its passes. Its calling surface, and
the five level-of-detail entry points every material is asked for, live here because what
a pass *is* differs per backend. What a pass declaration compiles into on this filling is
six stage programs, a merged constant table, a named sampler list with fixed presets and a
stencil vocabulary.

The mechanism a rebuild must copy is the **variant mangling**: a single named program in a
material becomes one of several compiled objects. A vertex program's name is mangled with
the current skinning mode (the skeletal formulations differ by bone count and weight
layout); a pixel program's with the multisample sample count (per-sample execution differs
from per-pixel). The *variant* name is the cache key, the *original* name is what gets
compiled, and the variant is expressed as a macro. One source file, many compiled objects.
The macro set itself is in
[`../xrRenderPC_R4/r4_shaders.cpp`](../xrRenderPC_R4/r4_shaders.cpp.md).
[`dx11ResourceManager_Scripting.cpp`](dx11ResourceManager_Scripting.cpp.md) ·
[`Blender_Recorder_R3.cpp`](Blender_Recorder_R3.cpp.md) ·
[`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md)

---

## Deduplicate everything, find by content

Every factory in [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md)
has one shape — *search for an equal object, return it if found, otherwise build, register
and return* — and the interesting content of each is what "equal" means, because that
choice decides how much sharing happens. Programs compare by compiled blob; constant
blocks by layout and member names; vertex declarations by element list; passes by the
whole tuple of state, programs, table, textures and samplers.

Passes are what the draw stream is sorted by, so fewer distinct passes is fewer state
changes per frame. Deduplication is therefore not hygiene — it is the frame budget.

---

## Two subsystems with their own directories

- [**`StateManager/`**](StateManager/README.md) — interning and shadowing device state.
  Five ideas, none of them API-specific, that turn thirty thousand state settings per
  frame into a few hundred device calls.
- [**`3DFluid/`**](3DFluid/README.md) — an Eulerian fluid simulation on a regular grid,
  used for volumetric fog and fire, together with the ray-marched volume renderer that
  draws it. It lives here rather than in chapter 19 because it exists only on this
  filling; a rebuild may omit it entirely and lose only fog volumes.

---

## Files

| File | Role |
|---|---|
| [`CommonTypes.h`](CommonTypes.h.md) | **The translation table**: one name per device concept, mapped onto whatever the API calls it. A port replaces this file and little else |
| [`dx11HW.h`](dx11HW.h.md) | Declares the device object: bring-up, teardown, presentation, format probing, device-state query |
| [`dx11HW.cpp`](dx11HW.cpp.md) | Device and presentation-chain bring-up, capability-level negotiation, and the lost/reset lifecycle everything else hangs off |
| [`dx11HWCaps.cpp`](dx11HWCaps.cpp.md) | Fills the capability record the whole renderer branches on, including how many graphics processors the frame is pipelined across |
| [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) | **The draw path**: how a state change, a constant-table swap and a draw reach the device |
| [`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md) | The two buffer lifetimes, and the translation of the frozen vertex-layout description |
| [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md) | The deduplicating factory: everything exists once, found by content; and the program-variant mangling |
| [`dx11ResourceManager_Scripting.cpp`](dx11ResourceManager_Scripting.cpp.md) | The material description language and the five level-of-detail entry points every material answers |
| [`Blender_Recorder_R3.cpp`](Blender_Recorder_R3.cpp.md) | What a pass declaration turns into here: six programs, a merged constant table, named samplers, the stencil vocabulary |
| [`dx11r_constants.cpp`](dx11r_constants.cpp.md) | **By-name binding, resolved once**: a program's self-description becomes the engine's constant table |
| [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md) | The write side: one call fans a named value out to every stage that declared it |
| [`dx11r_constants_cache.cpp`](dx11r_constants_cache.cpp.md) | Routing a write to the right block for the right stage, and uploading every dirty bound block at draw time |
| [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) | Declares the constant block: typed setters, direct-write escape hatch, identity test |
| [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) | One constant block: host shadow, device buffer, and the layout fingerprint that lets programs share it |
| [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md) | A typed value to bytes at an offset — including the matrix storage convention the shipped shaders depend on |
| [`dx11SH_Texture.cpp`](dx11SH_Texture.cpp.md) | A bindable texture: four kinds of content behind one name, resolved to a per-frame binding function |
| [`dx11Texture.cpp`](dx11Texture.cpp.md) | Loading a texture from the virtual filesystem: search order, fallback, mip dropping, retry |
| [`dx11TextureUtils.h`](dx11TextureUtils.h.md) | Declares the two-way pixel-format translation |
| [`dx11TextureUtils.cpp`](dx11TextureUtils.cpp.md) | The pixel-format translation table, both directions |
| [`dx11SH_RT.cpp`](dx11SH_RT.cpp.md) | A render target: written by the output stage and sampled by a later pass at once, plus its device-loss rebuild |
| [`dx11StateUtils.h`](dx11StateUtils.h.md) | Declares the state-description vocabulary: conversions, defaults, normalization, comparison, hashing |
| [`dx11StateUtils.cpp`](dx11StateUtils.cpp.md) | **The rules that make state interning work**: every field's default and which fields are irrelevant given which others |
| [`dx11DetailManager_VS.cpp`](dx11DetailManager_VS.cpp.md) | Drawing the grass-and-debris layer: many copies of few meshes, instanced through a constant array, wind evaluated in the vertex program |
| [`dx11r_screenshot.cpp`](dx11r_screenshot.cpp.md) | Reading the finished frame back: a picture for the player, a save-file thumbnail, a tile for the tools |
| [`StateManager/`](StateManager/README.md) | State interning, shadowing and reconciliation — eleven files, its own README |
| [`3DFluid/`](3DFluid/README.md) | The Eulerian fluid solver and its volume renderer — sixteen files, its own README |
