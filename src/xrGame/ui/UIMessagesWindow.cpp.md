# src/xrGame/ui/UIMessagesWindow.cpp

> The message overlay: one log in single player, three widgets in multiplayer, and the layout
> arithmetic that makes a news item's row fit its icon, its timestamp and its wrapped body.

**Needs** — [`UIMessagesWindow.h`](UIMessagesWindow.h.md) · [`UIGameLog.h`](UIGameLog.h.md) · [`UIChatWnd.h`](UIChatWnd.h.md) · [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`game_news.h`](../game_news.h.md) · [`game_type.h`](../game_type.h.md)
**Used by** — [`UIMessagesWindow.h`](UIMessagesWindow.h.md)
**Tier floor** — T3.

## Purpose

The transient text that appears over the world: story messages in single player, chat and kill
reports in multiplayer. Its structure is decided entirely by the game kind, and the two shapes
share only the file.

## State

```text
RECORD MessageOverlay EXTENDS Window        # always the full 1024x768 canvas
  game_log  : GameLog            # single player: PDA messages. multiplayer: kill reports
  chat_log  : optional<GameLog>  # multiplayer only
  chat_wnd  : optional<ChatWnd>  # multiplayer only: the entry field
  pending_rect, inprogress_rect : Rect      # the chat log's two authored positions
  in_pending_mode : bool
```

**Invariants**

- The overlay is always exactly the virtual canvas. It does no layout of its own; every child
  is placed absolutely by the layout document.
- In single player the chat log and chat window do not exist, and every use is guarded. The
  game log exists in both kinds but is configured from a *different* element and, in
  multiplayer, additionally given a font — because kill reports and story messages are not
  styled alike.

## `Init`

**Contract** — open the overlay's layout document. Always create the game log and configure it
from `sp_log_list` in single player, or from `mp_log_list` plus its font in multiplayer. In
multiplayer additionally create the chat log and the chat entry, configure the chat log from
`chat_log_list`, remember that rectangle as the in-progress position, read the pending
position from a separate element — falling back to the in-progress one when the document omits
it — and configure the chat log's font.

**Notes** — the pending rectangle is read as four raw attributes and assembled by hand rather
than through the layout reader, because the element is a *position record*, not a widget. A
rebuild whose layout format can express "a named rectangle" needs no special case.

## `PendingMode`

**Contract** — move the chat log to its pending rectangle and put the chat entry into pending
mode, or back. Idempotent in both directions.

**Notes** — "pending" is the pre-match lobby, where the round has not started and the chat gets
more room because nothing else is on screen. Two authored rectangles and a flag is the whole
mechanism.

## `AddIconedPdaMessage`

**Contract** — append a message row and fill it in: the receipt time rendered to the minute and
shrunk to its text; the caption placed three units to the right of the time; the body text set
and grown to its wrapped height; the row's fade animation started with the record's own
duration; the icon bound; and the row's height set to the greater of the icon's height and the
bottom of the body, plus three units. Then tell the log its child changed size, which is what
makes the log re-lay-out.

```text
FUNCTION add_news(news)
  row <- game_log.new_message_row()
  row.time.text <- format_clock(news.received_at, to_minutes)
  row.time.shrink_to_text()
  row.caption.position.x <- row.time.right + 3
  row.caption.text <- localize(news.caption)
  row.body.text <- localize(news.text);  row.body.fit_height_to_text()
  row.start_fade(named "ui_main_msgs_short", over news.show_time seconds)
  row.icon.bind(news.texture)
  row.height <- max(row.icon.height, row.body.bottom) + 3
  game_log.on_child_resized(row)
```

**Notes**

- **The row fades itself out; the log does not expire it.** The record carries its own display
  duration and the row runs a named alpha animation for exactly that long. A rebuild that
  instead removes rows on a timer changes the look — the original's messages fade, they do not
  vanish.
- The three-unit gap after the timestamp and the three units of bottom padding are the only
  two numbers here, and both are plain visual spacing.
- The caption is moved but not resized, so a long timestamp can push the caption past the row's
  edge. The shipped clock format is fixed-width, so it never does.
- The explicit "child changed size" notification is required because the row's height was set
  after it was already in the log. A rebuild whose container observes its children does not
  need it.

## The remaining surface

**Contract** — the two log-append forms and the chat-append form pass straight through to the
log that owns that kind of message. `Show` propagates to whichever of the three children
exist — note that it does *not* set the overlay's own visibility, so the overlay is always
"shown" and its emptiness is what hides it.
