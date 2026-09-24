# src/xrGame/HudItem.h

> Declares the base of every item the player can hold, implemented in [`HudItem.cpp`](HudItem.cpp.md).

**Needs** — [`HudSound.h`](HudSound.h.md) · [`actor_defs.h`](actor_defs.h.md) · [`inventory_space.h`](../xrServerEntities/inventory_space.h.md)
**Used by** — [`ActorInput.cpp`](ActorInput.cpp.md) · [`HudItem.cpp`](HudItem.cpp.md) · [`Level_input.cpp`](Level_input.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md) · [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`hud_item_object.cpp`](hud_item_object.cpp.md) · [`hud_item_object.h`](hud_item_object.h.md) · [`player_hud.cpp`](player_hud.cpp.md) · [`player_hud_tune.cpp`](player_hud_tune.cpp.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md)
**Tier floor** — T3: a declaration only, but it fixes the state enumeration every held item extends

## Purpose

Declares two classes. Substance is in [`HudItem.cpp`](HudItem.cpp.md).

`CHUDState` is the state machine's storage and its two clocks, plus the *shape* every
held item obeys: a state, an intended next state, and two abstract operations —
request a state, and apply one. Its enumeration names the five states the base knows —
hidden, idle, showing, hiding, bore — and, crucially, names the **last base state**, so
that a subclass may extend the enumeration from there. That marker is the extension
contract: a weapon's reloading and firing states are values past it.

`CHudItem` is everything a held item has.

Exported units:

- `CHUDState` — `GetState`, `GetNextState`, `SetState`, `SetNextState`,
  `CurrStateTime`, `ResetSubStateTime`, and the two abstract `SwitchState` /
  `OnStateSwitch`.
- `CHudItem` — the held item.
  - `Load`, `OnEvent`, `UpdateCL`, `renderable_Render` — lifecycle and per-frame.
  - `SwitchState`, `OnStateSwitch`, `OnAnimationEnd`, `OnMotionMark` — the state
    machine, driven by the network even in single player and advanced by animation ends.
  - `PlayHUDMotion` (two forms), `PlayHUDMotion_noCB`, `StopCurrentAnimWithoutCallback`
    — playing a motion with or without arming the end transition.
  - `PlayAnimIdle`, `TryPlayAnimIdle`, `PlayAnimIdleMoving`, `PlayAnimIdleMovingCrouch`,
    `PlayAnimIdleSprint`, `PlayAnimBore`, `MovingAnimAllowedNow`, `OnMovementChanged` —
    the movement-driven idle selection.
  - `isHUDAnimationExist`, `WhichHUDAnimationExist` — data-driven fallback between
    animation names and between aspect-ratio variants.
  - `PlaySound` (two forms) and the alias-keyed sound bank.
  - `ActivateItem`, `DeactivateItem`, `SendDeactivateItem`, `SendHiddenItem`,
    `OnActiveItem`, `OnHiddenItem`, `OnMoveToRuck` — drawing and putting away.
  - `OnH_A_Chield`, `OnH_B_Chield`, `OnH_A_Independent`, `OnH_B_Independent`,
    `on_a_hud_attach`, `on_b_hud_detach` — parentage and first-person attachment,
    including resuming a motion that was running when the view was taken away.
  - `HudItemData`, `GetHUDmode`, `HudSection`, `animation_slot` — is this item in the
    player's own hands right now, and where.
  - `TransformPosFromWorldToHud`, `TransformDirFromWorldToHud` — world space into the
    first-person view's own projection.
  - `RenderHud`, `EnableHudInertion`, `AllowHudInertion`, `HudInertionEnabled`,
    `HudInertionAllowed`, `GetInertionFactor`, `GetInertionPowerFactor` — the held
    model's sway.
  - `IsPending`, `SetPending`, `IsHidden`, `IsHiding`, `IsShowing` — the command gate
    and the three state predicates.
  - `UpdateXForm`, `on_renderable_Render` — the two operations every subclass must
    answer.
  - `UpdateHudAdditonal`, `render_hud_mode`, `render_item_3d_ui`,
    `render_item_3d_ui_query`, `need_renderable`, `CheckCompatibility`,
    `GetCurrentHudOffsetIdx`, `Action` — the remaining extension points.
  - `object`, `item` — the same instance seen as a physical object and as an inventory
    item, captured at construction.
  - `cast_hud_item` — the capability answer.

## Notes

The two frame stamps for the transform and the fire position are caching keys: each is
recomputed at most once per frame and the stamp records which frame. The mechanism is
incidental; the decision it encodes is that both are expensive enough to be worth caching
and are asked for several times a frame.
