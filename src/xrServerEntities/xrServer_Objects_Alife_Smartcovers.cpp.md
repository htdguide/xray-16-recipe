# src/xrServerEntities/xrServer_Objects_Alife_Smartcovers.cpp

> The smart-cover record: four numbers, a volume and a name — plus the editor machinery that runs the script table that name points at, so an author can see the cover they are placing.

**Needs** — [`xrServer_Objects_Alife_Smartcovers.h`](xrServer_Objects_Alife_Smartcovers.h.md) · [`script_value_container_impl.h`](script_value_container_impl.h.md) · [`ShapeData.h`](ShapeData.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Data: save format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`xrServer_Objects_Alife_Smartcovers.h`](xrServer_Objects_Alife_Smartcovers.h.md)
**Tier floor** — T1 for the record's layout; the editor half is T3.

## Purpose

A **smart cover** is an authored place a creature can take up a position in: a window to
shoot from, a wall to lean around, a crate to crouch behind. The record places it and names
it. Everything about what a creature can *do* there lives in a script table, and the record
carries only the name of that table.

## State

```text
RECORD SmartCover EXTENDS DynamicObject, Shape
  volume                   : shape          # the cover's extent
  description              : text           # names a table in the script data
  hold_position_time       : real           # how long a creature stays put
  enter_min_enemy_distance : real           # refuse to enter with an enemy closer than this
  exit_min_enemy_distance  : real           # refuse to leave with an enemy closer than this
  is_combat_cover          : bool           # written as one byte
  can_fire                 : bool           # written as one byte
  available_loopholes      : script table   # never serialized; set from script
```

**Invariants**

- **A combat cover can always fire.** Construction forces `can_fire` true whenever
  `is_combat_cover` is true, and only consults the section for it otherwise. The two are not
  independent.
- **A smart cover never switches offline.** It can come online and it can be saved, but it
  cannot leave — it is scenery the AI reasons about, and a creature approaching one needs it
  to exist.
- It is **not interactive**: the player cannot use it. And it **does** occupy navigation-graph
  locations, because a creature must path to it.

## Save write

The dynamic-object level, the collision form, the description name, the hold time, the two
enemy distances, then the two flags as one byte each.

## Save read — the version changelog

| From version | Field |
|---|---|
| always | the dynamic-object level, the collision form, the description, the hold time |
| ≥ 120 | the two enemy-distance thresholds |
| ≥ 122 | the combat-cover flag |
| ≥ 128 | the can-fire flag |
| < 128 | **can-fire is set equal to the combat-cover flag** |

**Notes** — the fallback below 128 is annotated in the source with an admission that the
combat-cover flag "seems to be changed in scripts" and that the assignment is a
synchronization. That is the honest reading: the derived relationship the constructor
establishes can be broken at run time by script, and an old save has no record of the
broken state, so the read re-derives it. A rebuild should store both and not re-derive.

## The editor half — reading a cover out of the script data

Everything below is absent from the shipping build. It exists so that placing a smart cover
in the editor shows the author the loopholes, their view cones and a posed figure in each
enterable one.

### `load_draw_data`

**Contract** — evaluates the script table `smart_covers.descriptions.<description>.loopholes`
and turns each entry into one drawable loophole. A missing or malformed table is **logged and
skipped**, not fatal — the author is mid-edit and has probably just typed a name that does
not exist yet.

```text
FUNCTION load_loopholes(cover)
  loopholes = script_table("smart_covers.descriptions." + cover.description + ".loopholes")
  IF absent
    LOG and RETURN                      # an unfinished description is not an error
  cover.draw_data = empty
  FOR EACH entry IN loopholes
    IF cover.available_loopholes names entry.id AND says false
      CONTINUE                          # this instance disabled this loophole
    IF entry is not a table
      CONTINUE
    h.id             = entry.id
    h.position       = entry.fov_position
    h.view_direction = normalize(entry.fov_direction) OR forward if degenerate
    h.enter_direction= normalize(entry.enter_direction) OR forward if degenerate
    h.fov            = entry.fov     clamped to [0, 360]
    h.range          = entry.range   clamped to non-negative
    h.enterable      = FALSE          # decided below, not here
    h.animation      = "loophole_" + h.id + "_visual"
    APPEND h
  mark_enterable(cover)
  build_figures(cover)
```

