# src/xrCore/_stl_extensions.h

> An aggregation header: pulls in every container alias, every math type and the two hand-written collections, so that older code can include one file.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`xrstring.h`](xrstring.h.md) · [`xrMemory.h`](xrMemory.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`buffer_vector.h`](buffer_vector.h.md) · [`FixedVector.h`](FixedVector.h.md) · [`_vector2.h`](_vector2.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_color.h`](_color.h.md) · [`_rect.h`](_rect.h.md) · [`_plane.h`](_plane.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [`../xrCommon/xr_map.h`](../xrCommon/xr_map.h.md) · [`../xrCommon/xr_set.h`](../xrCommon/xr_set.h.md) · [`../xrCommon/xr_list.h`](../xrCommon/xr_list.h.md) · [`../xrCommon/xr_deque.h`](../xrCommon/xr_deque.h.md) · [`../xrCommon/xr_stack.h`](../xrCommon/xr_stack.h.md) · [`../xrCommon/xr_array.h`](../xrCommon/xr_array.h.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [`../xrCommon/xr_unordered_map.h`](../xrCommon/xr_unordered_map.h.md) · [`../xrCommon/predicates.h`](../xrCommon/predicates.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: it decides nothing.

## Purpose

Historical convenience. It exists so that a file can include one header and get containers, strings, math types and the two collections the engine wrote itself. It contributes no declarations of its own.

The source carries its own verdict — a note that the header "includes pretty much every standard collection there is" and is a compiler hog that should be fixed. A rebuild should not create it: the two collections it introduces that nothing else does are the caller-buffer dynamic array ([`buffer_vector.h`](buffer_vector.h.md)) and the fixed-capacity one ([`FixedVector.h`](FixedVector.h.md)), and consumers should include those directly.

## Exported units

None of its own.
