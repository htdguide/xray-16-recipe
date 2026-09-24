# src/xrGame/UIGameCTA.h

> Declares the capture-the-artefact interface, including the record that maps a buy-menu selection to a slot and item, implemented in [`UIGameCTA.cpp`](UIGameCTA.cpp.md).

**Needs** — [`UIGameMP.h`](UIGameMP.h.md) · [`game_base.h`](game_base.h.md) · [`Inventory.h`](Inventory.h.md) · [`xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md)
**Used by** — [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md) · [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md) · [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the objective mode's interface. It derives from the multiplayer layer directly
rather than from the deathmatch, so it re-declares the whole caption and readout set instead
of inheriting it — which is duplication a rebuild should collapse.

Substance in [`UIGameCTA.cpp`](UIGameCTA.cpp.md), where the one genuinely interesting part
lives: turning a live inventory back into a priced shopping list.

Exported units:

- `CUIGameCTA` — the interface.
- `PresetItem` — one buy-menu selection, holding the same value twice: as a (slot, item)
  pair and as a single packed identifier, kept in step by its own setters. The packing is
  `(slot << 8) | item`. Both views exist because the menu speaks pairs and the default-items
  list is compared by identifier.
- `BuyMenuItemPair`, `BuyMenuItemsCollection` — how purchases travel out to the game mode.
- `UpdateBuyMenu`, `ShowBuyMenu`, `HideBuyMenu`, `CanBuyItem`, `GetBuyMenuItem`,
  `GetBuyWnd` — the purchase screen. Rebuilt only when the team changes.
- `GetPurchaseItems` — read the final selection back out, with the money difference
  (original cost minus final cost, so an underspend is a refund).
- `ReInitPlayerDefItems` — rebuild the free loadout after a rank change.
- `TryToDefuseAllWeapons`, `AdditionalAmmoInserter`, `BuyMenuItemInserter`,
  `SetPlayerItemsToBuyMenu`, `SetPlayerParamsToBuyMenu`, `SetPlayerDefItemsToBuyMenu`,
  `LoadTeamDefaultPresetItems`, `LoadDefItemsForRank`, `GetBuyMenuItemIndex` — the private
  half of the round trip. Private, but load-bearing: the rules about which items may be
  re-bought, and how a partly-used weapon is priced, live nowhere else.
- `UpdateSkinMenu`, `ShowSkinMenu`, `GetSelectedSkinIndex` — appearance choice, likewise
  rebuilt only on a team change.
- `ShowTeamSelectMenu`, `IsTeamSelectShown`, `ShowTeamPanels`, `IsTeamPanelsShown`,
  `UpdateTeamPanels`, `AddPlayer`, `RemovePlayer`, `UpdatePlayer` — team choice and the
  scoreboard.
- `ShowBuySpawn`, `HideBuySpawn`, `IsBuySpawnShown` — the pay-to-respawn prompt.
- `SetScore`, `SetRank`, `SetReinforcementTimes`, `ChangeTotalMoneyIndicator`,
  `DisplayMoneyChange`, `DisplayMoneyBonus` — the readouts.
- the eight caption setters, `ResetCaptions`, `SetVoteMessage`, `SetVoteTimeResultMsg`.
- `IR_UIOnKeyboardPress`, `IR_UIOnKeyboardRelease` — one raw scancode and six bound actions.
