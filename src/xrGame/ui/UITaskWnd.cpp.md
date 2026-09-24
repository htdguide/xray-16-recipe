# src/xrGame/ui/UITaskWnd.cpp

> The PDA's task page: the map, one or two active-task panels, and the hub every task notification
> in the PDA passes through on its way to the map.

**Needs** — [`UITaskWnd.h`](UITaskWnd.h.md) · [`UISecondTaskWnd.h`](UISecondTaskWnd.h.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`UIMapFilters.h`](UIMapFilters.h.md) · [`UIMapLegend.h`](UIMapLegend.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`../GameTask.h`](../GameTask.h.md) · [`../GametaskManager.h`](../GametaskManager.h.md) · [`../map_location.h`](../map_location.h.md) · [`../map_location_defs.h`](../map_location_defs.h.md) · [`../map_manager.h`](../map_manager.h.md) · [`../Level.h`](../Level.h.md) · [`../Actor.h`](../Actor.h.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UITaskWnd.h`](UITaskWnd.h.md)
**Tier floor** — T3: task and map-spot bookkeeping over the map widget

## Purpose

The page the player opens to see where they are supposed to go. Structurally it is the map with
furniture around it, but its real job is to be the **one place** that knows both the task registry
and the map: the task list, the task panels and the filter widget all describe what they want as
notifications, and this page translates them into map operations. Nothing below it has a reference
to the map.

The page also owns the filter state that decides which categories of map spot are visible at all,
and re-applies it to every spot in the level each time the tasks change.

## State

```text
RECORD TaskPage extends Window
  map               : MapWidget                 # owned outright, not auto-deleted
  storyline_panel   : TaskPanel                 # always present
  secondary_panel   : optional<TaskPanel>       # present only if the layout declares it
  focus_button      : Button                    # point the map at the storyline task
  focus_button_2    : optional<Button>          # the same for the secondary task
  task_list         : TaskListPanel             # a child OF THE MAP, so it floats over it
  legend            : LegendPanel               # likewise
  filters           : optional<FilterWidget>
  index_label       : optional<Widget>          # "3 / 7"
  hint              : HintWindow                # supplied by the PDA, not owned
  last_seen_change  : int                       # the task manager's change counter
```

Invariants:

- The pop-over task list and the legend are attached to the **map**, not to the page, so they are
  drawn above the map and clipped to it. Attaching them to the page would put them behind it.
- `filters` may be absent — the older game ships no filter widget — and **every filter query
  answers "enabled" when it is absent**, so a layout without filters shows everything. This is the
  single most reused line in the file.
- `secondary_panel`'s presence is what puts the task manager into multiple-active-task mode; the
  layout decides how many tasks the game tracks at once.
- `last_seen_change` is compared against the task manager's counter each frame; refresh is
  change-driven, not periodic.

## `Init`

**Contract** — Builds the page. Returns failure if the task layout document is absent, which is how
a data set without a task page is tolerated rather than fatal. Everything else is required or
explicitly optional.

```text
FUNCTION Init() -> bool
  document <- load the task layout; IF absent THEN RETURN false
  configure self from document["main_wnd"]

  optional decorations: a frame window or a frame line named "background", a "task_split" line

  filters <- new FilterWidget; IF it fails to build THEN drop it
             ELSE attach it and point its notifications here

  map <- new MapWidget over the hint window; build from document["map_wnd"]; attach

  storyline_panel <- new TaskPanel from document["storyline_task_item"]
  bind storyline_panel DOUBLE_CLICK -> point the map at the storyline task

  IF document has "secondary_task_item" THEN
    task_manager.allow_multiple_active_tasks(true)
    secondary_panel <- new TaskPanel from it
    bind secondary_panel DOUBLE_CLICK -> point the map at the secondary task

  focus_button   <- button "btn_task_focus";  bind BUTTON_DOWN -> same as above
  focus_button_2 <- optional button "btn_task_focus2"; bind likewise

  list_button <- button "btn_second_task"; bind BUTTON_CLICKED -> toggle the task list
  give it the scores action and the UI action-1 as accelerators

  index_label <- optional static "second_task_index"

  task_list <- new TaskListPanel; hint <- the page's hint
  task_list.init from document["second_task_wnd"]
  task_list.secondary_only <- (secondary_panel EXISTS)
  attach task_list TO THE MAP; point its notifications here; start hidden

  legend <- new LegendPanel from document["map_legend_wnd"]
  attach legend TO THE MAP; point its notifications here; start hidden
  RETURN true
```

**Notes** — The task list is told to show only secondary tasks exactly when the page has a separate
secondary panel: with two panels the storyline task is already on screen, so listing it again would
be redundant. With one panel the list must show everything.

The frozen element names are `main_wnd`, `background`, `task_split`, `map_wnd`,
`storyline_task_item`, `secondary_task_item`, `btn_task_focus`, `btn_task_focus2`,
`btn_second_task`, `second_task_index`, `second_task_wnd`, `map_legend_wnd`.

## `SendMessage` — the hub

**Contract** — Six task notifications are intercepted here and turned into map operations; anything
else falls through to normal propagation and this page's own bindings. This is the translation layer
the whole chapter's event discipline exists for.

```text
FUNCTION SendMessage(sender, message, payload)
  IF message == KEYBOARD_CAPTURE_LOST AND sender == self THEN
    IF payload is the filter widget OR the task list THEN
      IF the input device is a controller THEN warp the cursor onto the map
      RETURN                                     # focus goes back to the map, not onward

  IF message == TASK_SET_TARGET_MAP   THEN TaskSetTargetMap(payload); RETURN
  IF message == TASK_SHOW_MAP_SPOT    AND secondary tasks are enabled
                                       THEN TaskShowMapSpot(payload, true);  RETURN
  IF message == TASK_HIDE_MAP_SPOT    THEN TaskShowMapSpot(payload, false); RETURN
  IF message == TASK_SHOW_HINT        THEN map.show_task_hint(payload, sender); RETURN
  IF message == TASK_HIDE_HINT        THEN map.hide_hint(); RETURN
  IF message == TASK_RELOAD_FILTERS   THEN ReloadTaskInfo(); RETURN

  base.SendMessage(...); run this page's own bindings
```

**Invariants** — Showing a spot is gated on the secondary-task filter; **hiding is not**. That
asymmetry is deliberate: a filter that is off must not be able to reveal a spot, but the player must
always be able to hide one.

The capture-lost branch is what returns control to the map when a pop-over closes: on a controller
there is no pointer to fall back to, so the cursor is warped onto the map.

## `Update`

**Contract** — Refreshes the page when the task manager's change counter moved, then resolves which
panel's tooltip the map should show: the storyline panel's wins, the secondary's is second, and
otherwise the map's current tooltip is dismissed. When the secondary's wins, the storyline panel's
pending tooltip is explicitly cleared, so the two cannot both stay armed.

## `ReloadTaskInfo`

**Contract** — The page's refresh. Re-binds the panels to the currently active tasks, decides
whether each focus button is usable, re-applies every filter to every map spot in the level, and
updates the "n of m" label.

```text
FUNCTION ReloadTaskInfo()
  storyline_panel.bind(task_manager.active_task(storyline))
  IF secondary_panel EXISTS THEN secondary_panel.bind(task_manager.active_task(additional))

  # a focus button is useless for a task with no map location
  focus_button.shown   <- the storyline task exists AND has a map object AND a location name
  focus_button_2.shown <- the same for the secondary task

  FOR EACH location IN the level's map locations
    CASE its spot type OF
      contains "treasure"                  -> shown IF treasures are enabled
      "primary_object"                     -> shown IF primary objects are enabled
      "secondary_task_location" or its
        complex-timer variant              -> shown IF secondary tasks are enabled
      any of the seven quest-character
        spot types                         -> shown IF quest characters are enabled
      # anything else is left alone

  IF a task changed THEN
    last_seen_change <- task_manager.change_counter
    IF the task list is open THEN task_list.rebuild()

  IF index_label EXISTS THEN
    # whichever of the two tasks exists supplies the index; the secondary wins
    show "<index> / <count>" among in-progress tasks of that type, or hide the label when the
    count is zero
```

**Invariants** — The spot types are matched by name against the shipped map-spot vocabulary, and the
treasure test is a **substring** match while the others are exact — treasure spots come in several
named variants. Unrecognised spot types are deliberately untouched, so a mod's own spot types are
never hidden by a filter they do not participate in.

**Notes** — The change counter is only advanced when at least one of the two tasks exists, so a
player with no active task re-runs this every frame. Harmless — the loop is over a few dozen
locations — but it is why the guard reads the way it does rather than being unconditional.

When both a storyline and a secondary task exist, the label shows the *secondary* task's index,
because the second branch overwrites the first. That is the shipped behaviour.

## `TaskSetTargetMap`

**Contract** — Point the map at a task: enable its spot, recompute the spot's position, and ask the
map to centre on that level and position, jumping levels if necessary. Refused entirely when the
secondary-task filter is off, and refused for a task with no map location.

## `TaskShowMapSpot`

**Contract** — Enable or disable a task's map spot. Enabling also recentres the map on it, so
turning a spot on takes the player there; disabling only hides it. Refused when the secondary-task
filter is off.

**Notes** — That enabling also recentres is a surprise in the name, and it is what makes the task
list's view toggle feel like a "go here" button. A rebuild that separates the two loses that.

## `Show`

**Contract** — Show or hide the page and the map together, dismiss any map tooltip, always close the
legend, and refresh on open. The legend closing on every show — rather than persisting — is
deliberate: it is a reference card, not a mode.

## The filter queries

**Contract** — Four queries and four setters over the four map-spot categories: treasures, quest
characters, secondary tasks, primary objects. Every query answers **enabled** when there is no
filter widget, and every setter is a no-op then. That single convention is what lets the same page
serve a game that ships filters and one that does not.

## `IsUsingCursorRightNow`

**Contract** — Always true. The task page always wants a pointer, because the map is
pointer-driven even when everything else is on a controller.

## `OnKeyboardAction` / `OnControllerAction`

**Contract** — When a child holds the keyboard capture *and* the input device is a controller, input
goes straight to that child, bypassing the page's own handling. On a keyboard the normal routing
applies. This is what makes a pop-over genuinely modal on a controller, where there is no pointer to
disambiguate what the player means.

## `FillDebugTree`

**Contract** — Development only: adds the hint window to the inspector tree alongside the normal
widget subtree. The hint is not a child of anything, so the tree would not otherwise reach it.

## `CUITaskItem` — the active-task panel

**Contract** — One task's icon and caption, plus a dwell-timed tooltip request. Its child widgets
are held in a name-keyed table rather than as fields, so the layout may omit any of them.

```text
FUNCTION Init(document, path)
  configure self from document[path]
  dwell <- document[path].attribute "hint_wt"  (default 500 ms)
  t_icon      <- optional static "t_icon"
  t_icon_over <- optional static "t_icon_over"
  t_caption   <- required static "t_caption"
  IF t_icon_over IS absent THEN t_icon_over <- t_icon     # one icon serves both roles
```

**Invariants** — The over-icon aliasing is why the table holds widgets rather than owning them: two
keys may name the same widget, and destroying the table must not destroy it twice.

`InitTask` with no task turns the icon off and hides the over-icon and blanks the caption, so an
empty panel is a valid state rather than a hidden one.

The tooltip rule is the same as the task list's row: armed on focus received, disarmed on focus lost
or on any pointer press, and fired once the cursor has dwelt for the authored delay scaled by the
game's time factor. The panel sets a flag; the page reads it in `Update` and asks the map to show
the tooltip. The panel does not draw it.

**Notes** — The default dwell here is 500 milliseconds against the task list row's 700. Nothing
explains the difference; both are authored feel.
