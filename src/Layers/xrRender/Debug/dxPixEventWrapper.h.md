# src/Layers/xrRender/Debug/dxPixEventWrapper.h

> Names a region of the frame so a GPU capture tool shows a labelled tree instead of a flat list of draws.

**Needs** — [`../R_Backend.h`](../R_Backend.h.md) · [Seam: Profiler and GPU debugging](../../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1 as written, because the marker's extent is tied to a scope's lifetime; the *idea* is T4 — a begin/end pair around a region of work.

## Purpose

Every significant stage of the frame — the g-buffer fill, each shadow cascade, the light accumulation, each post-process — opens a named marker. A GPU capture tool renders those markers as a hierarchy, which is the difference between a capture you can read and four thousand anonymous draw calls.

The entire mechanism compiles to nothing in a shipping build. That is the decision worth recording: the markers are *everywhere* in the render path, they cost a driver call each, and the engine refuses to pay for them in a release the player runs.

## `scoped_gpu_marker(name)`

**Contract** — opens a named marker on a command list and closes it when the enclosing region of work ends. Nesting is expected and must be balanced. The name is a compile-time literal, never a formatted string — the markers sit in inner loops and building a name per frame would cost more than the marker.

```text
FUNCTION gpu_marker(command_list, name) DURING region_of_work
  command_list.mark_begin(name)
  ... the work ...
  command_list.mark_end()
```

**Invariants**

- Markers must nest, never overlap. Because the original ties the close to a lexical scope's end, this is structurally guaranteed; a rebuild with explicit begin/end calls must guarantee it some other way, since an unbalanced marker corrupts the capture's tree for the rest of the frame.
- The marker is opened against a *specific* command list, and the one that opens it must be the one that closes it. The renderer records several command lists in parallel, so a marker taken on the wrong one is a silent cross-thread bug.

**Notes** — The original spells this as two macros, one implicitly using the renderer's current command list and one taking it explicitly, whose only real job is to manufacture a unique local name so that two markers in one scope do not collide. That name manufacturing is pure C++ ceremony and survives as nothing.

The two backends reach two different tooling interfaces — one per-command-list, one global on the device — and the wrapper exists to hide that difference from the several hundred call sites. A rebuild has one such interface per graphics API and should assume it has to write this shim again.
