# `src/Layers/xrRenderGL` — the portable filling of the graphics-device seam

Chapter 21 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order), with
[`../xrRenderPC_GL`](../xrRenderPC_GL/README.md), which registers it as a selectable
renderer and supplies its final composition.

This directory is the body of the
[graphics-device contract](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) for
OpenGL. It is the filling that runs **everywhere the engine runs** — Windows, Linux, the
BSDs, macOS, Haiku — where the sibling [`../xrRenderDX11`](../xrRenderDX11/README.md) runs
on one operating system only. Both compile the same chapters
[18](../xrRender/README.md) and [19](../xrRender_R2/README.md) above them; nothing in those
chapters knows which of the two it is talking to.

Read the sibling chapter for *what the seam demands*: it states the eight obligations once,
and this filling meets the same eight. Read **this** chapter for a different question, the
one a rebuilder actually faces:

> **What does it cost to fill a seam that was cut to the shape of a different API?**

Because that is the honest description of this directory. The renderer interface the engine
declares carries Direct3D 9 heritage in its bones — the resource model, the granularity of a
state object, the way a constant is addressed, the vertex-declaration vocabulary, the
texture-origin and winding conventions, the sampler-attached-to-a-stage model, the
device-lost lifecycle. None of that is a fact about graphics hardware; all of it is a fact
about the interface's birthplace. This directory is a continuous act of translation, and the
translations are enumerated below so that a rebuilder can see, item by item, which ones exist
because of the *hardware* (keep them) and which exist because of the *seam's shape* (delete
them, and the whole file they live in with them).

## Where it sits

It rests on the core layer, the math layer, the renderer core of chapter 18 — whose command
list, resource registry, material compiler and texture model it supplies the bodies for — and
the deferred frame graph of chapter 19, which it compiles unchanged. It reaches the
windowing seam for its window and its drawing context, and the
[image-codec seam](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) for every shipped image
file, which on this path does considerably more work than on the other one.

It does *not* contain a frame graph, a visibility walk, a material compiler or a model
loader. Those are chapters 18 and 19 and are shared. What is here is: a device, a
translation of every value the shared core speaks in the older API's numbering, and the
bodies of the per-draw calls.

---

## The translations, named once

Each is a real decision with a real cost. The twins assume these and stay terse.

### 1. A state block is a *recording*, not an object

The interface hands the backend state in units: a pass resolves to one block covering depth,
stencil, blend, rasterizer and every sampler it uses, and chapter 18 interns blocks so that
two passes resolving to the same state are one block compared by identity. The other filling
turns each block into device objects created once at load and bound by handle.

This API has no such objects. A block here is a **passive key/value recording** of the older
API's render-state and sampler-state writes, played back on every bind. Playback splits three
ways, and the split is the load-bearing part:

- **Through the command list's redundancy filter** — cull mode, depth test, depth comparison,
  the whole stencil group, colour write mask. These cost a comparison when unchanged.
- **Straight at the global state machine** — depth write mask, blend enable, the separate
  colour and alpha blend factors, the separate blend equations. These are re-issued on every
  bind, changed or not.
- **Through sampler objects** — one per texture unit, created lazily the first time the block
  writes to that unit, bound to the unit at playback.

So chapter 18's interning still saves the *allocation*, but only part of the *traffic*. A
rebuilder porting to an API with pipeline objects gets the other part back for free.

Two details inside the translation are easy to get wrong and expensive to debug:

- **Winding is inverted.** The engine's "cull clockwise" becomes this API's "cull back face"
  and "cull counter-clockwise" becomes "cull front face". One table entry, one flipped world.
- **Three filter settings fold into one.** The interface sets minification, magnification and
  mip filtering as three independent values. This API expresses a sampler's minification and
  mip behaviour as a *single* enumerant whose numbering is bit-structured — one bit for linear
  filtering, one for linear mip interpolation, one for whether mips are consulted at all. So
  each of the three incoming writes must read the sampler's current value back and edit the
  relevant bit. A rebuild that writes the whole enumerant on each of the three loses two of
  them.