**Invariants** — a degenerate direction is **replaced with a fixed forward vector and
logged**, not rejected. Both directions must be unit length for the view cone and the figure
placement to mean anything, and an author writing a zero vector should see something wrong
rather than nothing at all.

**Notes** — the animation name is **synthesized from the loophole's identifier** by a naming
convention. The source contains, commented out, the code that used to derive it properly: it
walked the loophole's transition list looking for the idle-to-fire transition and took that
transition's animation. The convention replaced the derivation. A rebuild should be aware
that the loophole identifier and the animation asset name are coupled by a string rule and
nothing checks it.

**`available_loopholes` is a per-instance override**: the script table on the *record* can
switch individual loopholes of a shared description off. This is how one authored cover
description serves several placements that differ only in which positions are usable.

### `check_enterable_loopholes`

**Contract** — evaluates the description's transition list and marks a loophole enterable
when a transition leads into it **from the outside**. Missing transitions are a hard failure
in a debug build.

```text
FUNCTION mark_enterable(cover)
  transitions = script_table("smart_covers.descriptions." + cover.description + ".transitions")
  FOR EACH t IN transitions
    IF t.vertex0 is not the exterior vertex
      CONTINUE                     # an internal transition says nothing about entry
    mark the loophole named by t.vertex1 as enterable
```

**Invariants** — **"the exterior" is a named vertex with an empty identifier**, normalized
through the same transformation the script side uses. That convention — the empty name means
outside the cover — is the only thing distinguishing an entry transition from a movement
between two loopholes, and it is not written down anywhere else.

A transition naming a loophole that is not in the draw list is **ignored**, with the
assertion that would have caught it commented out. A description may legitimately reference
a loophole this instance disabled.

### `fill_visuals`

**Contract** — builds one posed figure per enterable loophole, so the author sees a body in
each usable position rather than a marker.

```text
FUNCTION build_figures(cover)
  discard the previous figures
  FOR EACH h IN cover.draw_data
    IF NOT h.enterable
      RETURN                        # see below
    IF h.animation is empty
      LOG and RETURN
    figure = a fixed stalker model, posed with h.animation
    place it at h.position, facing h.enter_direction, up = world up
    APPEND figure
```

**Notes** — **both early exits stop the whole loop rather than skipping the entry**, so a
single non-enterable loophole hides every figure after it. That is a defect, and it is
visible in the editor as covers that draw fewer figures than they should. A rebuild should
continue.

The figure is a **hard-coded neutral stalker model path**. It is a stand-in for "a person",
and a rebuild may use anything humanoid.

### `on_render`

**Contract** — draws, for the selected cover only, each loophole's identifier at its position
and its view cone as a wireframe frustum. Reparses the description lazily on the first render
after it changed.

**Notes** — the frustum drawing constructs the four corner rays from a field of view, an
aspect ratio and a range and draws eight lines. It is ordinary camera-frustum geometry, in
this file because the editor needed it and nowhere else had it.

The reparse flag is set by two things — construction, and any edit to the loophole table —
and is consumed on the next render. Deferring the parse to render time is what keeps the
editor responsive while an author types a description name character by character.

### `is_combat_cover` — the editor's probe

**Contract** — asks the script table whether this description is a combat cover, to decide
whether to *show* the two flag rows at all. An empty name answers no. A description whose
table is missing the flag **answers yes and logs**.

**Notes** — defaulting a missing flag to "combat cover" is the safe direction: a cover the AI
treats as combat but which is not will be under-used, while the reverse gets creatures shot.

### Property rows

Hold time, the description (chosen from the list of description names the editor knows),
the two enemy distances, and — only for a combat cover — the two flags. Changing the
description re-runs the parse and marks the record's visual as changed; changing the
loophole table writes the shadow cells back (see
[`script_value_container.h`](script_value_container.h.md)) and schedules a reparse.

**Notes** — construction and destruction bump a **global count of records contributing
property data**, which the editor uses to know when its description list must be rebuilt.
It is a refresh trigger, not state.
