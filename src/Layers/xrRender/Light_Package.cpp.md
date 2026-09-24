# src/Layers/xrRender/Light_Package.cpp

> Order the frame's lights so that the ones whose occlusion answer has not come back yet are drawn last, and the rest brightest-first.

**Needs** — [`Light_Package.h`](Light_Package.h.md) · [`light.h`](light.h.md) · [`r__occlusion.h`](r__occlusion.h.md)
**Used by** — reached through its declarations in [`Light_Package.h`](Light_Package.h.md); callers name that, not this file.
**Tier floor** — T2: a stable sort over three small lists.

## Purpose

Nine lines of real content, and the whole of it is one comparison function that encodes how the deferred lighting pass wants its work ordered.

## `sort()`

**Contract** — stably sorts each of the three buckets by the ordering below. Called once per frame after the visibility pass has filled them; allocates nothing.

```text
COMPARE a, b:
  IF a is awaiting an occlusion answer
    IF b is too          -> the one issued EARLIER comes first
    ELSE                 -> b comes first
  ELSE
    IF b is awaiting     -> a comes first
    ELSE                 -> the one with the LARGER range comes first
```

**Invariants**

- **Lights still waiting on a hardware occlusion query go last.** The query was issued earlier in the frame and its answer is not back; drawing everything else first gives the device the maximum time to finish it before the renderer has to either read it or give up. This is latency hiding, and it is the only reason the ordering is not simply by range.
- **Among the waiting lights, the earliest-issued comes first** — the one most likely to have an answer by the time it is reached. The query order is a monotonically increasing counter per frame; see [`r__occlusion.cpp`](r__occlusion.cpp.md).
- **Among the settled lights, the largest range comes first.** A big light covers more of the screen, so drawing it first fills the depth buffer and lets the smaller lights' own depth bounds reject more pixels.
- **The sort is stable.** Two lights with the same range keep the order the visibility pass produced, which is spatially coherent. An unstable sort here would shuffle them frame to frame and make the frame time noisier for no reason.

**Notes** — The whole function exists only on the renderer generations that have a deferred lighting pass. The oldest one lights forward, with a fixed number of lights per object chosen elsewhere, and has no use for a frame-wide light list at all.

## `clear()`

**Contract** — empties the three buckets, keeping their capacity. Called once per frame from the light database's update; see [`Light_DB.cpp`](Light_DB.cpp.md).
