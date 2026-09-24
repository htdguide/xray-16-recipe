# `src/Layers/xrRender/Utils/` — the state-description key

The renderer builds far more graphics-state descriptions than there are distinct ones:
every pass every material compiles names a rasterizer, blend, depth-stencil and sampler
configuration, and the overwhelming majority of them repeat. The state cache deduplicates
them, and comparing a few dozen fields at every lookup is wasteful — so each description is
reduced to a 32-bit key, the cache buckets on the key, and a full comparison runs only on a
collision. That reduction is all this directory is.

## Where it sits

A leaf of chapter 18 with no dependencies at all — not on the device, not on the core
layer. It is a separate directory because it is the only part of the backends' state
deduplication that touches nothing.

The construction is a reflected checksum with the common archive polynomial, and **none of
that is load-bearing**: nothing persists these values, nothing outside the process sees
them, and any hash with good avalanche over a few dozen bytes does the job. The engine
already has a checksum of its own for archive verification, and this being a second copy of
the same idea is duplication with no reason behind it; a rebuild should have one.

The one rule that does survive is a trap. The key must depend on **every byte the caller
feeds it and on nothing else** — which means a caller may not hand over a state description
wholesale, because it would fold in the layout padding between fields, which is
uninitialized, which would make two identical states produce different keys and defeat the
cache entirely. Callers therefore add each field by name, one at a time. A rebuild must
keep that discipline however it hashes, either by hashing field by field or by choosing a
representation with no padding in it.

| File | Role |
|---|---|
| [`dxHashHelper.cpp`](dxHashHelper.cpp.md) | The running fold and its shared lookup table: start from all-ones, add bytes, complement at the end |
| [`dxHashHelper.h`](dxHashHelper.h.md) | Declares the hasher, with the per-byte step placed for inlining because it runs once per byte of every state description built during level load |
