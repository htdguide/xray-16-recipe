# src/xrGame/HUDManager.cpp

> The game's filling of the engine's heads-up-display hook: it owns the in-game screen set, the first-person weapon's two render passes, the look-at target and the damage-direction markers.

**Needs** — [`HUDManager.h`](HUDManager.h.md) · [`HUDTarget.h`](HUDTarget.h.md) · [`HitMarker.h`](HitMarker.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`player_hud.h`](player_hud.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`Car.h`](Car.h.md) · [`Spectator.h`](Spectator.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`xrEngine/CustomHUD.h`](../xrEngine/CustomHUD.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: dispatch and flag tests; the two render brackets are ordering decisions, not device work

## Purpose

The engine draws the world and then asks a single object to draw everything that is *not*
the world. This is that object for the game module. It holds four things and is otherwise
a router:

- the **in-game screen set** — the whole player-facing UI, built by the current game mode
  so that deathmatch and single player get different screens from the same hook;
- the **look-at target** (see [`HUDTarget.cpp`](HUDTarget.cpp.md));
- the **hit marker** ring showing which direction damage and nearby grenades came from;
- the two render brackets that let the first-person weapon be drawn with different
  visibility rules from the rest of the scene.

The interesting content is entirely in *when* each of those is allowed to draw. There are
four independent gates — a global draw flag, a connection state, a per-pass weapon flag
set, and a predicate about what the camera is attached to — and they do not nest.

## State

```text
RECORD HudManager
  game_ui     : optional<GameScreens>   # created by the current game mode, not by this file
  target      : LookAtTarget
  hit_marker  : HitMarker
  online      : bool                    # a session is connected; gates the UI and its frame tick
  render_lock : mutex                   # see Notes
```

Invariant: the screen set is created once on load and re-bound (not recreated) on a
subsequent load, so screens survive a level change within one session. It is torn down
only with the manager.

## `Render_First`

**Contract** — the pass drawn *before* the world: the first-person weapon's contribution
to shadowing. Runs only when some weapon-draw flag is set, a screen set exists, the
camera is attached to an actor, and that actor is in first-person view. Brackets the
draw by making the view entity's root **invisible for the oldest renderer generation
only**, then drawing it, then restoring visibility.

**Invariants** — on the oldest renderer the first-person model contributes *only* a
shadow and must not appear in the colour pass; on the newer ones it is drawn normally in
this pass too. That is the entire meaning of the invisibility toggle, and it is why the
renderer generation is queried here rather than being hidden behind the renderer
interface.

The invisibility flag is set on the **root**, not on the view entity, because an item
being held is a child of the actor and the whole chain must be suppressed together.

## `Render_Last`

**Contract** — the pass drawn *after* the world: the first-person weapon itself. Gated by
the same weapon flags and screen set, plus the shared predicate below. Brackets the draw
by marking the root as "this is the heads-up model", which selects the separate near-field
projection the weapon is drawn with, then asks the view entity to draw its heads-up
representation, then clears the mark.

**Invariants** — the mark must be cleared on every exit path, because it changes the
projection used by anything drawn afterwards.

## `need_render_hud`

**Contract** — the shared predicate for "should the first-person model be drawn at all".
False while the screenshot-capture mode has taken over the camera, false with no view
entity, false when the view entity is an actor that is not in first-person view or is
dead, and false when the camera is attached to a vehicle or to a spectator.

**Notes** — death and vehicles are excluded for the same reason: in both cases the camera
is no longer at the eyes of someone holding a weapon, so a first-person model would float.
Listing the two classes by name here is the incidental part; the decision is that the
model is drawn only when the camera *is* the holder's eyes.

## `RenderUI`

**Contract** — draw the two-dimensional layer: the hit marker ring, the screen set, the
queued font output, and the look-at cursor; then, if the game is paused and the pause
banner is enabled, a centred translated caption. Gated by the global draw flag and by
being online.

**Invariants** — the order is load-bearing: markers under the screens, screens under the
text the screens queued, and the cursor on top of all of it. The pause banner is drawn
last of all, over everything.

**Notes** — the banner's text comes from the string table by identifier, never as a
literal, and its font is one of the fixed named faces. A rebuild must keep the string
table indirection; the caption is localized.

## `RenderActiveItemUIQuery` / `RenderActiveItemUI`

**Contract** — a separate opt-in pass for user interface attached to the *held item* — a
detector's dial, a scope's display. The query asks whether anything wants it, so the
engine can skip binding a render target when nothing does; the draw runs it. Gated by the
global flag, the weapon flags and the shared predicate.

## `OnFrame`

**Contract** — tick the screen set and re-cast the look-at ray, once per frame, only when
drawing is enabled and a session is online.

**Notes** — the look-at ray is cast from the *frame* hook rather than the render hook, so
that everything reading the target — the depth-of-field effector especially — sees a
result computed before rendering begins rather than one frame stale.

## `Load`

**Contract** — ask the current game mode to build the screen set, or, if one already
exists, re-bind it to the new game mode. The second path is what makes a level change
within a session cheap.

## `OnConnected` / `OnDisconnected`

**Contract** — connecting marks the manager online and registers the screen set with the
engine's per-frame sequence at a low priority; disconnecting unregisters it and marks
offline. Idempotent in both directions.

**Invariants** — the screen set receives its own frame callback from the engine *in
addition to* being ticked here, and it is registered late in the frame order so that it
sees a world already advanced. The manager's own online flag is the gate for everything
else; nothing draws or ticks while offline, which is what keeps the in-game UI off the
main menu.

## `OnUIReset`

**Contract** — rebuild every screen in place after something invalidated them — a
resolution change, a language change, a reload of the UI data. Hides open dialogs,
unloads, loads and re-runs the connected notification so screens re-subscribe to the
session.

## `net_Relcase`

**Contract** — forward an impending object destruction to the hit marker, the look-at
target and the physics debug overlay, each of which may hold a reference to it.

## Accessors and pass-throughs

**Contract** — `GetCurrentRayQuery` hands out the look-at result; `SetCrosshairDisp`
pushes a weapon's dispersion into the reticle; `ShowCrosshair` switches reticle for
cursor; `HitMarked`, `AddGrenade_ForMark`, `Update_GrenadeView`, `SetHitmarkType` and
`SetGrenadeMarkType` forward to the hit marker; `GetGameUI` hands out the screen set;
`SetRenderable` flips the global draw flag.

**Invariants** — `SetCrosshairDisp` takes *two* dispersions, a dynamic one and a static
one, and chooses between them by a display setting. With the dynamic reticle off the
weapon's live cone is ignored and a fixed value is shown instead, so the player who turns
the setting off gets a still reticle rather than no information.

## `CurrentGameUI` / `CurrentDialogHolder`

**Contract** — two free functions the rest of the chapter reaches the UI through.
`CurrentGameUI` is the screen set. `CurrentDialogHolder` answers "who owns modal dialogs
right now" — the main menu when it is open, otherwise the in-game screens. Every dialog
in the game is pushed onto whichever of those two is current, which is what lets the same
dialog class work in the menu and in the world.

## Notes

The render brackets are guarded by a mutex, with the source's own comment saying it
believes the lock can be avoided. The problem it solves is real: both passes mutate
visibility flags on a shared object and restore them, and the renderer may issue passes
from more than one thread. A rebuild should make the flag a parameter of the draw call
rather than a mutation of shared state, which removes the need for the lock entirely.
