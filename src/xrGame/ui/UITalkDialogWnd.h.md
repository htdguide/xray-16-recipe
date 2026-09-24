# src/xrGame/ui/UITalkDialogWnd.h

> Declares the dialogue screen's visual half — the two lists, the two portraits, the trade button —
> and the three list item types it fills them with.

**Needs** — [`UITalkDialogWnd.cpp`](UITalkDialogWnd.cpp.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`../InfoPortion.h`](../InfoPortion.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Windows/UIFrameLineWnd.h`](../../xrUICore/Windows/UIFrameLineWnd.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UITalkDialogWnd.cpp`](UITalkDialogWnd.cpp.md) · [`UITalkWnd.cpp`](UITalkWnd.cpp.md)
**Tier floor** — T3: declarations of a screen and its list items

## Purpose

Declares the surface implemented in [`UITalkDialogWnd.cpp`](UITalkDialogWnd.cpp.md). The dialogue
screen is split in two: this half owns the widgets and knows nothing about phrase graphs, and
[`CUITalkWnd`](UITalkWnd.cpp.md) owns the conversation and knows nothing about widgets. They
communicate by notification in one direction and by method call in the other.

Exported units:

- `CUITalkDialogWnd` — the visual half: answer log, question list, two portraits, trade/exit
  buttons.
  - `InitTalkDialogWnd()` — build, tolerating two different shipped layout vocabularies.
  - `AddQuestion` / `AddAnswer` / `AddIconedAnswer` / `ClearAll` / `ClearQuestions`.
  - `SetOurName` / `SetOthersName` / `SetOsoznanieMode` / `SetTradeMode` / `UpdateButtonsLayout`.
  - `TryScrollAnswersList`, `FocusOnNextQuestion`, `FocusOnFirstQuestion`, `FocusOnLastQuestion`.
  - `m_ClickedQuestionID` — the identifier of the question just clicked; read by the conversation
    half after the notification arrives.
  - `mechanic_mode` — whether the trade button means "upgrade" instead.
- `CUIQuestionItem` — one selectable question row, carrying its phrase identifier.
- `CUIAnswerItem` — one logged utterance: speaker name and text.
- `CUIAnswerItemIconed` — an answer row with a picture, used for item transfers.
