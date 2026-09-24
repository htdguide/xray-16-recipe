# src/xrGame/ui/UIMainIngameWnd.cpp

> The heads-up overlay: it samples the actor's condition, equipment and surroundings once
> every ten frames and turns the result into a strip of graded warning icons, a minimap, a set
> of quick-use slots and a pick-up preview.

**Needs** — [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`UIZoneMap.h`](../UIZoneMap.h.md) · [`UIMotionIcon.h`](UIMotionIcon.h.md) · [`UIHudStatesWnd.h`](UIHudStatesWnd.h.md) · [`UIArtefactPanel.h`](UIArtefactPanel.h.md) · [`UIMessagesWindow.h`](UIMessagesWindow.h.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md) · [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIGameSP.h`](../UIGameSP.h.md) · [`Actor.h`](../Actor.h.md) · [`ActorCondition.h`](../ActorCondition.h.md) · [`Inventory.h`](../Inventory.h.md) · [`CustomOutfit.h`](../CustomOutfit.h.md) · [`ActorHelmet.h`](../ActorHelmet.h.md) · [`WeaponMagazined.h`](../WeaponMagazined.h.md) · [`Level.h`](../Level.h.md) · [`game_news.h`](../game_news.h.md) · [`game_cl_capture_the_artefact.h`](../game_cl_capture_the_artefact.h.md) · [`HudSound.h`](../HudSound.h.md) · [`xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md)
**Used by** — [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md)
**Tier floor** — T3: polling game state and driving widgets. Nothing here is device-facing.

## Purpose

The one screen that is on while the player is playing. It is not a menu: it never takes
input, it is disabled at construction so no pointer event can reach it, and everything it
shows is *derived* — a projection of the actor's condition, inventory and surroundings onto a
fixed set of widgets.

Its layout document, `maingame.xml`, is the single largest frozen element-name surface in the
chapter: this file reads about forty elements by name, and the shipped document supplies them.
Elements the document omits are simply absent — most of the icon lookups are optional, and an
absent widget is silently skipped. That is what lets one implementation serve three games with
different heads-up designs.

## State

```text
RECORD Overlay EXTENDS Window
  minimap            : ZoneMap            # not a child window; drawn out of band
  motion_icon        : MotionIcon         # noise/luminosity indicator, possibly parented to
                                          # the minimap frame instead of to the overlay
  hud_states         : HudStatesWnd       # element "hud_states": health, stamina, armour
  artefact_panel     : optional<ArtefactPanel>   # single player only, element "artefact_panel"

  icon_strip         : ScrollView         # element "icons_scroll_view": the warning icons,
                                          # packed in the order they were turned on
  warning_icons      : map<WarningIcon, Static>  # jammed weapon, invincible, artefact
  thresholds         : map<WarningIcon, list<real>>  # from configuration; see below

  condition_icons    : the seven graded indicators — bleeding, radiation, starvation,
                       weapon damage, helmet damage, outfit damage, overweight
  booster_icons      : the eight active-effect indicators
  flashing_icons     : map<FlashingIcon, Static>  # built from the document, not from a
                                                  # fixed list

  quick_slot_icons   : list<Static>       # elements "quick_slot0…N"; each owns a "counter"
  quick_slot_texts   : list<Static>       # elements "quick_slot0_text…"; the key legends
  pick_up_item       : optional<InventoryItem>    # what the crosshair is over
  pick_up_icon       : Static             # element "pick_up_item"
  pick_up_box        : Rect               # the authored rectangle, captured at construction
  quick_help         : Static             # element "quick_info": the look-at action verb
  disk_io            : Static             # element "disk_io": the loading-activity light
  contact_sound      : Sound
  chat_wnd, log_wnd  : optional<Window>   # multiplayer, adopted for updating only
```

**Invariants**

- The overlay is **disabled** immediately after its layout is applied and stays disabled. It
  is a display surface, not an input target; anything clickable over the world belongs to a
  different screen.
- The minimap and the motion icon are drawn **outside** the normal child traversal. The
  minimap is not a child at all, and the motion icon is hidden for the duration of the child
  traversal and restored afterwards, so that both land *under* the rest of the overlay
  regardless of where they sit in the child list. This is the one place in the chapter where
  draw order is not list order, and it exists because the motion icon may be re-parented into
  the minimap's frame.
