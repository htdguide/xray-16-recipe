# src/Layers/xrRenderDX11/dx11ConstantBuffer_impl.h

> How a typed value becomes bytes at an offset inside a constant block — including the matrix storage convention the shipped shaders depend on.

**Needs** — [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) · [`xrRender/r_constants.h`](../xrRender/r_constants.h.md)
**Used by** — [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) · [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md)
**Tier floor** — T1: it writes floats at computed byte offsets into a shared image; every call is on the per-draw path and must inline.

## Purpose

The bodies of the constant block's setters live here rather than in the implementation file for one reason: they are called several times per draw and must inline into the renderer's inner loops. What they encode, though, is not a performance trick but a **storage convention**, and that convention is frozen because the shipped shader sources were written against it.

## `write access`

**Contract** — given a byte offset, returns the address inside the host image and marks the block dirty. Bounds-checked in debug builds only; the caller is responsible for the *span*, because the block cannot know how many bytes the caller is about to write.

**Notes** — Every write marks the whole block dirty. The source flags the missing refinement: a write should first compare against the stored value and skip the dirty mark when nothing changed. As written, setting a constant to the value it already holds costs a full block upload.

## `set` — scalar and vector

**Contract** — writes a scalar float, a scalar integer, or the first two, three or four components of a vector, at the constant's byte offset. The declared element type is asserted against the value's type (a float value into an integer constant is a programming error, not a conversion), and the declared shape decides how many components are copied. A three-component constant receives three floats and the fourth byte group is left as it was — the shader does not read it.

## `set` — matrix

**Contract** — writes a matrix into a constant declared as two, three or four rows of four. **The matrix is transposed on the way in**: each stored line is a *column* of the source matrix.

```text
FUNCTION write_matrix(offset, M, shape)
  # each destination line is 4 floats; the source is row-major in memory
  line[0] = (M.m11, M.m21, M.m31, M.m41)
  line[1] = (M.m12, M.m22, M.m32, M.m42)
  IF shape has 3 or more lines THEN line[2] = (M.m13, M.m23, M.m33, M.m43)
  IF shape has 4 lines         THEN line[3] = (M.m14, M.m24, M.m34, M.m44)
```

**Invariants** — This is the whole reason the engine's matrices and the shipped shaders agree. The engine stores matrices row-major; the shader compiler lays constants out expecting the transpose. A rebuild must either reproduce this transpose at write time or transpose in the shader — and the shader sources ship with the game, so it must be done here.

The two-line and three-line shapes exist because an affine transform's last column is always (0,0,0,1) and a shader that only needs the upper rows should not pay to upload the rest. A block declared with a shape other than 2×4, 3×4 or 4×4 rows is a fatal error, as is a column-major declaration or a structure — the engine simply does not support them, and a shader author who writes one gets told by name which constant is at fault.

## `seta` — array element

**Contract** — the same writes, addressed at `base_offset + element_index * bytes_per_element`, where bytes_per_element is four floats for a vector member and two, three or four lines for a matrix member. This is how the engine uploads bone matrix palettes for skinned meshes and per-light arrays: one constant declaration, many elements, each written individually.

## `AccessDirect`

**Contract** — returns the address of a span inside the host image for the caller to fill itself, marking the block dirty, or nothing when the span would cross the end of the block. Used where the data is already laid out correctly in memory and a per-element copy would be wasted work.
