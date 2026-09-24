# src/xrGame/InfoPortion.h

> Declares the info portion — the game's unit of authored knowledge — and the shared definition it resolves to.

**Needs** — [`shared_data.h`](../xrServerEntities/shared_data.h.md) · [`PhraseScript.h`](PhraseScript.h.md) · [`xml_str_id_loader.h`](../xrServerEntities/xml_str_id_loader.h.md) · [`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md) · [`PhraseDialogDefs.h`](PhraseDialogDefs.h.md)
**Used by** — [`InfoPortion.cpp`](InfoPortion.cpp.md) · [`PhraseScript.cpp`](PhraseScript.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`inventory_owner_info.cpp`](inventory_owner_info.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`UIInventoryUtilities.cpp`](ui/UIInventoryUtilities.cpp.md) · [`UITalkDialogWnd.h`](ui/UITalkDialogWnd.h.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CInfoPortion` and its shared payload `SInfoPortionData`. The loading substance is
in [`InfoPortion.cpp`](InfoPortion.cpp.md); what this header decides is the *shape*: a
portion is a lightweight handle carrying only an identifier, and everything authored about
that identifier lives in one shared, reference-counted definition that all holders of the
fact point at. Thousands of portions exist and a definition is immutable after load, so
copying one per holder would be pure waste.

Exported units:

- `SInfoPortionData` — the shared definition: dialog names made available, encyclopedia
  articles revealed and retracted, tasks started, script actions to run on the receiver, and
  the portions this one erases.
- `CInfoPortion` — the handle. It is simultaneously a participant in the shared-definition
  scheme and a client of the string-identifier index that maps a portion name to a position
  in one of the XML files named by configuration.
- `Load` — bind to an identifier and pull the shared definition, loading it if this is the
  first use.
- `Articles` / `ArticlesDisable` / `GameTasks` / `DialogNames` / `DisableInfos` — read-only
  views of the definition's lists. The receiver walks all five when a fact lands.
- `RunScriptActions` — fire the authored script effect at the owner who just received the
  fact, with no speaker on the other side; this is the one path by which acquiring knowledge
  can change arbitrary world state.
- `GetText` — declared as the portion's display string, but **no implementation exists** anywhere in the tree and nothing calls it. A rebuild should drop it, or supply the obvious meaning (look the identifier up in the string table).
- `InitXmlIdToIndex` — names the element and the file list the index scan uses.
