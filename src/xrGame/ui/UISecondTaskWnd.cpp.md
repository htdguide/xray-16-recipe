# src/xrGame/ui/UISecondTaskWnd.cpp

> The PDA's pop-over task list: a modal-ish panel of in-progress tasks, each row able to make its
> task active, toggle its map spot, or send the map to it — all by notification, never by calling
> the map.

**Needs** — [`UISecondTaskWnd.h`](UISecondTaskWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Buttons/UICheckButton.h`](../../xrUICore/Buttons/UICheckButton.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [`../GameTask.h`](../GameTask.h.md) · [`../GameTaskDefs.h`](../GameTaskDefs.h.md) · [`../GametaskManager.h`](../GametaskManager.h.md) · [`../map_location.h`](../map_location.h.md) · [`../Level.h`](../Level.h.md) · [`../Actor.h`](../Actor.h.md)
**Used by** — [`UISecondTaskWnd.h`](UISecondTaskWnd.h.md)
**Tier floor** — T3: screen logic over the task registry; nothing device-facing

## Purpose

The task page shows one or two *active* tasks at a time. This panel is how the player sees all of
them and changes which is active. It pops over the map, takes the keyboard and the navigation focus
while open, and gives both back when closed.

Every action a row offers — make this task active, show or hide its map spot, point the map at it —
is issued as a **notification to the page**, never as a call into the map. The row does not know
there is a map. That indirection is what lets the same row type live inside pages with different
map arrangements, and it is the chapter-wide rule this file is the clearest example of.

## State

```text
RECORD TaskListPanel extends Window
  background        : FrameWindow
  caption           : Widget
  close_button      : Button          # accelerated to the UI "back" action
  list              : ScrollContainer # sorted by task priority, descending
  original_height   : real            # the authored height, captured at build
  secondary_only    : bool            # filter storyline tasks out of the list
  hint              : HintWindow      # supplied by the page, not owned

RECORD TaskListRow extends Window
  task          : Task                # the row is meaningless without one
  title         : Button              # clicking makes the task active
  view_toggle   : optional<CheckBox>  # checked == spot hidden  (note the inversion)
  focus_button  : optional<Button>    # present when there is no view toggle
  story_marker  : optional<Widget>    # storyline vs secondary icon
  colours       : [active, unread, read]
```

Invariants:

- A row without a task is never constructed: the builder refuses and the list drops it.
- `view_toggle` and `focus_button` are alternatives, not both — the layout supplies one or the
  other, and the row's behaviour branches on which it got. This is how one row type serves layouts
  from two different games.
- **The check box reads inverted**: checked means the spot is *hidden*. Getting this backwards
  silently inverts the whole map-spot feature.
- The sort key is the task's priority and the order is descending, recomputed by the container on
  every insertion.

## `UITaskListWnd::init_from_xml`

**Contract** — Builds the panel from a named subtree of the task layout document. Requires the page
to have already supplied the hint window. Descends into the subtree so child element names are
relative, and restores the document root afterwards.

Three decisions worth keeping:

- The close button is given the UI **back** action as an accelerator, so the panel closes with the
  same key that backs out of anything else.
- The close button is then **removed from the navigation focus registry**. It is reachable by its
  accelerator and by the pointer, but directional navigation skips it — otherwise it would steal
  focus from the list, which is the only thing in the panel worth navigating.
- The list is given a sort function over task priority, descending, so rows land in priority order
  whatever order they are added in.

The element names are frozen: `background_frame`, `t_caption`, `btn_close`, `task_list`.

## `UITaskListWnd::Show`

**Contract** — The whole open/close protocol, and the densest few lines in the file.

```text
FUNCTION Show(open)
  base.Show(open)
  IF open THEN
    UpdateList()
    message_target.keyboard_capture <- self      # keys come here while open
    focus.lock_to(self)                          # nothing outside is navigable
    IF input is a controller THEN
      IF list is empty THEN
        focus.clear()
        warp the cursor onto the list            # so the player is pointing at something
      ELSE
        focus the first row's title
  ELSE
    IF message_target.keyboard_capturer == self THEN message_target.keyboard_capture <- none
    IF focus.locker == self THEN focus.unlock()
    notify message_target: KEYBOARD_CAPTURE_LOST, payload = self
  notify message_target: TASK_HIDE_HINT
  Enable(open)
```

**Invariants** — Both the keyboard capture and the navigation lock are released only if this panel
is the one holding them. Releasing unconditionally would tear down a capture some other widget took
while this panel was open.

**Notes** — The final `Enable(open)` looks redundant next to `Show`, and is not: a *disabled* panel
is skipped by accelerator matching, and without it the panel's close button would keep answering
the back action after the panel was hidden — closing the PDA would instead be swallowed by an
invisible button. Disabling the whole subtree is the cheapest way to make an invisible panel
inert.

The `KEYBOARD_CAPTURE_LOST` notification carries this panel as its payload, which is how the page
learns *which* child released the capture and can decide where the cursor should go next.

## `UITaskListWnd::UpdateList`

**Contract** — Rebuilds every row from the task manager's current task set, preserving the scroll
position across the rebuild. Includes only tasks in progress, and, when the panel is in
secondary-only mode, excludes storyline tasks. A row that fails to bind its task is dropped.

```text
FUNCTION UpdateList()
  saved_scroll <- list.scroll_position
  list.clear()
  FOR EACH task IN task_manager.tasks
    IF task IS none OR task.state != in_progress THEN CONTINUE
    IF secondary_only AND task.type == storyline THEN CONTINUE
    row <- new TaskListRow
    IF row.init_task(task, self) THEN list.insert(row, owned)
  list.scroll_position <- saved_scroll
```

**Notes** — Rebuilding the whole list rather than diffing it is what makes the preserved scroll
position necessary, and is affordable because the list is at most a few dozen rows and is rebuilt
only when the task set changes or the panel opens.

## `UITaskListWnd::SendMessage`

**Contract** — Forwards every notification *upward to the page first*, then lets the base
propagation and this panel's own bindings run. The panel is a relay: a row's notification about a
task must reach the page, which owns the map, whether or not the panel itself cares about it.

## `UITaskListWnd::OnMouseScroll`

**Contract** — Wheel up and wheel down step the list's scroll bar by one. The panel intercepts the
wheel rather than letting it fall through to the map behind, which would zoom the map instead.

## `UITaskListWnd::OnFocusReceive` / `OnFocusLost`

**Contract** — Both tell the page to hide the tooltip. Any change in what has focus invalidates
whatever the tooltip was describing.

## `UITaskListWndItem::init_task`

**Contract** — Binds the row to a task, points its notifications at the panel, loads the task layout
document afresh, and builds the four child elements from the fixed element path
`second_task_wnd:task_item`. Reads three text colours from the same subtree: active, unread, read.
Returns failure for a missing task, and the list then drops the row.

Both the view toggle and the focus button are **removed from the navigation focus registry** — only
the row's title is navigable, so directional navigation moves one row at a time rather than
three stops per row. The secondary actions stay reachable by their bound UI actions (see
`OnKeyboardAction`).

The element names are frozen: within `second_task_wnd:task_item`, the children `name`, `btn_view`,
`st_story`, `btn_focus`, and the three colour entries `activ`, `unread`, `read`.

## `UITaskListWndItem::update_view`

**Contract** — Runs every frame. Reconciles the row's appearance with the task's current state:
the spot toggle (or the focus button's visibility) with whether the task's map spot is enabled, the
marker icon with storyline versus secondary, the title text with the task's title, the row's height
with the title's wrapped height, and the title's colour with the task's status.

```text
FUNCTION update_view()
  spot <- task.linked_map_location
  spot_shown <- spot exists AND spot.enabled
  IF view_toggle exists THEN view_toggle.checked <- NOT spot_shown   # inverted, see State
  ELSE                       focus_button.shown  <- spot_shown

  IF story_marker exists THEN
    story_marker.texture <- primary-mission icon IF task.type == storyline
                            ELSE secondary-mission icon

  title.text <- localized(task.title)
  title.fit_height_to_text()
  self.height <- max(self.height, title.y + title.height + 10)   # grow only

  IF task is the active storyline task OR the active secondary task THEN
    title.colour <- colours.active
  ELSE IF task.read THEN title.colour <- colours.read
  ELSE                   title.colour <- colours.unread
```

**Notes** — The row grows to fit its title and never shrinks, so a long title permanently widens the
row's slot in the list; the rows are rebuilt from scratch whenever the list is, so this does not
accumulate. The 10-unit gap below the title is authored spacing. The icon names
(`ui_inGame2_PDA_icon_Primary_mission`, `ui_inGame2_PDA_icon_Secondary_mission`) are registered icon
names in shipped data.

## `UITaskListWndItem::SendMessage`

**Contract** — Turns the row's three widgets into four notifications to the panel. This is the
mapping the whole panel exists to perform:

```text
FUNCTION SendMessage(sender, message, payload)
  IF sender == focus_button AND message == BUTTON_DOWN THEN
    notify: TASK_SET_TARGET_MAP, payload = task
  IF sender == view_toggle THEN
    IF message == BUTTON_CLICKED THEN
      notify: TASK_HIDE_MAP_SPOT IF view_toggle.checked ELSE TASK_SHOW_MAP_SPOT, payload = task
      RETURN
  IF sender == title THEN
    IF message == BUTTON_DOWN THEN
      task_manager.set_active(task)          # the one direct call: activation is not a UI concern
      RETURN
    IF message == DOUBLE_CLICK THEN
      notify: TASK_SET_TARGET_MAP, payload = task
  base.SendMessage(sender, message, payload)
```

**Notes** — The check box's state is read *before* the click is applied, which is why the branch
reads the way it does: checked-at-click means the player is turning the spot off. Activating a task
is the one action taken directly rather than by notification, because it changes game state (which
task is active) rather than screen state.

## `UITaskListWndItem::OnKeyboardAction`

**Contract** — While the cursor is over the row, three bound UI actions are translated into pointer
presses on the row's three widgets: accept presses the title, action-1 presses the focus button,
action-2 presses the view toggle. This is what makes the row fully operable from a controller, where
only the title is in the focus registry.

**Notes** — Synthesising a pointer press at the cursor's current position, rather than calling the
widget's handler, is what keeps the button's own visual state machine (pressed, released, clicked)
in step. A rebuild that calls the handler directly gets the action without the button ever looking
pressed.

## The tooltip dwell rule

**Contract** — A row shows its tooltip when the cursor has been over its title continuously for
**700 milliseconds of game time** — scaled by the game's time factor, so the dwell follows slow
motion rather than wall clock. Any pointer press on the row, and any focus change, cancels the
pending tooltip. The row does not draw the tooltip: it notifies the page, which owns the hint window
and draws it above everything.

```text
FUNCTION Update()
  base.Update()
  update_view()
  IF task exists AND cursor is over title AND dwell_armed THEN
    IF now > title.focus_received_at + 700 * time_factor THEN
      notify: TASK_SHOW_HINT, payload = task
```

**Notes** — `dwell_armed` is set when the row receives focus and cleared by any press or focus loss,
which is what stops the tooltip from reappearing immediately after the player clicks through it.
