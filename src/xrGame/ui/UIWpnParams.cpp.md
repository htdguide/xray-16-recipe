# src/xrGame/ui/UIWpnParams.cpp

> The weapon comparison panel: four statistics computed by script, each drawn as the hovered
> weapon's value against the equipped one's on a single track.

**Needs** — [`UIWpnParams.h`](UIWpnParams.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`../Level.h`](../Level.h.md) · [`../inventory_item_object.h`](../inventory_item_object.h.md) · [`../Weapon.h`](../Weapon.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIWpnParams.h`](UIWpnParams.h.md)
**Tier floor** — T2: caches script function references that must be dropped when the script machine
is reset

## Purpose

When the player hovers a weapon in the inventory or a trade list, this panel answers "is it better
than what I am carrying?" for four qualities at once. The qualities are **not computed here**: each
is a named script function taking a weapon's configuration section and its installed upgrades, so
the balance of the game is data and script, not engine code.

The file also holds the condition panel, which does the same thing for an item's wear with no script
involved.

## State

```text
RECORD ScriptQualities                 # process-wide, created on first use
  rate_of_fire, accuracy, damage, damage_multiplayer, handling : script function references

RECORD WeaponPanel extends Window
  accuracy, handling, damage, rate_of_fire : TwoPositionBar
  captions for each                        : Widget
  icons for each                           : optional<Widget>
  property_line                            : optional<Widget>
  # single player only, all optional:
  ammo_icon, ammo_types_caption, ammo_used_type_caption,
  ammo_count, ammo_count_2, ammo_type_1, ammo_type_2 : optional<Widget>

RECORD ConditionPanel extends Window
  bar : TwoPositionBar
  caption : Widget
```

Invariants:

- The script function references are **cached process-wide and released when the script machine is
  reset**. Holding them across a reset would leave references into a dead machine. The release is
  arranged by subscribing to the script engine's reset event at the moment the cache is created, and
  unsubscribing as it is dropped.
- The ammunition widgets exist only in single player; in multiplayer they are never built, and every
  use is guarded.
- A two-position bar always gets both values. When there is no comparison — nothing equipped, or the
  hovered item *is* the equipped one — the comparison value is set equal to the current one, which
  draws as a single position rather than as a comparison.

## `InitFromXml`

**Contract** — Builds the panel, or reports absence if the layout has no weapon-parameters element
at all — which is how a tooltip omits the panel entirely rather than showing an empty one. Nearly
every child is optional; only the four bars and the four captions are required.

The frozen element names, all beneath `wpn_params`: `prop_line`, `static_accuracy`, `static_damage`,
`static_handling`, `static_rpm`, `cap_accuracy`, `cap_damage`, `cap_handling`, `cap_rpm`,
`progress_accuracy`, `progress_damage`, `progress_handling`, `progress_rpm`, and in single player
`static_ammo`, `cap_ammo_count`, `cap_ammo_count2`, `cap_ammo_types`, `cap_ammo_used_type`,
`static_ammo_type1`, `static_ammo_type2`.

## `SetInfo`

**Contract** — Fills all four bars from the hovered weapon and the equipped one, then, in single
player, the magazine size and the ammunition pictures. Ensures the script cache exists first.

```text
FUNCTION SetInfo(equipped, hovered)
  IF the script cache does not exist THEN
    create it and subscribe to the script engine reset, dropping it when that fires

  section  <- hovered.configuration section
  upgrades <- hovered's installed upgrade list, as a string
  FOR EACH quality IN (rate_of_fire, accuracy, handling)
    current[quality] <- quantise(script[quality](section, upgrades))
  current[damage] <- quantise((single player ? script.damage : script.damage_multiplayer)
                              (section, upgrades))

  comparison <- current                          # default: no comparison
  IF equipped EXISTS AND equipped is not the hovered item THEN
    recompute the same four from the equipped item's section and upgrades

  FOR EACH quality: bar[quality].set_two(current[quality], comparison[quality])

  IF NOT single player THEN RETURN
  weapon <- hovered as a weapon; IF it is not one THEN RETURN

  size <- weapon.magazine_size
  other <- equipped's magazine size, or size when there is none
  IF the second count label EXISTS THEN
    colour it neutral when equal, red when the hovered holds less, green when more
    set its text to the hovered weapon's magazine size
  ammo <- weapon's ammunition types; IF empty THEN RETURN
  IF the used-type caption EXISTS THEN set it from the first type's short name
  IF ammo picture 1 EXISTS THEN show the first type's inventory grid cell
  IF ammo picture 2 EXISTS THEN
    show the second type's cell, or a degenerate 1x1 rectangle when there is only one type
```

**Invariants** — Every script result is **quantised to 1/53** — multiplied by 53, floored, divided
by 53. The number is the track length in the shipped bar artwork, so quantising makes a bar land
exactly on a tick and makes two nearly-equal weapons read as equal rather than as a sliver of
difference. The constant has no other justification in the source and must be copied to reproduce
the original's appearance.

Damage alone has separate single-player and multiplayer script functions, because the games balance
damage differently between the two; the other three qualities are shared.

**Notes** — The colour comparison on the magazine label is **inverted relative to the label's own
text**: the label shows the *hovered* weapon's magazine size, while the colour is chosen by
comparing it against the equipped one. Red means the hovered weapon holds less. The three colours
are literal — grey, red, green — not theme entries.

An ammunition picture is placed by converting the item's inventory grid coordinates into atlas
pixels and then stretching the widget to the cell's size **scaled horizontally by the canvas
factor**, which is what keeps the picture square on a wide display; see chapter 15 on the
non-uniform canvas scale.

The single-type case sets a degenerate texture rectangle rather than hiding the second picture,
which is the same "shrink instead of hide" idiom as [`CUISleepStatic`](UISleepStatic.cpp.md).

## `Check`

**Contract** — Decides whether an item section is a weapon this panel can describe. The test is
"does it have a base dispersion value" plus four exclusions, each of which is an item that has one
and is not a weapon in the sense the panel means.

```text
FUNCTION Check(section) -> bool
  IF section has no base dispersion THEN RETURN false
  IF section declares a magazine size of zero THEN RETURN false   # a fake or melee weapon
  IF section's class is the knife class       THEN RETURN false
  IF section is the silencer attachment       THEN RETURN false
  IF section is either binocular section      THEN RETURN false
  RETURN true
```

**Notes** — The exclusions are a list of specific shipped item section names, not a category test,
because the data has no flag that distinguishes them. A mod adding a non-weapon with a dispersion
value would need to be added here — which is a real limitation of the shipped design, recorded
rather than fixed.

## `CUIConditionParams`

**Contract** — One two-position bar showing an item's wear against the equipped item's, with a
caption. Builds from either of two layout vocabularies — a grouped `condition_params` element in the
newer games, or a loose `condition_progress` bar plus a `static_condition` caption in the oldest —
and reports absence when neither is present.

The value shown is the item's condition as a percentage, **plus one, less an epsilon** — so an item
at full condition reads as 100 rather than as 99.9 after flooring, and an item at zero still shows a
sliver. As with the quantisation above, the adjustment is presentational and must be copied to match
the original.

When there is no comparison item, or the hovered item is the equipped one, both positions get the
same value.
