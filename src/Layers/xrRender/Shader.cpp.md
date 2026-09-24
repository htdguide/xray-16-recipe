# src/Layers/xrRender/Shader.cpp

> The compiled form of a material: a shader is a set of render-mode variants, each a short list of passes, each pass a bound set of device state, programs, textures and animated values — plus the equality rules that let identical ones be shared.

**Needs** — [`Shader.h`](Shader.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Constant.h`](SH_Constant.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: a pass is a bundle of device handles whose release order is load-bearing, and every one of these records unregisters itself from the resource registry as it dies.

## Purpose

A **blender** (see [`Blender.cpp`](Blender.cpp.md)) is a template; this file holds what a template compiles *to*. The compiled form is a four-level tree — shader, element, pass, and the four bound lists a pass carries — and its shape is the single most important data structure in the chapter, because every draw item in the frame points at one element of one shader and the sort key is built from that element's flags.

The file itself is thin. It contains the equality predicates (which decide whether two compilations may be shared) and the destructors (which unregister). Everything that *builds* this tree is in the resource manager and the blender compiler. It is a separate file because the tree is the shared vocabulary of the whole renderer, not because the code here needs its own home.

## State

```text
RECORD Shader                       # what a material name resolves to
  E : list<ShaderElement> of exactly 6 slots, any of which may be absent
```

The six slots are **render-mode variants of one material**, and their meanings differ by renderer generation. This is a frozen convention: the shipped material scripts fill slots by index, and the draw code selects by index.

```text
generation 1 (forward)            generation 2+ (deferred)
  0  normal, high detail            0  deferred fill (geometry buffer)
  1  normal, low detail             1  normal, low detail / forward
  2  additive pass: point light     2  point-light shadow map
  3  additive pass: spot light      3  spot-light shadow map
  4  lighting for models            4  directional shadow map
  5  (unused in both)
```

```text
RECORD ShaderElement                # one render mode of one material
  flags.priority     : int in [0,3]  # coarse draw-order bucket; default 1
  flags.strict_b2f   : bool          # sort back-to-front within the bucket
  flags.emissive     : bool          # this surface adds light; skip it in the light-only passes
  flags.distort      : bool          # this surface writes into the screen-distortion buffer
  flags.wallmark     : bool          # this surface may receive decals
  passes             : list<SPass>, at most 2

RECORD SPass                        # one draw of the surface
  state      : StateBlock            # depth, blend, raster, samplers — set as one unit
  vs, ps     : Program, optional     # absent means the fixed-function path
  gs         : Program, optional
  hs, ds, cs : Program, optional     # tessellation and compute, newest backend only
  constants  : ConstantTable, optional  # the merged named-constant table of the bound programs
  T          : TextureList           # (stage -> texture) bindings
  C          : ConstantList          # up to 4 animated colour constants
  M          : MatrixList            # up to 4 animated texture-coordinate matrices

RECORD STextureList  = list<(stage : int, texture : Texture)>
RECORD SConstantList = list<Constant>, at most 4
RECORD SMatrixList   = list<Matrix>,  at most 4

RECORD SGeometry                     # a drawable binding, not part of the material
  declaration : VertexDeclaration
  vertices    : VertexBuffer
  indices     : IndexBuffer
  stride      : int                  # bytes per vertex; cached from the declaration
```

**Invariants**

- **At most two passes per element.** This is a hard cap, not a convention: the shipped materials never need more, and the draw stream's batching assumes the count is small and fixed. A rebuild may raise it; nothing in the shipped data requires it to.
- `priority` occupies two bits and `strict_b2f` one, and they sit adjacent in the flags on purpose — the sort key is built by concatenating them (see the draw-stream discussion). The default element is priority 1 with every other flag clear.
- Every record here is reference counted and **interned**: two compilations that compare equal share one instance. Equality is therefore not a convenience, it is the identity relation of the cache.
- Equality at every level is **by identity of the children**, never by value: two passes are equal when they point at the *same* state block, the *same* programs, the *same* texture list. This works only because the children are themselves interned, and it is what makes the comparison cheap enough to run on every compilation.
- A `Shader` comparison walks only the **first five** slots. The sixth is compared by nobody, which means two shaders differing only in slot 5 would be merged. Slot 5 is unused in both generations, so this is latent rather than wrong.
- A texture list's stage numbers are not dense and not sorted by construction; they are whatever stages the material bound. Comparison is positional, so two lists with the same bindings written in a different order do *not* compare equal and will not be shared. That is a missed sharing opportunity, not a correctness problem.
- Geometry is *not* part of a material. It appears in this file because it is interned by the same registry and released by the same discipline.

## `Shader.equal` · `ShaderElement.equal` · `SPass.equal`

**Contract** — structural equality used by the resource registry to decide whether a freshly compiled object may be replaced by an existing one. Pure; no side effects. Compares flags by value and children by identity, as described above.

```text
FUNCTION pass_equal(a, b) -> bool
  RETURN a.state = b.state AND every program slot identical
         AND a.constants = b.constants
         AND a.T = b.T AND a.C = b.C
         # the matrix list is compared only in the authoring build

FUNCTION element_equal(a, b) -> bool
  RETURN flags equal in all five bits
         AND same pass count
         AND passes identical position by position

FUNCTION shader_equal(a, b) -> bool
  RETURN element_equal for each of slots 0..4, where absent = absent counts as equal
```

**Notes**

- The merged constant table is compared even though it is derived from the programs, which already compare equal — the source itself flags this as possibly redundant. It is redundant in the game build; a rebuild may drop it.
- The animated *matrix* list is compared only in the authoring tools. In the game build two passes differing only in their texture-coordinate animation are considered equal and merged, which would be a visible bug if the game's materials ever bound matrices — they do not, because on the deferred path the animation is folded into the program. A rebuild that keeps the matrix path must compare it everywhere.

## `STextureList`

**Contract** — the (stage, texture) bindings of one pass, as an ordered list rather than a map. Owns references to its textures; clearing it releases every one.

### `find_texture_stage`

**Contract** — given a texture *name*, return the stage it is bound to, or the invalid marker. Linear. Asserts by default when the name is absent, because the callers are code that knows the material's own binding and a miss is a mismatch between code and data.

**Notes** — Both this and the next function carry a warning in the source not to use them: they reverse a lookup that the compilation already knows the answer to, at a cost paid per call. They exist for code that receives a finished material and wants to substitute one texture — the detail-texture and terrain paths. A rebuild should let the compiler emit named slots so the reverse lookup is unnecessary.

### `create_texture`

**Contract** — bind a texture by name into a given stage, either filling an empty slot or replacing whatever is there. Only touches slots whose stage number matches; a stage that the material never declared is not added. That is the important part: this *overrides* a binding the material made, it does not extend the material.

## `SGeometry` creation

**Contract** — build a drawable binding from a vertex declaration (or a legacy packed format identifier), a vertex buffer and an index buffer. Delegates to the registry so identical bindings are shared, and caches the vertex stride so the draw path never re-derives it.

**Notes** — Two spellings exist, one taking a declaration and one taking a packed format word from the older graphics generation. The shipped model formats use the packed word; the renderer's own geometry uses declarations. Both must exist because both appear in the data.

## Destruction

**Contract** — every record in this file unregisters itself from the resource registry when its last reference goes. The registry is a map from key to a raw pointer that it does *not* own; the object removing itself is what keeps the map from holding a dangling entry.

**Notes** — This is the incidental half of the file: the whole set of destructors is one idea, "a registered resource removes itself on death", written seven times. A rebuild expresses it once. What must survive is the *ordering hazard* the source flags in a comment: these objects can outlive the registry when the script layer holds references, and the teardown must release script-held materials before the registry goes. A rebuild that ties resource lifetime to an explicit scope avoids the hazard entirely.
