# src/xrCore/xr_types.h

> The scalar vocabulary: fixed-width integer names, the pointer-to-text names, the numeric limits, and the fixed-size character buffer names that the whole engine's stack-allocated strings are declared with.

**Needs** — _(none — it is the base of the chain)_
**Used by** — [`object_interfaces.h`](../Common/object_interfaces.h.md) · [`data_storage_binary_heap.h`](../xrAICore/Navigation/data_storage_binary_heap.h.md) · [`vertex_path.h`](../xrAICore/Navigation/vertex_path.h.md) · [`FixedVector.h`](FixedVector.h.md) · [`_bitwise.h`](_bitwise.h.md) · [`_color.h`](_color.h.md) · [`_compressed_normal.h`](_compressed_normal.h.md) · [`_flags.h`](_flags.h.md) · [`_math.h`](_math.h.md) · [`_random.h`](_random.h.md) · [`_stl_extensions.h`](_stl_extensions.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_vector4.h`](_vector4.h.md) · [`client_id.h`](client_id.h.md) · _and 8 more_
**Tier floor** — T1: it names types by exact bit width because the on-disk and on-wire formats are byte images of them.

## Purpose

Every width in this engine is explicit. Level files, network packets, archive headers and vertex layouts are all byte images, so "an integer" is never good enough — a field is 8, 16, 32 or 64 bits and signed or not, and this file is where those names come from. The file also fixes a convention a rebuild will find everywhere: strings are stack buffers of a declared size, and the size is part of the type's name.

## Exported units

- **Fixed-width integers** — signed and unsigned at 8, 16, 32 and 64 bits.
- **Floating-point** — 32-bit and 64-bit names. 32-bit is the working type; 64-bit appears only in the physics solver.
- **Text pointers** — mutable text, constant text, and the two constness variations of the pointer itself. The distinction between "pointer to constant text" and "constant pointer to text" is spelled out because both appear in the interfaces.
- **Numeric limits** — maximum, minimum, smallest-magnitude and epsilon, as a family parameterized by type, with named shorthands for the integer, float and double cases.
- **Maximum path length** — the platform's limit, named once.
- **Fixed-size character buffer names** — at 16, 32, 64, 128, 256, 512, 1024, 2048 and 4096 bytes, plus a path-sized one at twice the platform's path limit.

## Notes

**The "minimum" of a type is defined as the negative of its maximum, not as the type's true lowest value.** For a signed integer those differ by one; for a float they are the same. Code that uses the integer minimum as a sentinel is therefore one away from the real bound. Nothing in the engine depends on the difference, but a rebuild copying the definition should know it is not the language's own minimum.

**The "zero" of a type is the language's smallest positive value** — which for a float is the smallest positive subnormal and for an integer is the true minimum. The name is misleading for both. It is used as a "negligibly small" threshold for floats and essentially not at all for integers.

**The path buffer is twice the platform's path limit.** That is a deliberate margin: the engine builds paths by concatenating a resolved root with a relative path and wants the intermediate result to fit even when the final one would not. The platform assumption that paths must not exceed the limit applies to the *final* path, not to this buffer.

The fixed-size buffer names are how every stack-allocated string in the engine is declared, and the size-deducing string helpers in [`_std_extensions.h`](_std_extensions.h.md) recover the size from the type — so the declared size is the bound that is actually enforced. A rebuild with owning strings deletes the whole family; a rebuild that keeps stack strings must keep the size in the type or lose the bound.
