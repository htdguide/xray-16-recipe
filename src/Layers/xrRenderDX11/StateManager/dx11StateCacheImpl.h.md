# src/Layers/xrRenderDX11/StateManager/dx11StateCacheImpl.h

> The interning algorithm shared by the rasterizer, depth-stencil and blend caches: normalize, hash, confirm, create-or-reuse.

**Needs** — [`dx11StateCache.h`](dx11StateCache.h.md) · [`../dx11StateUtils.h`](../dx11StateUtils.h.md)
**Used by** — [`dx11StateCache.h`](dx11StateCache.h.md) · [`dx11StateUtils.cpp`](../dx11StateUtils.cpp.md)
**Tier floor** — T1: it hashes a device description structure by its bytes.

## Purpose

One algorithm serves three kinds of state object, so it is written once against a description type and a device-object type. What matters to a rebuild is not the parameterization but the four steps and why each exists.

## `intern a description`

**Contract** — given a description, returns the single shared device object for it, creating one on first sight. Never fails; a creation failure is fatal.

```text
FUNCTION intern(desc) -> handle
  normalize(desc)                 # see Notes: "validate"
  key = hash(desc)                # over the meaningful fields only
  FOR EACH record IN cache
    IF record.key == key THEN
      candidate = describe(record.object)      # read the description back from the object
      IF candidate equals desc THEN RETURN record.object
  object = device.create_state(desc)
  cache.append({key, object})
  RETURN object
```

**Invariants** — The hash is a filter, never the identity: a hash hit is confirmed by a full field comparison, so a collision costs a comparison rather than a wrong state. Equally important, the comparison is against the description **read back from the created object**, not against a stored copy — the driver may legitimately normalize fields, and comparing against the read-back form is what stops the cache creating a second object that the driver would fold into the first anyway.

**Notes** — *Normalize* (`ValidateState` in the source) is the step that makes the cache effective rather than merely correct, and a rebuild that skips it will create many near-duplicate objects. Description structures contain fields that are *irrelevant given other fields* — a depth bias when depth bias is off, an anisotropy level when the filter is not anisotropic, a per-target blend equation when blending is disabled. Two descriptions that differ only in such a field describe the same behaviour. Normalization forces every irrelevant field to a fixed value first, so those two descriptions hash and compare equal. See [`../dx11StateUtils.cpp`](../dx11StateUtils.cpp.md), which owns the rules for which field is relevant when.

A second entry point takes the material's *recorded state block* instead of a description: it resets a description to the defaults and replays the block into it, then interns. That is the path [`dx11State.cpp`](dx11State.cpp.md) uses, and it is where the default value of every unset field is decided.

The search is a linear scan. The number of distinct states in a level is in the tens, and interning happens at load time, so a map would buy nothing.
