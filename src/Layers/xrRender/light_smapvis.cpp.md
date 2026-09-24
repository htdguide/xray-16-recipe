# src/Layers/xrRender/light_smapvis.cpp

> Learns, one candidate per frame, which visuals inside a light's volume cast no visible shadow, and then skips them for as long as the light does not move.

**Needs** — [`light_smapvis.h`](light_smapvis.h.md) · [`light.h`](light.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`r__occlusion.h`](r__occlusion.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`light_smapvis.h`](light_smapvis.h.md)
**Tier floor** — T1: the marking step writes a per-context frame stamp directly into each visual's visibility record, which is the same field the scene-graph builder tests.

## Purpose

Rendering a shadow map means drawing every shadow caster in the light's volume. Many of those casters are behind others and contribute nothing: a crate behind a wall, from the lamp's point of view, changes no shadow texel. Testing all of them every frame with occlusion queries would cost more than drawing them.

This file is the compromise: **one** caster is tested per frame, and the answer is remembered until the light moves. Over a couple of seconds a static light learns its whole invisible set, and thereafter pays nothing. One of these lives per light per render context.

## State

```text
ENUM Phase { counting, learning, using_cache }

RECORD ShadowCasterVis
  phase           : Phase
  invisible       : list<Visual>   # learned; skipped every frame
  sleep_until     : int            # frame before which learning does not start
  candidate_total : int            # how many casters there were when learning began
  candidate_index : int            # which one is being tested now
  pending_visual  : optional<Visual>
  pending_query   : int
  pending_frame   : int            # the frame the query's result becomes available
  context_id      : int            # which render context this instance serves
```

Invariants:

- `context_id` is stamped at construction and never changes. It selects which render context this instance marks visuals in, and the mark is written into a **per-context slot** of the visual's visibility record — that is what lets several contexts build scene graphs concurrently without fighting over one visual.
- `invisible` is valid only while the light has not moved. Any placement change calls `invalidate`, which clears it and restarts at `counting`. This is the correctness condition of the whole scheme.
- `pending_visual` is set only in the `learning` phase, and at most one query is outstanding per instance.
- Entering `invalidate` sets `sleep_until` to a fixed number of frames ahead. Learning does not begin until then.

## Phases

The three phases are a state machine driven from `begin`/`end` around each shadow-map build.

```text
counting     -> learning       once the sleep interval has passed; records how many
                               casters the graph produced as the candidate total
learning     -> using_cache    once every candidate has been tested
any          -> counting       on invalidate (the light moved, or a setting changed)
```

**Notes** — The sleep interval exists because a light that has just appeared or just moved is likely to move again — a thrown flare, a swinging lamp, a muzzle flash. Spending queries on it would be wasted. The interval is a configured constant; nothing derives it.

## `begin`

**Contract** — Called before the light's shadow-caster graph is built, and before the graph's frame marker is advanced. Resets the context's caster counters, and then, depending on phase:

```text
FUNCTION begin()
  context.reset_counters()
  CASE phase OF
    counting:
      # nothing: this pass exists only to find out how many casters there are
    learning:
      forget any pending candidate
      mark_invisible()
      # Ask the graph to call back when it reaches the candidate-th caster.
      context.set_feedback(this, at_index = candidate_index)
    using_cache:
      mark_invisible()
```

## `mark_invisible`

**Contract** — Stamp every learned-invisible visual with the frame marker the graph is *about* to use, in this context's slot. The graph builder skips any visual already stamped with the current marker, so this pre-stamping is exactly "pretend these were already inserted". Also adds the skipped count to the frame's culling statistics.

**Invariants** — It must run *before* the marker is advanced, and it stamps `marker + 1` for that reason. Stamping the current marker would skip nothing; stamping after the advance would skip the wrong frame.

## `end`

**Contract** — Called after the shadow-caster graph is built. Harvests the caster count into the frame statistics, clears the feedback hook, and advances the state machine.

```text
FUNCTION end()
  (static_count, dynamic_count) = context.get_counters()
  frame_stats.casters_total += static_count
  context.set_feedback(none)

  CASE phase OF
    counting:
      IF the sleep interval has passed
        candidate_total = static_count
        candidate_index = 0
        phase = learning

    learning:
      # The feedback hook fired during the build and left us a candidate.
      # Draw that one visual alone, wrapped in an occlusion query, into the
      # shadow map that was just rendered. The query counts how many of its
      # fragments survived the depth test — that is, how many shadow texels
      # it actually owns.
      IF pending_visual is present
        begin_occlusion_query(pending_query)
        advance the graph's marker
        insert pending_visual into the graph
        render the graph at priority zero
        end_occlusion_query(pending_query)
        pending_frame = current_frame + 1     # read the answer next frame

    using_cache:
      # nothing left to learn
```

**Notes** — The candidate is drawn *after* the full shadow map, against the depth buffer that map left behind. That is what makes the query meaningful: a caster fully behind another writes no fragments. It also means the test is only valid for the light's current shadow map, which is another reason the learned set dies when the light moves.

## `flush_query`

**Contract** — Read the outstanding query's result, if it is due this frame, and act on it. Called when the light will not be rendered this frame, so that a query issued last frame does not stay outstanding forever. Blocks on the query result.

```text
FUNCTION flush_query()
  IF pending_frame is not the current frame THEN RETURN
  IF phase is not learning OR no pending_visual THEN RETURN

  fragments = read_occlusion_query(pending_query)
  IF fragments = 0
    # Invisible. Remember it, and do NOT advance the candidate index: this
    # visual will no longer be produced by the graph, so the visual that was
    # next now occupies this index. Shrink the total to match.
    invisible.append(pending_visual)
    candidate_total -= 1
  ELSE
    # Visible. It will keep being produced, so step past it.
    candidate_index += 1

  pending_visual = none
  IF candidate_index = candidate_total
    phase = using_cache
```

**Invariants** — The index-does-not-advance rule is the subtle part. The candidate index is a position in the *sequence the graph produces*, and that sequence shrinks by one whenever a visual is learned invisible. Advancing the index in that case would skip a caster permanently.

## `reset_query`

**Contract** — Force a pending query that was scheduled for *next* frame to be harvested now, by pulling its due frame back one, then flushing. Used when the frame is being abandoned and the query must not outlive it.

## `invalidate`

**Contract** — Return to the `counting` phase, drop the learned set, forget any candidate, and push the sleep deadline forward. Called from the light's placement change and from teardown.

## `caster_feedback`

**Contract** — The callback the graph builder invokes when it reaches the requested caster index. Records that visual as the candidate and immediately clears the hook so no further callback fires this build.
