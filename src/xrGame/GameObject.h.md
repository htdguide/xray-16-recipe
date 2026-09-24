# src/xrGame/GameObject.h

> Declares the base class every client object in the game derives from, implemented in [`GameObject.cpp`](GameObject.cpp.md).

**Needs** — [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [`xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`script_binder.h`](script_binder.h.md) · [`Hit.h`](Hit.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`AnselManager.h`](AnselManager.h.md) · [`GameObject.cpp`](GameObject.cpp.md) · [`HangingLamp.h`](HangingLamp.h.md) · [`IKFoot.cpp`](IKFoot.cpp.md) · [`IKLimbsController.cpp`](IKLimbsController.cpp.md) · [`InfoPortion.cpp`](InfoPortion.cpp.md) · [`InventoryBox.cpp`](InventoryBox.cpp.md) · [`InventoryBox.h`](InventoryBox.h.md) · [`PHShellCreator.cpp`](PHShellCreator.cpp.md) · [`Phrase.cpp`](Phrase.cpp.md) · [`PhraseDialog.cpp`](PhraseDialog.cpp.md) · [`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md) · [`PhraseScript.cpp`](PhraseScript.cpp.md) · _and 46 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGameObject`, the one class that simultaneously satisfies every role the engine
imposes on an entity: the engine's abstract object interface, the object factory's
product, the spatial registry's member, the scheduler's client, the renderable and the
collidable. Substance — the lifecycle ordering, the navigation bookkeeping, the callback
table — is in [`GameObject.cpp`](GameObject.cpp.md).

Two things in this header are load-bearing rather than declarative and are worth naming
here:

- **The cast table.** Some three dozen `cast_*` predicates, each defaulting to "I am not
  one of those", let any holder of an entity ask "are you an inventory owner / an
  artefact / a stalker / a zone" without a language-level dynamic type query. A rebuild
  should read this as a **capability query**: the base answers *no* to every capability
  and each subclass overrides the one or two it actually has. Which capabilities exist
  at all is the interesting content; the mechanism is incidental.
- **The default-nothing overrides.** A large fraction of the declarations here are empty
  bodies — net export, net import, hit, physics-correction hooks, HUD draw, forced
  transform. They exist so that the frame loop and the network layer may call *every*
  object uniformly and pay nothing for objects that do not participate. The decision is
  that participation is opt-in per entity class, not a per-frame test.

Exported units:

- `CGameObject` — the base client object.
- Identity and naming — `ID`, `setID`, `cName`, `cNameSect`, `cNameVisual` and their
  setters, `Name`. The section name is the entity's configuration key; the visual name
  its model.
- Parentage — `H_Parent`, `H_Root`, `H_SetParent`, and the four transition hooks
  `OnH_B_Chield` / `OnH_A_Chield` / `OnH_B_Independent` / `OnH_A_Independent` that
  bracket a change of owner (an item entering or leaving an inventory).
- Transform and space — `XFORM`, `Position`, `Direction`, `Center`, `Radius`,
  `BoundingBox`, `Sector`, `Orientation`, the four `spatial_*` registry hooks, and
  `on_matrix_change`.
- Lifecycle — `Load`, `PostLoad`, `net_Spawn`, `net_Destroy`, `reinit`, `reload`,
  `DestroyObject`, `NeedToDestroyObject`, `setDestroy`, `object_removed`.
- Serialization — `net_Save`, `net_Load`, `net_SaveRelevant`, `save`, `load` (the save
  game) and `net_Export`, `net_Import`, `net_ImportInput`, `net_Relevant` (the wire).
- Per-frame — `PreUpdateCL`, `UpdateCL`, `PostUpdateCL`, `shedule_Update`,
  `shedule_Needed`, `shedule_Scale`, `renderable_Render`, `OnEvent`.
- Flags — visible, enabled, destroy, local/remote, server-update, ready, and the
  processing counter (`processing_activate` / `processing_deactivate`) that decides
  whether the object receives a per-frame update at all.
- The crow protocol — `MakeMeCrow`, `AmICrow`, `IAmNotACrowAnyMore`, `AlwaysTheCrow`:
  the scheduler's name for an object that has asked for a cheap once-per-frame pass.
- Navigation — `ai_location`, `UsedAI_Locations`, `use_parent_ai_locations`,
  `is_ai_obstacle`, `obstacle`.
- Script surface — `lua_game_object`, `clsid`, `callback`, `clear_callbacks`,
  `spawn_ini`, `GetScriptBinderObject` / `SetScriptBinderObject`, `story_id`.
- The `cast_*` capability table.
- The `ef_*` family — creature, equipment, weapon, anomaly and detector *type codes*
  used by the evaluation-function layer that scores one entity's interest in another.
  Every one defaults to "none of the above".
- Animation-driven movement — `create_anim_mov_ctrl`, `destroy_anim_mov_ctrl`,
  `update_animation_movement_controller`, `animation_movement`, and the visual-callback
  list that lets an owner run code when the model's pose is rebuilt.
- Use and tip text — `use`, `tip_text`, `set_tip_text`, `nonscript_usable`, which
  together decide whether the crosshair offers an action on this object.
- Physics correction/prediction bracket — `PH_B_CrPr`, `PH_I_CrPr`, `PH_A_CrPr`,
  `make_Interpolation`, and the activation-step accessors used by the multiplayer
  reconciliation pass.
- `u_EventGen` / `u_EventSend` — build and dispatch an entity event packet. Static
  helpers on this class only because every entity needs them.

## Notes

The class is declared with a fixed structure packing directive. That is not a rebuild
decision — nothing reads this object as a memory image — but it is a hint that in the
original the per-entity footprint was watched closely: hundreds of thousands of these
exist across a save, and the property block is deliberately a bitfield.

The position stack is a short bounded history of recent transforms, used by the
multiplayer correction path to ask where the object was a few steps ago. Its depth is
four; nothing in the source explains why four rather than any other small number.
