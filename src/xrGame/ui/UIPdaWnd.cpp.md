# src/xrGame/ui/UIPdaWnd.cpp

> The PDA: a frame, a tab strip and a slot into which exactly one sub-screen is attached —
> where the sub-screen may equally well be one the script layer supplies.

**Needs** — [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`UITaskWnd.h`](UITaskWnd.h.md) · [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md) · [`UIActorInfo.h`](UIActorInfo.h.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md) · [`UILogsWnd.h`](UILogsWnd.h.md) · [`UIScriptWnd.h`](UIScriptWnd.h.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`UIMessagesWindow.h`](UIMessagesWindow.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIGameCustom.h`](../UIGameCustom.h.md) · [`PDA.h`](../PDA.h.md) · [`Level.h`](../Level.h.md) · [`xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [`xrUICore/Static/UIAnimatedStatic.h`](../../xrUICore/Static/UIAnimatedStatic.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIPdaWnd.h`](UIPdaWnd.h.md)
**Tier floor** — T3.

## Purpose

The player's in-fiction computer, and the container for most of the game's non-inventory
screens. Three decisions carry the file.

1. **One active sub-screen, addressed by a section name.** The tab strip does not own the
   pages; it emits a name, and the PDA looks that name up in a table. Everything else —
   opening at a page, restoring the last page, a script supplying its own page — is the same
   lookup.
2. **Every sub-screen is optional and construction is a build-and-test.** Each is constructed,
   asked to initialise, and deleted if it declines. A game whose data lacks a page simply does
   not have it, and every use is guarded.
3. **The PDA does not stop the player moving.** It is modal for input routing but the world
   continues; that is why the pointer handler reports every event as consumed regardless.

## State

```text
RECORD PdaScreen EXTENDS Dialog
  frame            : Static      # element "background_static"; the SUB-SCREEN'S PARENT
  noise            : optional<Static>   # a scanline overlay drawn above everything
  tab_control      : TabControl
  caption          : optional<Static>
  caption_prefix   : text        # the caption widget's authored text, kept as a prefix
  clock            : optional<Static>
  close_button     : optional<Button>
  hint             : optional<Hint>     # shared by every sub-screen

  map, tasks, faction_war, actor_info, ranking, logs : each optional
  active_dialog    : optional<Window>   # may be a script-supplied page
  active_section   : text
```

**Invariants**

- **Sub-screens are children of the frame, not of the PDA.** The frame is the bezel; attaching
  inside it is what makes a page sit in the screen area. Swapping pages detaches from the frame
  and re-attaches.
- The frame also holds the **keyboard capture**, handed to whichever page is active and cleared
  when a page leaves. A page therefore gets keys without being the modal dialog.
- The active page is remembered across closings, so reopening returns to it.
- There is exactly one hint for the whole PDA, created here and passed *into* the sub-screens
  that need one at construction.

## `Init`

**Contract** — load the PDA document, apply its root, and build: the frame; the document's
declared decorative statics as a group; the caption, whose authored text is kept as a prefix;
the tab strip, parented either to the PDA or to the frame depending on which background element
the document defines; the clock, either as its own element or as a frame line's title text; an
optional animated decoration; the close button; and the hint. In single player, construct each
of the six sub-screens and discard any that declines to initialise. Finally normalise the tab
identifiers and, for the second game, re-flow the tab strip.

```text
FUNCTION init()
  doc <- load_layout("pda.xml")
  apply(doc, "main", self)
  frame <- static from "background_static"
  build the document's auto-static group
  caption <- optional static from "caption_static";  caption_prefix <- its authored text

  buttons_parent <- self
  IF doc HAS "mbbackground_frame_line" THEN build it in the frame; buttons_parent <- frame

  clock <- optional static from "clock_wnd"
           OR the title text of an optional frame line "timer_frame_line" in the frame
  IF doc HAS "anim_static" THEN build an animated decoration
  close_button <- optional; bound to the back action and REMOVED from navigation focus
  hint <- optional, from "hint_wnd"

  IF single player THEN
    FOR EACH of (map from "pda_map.xml", tasks, faction_war, actor_info, ranking, logs)
      construct it; IF it declines to initialise THEN discard it

  tab_control <- from "tab", adopted by buttons_parent, reporting to self, accelerators on
  normalise_tab_ids(tab_control)
  noise <- optional static from "noice_static"
  IF this is the second game THEN reflow_tabs(tab_control)
```

**Notes**

- **The close button is deliberately removed from directional navigation** after being bound to
  the back action. It is reachable by that action from anywhere, so leaving it in the focus ring
  would put a redundant stop between the page's own controls.
- The tab-identifier normalisation maps the first game's numeric tab identifiers — zero through
  six — onto the symbolic names the rest of the code uses, and only for buttons whose identifier
  was defaulted rather than authored. The first game's document numbers its tabs; the later ones
  name them. This is the chapter's clearest case of a compatibility shim between shipped data
  generations, and a rebuild that normalises at load keeps one vocabulary downstream.
- Which parent the tab strip gets, and where the clock comes from, are both decided by which
  element the document happens to define. Neither has a flag; the document's shape is the
  switch.

## `SetActiveSubdialog` — the page swap

**Contract** — detach and hide whatever is active, release the frame's keyboard capture, then
resolve the section name against a fixed table of the six built-in pages. Then give the script
layer a chance to override: a named script function is called with the section name and, if it
returns a window, that window becomes the active page instead, bound to the current screen
stack. If a page was resolved, notify the actor that this section was opened, attach the page to
the frame, give it the keyboard capture, show it, record the section and update the caption.
Otherwise leave nothing active.

```text
FUNCTION set_active_subdialog(section)
  IF active_dialog EXISTS THEN
    detach it from the frame if it is there; clear the frame's keyboard capture; hide it

  page, legacy_info <- lookup section IN
    { "eptMap": map,        "eptTasks": tasks,     "eptFractionWar": faction_war,
      "eptStatistics": (actor_info, also notify "ui_pda_actor_info"),
      "eptRanking": ranking, "eptLogs": logs }
  # a page that failed to initialise is absent, and the lookup yields nothing

  IF script "pda.set_active_subdialog"(section) returns a window THEN
    bind it to the current screen stack;  page <- it            # the script wins

  IF page EXISTS THEN
    notify the actor of `section`, and of legacy_info when there is one
    frame.adopt(page);  frame.keyboard_capture <- page;  page.show()
    active_section <- section;  refresh the caption
  ELSE
    active_section <- ""
```

**Notes**

- **The script override runs after the built-in lookup and replaces its result.** So a mod can
  supply a page for an existing tab as easily as for a new one, and a tab whose built-in page is
  missing is where a mod's page naturally lands. This one function is the entire extension
  point for the PDA, and it is the chapter's strongest example of the script layer owning a
  screen.
- **Opening a page notifies the actor.** The section name is sent as an information packet, and
  one page additionally sends a second, older packet name for compatibility with the first
  game's scripts. Story scripts listen for these, so "the player looked at the map" is a game
  event. A rebuild that skips the notification silently breaks scripted sequences.
- The early-out for "already on this section" is commented out, so re-selecting the current tab
  performs a full detach-and-reattach. That re-fires the notification and re-runs the script
  hook, which at least one shipped script relies on.

## `Show`

**Contract** — opening notifies the actor that the PDA was opened and restores the last section,
or — on the very first opening — chooses the map when there is a map and no task page, and the
task page otherwise, setting the tab strip to match. Closing notifies the actor, clears the
overlay's new-task attention icon, hides the active page, discards the two tooltip windows, and
**sets the active page to the task page regardless of what it was**.

**Notes** — that last step is labelled a hack in the source and is: it exists so that a
script-supplied page, whose lifetime the PDA does not control, is not still referenced after the
PDA closes. A rebuild with owned references clears the reference instead.

## `Update` and `Draw`

**Contract** — the update pumps the active page and refreshes the clock: the time to the minute,
plus the date when the clock is the first game's frame-line title rather than its own element.
It also queues the news log's background work onto the frame's parallel task list. The draw
renders the tree, then the hint out of band, then the scanline overlay above everything.

**Notes**

- **The clock's format is chosen by where the widget lives.** A clock parented to the PDA is
  time only; one that is a frame line's title also gets a date. That is a proxy for the game
  generation and is the file's least defensible switch.
- The news log's work is pushed onto a *parallel* queue every frame, so it is executed off the
  main thread alongside the frame. It is the only sub-screen with background work.

## `DrawHint`

**Contract** — ask whichever of the three hint-owning pages is active to draw its own hints,
then draw the shared hint. Only the task page, the map and the ranking page have any.

## The rest

**Contract** — `SetActiveCaption` finds the tab button whose identifier matches the active
section and sets the caption to the authored prefix followed by that button's own text, so the
title bar always names the visible page in the player's language. `Show_MapWnd`,
`Show_SecondTaskWnd` and `Show_ContactsWnd` open the PDA at a named section, the task form also
opening its list panel. `UpdatePda` refreshes the news log and, when the task page is showing,
its task information. `Reset` resets every sub-screen that exists. `NeedCursor` defers to the
active page first, so a page that wants no pointer gets none.

**Notes** — the contacts opener has no page to check for and is written as an unconditional
branch awaiting one; the contacts page is not implemented in this build. The tab exists and the
section name is reserved for a script-supplied page.

## `RearrangeTabButtons`

**Contract** — re-flow the tab strip horizontally for the second game's layout: place each
button at the running cursor, shrink it to its text plus thirty units of padding, and advance
the cursor by that width less six units of overlap. Then size the strip to the total and move it
left by that total, so the strip ends where it began — that is, it grows leftward from its
authored right edge.

**Notes** — the re-flow exists because that game's tab labels are localized and of very
different lengths, and its layout authors the strip's *right* edge. The three constants — thirty
units of padding, six of overlap, five of trailing slack — are the shipped tab art's own metrics
and have no derivation beyond it.