- Every warning icon's alpha byte doubles as its on/off flag. There is no separate visibility
  state: a colour with a zero alpha means *off*, and turning one off means writing a
  fully-transparent white. A rebuild that splits colour from visibility must keep the two in
  step, because the only call the game layer makes is "set this icon to this colour".
- The icon strip owns the *order* of the warning icons. An icon is appended to the strip when
  it turns on and removed when it turns off, so the strip compacts and the icons are ordered
  by when they appeared, not by their enumeration order.
- Most widget lookups are optional. A missing element yields nothing and every later use is
  guarded. Only a handful — the pick-up icon, the look-at hint, the icon strip, the disk light
  — are required and would fail loudly.

## `Init`

**Contract** — open the overlay layout document, apply its root section to this window,
disable the window, then create every element listed in the state record above, read the
warning thresholds from configuration, build the flashing icons from a repeated element, bring
up the minimap and motion icon, discover the quick-use slots by counting upward, and load the
new-contact sound. Allocates heavily; runs once per session.

```text
FUNCTION init()
  doc <- load_layout("maingame.xml")
  apply(doc, "main", self)
  self.enabled <- false                    # the overlay never takes input

  pick_up_icon <- create_static(doc, "pick_up_item")
  pick_up_icon.shader <- equipment_icon_atlas
  pick_up_box <- pick_up_icon.rect         # captured before anything resizes it
  quick_help  <- create_static(doc, "quick_info")
  icon_strip  <- create_scroll_view(doc, "icons_scroll_view")

  FOR EACH name IN the seven condition indicators AND the eight booster indicators
    create optionally; booster indicators start hidden

  warning_icons  <- the jammed-weapon and invincibility icons, both hidden, both
                    UNATTACHED — the strip adopts them when they turn on
  IF game_type IS artefact_hunt OR capture_the_artefact THEN
    warning_icons.artefact <- likewise

  load_thresholds()
  build_flashing_icons(doc.subtree("flashing_icons"))

  motion_icon.init() -> attached_to_minimap
  minimap.init(attached_to_minimap)
  IF attached_to_minimap THEN
    minimap.frame.adopt(motion_icon); motion_icon.fit_to(minimap.frame.rect)
  ELSE
    self.adopt(motion_icon)

  disk_io <- create_static(doc, "disk_io")
  IF single_player AND doc HAS "artefact_panel" THEN artefact_panel <- from(doc)
  hud_states <- from(doc, "hud_states")
  discover_quick_slots(doc)
  contact_sound <- load_sound("maingame_ui", "snd_new_contact")

FUNCTION discover_quick_slots(doc)
  i <- 0
  WHILE true
    slot <- create_static_optional(doc, "quick_slot" + i)
    IF slot IS none THEN BREAK              # the count is however many the document defines
    create_static(doc, "quick_slot" + i + ":counter", parent: slot)
    quick_slot_icons.append(slot)
    quick_slot_texts.append(create_static(doc, "quick_slot" + i + "_text"))
    i <- i + 1
```

**Notes**

- The quick-slot count is **discovered, not declared**: the loop stops at the first missing
  element. Three games with three slot counts therefore need no code change.
- The threshold loader walks the warning-icon enumeration from the first real icon up to but
  not including the invincibility icon, and indexes a parallel table of configuration keys by
  *the enumeration value minus one*. That offset only works because the wildcard value is
  zero and comes first. The key table has seven entries while the loop reads one; the other
  six name icons that were removed from the overlay and whose configuration entries still
  ship. Nothing reads them, and this is the clearest example in the chapter of shipped
  configuration outliving its consumer.
- The flashing icons are built from *repeated* elements rather than from named ones: each
  carries a `type` attribute naming which of the two it is, duplicates are rejected, and an
  unrecognised type is fatal. A rebuild may keep the two as fixed elements; the repeated form
  buys nothing here.

## `Draw`

**Contract** — runs every frame. Fades the disk-activity light, feeds the motion icon its two
inputs, renders the minimap, renders the child tree with the motion icon suppressed, and then
renders the look-at hint. Draws nothing beyond the disk light when there is no living actor to
view.

