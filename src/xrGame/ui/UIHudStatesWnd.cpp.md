# src/xrGame/ui/UIHudStatesWnd.cpp

> The heads-up panel: health, stamina, suit condition, the active weapon's ammunition, and the four hazard indicators — and, uniquely among screens, it also *writes back into the game*, feeding the actor's danger state and driving the anomaly detector's clicking.

**Needs** — [`UIHudStatesWnd.h`](UIHudStatesWnd.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [`xrUICore/ProgressBar/UIProgressShape.h`](../../xrUICore/ProgressBar/UIProgressShape.h.md) · [`xrUICore/arrow/ui_arrow.h`](../../xrUICore/arrow/ui_arrow.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [`Actor.h`](../Actor.h.md) · [`ActorCondition.h`](../ActorCondition.h.md) · [`CustomOutfit.h`](../CustomOutfit.h.md) · [`ActorHelmet.h`](../ActorHelmet.h.md) · [`Inventory.h`](../Inventory.h.md) · [`CustomZone.h`](../CustomZone.h.md) · [`PDA.h`](../PDA.h.md) · [`WeaponMagazinedWGrenade.h`](../WeaponMagazinedWGrenade.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`UIHudStatesWnd.h`](UIHudStatesWnd.h.md)
**Tier floor** — T3.

## Purpose

The permanent overlay. It is the busiest file in this chapter and the only one that is not a
pure view: the hazard indicator's verdict is **written back into the actor's condition** as a
per-type danger level, and the zone sweep that feeds the indicator is also what makes the
anomaly detector click. Removing this window would change gameplay, which is why it updates
every frame whether or not it is drawn.

Three subsystems share the file, and they are independent:

1. **Vitals and the weapon read-out** — a straight projection of actor state onto widgets.
2. **The zone sweep** — proximity to nearby anomalies, accumulated per hazard type, decayed,
   and turned into detector clicks.
3. **The indicator ladder** — four hazard triangles whose colour compares the accumulated
   hazard against the player's protection.

## State

```text
RECORD HudPanel EXTENDS Window
  # widgets: nearly all optional; see "Everything is optional" below
  vitals      : health bar, optional stamina bar, optional armour bar + its label
  weapon      : icon, fire mode, up to four ammunition counters, a combined counter,
                a grenade counter
  hazard      : four indicator pictures (radiation, fire, chemical, psionic)
                four optional backing pictures
                an optional needle and its shadow, over an optional dial
  radiation   : an optional circular gauge for absorbed dose

  zone_power     : real per influence type   # accumulated, decaying
  zone_radius    : real per influence type   # from configuration
  zone_threshold : real per influence type   # from configuration
  zone_hit_type  : the hit kind each influence type corresponds to
  max_zone_radius: real                      # the sweep radius
  blink_state    : bit per influence type    # is the red animation running
  health_blink   : real                      # change needed to restart the health animation
  last_health    : real
  fake_mode      : bool                      # indicators driven from script instead
  force_ammo_recount : bool                  # once, on first update
```

**Invariants**

- There are **five** influence types but only **four** have widgets. Electric hazard is
  tracked, accumulated and protected against, and has no indicator — the indicator loop runs
  over one fewer element than the data arrays, and that off-by-one is intentional and named.
- `zone_power` is a **decaying maximum**: a sweep never lowers it, only raises it; time
  lowers it. That is what makes an indicator flare as the player passes an anomaly and settle
  afterwards, instead of flickering with every distance sample.
- The hazard-to-hit-type mapping is many-to-one: light burn and burn both map to the fire
  indicator, and the six purely-physical hit kinds map to no indicator at all.

## Everything is optional

**Contract** — almost every widget is created with the optional flag, and the code checks for
its absence at every use. Only a handful are required: the backing picture, the four hazard
indicators, the fire-mode label, the weapon icon and the health bar.

**Notes** — this is the three-games problem at its worst. Each game's overlay shows a
different subset — one has a needle gauge, one has an armour bar, one has three ammunition
counters — and one class renders all three from whichever elements the mounted data defines.
A rebuild cannot simplify this into one layout; the optionality *is* the compatibility.

One paired constraint is enforced rather than tolerated: the armour bar and its label must
both be present or both absent, and a layout with one of them is rejected by name. Every
other element stands alone.

Two containers are chosen by presence: when the health and armour labels exist, the bars are
attached **to them** rather than to the panel, so a layout can move a label and carry its bar
with it. Otherwise both go on the panel. The same for the weapon group.

## Vitals

**Contract** —

```text
FUNCTION update_vitals(actor)
  health := actor.health
  bar.position := ceil(health * 100 * 35) / 35       # quantized to 35 steps
  IF |health - last_health| > health_blink
    last_health := health
    restart the bar's colour animation                # the damage flash
  IF a stamina bar exists
    bar.position := ceil(power * 100 * 35) / 35
    IF the actor can still sprint THEN restart its colour animation
  IF an armour bar exists
    show it only while an outfit is worn; its position is the outfit's condition
  IF a bleeding icon exists
    show it while the bleeding rate exceeds a hundredth
  IF a dose gauge exists
    set it to the actor's absorbed radiation
```

**Notes** — the **quantization to 35 steps** is the shipped bar texture's segment count, the
same idea as the cell board's fifteen-step condition bar and a different number because a
different texture. Ceiling rather than rounding means a sliver of health always shows as one
segment rather than as none.

The health flash is *restarting an animation*, not starting one: the bar always has a cyclic
animation, and a hit restarts it from the top. The threshold comes from configuration and
defaults to zero, which makes the animation restart on **every** change — that is the shipped
default and it is what makes the bar pulse continuously while the player is healing.

The stamina bar's animation restarts every frame *except* while the player is too tired to
sprint, so the animation visibly stops at exactly the moment sprinting becomes impossible.
Inverted logic that reads as a mistake and is the intended signal.

## The weapon read-out

**Contract** — the active item is asked for a **brief-information record** — fire mode, icon
section, current magazine, each ammunition pool, total, grenade count — and the widgets are
filled from it. With no active item, every weapon widget hides.

Three details carry decisions:

- **A one-shot forced recount.** The first update after the panel appears tells the weapon to
  recompute its ammunition totals. Those totals are otherwise maintained incrementally, and
  the panel may have been created after the last change.
- **The active ammunition type is opaque; the others are dimmed.** All counters share one
  colour at an alpha of 150, and the one matching the weapon's selected type is redrawn at
  full alpha. The grenade counter does the same for grenade-launcher mode. That is the only
  indication of which ammunition is loaded.
- **The combined counter picks its font by fit.** The "current/total" string is drawn in a
  large display font, dropping to a smaller one on a widescreen display unconditionally, and
  on a 4:3 display only when the string exceeds five characters. Hard-coded font names and a
  hard-coded length; the source calls it a hack. A rebuild measures the string.

## The weapon icon

**Contract** — the icon's sub-rectangle is read from the item's configuration section in
icon-grid units and multiplied by the atlas cell size, then the widget is sized by a
**per-game scale table**:

| Mounted game | Height | Width | Extra |
|---|---|---|---|
| first | ×0.8 | ×0.8, or ×0.7 widescreen | fixed pixel offsets, differing by whether the icon is narrower than one grid cell |
| second | ×0.65 | ×0.65, ×0.833 again if widescreen | offset only for sub-cell icons |
| third | ×0.8 | ×0.8 × the runtime horizontal factor | none |

and in every game an icon wider than two grid cells is clamped to a fixed width (1.6 cells in
the first game, 1.5 in the others) so that a long rifle does not overrun the panel.

**Notes** — none of these numbers is derivable. They are per-game visual tuning, they are in
code rather than in the layouts, and a rebuild must carry the table verbatim to reproduce any
of the three games' overlays. Recorded as unrecovered rationale.

## The zone sweep

**Contract** — once per update, from the camera position:

```text
FUNCTION sweep_zones(actor)
  # the PDA's own proximity sense drives monster detector chirps
  FOR EACH living monster the PDA can feel
    that monster plays its detector sound

  absorbed_dose := actor.radiation
  # material damage the player is standing in counts as radiation
  raise zone_power[radiation] to clamp(material_damage / max_radiation_power, 0, 1.1)
  needle_value := zone_power[radiation]
  drive the needle; the shadow needle copies the needle's drawn angle

  # decay, before this frame's samples
  FOR EACH influence type
    IF frame_time < 1 s THEN zone_power *= 0.9 * (1 - frame_time)
    IF zone_power < 0.01 THEN zone_power := 0

  refresh the proximity list of zones within max_zone_radius of the camera
  FOR EACH zone in that list
    measure distance from a point half a unit below the camera to the zone's surface
    type := the influence type of this zone's hit kind
    raise zone_power[type] to clamp(zone.power_at(distance), 0, 1.1)
    compute a nearness factor in 0..1 from the distance
    period := lerp(zone.click_period_far, zone.click_period_near, nearness²)
    accumulate time; when it exceeds period, play the zone's detect sound and reset
```

**Invariants** — the accumulation is `max`, the decay is multiplicative, and the decay runs
*before* the samples, so a zone still in range is re-raised every frame and only a zone left
behind actually fades.

**Notes** — several things here are only visible by reading it as a whole.

*The decay is skipped entirely when a frame takes longer than a second.* At `frame_time`
close to one the multiplier reaches zero and beyond it would go negative, so the guard is a
sign guard, not a performance one. A long hitch therefore freezes the indicators rather than
clearing them, which is the safer of the two.

*The click period is interpolated on the **square** of nearness*, so the detector's clicking
accelerates sharply only in the last part of the approach. The nearness factor is further
scaled down near the zone and again inside it, and inside a zone it is additionally scaled by
the zone's own strength — a strong anomaly clicks faster from the same distance. This is the
anomaly detector's entire feel, and it lives in the HUD rather than in the detector item.

*The camera point is lowered half a unit* before measuring, so the detector reads from
roughly chest height rather than eye height. Small, deliberate, and it changes when a
low-lying anomaly starts clicking.

*The nearness factor's denominator is wrong.* The expression that should divide by the zone's
radius plus the type's feel radius instead always divides by a constant 5, because an
addition binds tighter than the conditional it was meant to select between. Every zone type
is therefore scaled as though its feel radius were the same. The effect is a uniform detector
response across hazard types, which is not what the per-type configuration intends and is
what the shipped games actually do. A rebuild reproducing the original's feel must keep the
constant; one honouring the configuration must fix it and will sound different.

*Monster detector chirps are driven from here*, through the PDA's proximity sense rather
than through any detector item. The overlay updating is what makes the bio-detector work.

## The indicator ladder

**Contract** — for each of the four widgeted hazard types, compare the accumulated hazard
against the player's total protection and pick one of four rungs:

```text
FUNCTION update_indicator(actor, type)
  hazard  := zone_power[type]
  protect := outfit protection for this hit kind
           + helmet protection for this hit kind
           + protection from artefacts on the belt
           + the matching active booster's contribution, if any
  IF hazard < epsilon           THEN rung := white   ; danger := 0
  ELSE IF hazard <= protect     THEN rung := green   ; danger := 0
  ELSE IF hazard - protect < threshold[type] THEN rung := yellow ; danger := 0
  ELSE                               rung := red     ; danger := (hazard - protect) / max_power
  blink := (rung is red)
  paint the indicator; write `danger` back into the actor's condition for this type
```

**Invariants** — **only red writes a non-zero danger**, and that danger is what the game
layer reads to decide the player is in trouble. The three lower rungs explicitly write zero,
so the value cannot be left stale from a previous frame.

**Notes** — the ladder is *protection-relative*, not absolute: walking into the same anomaly
in a better suit shows green instead of red. That is the design and it is the reason the
protection sum gathers four sources — suit, helmet, belt artefacts and consumable boosters —
each of which a player changes independently.

Painting resolves a texture named `<type prefix><colour word>` and **falls back twice**: if
that texture is missing, the indicator is tinted with a flat colour instead. In the white
case there is a third path: if the white texture is missing but the green one exists, the
indicator is *hidden* rather than painted, because the game whose data that is does not draw
an idle indicator at all. Three games, three idle behaviours, resolved by probing the texture
registry — the same first-of pattern used elsewhere in this chapter.

The red rung also starts a cyclic colour animation on the indicator, named in the layout. The
blink is switched by a per-type bit and only on a transition, so the animation is not
restarted every frame.

## Fake indicators

**Contract** — a mode in which the ordinary indicator update is suppressed and the
indicators are driven by an explicit per-type power supplied from outside. The fake path
recomputes protection identically but **normalises protection by the type's maximum power**
first, and writes the raw difference as danger rather than a normalised one.

**Notes** — this exists for tutorials, which need to show a red indicator without an anomaly.
The two paths' arithmetic differs in both of the places named above, so the fake path does not
reproduce the real one exactly — a fake power that reads green would read yellow for real.
Recorded as a divergence with no recoverable justification.

## Drawing the indicators separately

**Contract** — a second entry point updates and draws only the four indicators, with no other
widget. It is for the case where the rest of the overlay is hidden but the hazard warnings
must stay — the player is in a screen, and an anomaly still matters.
