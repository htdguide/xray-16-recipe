# src/xrGame/ui/UIActorStateInfo.cpp

> Where the actor's condition becomes fourteen readouts: how protection is summed across
> outfit, helmet, belt artefacts and active boosters, and why the answer is a *fraction* of
> the worst thing the level can do to you.

**Needs** — [`UIActorStateInfo.h`](UIActorStateInfo.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHudStatesWnd.h`](UIHudStatesWnd.h.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`../../xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [`../../xrUICore/ProgressBar/UIProgressShape.h`](../../xrUICore/ProgressBar/UIProgressShape.h.md) · [`../../xrUICore/arrow/ui_arrow.h`](../../xrUICore/arrow/ui_arrow.h.md)
**Used by** — [`UIActorStateInfo.h`](UIActorStateInfo.h.md)
**Tier floor** — T2: polls simulation state and drives widgets

## Purpose

The condition panel is the player's only view of a fairly deep simulation: seven hit types,
four sources of protection that stack, a restoration rate, and a bleeding and radiation level.
This file is where that simulation is flattened into fourteen numbers a player can read at a
glance, and the flattening involves three decisions that are made nowhere else.

## State

See [`UIActorStateInfo.h`](UIActorStateInfo.h.md).

## The three decisions

### Protection is shown as a fraction of the level's worst

**Contract** — A protection readout is not an absolute value. It is
`accumulated protection / the maximum power of that hit type in this level`. The denominator
comes from the actor's condition model, which knows how bad the local anomalies get.

**Invariants** — This is why a suit that reads as fully protective on one level reads as
half-protective on the next, with no change to the suit. It is the single most important thing
about the panel and it is invisible in the numbers themselves. A rebuild that shows absolute
protection has changed what the screen means.

Two readouts use a different denominator supplied directly by the condition model — wound and
fire-wound protection — and one, the power-restoration readout, is normalised against the
actor's own maximum restoration rate rather than anything environmental.

### Protection accumulates from four sources, additively

**Contract** — For each hit type, the displayed protection is the sum of:

```text
  outfit protection for that hit type
+ helmet protection for that hit type
+ the sum over active boosters of that booster's value    (three types only)
+ the actor's total artefact protection from the belt
```

**Invariants** — Only three hit types have booster contributions — radiation, chemical burn
and telepathic — because only those three have boosters in the shipped data. The artefact
contribution is asked of the actor rather than computed here, because the belt's algebra is
the inventory's business.

**Fire-wound protection is the exception** and is computed from *bone armour*, not from hit
type: the spine bone's armour value times the outfit's condition, plus the head bone's armour
times the helmet's condition — or, when the outfit seals the head and there is no helmet, the
outfit's own head-bone armour. That is the only readout that knows about bones, and it is why
the panel asks the actor's model for two bone handles by name.

### Every readout quantises to its own artwork

**Contract** — Continuous values are floored onto the number of segments the bar was drawn
with:

- **health** quantises to 55 segments,
- **every protection bar** quantises to 31,
- the needle and the numeric caption take the unquantised fraction.

**Invariants** — The numbers are properties of the shipped bar textures. Skipping the
quantisation makes a bar render a partial segment; getting it wrong makes the last segment
unreachable. They are frozen alongside the art.

## `UpdateActorInfo`

**Contract** — The whole poll, called every frame the panel is shown. Returns early when the
owner is not the player.

```text
FUNCTION update_actor_info(owner)
  actor = owner as the player; RETURN IF not
  c = actor.conditions

  stamina.progress = c.power
  stamina.caption  = actor.power_restore_speed

  health.progress = floor(c.health * 55) / 55
  health.icon     = shown when bleeding speed > 0.01     # the bleeding drop icon

  # Bleeding and radiation each pick ONE of three icons by severity.
  FOR each of bleeding, radiation
    hide all three icons
    v = the level (bleeding speed, or radiation)
    IF v is not zero
      show icon 1 IF v < 0.35, else icon 2 IF v < 0.7, else icon 3

  zero all eight protection readouts

  accumulate burn, radiation, acid, psi, wound, shock from the outfit,
    the helmet and the three boosters that have them
  accumulate fire-wound from spine and head bone armour times condition

  IF an outfit is worn
    armour.progress = outfit.condition
    armour.caption  = outfit's spine bone armour
  ELSE
    armour.progress = 0; armour.caption = 0

  FOR each protection readout
    add the actor's belt-artefact contribution for that hit type
    show it as accumulated / the level's maximum for that hit type

  main sensor's shape = current radiation level
  drive the main sensor's needle from the overlay's zone sensing
```

**Invariants** — Zeroing all eight before accumulating is what stops a removed suit from
leaving its protection on screen; there is no change notification, only the poll.

The bleeding and radiation thresholds — 0.35 and 0.7 — divide the range into thirds with the
first third slightly wider. They are the same in both readouts.

## `update_round_states` — the presentation fallback

**Contract** — Show one protection value in whatever form the layout gave this readout.

```text
FUNCTION update_round_states(which, value, maximum)
  fraction = value / maximum
  IF the readout has a progress bar
    bar = floor(fraction * 31) / 31
    show it; done
  ELSE
    needle  = fraction                # unquantised
    caption = fraction                # scaled and rounded by the readout
```

**Invariants** — The dispatch is *by what exists*, not by a mode flag. That is what lets the
same panel code drive a bar-based layout and a dial-based one, and it is why every setter on a
readout returns a success flag.

There is no division guard: a hit type whose level maximum is zero produces a non-finite
fraction. The shipped configuration never has a zero maximum; a rebuild should guard anyway.

## `UpdateHitZone`

**Contract** — Drive the main sensor's needle from the in-game overlay's zone-sensing state,
by reaching across to the overlay's hud-states window, making it refresh, and reading its
aggregate sensor value.

**Notes** — The source flags this as ugly, and it is: a screen reaching into another screen to
make it update. What the coupling records is real, though — the anomaly sensing is a
continuous, decaying, peak-holding value owned by the overlay (see the chapter opener), and
the inventory panel must show the *same* value, not a second copy that would decay
differently.

## `ui_actor_state_item::init_from_xml`

**Contract** — Configure one readout from a named element. Returns immediately when the
element is absent and the caller said it was optional — which is how the same panel
description serves games with different readout sets.

Then, from the element's subtree, each of five optional parts is built **only if its child
element exists**: a progress bar, a progress shape, a needle, a shadow needle, and up to three
icons. Hint text and hint delay are read unconditionally. The needle is initialised to zero so
it does not start at an arbitrary angle.

**Notes** — The `magnitude` attribute is read from whichever icon element is present *last*,
overwriting earlier ones — so a readout with three icons takes the third's magnitude for its
numeric caption. That is almost certainly unintended; the shipped data gives all three the
same value, which is why it never showed.

## `ui_actor_state_item::set_text` / `set_progress` / `set_progress_shape` / `set_arrow` / `show_static`

**Contract** — Each writes one part and reports whether that part exists. `set_text` scales
the fraction by the readout's magnitude — 100 in the shipped data, making a percentage —
rounds to the nearest integer with a half-unit bias, and **clamps to 0…99** so the caption is
always two digits and never overflows its box. `set_arrow` returns how many needles it moved
(0, 1 or 2), the shadow needle simply mirroring the main one's resolved position.

## `init_from_xml` (plain) and the panel's own draw

**Contract** — The plain form configures three readouts by direct element name and leaves the
rest unconfigured; the full form walks a fixed list of fourteen element names, four of which
are required and ten optional.

The panel's `Draw` draws its children and then the shared hint window last, so a readout's
tooltip is over the whole panel. `Show` propagates explicitly to children, because the panel is
attached in different places in the two layout dialects and cannot rely on a parent to cascade.