```text
FUNCTION draw()
  IF filesystem opened anything since the last frame THEN disk_light_time <- now
  age <- now - disk_light_time
  IF age >= 1 second THEN disk_io.hide()
  ELSE disk_io.show(); disk_io.tint(white with alpha = 255 * (1 - age))
  reset the filesystem's open counter

  IF NOT single_player THEN
    motion_icon.luminosity <- smoothed(scene luminosity at the viewed entity)
  actor <- current_view_entity AS actor
  IF actor IS none OR actor IS dead THEN RETURN

  motion_icon.noise <- actor.sound_noise
  motion_icon.draw()          # explicitly, first: it must land under everything
  minimap.visible <- true
  minimap.render()

  was <- motion_icon.shown; motion_icon.hide()
  draw children normally
  motion_icon.shown <- was

  draw_look_at_hint()
```

**Notes**

- The disk light is a one-second decay from the last frame in which the virtual filesystem
  opened a file, and the counter it reads is reset here — this screen is the counter's only
  consumer. It is a *hitch* indicator: it lights when the game streamed something mid-play.
- The luminosity feed is multiplayer-only and heavily smoothed — one per cent of the new
  reading per frame — because it drives a visible stealth indicator and a raw reading flickers.
  It is read through a logarithm and an exponential whose factor is one, so the pair cancels;
  what survives is the clamp at one thousandth, which keeps the reading out of the logarithm's
  singularity. A rebuild can write the clamp alone.
- The motion icon is drawn and then hidden across the child traversal, as described in the
  invariants.

## `Update`

**Contract** — runs every frame, but does real work once in ten. Every frame it updates the
child tree, the minimap, the motion icon's stamina reading and the pick-up preview, and pumps
the adopted multiplayer chat and log windows. Every tenth frame it additionally recomputes the
invincibility icon, the condition strip, and — in the two artefact game types — the
artefact-carrier icon. Returns immediately when there is no actor.

**Notes**

- The ten-frame stride is the load-bearing decision: the condition strip reads a dozen
  inventory slots and configuration values, and it is not worth doing at frame rate. Anything
  that must be smooth — the minimap, the stamina arc, the pick-up preview — is deliberately on
  the *other* side of that early return.
- The invincibility icon follows the viewed player's invincibility flag, or the local god-mode
  setting, or is forced on when there is no player record at all. During demo playback the
  player watched is the demo's camera target rather than the local one.
- In capture-the-artefact the icon distinguishes three states by colour: red for carrying the
  *enemy's* artefact, green for carrying one's own, off otherwise. The file itself notes this
  belongs in that game mode's own screen; it is here because the icon is here.

## `UpdateMainIndicators` — the graded condition strip

**Contract** — recompute the seven condition indicators from the actor. Each is hidden when
its condition is benign and otherwise shown with one of three textures — green, yellow, red —
chosen by where the condition falls between two thresholds. Bleeding and radiation
additionally get a blinking colour animation whose speed matches the severity. Also refreshes
the quick-use slots and, in single player, the PDA's ranking page.

```text
FUNCTION update_condition_strip(actor)
  update_quick_slots()
  IF single_player THEN pda.ranking_page.refresh()

  # bleeding and radiation: three bands with a matching blink rate
  FOR EACH (indicator, value) IN [(bleeding, actor.bleeding_speed),
                                  (radiation, actor.radiation)]
    IF value IS zero THEN indicator.hide(); indicator.stop_blinking(); CONTINUE
    indicator.show()
    band <- value < 0.35 ? green : value < 0.7 ? yellow : red
    indicator.texture   <- the band's texture
    indicator.blink     <- slow / medium / fast to match the band

  # hunger: graded on a normalised distance from the critical point, not on the raw value
  k <- (satiety - critical) / (satiety >= critical ? 1 - critical : critical)
  IF k > 0.5 THEN hunger.hide()
  ELSE hunger.show() WITH green above 0, yellow above -0.5, red below

  # the three wear indicators: shown only below three quarters condition
  FOR EACH (indicator, item) IN [(outfit_worn, outfit slot), (helmet_worn, helmet slot)]
    indicator.hide()
    IF item EXISTS AND item.condition < 0.75 THEN
      indicator.show() WITH green above 0.5, yellow above 0.25, red below

  # the weapon indicator is graded against THAT WEAPON's misfire range, not a constant
  weapon_worn.hide()
  IF the active slot is one of the two weapon slots AND it holds a weapon THEN
    start <- weapon.misfire_start_condition;  end <- weapon.misfire_end_condition
    IF weapon.condition < start THEN
      weapon_worn.show() WITH green above (start+end)/2, yellow above end, red below

  # overweight: warns ten units before the walking limit
  overweight.hide()
  IF single_player AND carried_weight >= walk_limit - 10 THEN
    overweight.show() WITH red above the limit, yellow below it
```

