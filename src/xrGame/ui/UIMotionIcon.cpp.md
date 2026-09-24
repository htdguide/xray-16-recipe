# src/xrGame/ui/UIMotionIcon.cpp

> The stealth indicator: it takes one visibility contribution per creature that can see the
> actor, shows the worst of them, and eases the needle toward it so the reading never jumps.

**Needs** — [`UIMotionIcon.h`](UIMotionIcon.h.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md)
**Used by** — [`UIMotionIcon.h`](UIMotionIcon.h.md)
**Tier floor** — T3.

## Purpose

Answers the question the stealth game is built around: *can they see me right now*. The answer
is not a property of the actor — it is the maximum over every creature currently perceiving
the actor — so this widget owns a small table keyed by observer and the game's vision system
writes into it as perceptions begin and end.

The widget reaches the game through a process-wide handle and a free function, because the
vision code that reports a contribution has no path to the screen tree.

## State

```text
RECORD StealthIndicator EXTENDS Static
  current_state    : Posture                 # starts at the sentinel, so the first show works
  states           : map<Posture, Static>    # whichever postures the document defines
  power_progress   : optional<ProgressBar>   # stamina
  luminosity, noise : each EITHER a ProgressBar OR a ProgressShape, never both

  npc_visibility   : list<(observer_id, contribution)>
  changed          : bool                    # the table needs re-reducing
  luminosity       : real                    # the reduced target
  cur_pos          : real                    # the eased needle
  relative_size    : real                    # from the document; how much of the minimap
                                             # frame the arcs occupy
```

**Invariants**

- **A contribution of zero is an absence.** Writing zero for an observer removes its row
  rather than storing a zero. That keeps the table exactly the set of creatures who can see
  the actor at all, which is what makes the reduction a plain maximum with an empty case.
- Each of the two readouts is either a bar or an arc, decided by which element the document
  defines, and the bar is tried first. The two carry their range differently — an arc is always
  zero to one and the values are pre-scaled into percent, a bar carries its own range and the
  values are scaled into it — so every setter has two branches.
- The posture icons are mutually exclusive and are shown *and enabled* together, because a
  hidden-but-enabled widget still takes navigation focus.

## `Init`

**Contract** — load the indicator's own layout document. If it defines a window element, apply
it and read the relative size, and report that the caller should parent this widget into the
minimap; otherwise apply a background element instead and report that it stands alone. Then
create the stamina bar, the two readouts (bar first, arc as fallback), and one icon per posture
the document defines, each hidden. Finally select the normal posture.

**Notes** — the return value carries a *layout* decision out to the overlay: whether the
stealth indicator lives inside the minimap's circle or beside it. That is data, not code, and
it is the reason the overlay's construction branches on it.

## `AttachToMinimap`

**Contract** — take the minimap frame's size, place this widget at the frame's centre at that
size, and size the two arc readouts to the frame scaled by the relative size and the canvas
correction, centred likewise.

**Notes** — the arcs are scaled by the horizontal canvas correction so they stay circular on a
wide display, exactly as the minimap's own geometry is. The bars are not, because a bar is
axis-aligned and a stretch is invisible on it.

## `SetActorVisibility` — the contribution table

**Contract** — record one observer's contribution, scaled into whichever readout exists, then
mark the table dirty. A non-zero contribution from an unknown observer inserts a row; a zero
removes the row if there is one; anything else updates in place. Inert outside single player.

```text
FUNCTION set_actor_visibility(observer, value)  # value in 0..1
  IF the readout is an arc THEN value <- clamp(value, 0, 1) * 100
  ELSE                          value <- bar.min + value * (bar.max - bar.min)

  row <- npc_visibility WHERE observer matches
  IF row IS none AND value is non-zero THEN npc_visibility.append((observer, value))
  ELSE IF value IS zero                THEN remove row if present
  ELSE                                      row.contribution <- value
  changed <- true
```

**Notes** — there is a gap in the cases: a zero contribution for an unknown observer, and a
non-zero contribution when no readout exists at all, both fall into branches that assume the
row was found. The first is harmless; the second reaches a missing row. A rebuild should
resolve the row once and branch on it.

## `Update` — the reduction and the easing

**Contract** — in single player only: when the table is dirty, re-reduce it to the greatest
contribution (or zero when empty) and take that as the target. Then move the displayed value
toward the target, clamped to the readout's range. Outside single player, update as a plain
picture and nothing else.

```text
FUNCTION update()
  IF NOT single_player THEN update as a picture; RETURN
  IF changed THEN
    changed <- false
    luminosity <- maximum contribution in npc_visibility, or 0 when empty
  update as a picture

  diff <- abs(luminosity - cur_pos)
  IF the readout is an arc THEN
    cur_pos <- cur_pos ± diff * frame_delta          # approaches asymptotically
  ELSE
    span <- bar.max - bar.min
    cur_pos <- cur_pos ± min(span * frame_delta, diff)   # constant rate, one second full sweep
  clamp cur_pos into the readout's range and show it
```

**Notes** — **the two readouts ease differently, and the difference is visible.** The bar moves
at a constant rate that crosses its full range in one second and never overshoots, because its
step is clamped to the remaining distance. The arc moves by a *fraction of the remaining
distance* per second, which is an exponential approach that never quite arrives and is not
frame-rate independent — at a high frame rate it converges faster. This is an inconsistency in
the original, not a design; a rebuild should use the bar's rule for both, and should be aware
that doing so slightly changes how quickly the arc reacts.

The reduction is performed by sorting the whole table and taking the last element. A rebuild
takes the maximum directly; the sort is incidental.

## The rest of the surface

**Contract** — `ShowState` hides the old posture icon and shows the new one, doing nothing when
the posture is unchanged and tolerating a posture the document did not define. `SetPower` sets
the stamina bar. `SetNoise` sets the noise readout, clamped into whichever kind it is, and is
inert outside single player. `ResetVisibility` empties the table and marks it dirty.
`Draw` draws nothing at all in the second shipped game, where the indicator's background is
part of the minimap art and drawing it again would double it.

**Notes** — the free function beside the class is the entry point the vision system actually
calls: it checks the game kind and forwards to whichever indicator exists. That indirection
exists because perception is reported from deep in the AI layer, which holds no screen
reference. A rebuild replaces it with an event the screen subscribes to — which is the
chapter's stated rule, and this file is the exception that proves it.
