# src/xrGame/ui/UIChatWnd.cpp

> A one-line chat entry that has two authored positions, because the screen it sits on is
> laid out differently while a round is waiting to start.

**Needs** — [`UIChatWnd.h`](UIChatWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIGameLog.h`](UIGameLog.h.md) · [`../../xrUICore/EditBox/UIEditBox.h`](../../xrUICore/EditBox/UIEditBox.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`UIChatWnd.h`](UIChatWnd.h.md)
**Tier floor** — T3: an edit box, two rectangles and one send

## Purpose

The chat line is the smallest screen in the chapter and has exactly two ideas in it: a
destination that is chosen before the screen is raised, and two layouts that are switched
between by game phase.

## State

```text
RECORD ChatEntry
  prefix   : static widget         # "to all:", "to team:"
  edit     : edit box
  to_all   : bool                  # destination of the NEXT message
  pending  : bool                  # which of the two layouts is applied

  in_progress_prefix_rect, in_progress_edit_rect : rect
  pending_prefix_rect,     pending_edit_rect     : rect
```

Invariant: the two rectangle pairs are read once at build time and never recomputed. Switching
layouts assigns rectangles; it does not re-read the document.

## `Init`

**Contract** — Build the prefix and the edit box from the document and remember their
rectangles as the *in-progress* layout. Then look for two further elements describing the
*pending* layout; when **both** are present their rectangles are read, and when either is
missing both pending rectangles fall back to the in-progress ones — so a document that does
not describe the second layout simply never moves.

Finally, bind the edit box's commit and cancel notifications.

**Invariants** — All-or-nothing on the pending pair. Reading one and defaulting the other
would put the prefix and the edit box in different layouts.

## `PendingMode`

**Contract** — Apply one of the two layouts, and do nothing if it is already applied. The
pending layout is used while a round has not started; the in-progress one while it is running.
The chat line moves because the surrounding screen — the scoreboard, the spawn prompt — is
different in the two phases.

## `SetEditBoxPrefix`

**Contract** — Set the prefix text, shrink the prefix widget to fit it, move the edit box to
start just past it with a five-unit gap, and clear the edit box.

**Invariants** — The prefix is measured, not assumed: "to all" and "to team" are different
widths, and in a localized build they are different widths again. The five-unit gap is the
only constant.

Clearing the edit box here is what makes every chat session start empty; there is no
"remember what I was typing".

## `Show`

**Contract** — Showing captures the keyboard into the edit box; hiding releases it. Without
the capture the chat line would receive the key that raised it.

## `OnChatCommit` / `OnChatCancel`

**Contract** — Commit sends the edit box's text to the game's chat channel with the stored
destination flag, then closes. Cancel just closes.

**Invariants** — The destination is read from state set **before** the screen was raised, not
from a control inside it. The key the player pressed to open chat is what chose it; there is
no way to change destination once typing.

## `NeedCursor`

**Contract** — False. The chat line takes the keyboard but must not summon the pointer, which
would both obscure the game and, in a running round, be meaningless.