The mip level-of-detail bias is a console variable with no global equivalent here, so every
live sampler object is rewritten whenever it changes.

Settings that meant something only to the fixed-function pipeline the interface grew out of —
lighting enable, fog enable, alpha-test enable, alpha reference — are accepted and dropped.

### 2. Constants are program uniforms, set one at a time, immediately

This is the largest performance difference between the two fillings and the one a rebuilder
must plan around.

Chapter 18's model is *by name*: a material says "set `m_shadow`" and never learns where it
lives. That model survives intact, and it is resolved the same way at load — but against a
different oracle. The other filling reads a compiler-produced reflection blob and gets a
constant-block index and a byte offset. This filling asks the **linked program itself** to
enumerate its active uniforms, and each answer carries a name, a type, an array length and a
location. A name declared by two stages becomes one table entry holding both locations, so one
`set` still fans out to every stage that wants it, and setting a name no stage declared is
still a deliberate no-op.

What does *not* survive is everything below the name:

- **There are no uniform buffers.** A named value is written to its program the moment it is
  set. Nothing is batched, nothing is marked dirty, and the "upload every dirty block before
  the draw" step that the other filling depends on does nothing here.
- **Arrays are written element by element.** The grass layer's per-instance batch — a
  three-row transform plus a packed colour, per plant, up to a batch size per draw — reaches
  the device as four separate four-component writes per instance. The other filling maps one
  block and fills the whole batch in a single copy.
- **Matrices are transposed on every write.** The shipped sources are authored for the other
  storage order, so each matrix is gathered column-wise into scratch and uploaded with the
  transpose flag set. A matrix narrower than four columns is refused outright; only the
  4×4, 4×3 and 4×2 shapes exist.
- **Whether a program must be bound to be written depends on an extension.** Where separable
  programs are available, a uniform is set on a named program without binding it; otherwise
  the program must be current first, and the write order relative to the bind becomes
  load-bearing.

### 3. A sampler is a texture unit's second half

The other API gives every shader stage its own flat slot space and keeps textures and samplers
in *separate* spaces, so a material's pass declaration names both and resolves to two slot
numbers.

Here there is **one** shared space of texture image units, shared by all stages, and a unit is
where two independent objects meet: the texture is bound to the unit's target, the sampler
object is bound to the unit's index. The engine therefore partitions the one space by
convention — pixel-stage units first, then a fixed vertex-stage base, then a fixed
geometry-stage base, thirty-six units in all — and a constant of sampler type is assigned its
unit number from its position in the program's enumeration *plus* the base for its stage.

The capability record advertises a flag meaning "texture and sampler are one object", and the
material compiler reads it: instead of looking up a sampler name and an image name separately,
it looks the **image** name up once and uses that number for both. That flag is the whole
difference in the shared compiler between the two fillings.

Because a sampler uniform's binding to a unit is *program* state rather than device state, it
is re-established whenever the bound constant table changes, not once at load.

### 4. A vertex declaration is a vertex array object — and the semantics are a frozen table

The on-disk declaration form is the older API's element list and it stays: model files ship it,
so it is the truth (chapter 18 says this once). Interning a declaration creates a vertex array
object, and *that object is the interned declaration*. Unlike the other filling, it does not
depend on the bound program at all — no (declaration, program signature) memo, no lazy
creation on first draw.

That is possible only because of a decision a rebuilder must copy or consciously replace: this
API has no vertex *semantics*, so the engine publishes a **frozen numbering from semantic to
attribute index** — colour 0, position 3, tangent 4, normal 5, binormal 6, fog 7, texture
coordinates from 8 up — with the semantic's own index added on top. The declaration translator
and every shipped shader source for this backend must agree on that table, and it is as frozen
as any file format. Semantics with no number assigned — blend weights, blend indices, point
size, tessellation factor, depth — are dropped from the layout silently.

