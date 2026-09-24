# src/xrGame/UIGameCustom.cpp

> The in-game interface: the base every game mode's screen set derives from, owning the heads-up display, the inventory and handheld-computer screens, the message log, the script-driven text overlays — and, separately, the multiplayer map and weather catalogue.

**Needs** — [`UIGameCustom.h`](UIGameCustom.h.md) · [`UIDialogHolder.h`](UIDialogHolder.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UIPdaWnd.h`](ui/UIPdaWnd.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [`ui/UIMessagesWindow.h`](ui/UIMessagesWindow.h.md) · [`ui/UIHudStatesWnd.h`](ui/UIHudStatesWnd.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`ui/UICellItem.h`](ui/UICellItem.h.md) · [`xrEngine/CustomHUD.h`](../xrEngine/CustomHUD.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: window ownership, a draw order, and a filesystem scan of level archives

## Purpose

Two unrelated things share this file and a rebuild should separate them.

**The in-game interface.** One object per session, created when a level loads and
destroyed when it unloads, owning every screen the player can see while playing: the
heads-up display, the inventory screen, the handheld computer, the message log, and a set
of script-created text overlays with lifetimes. It inherits the screen stack and input
router, so the mode-specific subclasses get modality for free and add only their own
panels. Its most load-bearing content is the **draw order**, which is stated once in
`Render` and is otherwise invisible.

**The multiplayer map catalogue.** A single process-wide object that discovers, once,
which levels support which multiplayer game modes and which weather presets exist. It
lives here because the lobby screens ask for it, and for no better reason.

## State

```text
RECORD GameUI EXTENDS DialogHolder
  root_window     : Window            # everything mode-specific hangs off this
  message_config  : LayoutDocument    # the overlay text styles, one document
  overlays        : list<Overlay>     # script-created text, sorted live-first
  actor_menu      : InventoryScreen
  pda_menu        : HandheldScreen
  hud             : HudWindow
  message_log     : MessageWindow
  indicators_shown: bool

RECORD Overlay
  widget    : StaticText
  name      : text          # the identifier scripts address it by
  expires_at: real          # negative means never
```

**Invariants**

- The overlay list is compacted every frame: expired entries sort to the end and are
  destroyed. An overlay with no expiry lives until a script removes it.
- The interface is built exactly once per level and asserts each of its six owned windows
  is absent before creating it. A second load without an unload is a leak of the whole
  screen set.
- The heads-up display and the crosshair are two *separate* switches, because a screen may
  want to hide the readouts and keep the crosshair, or the reverse.

## `Load`

**Contract** — builds the whole interface. Does nothing when no level is loaded. Creates
the overlay style document, the inventory screen, the handheld screen, the root window,
the heads-up display and the message log — asserting each is not already present — then
runs the three-stage initialization.

**Invariants** — the **three stages** are the extension point and their contract is fixed:

```text
stage 0 : shared setup — the base builds what every mode needs
stage 1 : mode-specific setup — the subclass builds its own panels
stage 2 : attachment — panels built in stage 1 are parented and ordered
```

A subclass calls the base for stages 0 and 2 and does its own work in stage 1, so that
the base's widgets exist before the subclass reads them and the subclass's widgets exist
before the base attaches them. Collapsing the three into one constructor is what the
split exists to avoid, and a rebuild with a declarative screen description avoids it
differently.

## `Render`

**Contract** — draws the whole interface, in an order that is the file's most important
single statement:

```text
FUNCTION Render()
  FOR EACH overlay IN overlays: draw it          # script text sits behind everything
  root_window.Draw()                             # the mode's own panels

  entity <- the current view entity
  IF entity EXISTS THEN
    IF entity is the living player in first person AND weapon drawing is enabled THEN
      FOR EACH inventory slot from first to last
        item <- what is in it
        IF item wants to draw its own interface THEN item.draw_item_ui()
    IF indicators are shown AND heads-up drawing is enabled THEN hud.Draw()

  message_log.Draw()
  DoRenderDialogs()                              # the modal stack, on top of all of it
```

**Invariants**

- The modal screen stack is drawn **last**, unconditionally. An open inventory must cover
  the heads-up display, and nothing in the mode's own panels may paint over it.
- Per-item interfaces — a detector's display, a scope's reticle, a device's own readout —
  are drawn by iterating *slots*, not by iterating the inventory. Slot order is therefore
  the layering order between two items that both draw, and it is stable because slots are
  fixed positions.
- Those per-item interfaces are drawn only for a **living player in first person**. A dead
  player, a spectator, or a third-person view must not see a first-person device.

## `OnFrame`

**Contract** — the per-frame update: the screen stack's own work, then every overlay,
then the compaction of expired overlays, then the root window, the heads-up display (only
while shown and enabled), and the message log.

A process-wide flag can request that every overlay be discarded at once; it is consumed
here. That is how a script or a level transition clears the screen of leftover text in one
call.

## `AddCustomStatic` / `GetCustomStatic` / `RemoveCustomStatic`

**Contract** — the script-facing text overlay surface. Adding takes an identifier, a flag
saying whether the overlay is a singleton, and an optional lifetime; it returns the
overlay, creating one configured from the style document under that identifier. A
singleton request returns the existing overlay of that name if there is one, so a script
that calls it every frame gets one overlay rather than a thousand.

The lifetime comes from the style document's own attribute when the caller does not supply
one, so the *data* decides how long a message stays up. A non-positive lifetime means the
overlay persists until removed.

Lookup is by name and returns nothing if absent. Removal destroys the first match.

**Invariants** — the identifier is both the lookup key and the style document's node name.
One string does two jobs, which means a script cannot create two differently-styled
overlays with the same name, and that is precisely what makes the singleton flag
meaningful.

## `ShowActorMenu` / `HideActorMenu` / `UpdateActorMenu` / `CurrentItemAtCell`

**Contract** — the inventory screen. Showing toggles: if it is up it closes, otherwise the
handheld screen is closed first, the screen is bound to the current view entity as its
owner, put into inventory mode, and opened. The two screens are mutually exclusive by
construction rather than by a rule elsewhere.

`UpdateActorMenu` refreshes the displayed owner and the currently highlighted cell, for
scripts that change an inventory behind the screen's back. `CurrentItemAtCell` returns the
script facade of whatever item the highlighted cell holds, or nothing.

## `ShowPdaMenu` / `HidePdaMenu` / `UpdatePda`

**Contract** — the handheld computer screen, with the same toggle-and-exclude rule
mirrored. `UpdatePda` refreshes its contents.

## `ShowMessagesWindow` / `HideMessagesWindow` / `CommonMessageOut`

**Contract** — the message log's visibility, and appending one line to it.

## `ShowGameIndicators` / `GameIndicatorsShown` / `ShowCrosshair` / `CrosshairShown`

**Contract** — the two independent display switches. The crosshair's state lives in the
engine's shared display flags rather than here, because the weapon code reads it too.

## `OnInventoryAction`

**Contract** — forwards an inventory event to the inventory screen, but only while it is
shown. A closed screen must not react to inventory changes; it re-reads everything when it
opens.

## `update_fake_indicators` / `enable_fake_indicators`

**Contract** — lets a script drive the heads-up status readouts — radiation, bleeding,
hunger and the rest — with values of its own instead of the player's real condition. Used
by tutorials and scripted sequences to show a warning that is not yet true.

## `SetClGame` / `OnConnected`

**Contract** — hands this interface to the game mode so the mode can call back into it, and
— on connecting to a session — builds the interface if it does not exist yet and tells the
heads-up display to reconfigure for the connection.

## `UnLoad` / `OnUIReset`

**Contract** — destroys the six owned windows; and, on a display-mode change, rebuilds the
shared interface materials so that resolution-dependent resources are recreated.

## The overlay entry

**Contract** — a text widget with an expiry. It reports itself live while the expiry is in
the future or absent; it draws and updates only while shown. Setting its text also shows
or hides it — a null text hides it — and restarts its colour animation, so a repeated
message visibly flashes again rather than sitting still.

## `CMapListHelper`

**Contract** — the multiplayer map and weather catalogue: given a game mode identifier,
the list of levels that declare support for it; and the list of weather presets. Built
lazily on first use and never rebuilt.

```text
FUNCTION Load()
  FOR EACH level configuration file found under the levels root
    read its "map usage" section: each key is a game mode this level supports
    record (level name, level version) under each such mode

  # the interesting half: levels that are not currently mounted
  FOR EACH archive known to the virtual filesystem that is NOT open
    read the level name and version from the archive's own header
    mount it under a temporary root, read its level configuration, unmount it
    record it the same way
  restore the levels root

  read the weather presets from the multiplayer map-list configuration
```

**Invariants**

- The catalogue must include levels the player is **not currently able to load**, because a
  multiplayer server may name any level and the lobby must be able to show it. That is why
  unmounted archives are opened, read and closed again, under a scratch root so the live
  filesystem namespace is not disturbed. This temporary remount is the only thing in the
  file that is not window management.
- A level's version is taken from the archive header when one is given and from the level's
  own configuration otherwise, and the pair (name, version) is the identity: two versions
  of the same level are two entries.
- Duplicates are reported and dropped rather than being allowed to appear twice in a lobby
  list.
- A mode with no levels at all, or a catalogue that discovered nothing, degrades to an
  empty entry rather than failing — a missing map list must not prevent the game starting.

**Notes** — the catalogue is a process-wide singleton reached by name. It is data derived
from the filesystem and belongs to whatever owns the lobby, not to the in-game interface.

## `FillDebugTree` / `FillDebugInfo` / `GetDebugType`

**Contract** — debug-only. Renders the interface as a tree in the developer overlay — the
screen stack, the root window, both menus, the heads-up display, the message log and every
overlay, each contributing its own subtree — and exposes the indicator switch. Compiled
out of a release build.

## `CurrentGameUI`

**Contract** — the process-wide accessor for the one live interface. Another instance of
the service-locator pattern the preface describes; a rebuild should pass it.