**Notes**

- **The three bands are hard-coded, and the configured thresholds are not used here.** The
  thresholds read at construction feed only the warning strip, which in the shipped build has
  one live icon. This split is historical: the graded condition indicators were added later
  with their fractions written into the code. A rebuild is free to make them data, and should
  note that doing so changes nothing the shipped data can express today.
- **The weapon indicator is the only one graded against a per-item range.** A weapon declares
  the condition at which misfires begin and the condition at which they are certain; the icon
  divides that interval in half. So two weapons at the same condition can show different
  colours, which is correct and is the intended reading of the icon.
- The hunger measure is normalised *separately on each side* of the critical point, so the
  scale is continuous through the critical value but has different units above and below it.
  That is what makes a single set of comparisons work for satiety values whose critical point
  the configuration may put anywhere.
- The ten-unit overweight margin is a constant with no configuration entry, and the yellow
  band's comparison is degenerate — both branches select yellow — because the intermediate
  band was removed and its branch left behind. Green never appears on this indicator.

## `UpdateBoosterIndicators`

**Contract** — hide all eight active-effect indicators, then show one per effect currently
influencing the actor, mapping several related effect kinds onto one icon: the three
protective families each collapse to a single icon, and restoration effects get one icon each.
An effect with three seconds or less remaining blinks slowly; one with more does not.

**Notes** — the three-second warning is the only feedback the player gets that an effect is
about to lapse, and it is a constant. The collapse of immunity and protection onto one icon
per damage family is deliberate: the player cares which damage kind is covered, not by which
mechanism.

## `DrawMainIndicatorsForInventory`

**Contract** — recompute and draw, out of band, the parts of the overlay the inventory screen
wants to keep visible over itself: the quick-use slots, the active-effect icons and the zone
indicators. Called by the inventory screen from its own draw, not by the overlay's.

**Notes** — this is a deliberate violation of the tree: a second screen borrows widgets that
belong to this one rather than owning copies. It works because the overlay is disabled and
these widgets take no input. A rebuild that insists on one owner per widget must instead let
the inventory screen host its own instances and share the *state* — which is what the
recomputation call here already implies.

## `UpdateQuickSlots`

**Contract** — for each quick-use slot, set its legend from the localized key name and its
icon from the item the slot is bound to, with a count badge. A slot bound to nothing hides its
badge and becomes fully transparent; a slot bound to an item the actor does not carry is drawn
at forty per cent opacity with an invisible badge; a slot whose item is carried is drawn fully
opaque with the count.

```text
FUNCTION update_quick_slots()
  FOR i, legend IN quick_slot_texts
    full <- localize("quick_use_str_" + (i + 1))
    legend.text <- first 3 characters of full, cut at the first comma   # see note

  FOR i, slot IN quick_slot_icons
    badge <- slot.child("counter")
    section <- quick_use_binding[i]
    IF section IS empty THEN badge.hide(); slot.tint(transparent); CONTINUE
    count <- actor.inventory.count_of(section, include_nested: true)
    badge.text <- "x" + count
    slot.bind_icon_from(equipment atlas, grid cell of section)
    IF count IS 0 THEN badge.tint(transparent); slot.tint(white at 100/255)
    ELSE                badge.tint(opaque);     slot.tint(opaque)
```