One consequence costs a day if it is not known in advance: **the index buffer binding belongs
to the array object**, so changing declaration silently changes which indices are bound. The
command list must therefore forget its cached index buffer every time the declaration changes,
or the next draw reads the wrong indices.

### 5. Clip space, texture origin, and where the flip is paid

Every one of these is paid **on the processor, in the engine, before the value reaches the
shader**. A rebuilder looking for a shader-side fix will not find one.

- **Texture origin — the screen texgen matrix.** The matrix that turns a clip-space position
  into a coordinate for sampling a full-screen target has its vertical scale **negated on the
  other filling and left positive here**. That single sign is the entire texture-origin
  difference, and the jitter texgen carries it too.
- **Clip-space depth — the shadow texgen matrix.** The matrix that turns a world position into
  a shadow-map lookup carries the *second* flip: its depth row is **halved and biased by a
  half** here, because this API's clip depth spans minus-one to one and the shadow map stores
  zero to one, while the other API's clip depth already arrives in zero to one. The other
  filling's version of the same matrix has a plain range term and no bias. Both flips live in
  the same four matrices, and a rebuilder who copies one and forgets the other gets shadows
  that are half-right in a way that looks like a depth-bias problem and is not.
- **The sun's own projections are built by a different math layer.** The legacy sun phase is
  the one place the flip is not a sign in a matrix literal: that phase is rewritten against a
  vector-math library whose *default* depth convention is this API's, and the substitution is
  the compensation. See [`../xrRenderPC_GL`](../xrRenderPC_GL/README.md).
- **Position reconstruction.** The camera half-angle tangents handed to the g-buffer
  decompression are negated on the other filling and not here.
- **Full-screen quads** are emitted with their two middle vertices swapped and, where the quad
  carries texture coordinates rather than clip positions, with the vertical coordinate
  inverted — because the quad must come out the same way round after the origin flip.
- **Every rectangle crossing into the device** — the scissor rectangle, and the rectangles of a
  rectangle-limited clear — has its vertical coordinate mirrored against the frame height,
  because this API's window origin is at the bottom. The final present needs no flip at all,
  because the whole pipeline already lives in that space.
- **Fragment outputs are still named by the other API's target semantics.** The link step binds
  those names explicitly to colour attachment zero, one and two, so the shipped sources keep
  their original output names.

The **viewport's** own near and far pair is passed through unchanged; the depth fix is entirely
in the matrices above.

### 6. Two texture format vocabularies, handled in two different places

- **Shipped image files go through the codec seam whole.** The codec parses the container,
  reports the target shape (flat, cube, volume, array), the mip chain and the raw block data,
  *and translates the file's format into this API's internal/external/type triple together with
  a per-channel swizzle*. The engine allocates immutable storage for the whole mip chain up
  front, uploads each level into it — compressed levels as blocks, with no decode — and
  installs the swizzle as texture state. The swizzle is what makes a single-channel file keep
  behaving the way the shipped shaders expect, and it is the honest answer to "what happens
  when the device lacks the format": on this path the codec answers it, not a table here. The
  one deliberate exception is a file that is already a lone red channel, left unswizzled so the
  greyscale-plus-alpha font textures read correctly.
- **Engine-created surfaces do not.** Render targets and the lookup volumes are created by
  name and format from the shared core's older-API format vocabulary, so those need a small
  hand table of their own — and it covers only what the engine actually asks for, which is a
  few dozen entries, not a format matrix.

One honest gap: the texture-quality setting's mip-drop is **computed on this path and used only
to report memory**. Every level in the file is uploaded regardless.

### 7. There is no swap chain; there is one framebuffer

The device owns exactly one framebuffer object, created when the context comes up and bound for
the whole frame. "Set the render targets" means attaching textures to that one object's colour
attachment points, attaching one texture to its *combined* depth-and-stencil point, declaring
which attachments the fragment stage writes to, and checking that the result is a complete
framebuffer. Three colour attachments are declared together, which is exactly the g-buffer of
chapter 19.

Consequences worth stating flat:

- **A render target and a depth target are the same object.** A target's depth handle *is* its
  colour handle; this API does not distinguish them. Depth and stencil share one attachment
  point and one format.
- **Clearing an unbound target attaches it first.** There is no "clear this view" call.
- **Presenting is a blit, then a swap.** The frame is copied from the owned framebuffer to the
  window's own framebuffer and the window is swapped. The back-buffer count is one; there is no
  chain to rotate.
- **A multisample resolve is a blit between two attachments** of the same framebuffer.
- **Slices do not exist.** Binding one slice of an array target for reading or writing does
  nothing here, and the capability record says so, which is why the sun's cascades need
  their own implementation in [`../xrRenderPC_GL`](../xrRenderPC_GL/README.md).

### 8. The shader source problem — solved by shipping a second corpus

Chapter 19 renders through programs it never writes: shader *source* ships with the game data
([§5 Shaders](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)), authored in the other
API's high-level dialect. This is named in the requirements as the single largest compatibility
hazard in a port, and it is worth seeing exactly how this backend answers it, because the answer
is not the obvious one.

**It does not translate at load time.** It reads its programs from a *different directory of the
game data* than the Direct3D backend does, and the registration module in
[`../xrRenderPC_GL`](../xrRenderPC_GL/README.md) refuses to offer the renderer at all when that
directory is absent. In other words: the translation is done **ahead of time, by hand, once**,
and the result is shipped as a second corpus that the project maintains. That is a legitimate
answer to the seam — the requirements permit a translation layer, and this is one, moved from
run time to authoring time — and it is the answer a rebuilder should weigh first, because a
faithful run-time translator for the whole shipped corpus is a project in itself.

What happens at load, per stage: the source is read whole and handed to the driver to compile.
Where separable programs are available, each stage becomes its own single-stage program object
and a pass's three stages are joined by a *pipeline* object; where they are not, the three stage
objects are linked into one monolithic program, the individual stages are dropped, and the
constant table is re-derived from the linked program with every stage marked at once.

The **interned unit is the triple** (vertex, geometry, pixel), and its cache key is the three
mangled names concatenated. The mangling is the same variant scheme the other filling uses and
chapter 20 describes: a vertex program's name carries the skinning formulation, a pixel
program's carries the sample index, one source becomes several compiled objects. Compiled
programs can be retrieved from the driver and re-loaded as binaries, which is what makes a warm
start affordable; the monolithic path does not have that and recompiles from source every time.

A stage source that is missing falls back to a stub rather than failing the level, and a failed
compile retries through the same stub once.

### 9. Queries: one kind, and no "not yet"

Only occlusion queries exist — a count of samples that passed. Timestamp queries are refused
outright, so chapter 18's GPU-timing vocabulary has no body on this filling.

More consequentially, **reading a result has no "not ready yet" answer**. The shared caller is
written as a spin — ask, and retry while the answer is "pending" — and on this filling the first
ask blocks inside the driver until the query resolves. Chapter 18's discipline of reading a
query a frame or more after issuing it is therefore not an optimization here; it is the only
thing standing between the light-visibility pass and a full pipeline stall.

### 10. What is simply absent

These are parts of the interface that meant something to the other API and mean nothing here.
They are listed because a rebuilder reading the interface will look for them, and because the
list *is* the measure of how much of the seam is archaeology:

| Interface obligation | Here |
|---|---|
| Set a fixed-function transform slot | Nothing happens |
| Texture factor, ambient colour | Nothing happens |
| Alpha-test reference value | Refused |
| Bind one slice of an array target for read or write | Nothing happens; array targets are not supported |
| Device-lost / reset / removed state | The device always reports *normal*; the lifecycle is never entered |
| Begin-scene / end-scene bracket | Empty |
| Flush pending constants before a draw | Empty |
| Input-layout description record | Declared, and carries nothing — a layout here needs no description beyond the declaration itself |
| Compute shaders | Declared unsupported |
| Timestamp queries | Refused |
| Multisampling, and the stencil-bit edge scheme built on it | Disabled at option resolution |
| Volumetric fog, the compute ambient-occlusion paths, the stencil-bounds optimization | Disabled or unimplemented |
| Screenshot for a save thumbnail or for tooling | Absent; only the player-facing capture exists |

