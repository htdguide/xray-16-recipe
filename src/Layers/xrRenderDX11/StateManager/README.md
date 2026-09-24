# `src/Layers/xrRenderDX11/StateManager` — interning and shadowing device state

Part of chapter 20, [the Direct3D 11 filling](../README.md) of the
[Graphics device seam](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device).

This directory answers one question: **how does a frame set thirty thousand pieces of
render state while making a few hundred device calls?** Every file here is one half of
that answer. Nothing in it is specific to the graphics API it was written against — the
same five ideas apply unchanged to a rebuild on Vulkan, Metal or WebGPU, and on those
APIs at least three of them are not optional.

It rests on nothing but the device object ([`../dx11HW.h`](../dx11HW.h.md)) and the
field-level rules in [`../dx11StateUtils.cpp`](../dx11StateUtils.cpp.md). Everything
above it — the command backend, the material system, the frame graph — reaches state only
through this directory.

---

## The ideas you need before the twins make sense

**State comes in blocks, not fields.** The device does not accept "turn depth writes
off"; it accepts a *depth-stencil object* describing all of depth and stencil at once,
created ahead of time and bound as a unit. There are three such objects — rasterizer,
depth-stencil, blend — plus a sampler object per texture slot per shader stage. This is
the shape every modern graphics API has converged on, and it is why "the state manager"
is a module rather than a function.

**Interning is what makes blocks affordable.** A level's materials describe several
thousand passes. Their state descriptions are *nearly all duplicates*: a handful of
distinct rasterizer settings, a few dozen depth-stencil settings. Each description is
therefore looked up in a cache that returns an existing device object for an identical
description and creates one only on a miss. This happens once, at load time. The payoff
is not memory — state objects are tiny — it is that at draw time **binding reduces to a
pointer comparison**: two passes that share a state object need no device call between
them.

**Interning only works if descriptions are normalized first.** Descriptions carry fields
that are *irrelevant given other fields* — a depth bias when bias is off, an anisotropy
level when the filter is not anisotropic, a per-target blend equation when blending is
disabled. Two descriptions differing only in such a field describe identical behaviour
but hash differently, and a naive cache creates two objects for them. Forcing every
irrelevant field to a fixed value before hashing is the step that turns a correct cache
into an effective one. The rules for what is irrelevant when are the load-bearing content
of [`../dx11StateUtils.cpp`](../dx11StateUtils.cpp.md) and a rebuild must write them out.

**A hash is a filter, and identity is confirmed against the object, not the request.**
A hash hit is verified by a full field comparison, so a collision costs a comparison
rather than a wrong pipeline state. The comparison is made against the description read
back *out of the created object*, because a driver may legitimately normalize fields of
its own; comparing against a stored copy of the request would create a second object the
driver would have folded into the first.

**Handles, not pointers, where a global setting can change.** Two sampler fields —
maximum anisotropy and mip-level bias — are graphics options the player controls, not
properties of a material, and they change at run time. So the sampler cache hands out
*indices*. When the setting changes, every cached sampler is rebuilt and the array slot
overwritten; every stored handle stays valid and now names the new object. A rebuild that
hands out object references has to find and patch every reference instead.

**Shadow the slots, bind the range.** Textures change more often than anything else in a
frame. Binding one slot at a time is one device call each; binding a whole stage is one
call but re-sends every slot. The third option wins: shadow every slot, track the lowest
and highest slot changed since the last draw, and bind exactly that inclusive run once.
It covers unchanged slots in the middle, which is the right trade because the slots one
material uses are adjacent. This is the single most transferable idea in the directory,
and the constant-table diff in [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md)
is the same idea applied to constant blocks.

**Two callers want opposite things, and both are served.** A material pass hands over a
fully resolved block — three objects, ready to bind. Older engine code in chapters 18 and
19 sets *one field at a time*, in the vocabulary of the fixed-function generation ("depth
test off", "cull the other way"). The front end keeps both paths alive without either
paying for the other: the resolved path is a comparison and a bind; the field path
reconstructs a description, mutates the field and re-interns at draw time, which costs a
cache lookup rather than an object creation once that combination has been seen. A rebuild
that gets to delete the field-at-a-time vocabulary should; as long as shared upper layers
speak it, this reconciliation is the price.

---

## Ownership and lifetime — the trap

State objects are owned **by the caches and by nothing else**, for the life of the device.
Everything that stores one — every material pass, every command list's current binding —
holds a weak reference and must never release it.

The consequence is the rule that a rebuild will otherwise discover the hard way: **a
resolved pass state must not outlive a device reset.** When the device is lost and
rebuilt, the caches are cleared, and every material's resolved state must be rebuilt with
them. See [the device lifecycle](../dx11HW.cpp.md).

---

## Reconciliation, per pipeline-object kind

Each of the three pipeline kinds carries three flags, and the invariant between them is
what keeps the two calling styles from corrupting each other:

| Flag | Meaning |
|---|---|
| `need_apply` | the bound object is not what the device currently has |
| `changed` | the description was edited; re-intern it before binding |
| `desc_stale` | the description no longer describes the bound object; refresh it first |

`changed` and `desc_stale` are never both true: setting a resolved object clears `changed`
and sets `desc_stale`; editing a field refreshes the description first (clearing
`desc_stale`) and then sets `changed`. A rebuild that keeps only a dirty bit will
eventually intern a description belonging to a different object.

---

## Files

| File | Role |
|---|---|
| [`dx11StateCache.h`](dx11StateCache.h.md) | Declares the three interning caches — rasterizer, depth-stencil, blend — and the record each keeps |
| [`dx11StateCacheImpl.h`](dx11StateCacheImpl.h.md) | **The interning algorithm**: normalize, hash, confirm by read-back, create-or-reuse |
| [`dx11StateCache.cpp`](dx11StateCache.cpp.md) | Instantiates the three caches and supplies the one operation that differs between them |
| [`dx11State.h`](dx11State.h.md) | Declares the resolved per-pass state block: three pipeline objects, six sampler arrays, two reference values |
| [`dx11State.cpp`](dx11State.cpp.md) | Compiles a material pass's recorded state block into shared device objects at load time |
| [`dx11StateManager.h`](dx11StateManager.h.md) | Declares the per-command-list front end and its three-flag reconciliation |
| [`dx11StateManager.cpp`](dx11StateManager.cpp.md) | The front end: whole blocks and single fields reconciled lazily, at most three objects bound per draw |
| [`dx11SamplerStateCache.h`](dx11SamplerStateCache.h.md) | Declares the sampler cache: handles instead of pointers, one bind per stage, two globally overridden fields |
| [`dx11SamplerStateCache.cpp`](dx11SamplerStateCache.cpp.md) | Sampler interning, whole-stage binding, and rebuild-in-place when the filtering option changes |
| [`dx11ShaderResourceStateCache.h`](dx11ShaderResourceStateCache.h.md) | Declares the per-stage bound-resource shadow and its dirty range |
| [`dx11ShaderResourceStateCache.cpp`](dx11ShaderResourceStateCache.cpp.md) | The slot shadow: scattered texture changes collapsed into one contiguous rebind per stage per draw |