**Notes** — the legend is the localized *key name* truncated to three characters and then cut
at a comma in the third position. The shipped tables hold entries like `1` or `F1` or a
two-key list; the truncation exists to keep a long binding name from overflowing a small
widget, and the comma check keeps a truncated list from ending in a dangling separator. This
is fragile and obviously so: a binding whose name is longer than three characters displays a
prefix. A rebuild may size the widget to the text instead, at the cost of the shipped layout's
alignment.

## `UpdatePickUpItem` — the pick-up preview

**Contract** — when an item has been nominated and there is a living actor to view, draw that
item's inventory icon inside the authored preview box: the icon is the item's cell rectangle
out of the equipment atlas, scaled down to fit the box (never up), centred in it, and tinted
at three quarters opacity. When nothing is nominated, the preview is hidden.

```text
FUNCTION update_pick_up_preview()
  IF nothing nominated OR no living actor THEN pick_up_icon.hide(); RETURN

  cells_w, cells_h, cell_x, cell_y <- the item's inventory grid placement, from configuration
  scale <- min(1, box.width  / (cells_w * cell_pixels_w),
                  box.height / (cells_h * cell_pixels_h))
  pick_up_icon.texture_rect <- the item's cells in the equipment atlas
  pick_up_icon.stretch <- true
  pick_up_icon.width   <- cells_w * cell_pixels_w * scale * horizontal_canvas_correction
  pick_up_icon.height  <- cells_h * cell_pixels_h * scale
  pick_up_icon.position <- centred inside box
  pick_up_icon.tint(white at 192/255)
  pick_up_icon.show()
```

**Notes** — the width is multiplied by the toolkit's horizontal aspect correction and the
height is not. The canvas scale is non-uniform, so an icon sized in canvas units would be
stretched on a wide display; correcting the width alone restores the item's aspect on screen.
This is one of the two places the whole chapter admits the canvas is not square.

## `SetWarningIconColor` / `SetWarningIconColorUI` / `TurnOffWarningIcon`

**Contract** — set one warning icon, or all of them, to a colour. An icon whose new colour has
a non-zero alpha is tinted, appended to the icon strip if it was not already there, and shown;
an icon whose new colour is fully transparent is removed from the strip and hidden. Turning an
icon off is setting it to transparent white.

**Notes** — the "all icons" case is expressed as a fall-through chain: the wildcard enters at
the top and falls through every icon, while a named icon enters at its own label and breaks
immediately. The mechanism is incidental; the decision — *one call sets either one icon or all
of them* — is not, and a rebuild expresses it with a loop over the set.

## Flashing icons

**Contract** — `SetFlashIconState_` shows or hides one of the two attention icons by kind and
fails loudly on a kind the document did not define. The icons are built at construction and
destroyed explicitly at teardown.

**Notes** — these icons carry an authored colour animation, so "flashing" is a property of the
widget, not of this file; showing one is the whole of turning it on.

## `RenderQuickInfos` — the look-at hint

**Contract** — show the verb for the default action on whatever the crosshair is over, as a
localization identifier, and hide the hint when there is none. Restart the hint's colour
animation whenever the object under the crosshair changes, so that looking at a second object
of the same kind re-plays the fade-in.

**Notes** — the previously-looked-at object is held in a *process-wide* variable rather than
in the screen, which means the hint's animation is shared between however many overlays exist.
Exactly one does, so it works. A rebuild puts it in the record.

## The remaining surface

**Contract** — `ReceiveNews` appends a news record to the message window and wakes the PDA;
the record is required to name a texture. `AnimateContacts` restarts the minimap's contact
counter animation and optionally plays the new-contact sound at the listener's own position,
so it is heard flat rather than placed in the world. `SetMPChatLog` adopts the two multiplayer
windows for *updating only* — they are drawn by whoever owns them. `OnConnected` rebinds the
minimap to the newly loaded level and re-initialises the state panel; `OnSectorChanged`
forwards the player's new sector to the minimap; `reset_ui` clears the nominated pick-up item
and resets the motion icon and state panel, and is what runs between levels.

**Notes** — the missile-force indicator is a process-wide object this screen does not create
but does destroy, because it is drawn as part of the overlay and nothing else owns it. A
rebuild gives it an owner. The header also declares a development-only adjust mode that has no
implementation in this file; it is a stub left from a layout-tuning tool.