The capability record is the other half of this picture. The other filling *negotiates* a
capability level with the hardware and branches the whole renderer on the answer. This one
largely **asserts** one: the shader profile names, the maximum simultaneous targets, the stencil
width, scissor and cube support and the vertex-cache size are constants in the source, and the
number of graphics processors the frame is pipelined across is reported as a fixed two. Only a
handful of things are genuinely probed — vertex texture fetch, anisotropic filtering, separable
programs, attribute binding, retrievable program binaries — and each of those is probed because
the code path genuinely forks on it.

---

## What a rebuild on a modern API deletes outright

The distinction that makes this chapter worth reading: some of the above is the price of the
*hardware generation*, and some is the price of the *seam's shape*. Only the second kind goes
away.

**Deleted, because the seam's shape created them:**

- The entire state-block replay of item 1. A modern API compiles depth, blend, rasterizer and
  the program set into one pipeline object; the interning chapter 18 already does becomes
  interning of that object, and the three-way playback split disappears along with the
  read-modify-write of the folded filter enumerant.
- The winding inversion, both texgen flips, the quad vertex reordering and the scissor
  mirroring of item 5. A rebuild picks one clip-space depth range and one texture-origin
  convention, fixes it once in the projection matrix, and authors its shader corpus against it —
  and then no matrix in the engine needs two spellings. Modern APIs settle this argument by
  agreeing with the older one, so a port to any of them deletes the whole item.
- The semantic-to-attribute numbering of item 4. Every modern API carries explicit locations in
  both the vertex layout and the shader source, so there is no frozen table to agree on and no
  silently dropped semantic.
- The texture-unit partitioning and the combined-sampler flag of item 3. Descriptor sets
  separate textures from samplers again — deliberately, and per stage — so the flat shared space
  and its per-stage base offsets are not reconstructed.
- The matrix transposition of item 2, which is a one-time authoring decision in a rebuild's own
  corpus.
- The render-target-view / attachment juggling and the depth-is-colour conflation of item 7.
  Modern APIs describe an attachment set once, as data.
- **The whole device-lost lifecycle.** It is already a no-op here, which is the evidence: it was
  never about hardware, it was about one API's presentation model.
- The older API's format numbering, the primitive-count-versus-index-count conversion, and the
  `D3D`-prefixed type vocabulary that the shared core still speaks.

**Kept, because they are the engine's own requirements, not the API's:**

- **By-name constant binding**, because the shipped material data spells names and expects an
  answer. Only the mechanism under the name changes.
- **The variant mangling**, because the shipped material data names one program and means
  several — one per skinning formulation, one per sample count.
- **The program cache keyed by source and macro set**, because the shipped corpus is large
  enough that a cold compile is a visible load-time cost.
- **The two-lifetime buffer model** — built once and uploaded whole, versus rewritten every
  frame in a ring with a discard at wrap — because it is a statement about the engine's data,
  not about any API.
- **The shipped shader corpus itself**, in whichever dialect a rebuild chooses to maintain it.
  This is data, and data is frozen.

---

## The files

Every file here is either a value translation, a device lifecycle, or the body of a call the
shared core declared. None of them contains a rendering algorithm; those are chapters 18 and 19.

### The device and its capabilities

| File | Role |
|---|---|
| [`glHW.h`](glHW.h.md) | Declares the device: context creation, the single framebuffer it owns, presentation, and the make-current calls a loading thread needs |
| [`glHW.cpp`](glHW.cpp.md) | Bringing the drawing context up on the windowing seam, the owned framebuffer that stands in for a swap chain, present as blit-then-swap, and the vertical-sync policy including its adaptive variant |
| [`glHWCaps.cpp`](glHWCaps.cpp.md) | Fills the capability record the whole renderer branches on — here mostly asserted rather than negotiated, and the page says which entries are real measurements |
| [`CommonTypes.h`](CommonTypes.h.md) | The compile-time half of the translation: every type the shared core still names in the older API's vocabulary, given a local meaning. A port replaces this file and little else |

