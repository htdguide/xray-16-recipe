# src/xrGame/UIGameCTA.cpp

> The capture-the-artefact interface, and the only place in the game where a player's *live inventory* is turned back into a shopping list — which is what makes their loadout survive a round.

**Needs** — [`UIGameCTA.h`](UIGameCTA.h.md) · [`UIGameMP.h`](UIGameMP.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`Artefact.h`](Artefact.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`ui/UIMpTradeWnd.h`](ui/UIMpTradeWnd.h.md) · [`ui/UIBuyWndBase.h`](ui/UIBuyWndBase.h.md) · [`ui/UISpawnWnd.h`](ui/UISpawnWnd.h.md) · [`ui/UISkinSelector.h`](ui/UISkinSelector.h.md) · [`ui/UIMoneyIndicator.h`](ui/UIMoneyIndicator.h.md) · [`ui/UIRankIndicator.h`](ui/UIRankIndicator.h.md) · [`ui/UIVoteStatusWnd.h`](ui/UIVoteStatusWnd.h.md) · [`ui/UIMessageBoxEx.h`](ui/UIMessageBoxEx.h.md) · [`ui/UIHelper.h`](ui/UIHelper.h.md) · [`xrUICore/ProgressBar/UIProgressShape.h`](../xrUICore/ProgressBar/UIProgressShape.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — reached through its declarations in [`UIGameCTA.h`](UIGameCTA.h.md); callers name that, not this file.
**Tier floor** — T2: inventory traversal and a round-trip between live items and purchase records

## Purpose

Most of this file is the same caption-and-readout layer as
[`UIGameDM.cpp`](UIGameDM.cpp.md), built for a two-team objective mode rather than derived
from the deathmatch. Skip that half.

The half worth reading is the **buy menu round trip**. In this mode the player buys a
loadout at the start of a round, dies, and buys again — and what they already own must be
pre-selected and pre-paid so that they are charged only the difference. That requires
turning a live inventory back into the buy menu's own vocabulary of (group, item, addons)
selections, and it is genuinely hard for one reason: **a partly-used weapon is not the same
item as the one that was bought.**

The answer is *defusing*: before the inventory is read, every loaded weapon in it is
notionally unloaded, its rounds are folded back into whatever partial ammunition boxes the
player is carrying, and any whole boxes' worth are handed to the buy menu as separate
purchases. Only then does the menu see a loadout it can price.

## State

```text
RECORD CaptureUI
  captions           : nine named text widgets, as in the deathmatch
  money, rank, frag limit, reinforcement : readouts
  team icons and scores : four widgets
  team_panels        : the scoreboard, with its own shown flag
  team_select        : the team-choice window
  vote_status        : optional, created and destroyed per vote
  buy_spawn_prompt   : the pay-to-respawn message box

  buy_menu           : optional window, keyed by the team section it was built for
  skin_menu          : optional window, keyed by the same
  cost_section       : the configuration section holding this round's prices
  default_items      : list<PresetItem>    # the loadout a player gets for free

RECORD PresetItem                  # one buy-menu selection
  slot_id  : int (8-bit)
  item_id  : int (8-bit)
  big_id   : int (16-bit)          # invariant: (slot << 8) | item; the two views are kept
                                   #   in step by the record's own setters
```

Invariants:

- the buy menu and the skin menu are each **rebuilt only when the team changes**. Switching
  teams means a different catalogue and a different set of skins; staying on a team means the
  same window, reset.
- the default-items list is rebuilt whenever the player's rank changes, because rank
  substitutes better items into it.
- the team panel's own shown flag is tracked separately from the window's, because the window
  is added to and removed from a draw set rather than being shown and hidden.

## The buy menu round trip

### `ShowBuyMenu`

**Contract** — open the purchase screen with the player's current possessions already
selected.

```text
FUNCTION show_buy_menu()
  IF this is a recorded match THEN RETURN
  REQUIRE the menu exists
  IF it is already shown THEN RETURN
  tell it whether to ignore money and rank      # true during the warm-up period
  reset it; begin a player-items transaction
  SetPlayerItemsToBuyMenu()                     # what they have
  SetPlayerParamsToBuyMenu()                    # their rank and their money for this round
  end the transaction
  show it; tell the mode the menu opened
```

**Invariants** — the selection is wrapped in an explicit begin/end pair so the menu knows
these selections are *already owned* rather than newly chosen. That distinction is what makes
the final price a difference rather than a total.

During the warm-up period money and rank are ignored entirely, so players can try anything.

### `SetPlayerItemsToBuyMenu`

**Contract** — read the local player's inventory into the menu. A player who is dead beyond
respawning gets the default loadout instead.

```text
FUNCTION set_player_items()
  find the local player's actor
  REQUIRE it exists, OR the player is flagged as permanently dead
  IF the actor exists AND is not permanently dead THEN
    TryToDefuseAllWeapons(collecting loose ammunition)
    insert every item in a slot, then everything on the belt, then everything in the pack
    insert every whole box of ammunition the defusing produced
  ELSE
    SetPlayerDefItemsToBuyMenu()
```

**Invariants** — the traversal order — slots, belt, pack — is the order the menu will
re-equip them in, so a rebuild that reorders it changes which weapon the player spawns
holding.

### `TryToDefuseWeapon` and `TryToDefuseGrenadeLauncher`

**Contract** — unload one weapon's magazine back into the player's ammunition, so the weapon
prices as an empty weapon and the rounds price as boxes.

```text
FUNCTION defuse(weapon, all items, out loose ammunition)
  IF the weapon has a grenade launcher attached THEN defuse that separately first
  pick the ammunition type currently loaded  # the launcher's, if it is in grenade mode
  rounds = how many are in the magazine
  box = the configured box size for that ammunition type
  WHILE rounds >= box
    emit one full box into the loose list; rounds = rounds - box
  IF rounds remain THEN
    find a PARTIAL box of that exact type in the inventory that these rounds would FILL
    IF one is found THEN top it up to full
```

**Invariants** — the leftover rounds are only used to complete a box that they exactly fill
or overfill. Rounds that would leave a box still partial are **discarded**. That is the
compromise that keeps the pricing honest: the menu can only express whole boxes, so a
half-full magazine's rounds either complete something the player is carrying or are lost.

The grenade launcher is asserted to hold at most one round, which is a statement about every
shipped launcher and is what lets its loop be a special case of the same code.

**Notes** — the defused ammunition is collected into a buffer sized from the inventory's own
item count, allocated on the stack. That is an allocation decision; what matters is the
bound: at most two entries per inventory item.

### `BuyMenuItemInserter`

**Contract** — offer one item to the menu as an already-owned selection, or refuse it.

An item is refused if it is invalid, is a knife, is an artefact, has no price in this round's
cost section, cannot be traded, or is **a partially-filled ammunition box**.

**Invariants** — each refusal is a rule about the mode:

- the knife is free and always present, so it must never be bought;
- an artefact is the objective, not equipment;
- no price means the mode does not sell it, so it cannot be re-bought;
- a partial ammunition box is refused because the defusing pass has already had its chance to
  fill it, and the menu cannot price a fraction.

A weapon's **addon state** — which scope, launcher and silencer are fitted — travels with the
selection, so a player keeps their attachments across a round.

### `GetPurchaseItems`

**Contract** — read the player's final choices back out, as (addon state, item) pairs, along
with the money difference.

```text
FUNCTION get_purchases(out list, out money_difference)
  preset = the menu's LAST preset, or its DEFAULT preset if the last is empty
  FOR EACH entry IN preset
    resolve its item index by section name
    append (its addon state, that index) once per unit of its count
  IF the local player is permanently dead THEN append the knife explicitly
  money_difference = cost of the ORIGINAL preset - cost of the LAST preset
```

**Invariants** — the money difference is **original minus final**, so a player who bought
less than they started with is refunded. That single expression is the whole economic
consequence of the round trip above.

The knife is added back for a permanently dead player because for them the menu was seeded
with the default list, from which the knife was deliberately excluded.

**Notes** — the addon variable is used as scratch to receive the resolved group and then
immediately overwritten with the real addon state, with a comment saying so. It works because
only the item index is wanted from the resolution.

## The default loadout

### `LoadTeamDefaultPresetItems` and `LoadDefItemsForRank`

**Contract** — build the free loadout a player starts with: the team's configured list,
upgraded by each rank they have earned, plus two boxes of ammunition for each weapon in it.

```text
FUNCTION load_default_items_for_rank()
  LoadTeamDefaultPresetItems(the player's team section)      # reads "default_items"
  FOR rank FROM 1 TO the player's rank
    IF a section named "rank_<n>" exists THEN
      FOR EACH item currently in the default list
        IF that section declares "def_item_repl_<item name>" THEN
          replace that entry with the item it names
  FOR EACH item in the resulting list
    IF it is the knife, or declares no ammunition class, THEN skip
    append TWO boxes of its FIRST ammunition class
```

**Invariants** — the rank upgrades are applied **cumulatively from rank 1 upward**, each
rank's replacement table acting on the result of the previous. So a rank-3 player gets
rank 1's substitution, then rank 2's applied to that, then rank 3's. A rebuild that applies
only the player's own rank's table produces different loadouts.

Ammunition is appended *after* all substitutions, so an upgraded weapon gets its own
ammunition rather than the one it replaced. Two boxes is a compiled-in constant.

**Notes** — every selection in the default list is stored with **slot zero** rather than with
the slot the buy menu reported; the line that would have used the real slot is commented out
beside it in three places. The effect is that default items are always offered from the
menu's first group. Whether that is a fix or a regression is not recoverable.

## The rest

**Contract** — the remaining surface is the deathmatch pattern, re-implemented rather than
inherited:

- `Init(stage)` — two stages only (0 and 2; nothing happens at 1), building every widget from
  this mode's single layout file with no fallbacks. The reinforcement indicator is a progress
  shape if the layout declares one and a text widget otherwise, as in
  [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md).
- the eight caption setters, `ResetCaptions` (which clears six of the eight — the source marks
  the omission "bad"), `SetVoteMessage`, `SetVoteTimeResultMsg`.
- `SetScore` — writes both team scores and the score limit, and pushes the same pair into the
  scoreboard. A limit of zero or less prints as a dash.
- `ShowTeamPanels`, `IsTeamPanelsShown`, `UpdateTeamPanels`, `AddPlayer`, `RemovePlayer`,
  `UpdatePlayer` — the scoreboard.
- `ShowTeamSelectMenu`, `IsTeamSelectShown`, `UpdateSkinMenu`, `ShowSkinMenu`,
  `GetSelectedSkinIndex` — team and appearance choice, both suppressed while watching a
  recording.
- `ShowBuySpawn`, `HideBuySpawn`, `IsBuySpawnShown` — the pay-to-respawn prompt, whose text
  is a **string-table format string** filled with the player's money and the spawn cost. The
  format string comes from the translation, so a translator controls the argument order.
- `IR_UIOnKeyboardPress` / `Release` — one raw scancode is intercepted (the player-name
  overlay, held or toggled depending on a server setting) and six bound actions are forwarded
  to the game mode.
- `Render`, `OnFrame` — the team icons are drawn **before** the base's pass and the vote
  banner after it, because neither is a child of the main window.

**Notes** — the raw scancode in the input handler is the one place in the interface that reads
a physical key rather than an action, and it is therefore not rebindable. It is also the only
key whose press-and-hold versus toggle behaviour is a *server* setting rather than a client
one.
