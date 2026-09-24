# src/xrGame/PhraseDialog.h

> Declares the dialog: a shared authored phrase graph plus one conversation's traversal of it, implemented in [`PhraseDialog.cpp`](PhraseDialog.cpp.md).

**Needs** — [`shared_data.h`](../xrServerEntities/shared_data.h.md) · [`Phrase.h`](Phrase.h.md) · [`PhraseDialogDefs.h`](PhraseDialogDefs.h.md) · [`xml_str_id_loader.h`](../xrServerEntities/xml_str_id_loader.h.md) · [`xrAICore/Navigation/graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`PhraseDialog.cpp`](PhraseDialog.cpp.md) · [`PhraseDialogDefs.h`](PhraseDialogDefs.h.md) · [`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md) · [`PhraseDialog_script.cpp`](PhraseDialog_script.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`UITalkWnd.cpp`](ui/UITalkWnd.cpp.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `SPhraseDialogData` (the shared, immutable authored graph) and `CPhraseDialog` (one
live conversation over it). Substance is in [`PhraseDialog.cpp`](PhraseDialog.cpp.md).

The shape this header fixes is worth stating once: a conversation is a *reference-counted*
object that three parties hold at the same time — both speakers and the conversation screen
— and any of them can end it. That is why it is handed around by shared handle and why the
step function is a free operation over such a handle rather than a method.

Exported units:

- `SPhraseDialogData` — caption, phrase graph, whole-dialog preconditions, list priority.
- `CPhraseDialog` — the conversation, combining the shared-definition scheme, the
  string-identifier index, and reference counting.
- `Load` — bind an identifier, loading the shared definition on first use.
- `Init` / `IsInited` / `Reset` — bind two speakers and position at the entry phrase.
- `Precondition` — may this conversation start between these two.
- `PhraseList` / `allIsDummy` — the replies currently offered, and whether they are all
  textless.
- `SayPhrase` — the conversation step. Static, over a shared handle, because it can outlive
  its own side effects.
- `GetPhraseText` / `GetLastPhraseText` / `GetLastPhraseID` — text resolution, live and
  retrospective.
- `DialogCaption` / `Priority` / `GetDialogID` — the player's dialog-list surface.
- `IsFinished` — the graph ran out of edges.
- `FirstSpeaker` / `SecondSpeaker` / `CurrentSpeaker` / `OtherSpeaker` / `LastSpeaker` /
  `FirstIsSpeaking` / `SecondIsSpeaking` / `IsWeSpeaking` / `OurPartner` — turn bookkeeping.
- `AddPhrase` / `SetCaption` / `SetPriority` / `GetPhrase` — the construction surface, public
  because a dialog may be assembled by a script instead of parsed from a file.
- `InitXmlIdToIndex` — names the element and file list for the index scan.

## Notes

The copy constructor and assignment operator declared here assign the object to itself and
recurse forever. Nothing in the tree copies a dialog, so the defect is unreachable — but it
means a rebuild has no authored answer for what copying a conversation should do. Treat a
conversation as non-copyable; sharing is what the reference count is for.