### The draw path

| File | Role |
|---|---|
| [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) | **The hot path**: what a state change, a target change, a program change and a draw actually cost, the primitive-count-to-index-count conversion, and every place a rectangle's vertical coordinate is mirrored |
| [`glBufferUtils.cpp`](glBufferUtils.cpp.md) | The two buffer lifetimes, the map flags that mean *discard* and *append*, and **the frozen semantic-to-attribute numbering** that lets a declaration become a vertex array object without consulting a program |
| [`glr_constants.cpp`](glr_constants.cpp.md) | Building a pass's constant table out of the program's own uniform enumeration, and the one place a sampler constant is assigned its texture unit |
| [`glr_constants_cache.h`](glr_constants_cache.h.md) | The write side: a named value reaches the device immediately, once per stage that declared it, transposed if it is a matrix — and there is nothing to flush |

### State

| File | Role |
|---|---|
| [`glState.h`](glState.h.md) | Declares the state block: the recorded depth-stencil group, the blend group, the cull mode and one sampler object per texture unit |
| [`glState.cpp`](glState.cpp.md) | Applying a block: what is routed through the command list's redundancy filter, what is re-issued at the global machine every time, how a sampler object is created on first use, and how the mip bias is pushed to every live sampler when the setting changes |
| [`glStateUtils.h`](glStateUtils.h.md) | Declares the value translations |
| [`glStateUtils.cpp`](glStateUtils.cpp.md) | The value tables — comparison functions, stencil operations, blend factors and equations, address modes — including the **winding inversion** and the bitwise edit that folds three filter settings into one enumerant |

### Textures and targets

| File | Role |
|---|---|
| [`glTexture.cpp`](glTexture.cpp.md) | Loading a shipped image: where the file is looked for, the bump-map fallback, immutable storage plus per-level upload, and the channel swizzle that keeps a one-channel file behaving as the shipped shaders expect |
| [`glSH_Texture.cpp`](glSH_Texture.cpp.md) | A bindable texture: still image, video stream or timed sequence behind one name, each resolved once to a binding that activates a texture unit and binds to its target |
| [`glTextureUtils.h`](glTextureUtils.h.md) | Declares the format translation for engine-created surfaces |
| [`glTextureUtils.cpp`](glTextureUtils.cpp.md) | That table — deliberately small, because shipped images never pass through it |
| [`glSH_RT.cpp`](glSH_RT.cpp.md) | A render target: one texture that is simultaneously the colour attachment and the depth attachment, the size ceiling it is checked against, the rebuild-from-description path, and the blit that resolves one into another |

### Materials, resources, and the rest

| File | Role |
|---|---|
| [`Blender_Recorder_GL.cpp`](Blender_Recorder_GL.cpp.md) | The backend half of the material compiler: what a pass declaration turns into here — a program triple, a merged constant table, the stencil vocabulary, and the comparison-sampler preset every shadow lookup needs |
| [`glResourceManager_Resources.cpp`](glResourceManager_Resources.cpp.md) | The deduplicating factory for this filling: a declaration becomes a vertex array object, a pass's three programs become one interned pipeline under a mangled name, and the two linking strategies the extension set chooses between |
| [`glResourceManager_Scripting.cpp`](glResourceManager_Scripting.cpp.md) | The material description language and the five level-of-detail entry points every material answers |
| [`glDetailManager_VS.cpp`](glDetailManager_VS.cpp.md) | Drawing the grass-and-debris layer: wind evaluated in the vertex program, and the per-instance transform array written one four-component element at a time because there is no block to map |
| [`glr_screenshot.cpp`](glr_screenshot.cpp.md) | Reading the finished frame back for the player; the save-thumbnail and tooling captures are absent on this filling |
