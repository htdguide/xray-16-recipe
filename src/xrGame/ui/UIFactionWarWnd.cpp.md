# src/xrGame/ui/UIFactionWarWnd.cpp

> The PDA's faction-war page: the player's faction against its designated enemy, compared on power, membership and resources, with the war's stage shown as a centred row of markers and the scale for every bar asked of the script layer.

**Needs** — [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIWarState.h`](UIWarState.h.md) · [`FactionState.h`](FactionState.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [`xrUICore/Windows/UIFrameLineWnd.h`](../../xrUICore/Windows/UIFrameLineWnd.h.md) · [`Actor.h`](../Actor.h.md) · [`PDA.h`](../PDA.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md)
**Tier floor** — T3.

## Purpose

One PDA tab, shipped only with the game that has faction warfare. Its structure is a
**mirror**: every widget exists twice, once for the player's faction and once for its enemy,
and one update fills both from two faction records.

Three decisions make it worth a page. The **axis maxima are script functions**, not
constants. The **war-stage row centres itself** on however many stages are populated. And the
whole page **fails soft** on data it does not have: an absent layout, an unknown enemy or an
empty faction each hide part of the page rather than fault.

## State

```text
RECORD FactionWarPage EXTENDS Window
  our, enemy        : FactionRecord     # name, icon, power, members, resource, bonus,
                                        # a mission target, and up to N war stages
  max_members       : int               # from script
  max_resource      : real              # from script
  max_power         : real              # from script
  update_period     : int (ms)          # from the layout; default 3000
  last_update       : int (ms)
  stage_markers     : list<WarStage>    # fixed count, from the faction-state vocabulary
  stage_gap         : real              # authored spacing between markers
  stage_centre_x    : real              # authored centre line; default 511
  bonus_pips        : two lists of 6 pictures
  target_caption_pos, target_desc_pos : point   # authored origins, needed for reflow
```

**Invariants**

- The page shows information only when **all three** of these hold: the enemy faction is
  known, the player's faction has at least one member, and the player's faction has a name.
  Otherwise everything but the mission target is hidden. Those are the three ways the data
  can be mid-transition, and each would otherwise render as a blank or a zero bar.
- The mission target and its description are the exception: they stay visible even when the
  comparison is hidden, because a faction that has not started a war still has an objective.
- The stage count comes from the faction-state vocabulary, so adding a stage to the game
  logic adds a marker with no layout change.

## Construction

**Contract** — the whole page is read from one shipped layout document, and **a missing
document is not an error** — construction reports failure and the PDA simply omits the tab.
That is how one executable runs all three games from the same code: the two games without
faction warfare ship no such document.

Two background elements are looked up **twice, as two different control types**: first as a
nine-slice frame window, and if the layout does not define one, as a frame line and a plain
picture instead.

**Notes** — the double lookup is the same first-of pattern the key-binding cell uses for its
row texture, applied to *element types* rather than texture names. Different games' data
authored this page's background with different widget types, and the engine accepts either.
A rebuild needs the layout reader to be able to answer "is this element present and of this
type" without failing, which is a real requirement on the reader, not an implementation
detail.

Three repeated element groups — the war stages, and the two rows of bonus pips — are
instantiated by **reading the same layout element several times and spacing them in code**:
the first takes the authored position, each subsequent one is offset by its predecessor's
width plus an authored gap. One authored element, N instances. That is the chapter's standard
way of expressing a repeated row, because the layout vocabulary has no repetition construct.

## The update cycle

**Contract** — a full refresh at most every `update_period` milliseconds **scaled by the
game's time factor**, so the page refreshes in game time rather than wall time. Hiding the
page resets its state; showing it re-resolves the two factions and clears every stage marker.

```text
FUNCTION update()
  IF NOT shown THEN reset(); RETURN
  IF now - last_update > update_period * time_factor
    last_update := now
    refresh()
```

**Notes** — scaling the period by the time factor is deliberate and not obviously right: it
means the page updates *less* often when time is accelerated, which is the opposite of what a
player sleeping through a night wants. It is the original's behaviour.

The reset-while-hidden call also clears the cached faction identifiers, which is why showing
the page always re-resolves them — the player's faction can change between openings.

## Refreshing

```text
FUNCTION refresh()
  IF our faction is unresolved THEN resolve it, or fault
  max_members  := script "pda.get_max_member_count"
  max_resource := script "pda.get_max_resource"
  max_power    := script "pda.get_max_power"
  our.reload()
  show the mission target; fit its caption's height to its text
  reflow the description to sit 8 units under the caption's new bottom
  IF enemy unknown OR our.member_count = 0 OR our.name empty
    THEN hide the comparison; RETURN
  enemy.reload()
  show the comparison
  centre the war-stage row over the populated stages
  FOR EACH side
    name, big icon
    power bar    over 0 .. max_power
    member bar   over 0 .. max_members
    resource bar over 0 .. max_resource
    light `bonus` of the six pips
```

**Notes** — **the three maxima are Lua calls, not configuration.** Every bar on this page is
a fraction whose denominator the mod author controls in script, which is the point: the bars
are comparative, and what counts as "full" is a balance decision. The engine holds no default
and a missing function is fatal. See
[Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine).

The caption is **measured, then the description is moved** — the one place on the page where
layout is data-dependent, because a mission title can be one line or three. Its authored
origin is remembered at construction precisely so the reflow can be recomputed from scratch
each time rather than accumulated.

## Centring the war-stage row

**Contract** — stages are filled from the first until one reports it has no information;
that one and everything after it is left empty. The populated prefix is then centred on an
authored vertical line.

```text
FUNCTION centre_stages(faction)
  filled := 0; span := 0
  FOR i IN 0 .. stage_count - 1
    IF NOT stage[i].fill_from(faction.stage(i), faction.stage_hint(i)) THEN BREAK
    filled := filled + 1
    span   := span + stage[i].width + gap
  IF filled = 0 THEN RETURN
  span := span - gap                       # no gap after the last one
  row.x := centre_x - span / 2
```

**Notes** — the row is a *progress* display: a war passes through its stages in order, so the
populated ones are always a prefix and stopping at the first empty one is correct. Centring
by measured span rather than by a per-count authored position is what keeps a three-stage war
and a six-stage war both looking deliberate. The default centre line, 511, is the middle of
the 1024-unit canvas — half a unit off centre, which nobody will see.

## The bonus pips

**Contract** — six pictures per side. All are first dimmed to a low-alpha white, then the
first `bonus` of them are set to opaque green.

**Notes** — paint-all-then-repaint-some is one pass over a six-element row and needs no
condition per element. The two colours are hard-coded here rather than authored, which is the
only place on this page where a colour is not in the layout; there is no recoverable reason,
and a rebuild should move them to the document.
