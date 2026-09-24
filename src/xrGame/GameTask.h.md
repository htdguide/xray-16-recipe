# src/xrGame/GameTask.h

> Declares a quest and its objectives, implemented in [`GameTask.cpp`](GameTask.cpp.md).

**Needs** — [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`GameTask_script.cpp`](GameTask_script.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`UIGameSP.cpp`](UIGameSP.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`map_spot.cpp`](map_spot.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md) · [`UIMapWnd.cpp`](ui/UIMapWnd.cpp.md) · [`UISecondTaskWnd.cpp`](ui/UISecondTaskWnd.cpp.md) · [`UITaskWnd.cpp`](ui/UITaskWnd.cpp.md) · [`map_hint.cpp`](ui/map_hint.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the quest system's two classes — an objective, and a task which *is* an
objective plus a list of them. Substance is in [`GameTask.cpp`](GameTask.cpp.md); the
enumerations and the saved registry are in [`GameTaskDefs.h`](GameTaskDefs.h.md).

Exported units:

- `SGameTaskObjective` — one step of a quest: its text, icon, map marker description,
  deadline, and the four condition lists (checked-for-completion, checked-for-failure,
  fired-on-completion, fired-on-failure).
  - `UpdateState` — evaluate the conditions and report the state they imply, without
    changing anything.
  - `SetTaskState` — move to a state and run its consequences.
  - `CreateMapLocation`, `RemoveMapLocations`, `ChangeMapLocation`, `LinkedMapLocation`
    — the map marker's lifecycle.
  - `ChangeStateCallback` — the overridable notification hook.
  - `save` / `load`.
  - The script-facing property accessors and the eight condition-adding calls.
  - `CommitScriptHelperContents` — resolve script-added condition names into callable
    handles.
- `CGameTask` — a quest: an objective plus sub-objectives, an authored identifier, a
  display priority and a read flag.
  - `Load` — build from the authored XML definition.
  - `Objective`, `ObjectiveState`, `ActiveObjective`, `ActiveObjectiveIdx`,
    `SetActiveObjective`, `GetObjectivesCount`, `HasObjectiveInProgress` — addressing
    the tree, with index zero meaning the task itself.
  - `SetTaskState(state, objective)` — set one objective, cascading to in-progress
    siblings when the root is addressed.
  - `OnArrived` — hand the quest to the player.
  - `FillEncyclopedia` — unlock the articles the objectives name.
  - `AddObjective_script`, `GetObjective_script` — the script-side tree builder.
- `SScriptTaskHelper` — the four *name* lists for conditions a script added at runtime,
  and the resolution step that turns them into callable handles. It exists because a
  resolved handle cannot be saved and a name can.

## Notes

The objective's fields are public and the task's tree is private, which is an artifact of
how the script binding was written rather than a designed encapsulation boundary. Treat
the whole record as the module's own state.
