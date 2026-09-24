# src/xrGame/PhraseDialogDefs.h

> Names the shared handle to a conversation and the list-of-dialog-identifiers type, so parties to a dialog can refer to it without depending on its definition.

**Needs** — [`PhraseDialog.h`](PhraseDialog.h.md)
**Used by** — [`InfoPortion.h`](InfoPortion.h.md) · [`PhraseDialog.h`](PhraseDialog.h.md) · [`PhraseDialogManager.h`](PhraseDialogManager.h.md) · [`UITalkWnd.h`](ui/UITalkWnd.h.md)
**Tier floor** — T3: type aliases

## Purpose

A conversation is held simultaneously by both speakers and by the screen showing it, and any
of the three may be the last to let go — so it is reference-counted and passed by shared
handle everywhere. This file gives that handle a name, and gives the "list of dialog
identifiers" that info portions and characters carry a name of its own.

It exists to break a cycle: the parties to a conversation need the handle type, the
conversation needs the parties. A rebuild whose module system tolerates mutual reference
needs no separate file here.

## State

`Stateless.`

## `DIALOG_SHARED_PTR`

**Contract** — a counted handle to one live conversation. Shared ownership is the decision;
the counting mechanism is not. What matters is that releasing the last holder ends the
conversation, and that the conversation-step operation can be handed one of these and remain
valid while its own script effects release other holders.

## `DIALOG_ID_VECTOR`

**Contract** — an ordered list of dialog identifiers. Order is authored and preserved: it is
the order dialogs were declared in, which is the tiebreak when several share a priority.
