# src/xrGame/ui/UITalkWnd.h

> Declares the conversation half of the dialogue screen: who is talking, which phrase graph is
> active, and how a click becomes a spoken phrase.

**Needs** — [`UITalkWnd.cpp`](UITalkWnd.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`../PhraseDialogDefs.h`](../PhraseDialogDefs.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Buttons/UIButton.h`](../../xrUICore/Buttons/UIButton.h.md) · [`xrUICore/EditBox/UIEditBox.h`](../../xrUICore/EditBox/UIEditBox.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](../AI_PhraseDialogManager.cpp.md) · [`UIGameSP.cpp`](../UIGameSP.cpp.md) · [`actor_communication.cpp`](../actor_communication.cpp.md) · [`script_game_object_inventory_owner.cpp`](../script_game_object_inventory_owner.cpp.md) · [`stalker_animation_head.cpp`](../stalker_animation_head.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) · [`UIGameTutorialSimpleItem.cpp`](UIGameTutorialSimpleItem.cpp.md) · [`UITalkDialogWnd.cpp`](UITalkDialogWnd.cpp.md) · [`UITalkWnd.cpp`](UITalkWnd.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UITalkWnd.cpp`](UITalkWnd.cpp.md). This is the half of the
dialogue screen that knows about phrase graphs and speakers; the widgets are in
[`CUITalkDialogWnd`](UITalkDialogWnd.cpp.md).

Two declarations are behavioural and belong here: the screen **stops the player moving** while it is
open, and it plays voice lines — so it holds one sound handle.

Exported units:

- `CUITalkWnd` — the conversation.
  - `InitTalkWnd()` — build the visual half inside a full-canvas window.
  - `Show(open)` — start or end a conversation.
  - `UpdateQuestions()` / `NeedUpdateQuestions()` — recompute what the player may say, deferred to
    the next frame.
  - `AddQuestion` / `AddAnswer` / `AddIconedMessage` — narrowing wrappers over the visual half that
    also translate and speak.
  - `SwitchToTrade()` / `SwitchToUpgrade()` / `StopTalk()`.
  - `OthersInvOwner()` — who is being talked to.
  - `b_disable_break` — whether the player may walk away.
