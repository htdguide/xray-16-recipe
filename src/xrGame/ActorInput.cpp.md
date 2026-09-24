# src/xrGame/ActorInput.cpp

> Turns the player's actions into a wishful movement state, item commands, and the one gesture that does everything — *use*.

**Needs** — [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`actor_input_handler.h`](actor_input_handler.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`Car.h`](Car.h.md) · [`Torch.h`](Torch.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`Weapon.h`](Weapon.h.md) · [`HudItem.h`](HudItem.h.md) · [`player_hud.h`](player_hud.h.md) · [`InventoryBox.h`](InventoryBox.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: dispatch over an action set; no layout or timing concern

## Purpose

Nothing here reads a key. The engine's binding layer has already turned the physical input
into a named *action*
([`xrEngine/xr_level_controller.cpp`](../xrEngine/xr_level_controller.cpp.md)), and this
file is the actor's handler for that action set, in four phases — press, hold, release and
axis motion — across keyboard, mouse, gamepad and attitude sensors.

The load-bearing idea is that input never moves the actor. It sets a **wishful movement
state**, a bitset of what the player is asking for, which the movement code in
[`Actor_Movement.cpp`](Actor_Movement.cpp.md) reconciles against what is physically
possible. Nothing in this file may assume an input will take effect.

## State

```text
  wishful_movement_state : bitset   # what the player is ASKING for, not what is happening
  pickup_mode            : bool     # true while the use action is held, for item pickup
  drop_activated, drop_power : bool, real   # the charged throw
  external_input_handler : optional<handler> # a script or cutscene filtering input
  auto_clear_crouch      : bool     # global: crouch is hold-to-crouch, or a toggle
```

**Invariant** — strafing clears the sprint request. You cannot sprint sideways, and the
rule is enforced at the point of input rather than in the movement code, because the
movement code must not have to know which inputs conflict.

## The five gates every handler passes

Before any action is interpreted, five conditions can swallow it. The order is the same in
every handler and is itself the contract:

```text
  the heads-up tuning tool is open   -> swallow (a developer tool has the input)
  this actor is remote               -> swallow (a network peer's actor takes no local input)
  the actor is in a conversation     -> swallow (press and hold only; release still runs)
  an external handler refuses it     -> swallow (a cutscene or script filters the action set)
  the loading screen is up           -> swallow (press only)
```

**Notes** — that release is *not* gated on conversation while press is, is deliberate: a
key held when a conversation starts must still be released, or the actor keeps walking
into a wall for the length of the dialogue. This is the same problem the engine's input
stack solves with a synthetic release burst
([`xrEngine/IInputReceiver.cpp`](../xrEngine/IInputReceiver.cpp.md)); here it is solved by
letting release through.

## `IR_OnKeyboardPress`

**Contract** — one action begins. The action is offered to three consumers in a fixed
priority order and the first to take it wins:

```text
FUNCTION on_press(action)
  pass the five gates

  IF action is FIRE THEN
    IF leaning AND multiplayer THEN RETURN          # no shooting from a lean, online only
    IF a weapon is in either weapon slot THEN clear the sprint request
    IF authoritative THEN broadcast a "player fired" event     # for the statistics layer
  # note: this runs even when the actor is DEAD — see Notes

  IF NOT alive THEN RETURN

  IF in a holder AND the action is not USE THEN
    give it to the holder; then to the inventory if the holder allows weapons; RETURN
  ELSE IF the inventory consumes it THEN RETURN

  # what is left is the actor's own action set
  DISPATCH:
    jump                  -> request jump
    sprint toggle         -> flip the sprint request
    crouch / crouch toggle-> flip or begin crouch, depending on the toggle setting
    camera 1 / 2 / 3      -> switch view mode
    night vision, torch   -> switch the head-mounted light
    detector              -> toggle the hand-held detector
    use                   -> the use gesture (below)
    drop                  -> begin charging a throw
    next / previous slot  -> cycle the weapon slots
    quick slots 1..4,     -> consume an item by name or by class, and post a
    bandage, medkit          "used <item>" message on screen
```

**Invariants** — the inventory gets first refusal on every action *before* the actor's own
dispatch, which is why a weapon can bind the same action as the actor and win. A holder
gets first refusal ahead of the inventory, except for the use action, which the actor keeps
so the player can always get out.

**Notes**

- The fire action is handled *above* the alive check on purpose: the fired-event broadcast
  must happen even in the frame the actor dies, because the multiplayer statistics layer
  counts shots.
- Crouch has two mutually exclusive semantics selected by a setting, and both are
  implemented with one global flag that is *flipped* rather than set. That flag is process
  global rather than per-actor, which is harmless in single player and meaningless in
  multiplayer, where only one actor takes local input anyway.
- Consuming a quick-slot item resolves it by *name* from a four-entry table the interface
  owns; the bandage and medkit actions resolve by *class identifier* instead. Two
  mechanisms for the same job; a rebuild should keep only the first.
- The on-screen confirmation is shown for a fixed three seconds in the two older games and
  for the interface's default duration in the newest. That branch is the only behavioural
  difference between the games in this file.
- A flare-item action is present but commented out. It was a fourth quick-use path and a
  rebuild should omit it.

## `IR_OnKeyboardRelease` · `IR_OnKeyboardHold`

**Contract** — release ends a request: the jump request is cleared, the charged throw is
performed (only while the match is actually in progress), pickup mode ends, and crouch
returns to its default when hold-to-crouch is in force. Hold *maintains* one: the movement
direction bits, acceleration, leaning, and the free-look camera's own rotation.

**Invariants** — the direction bits are set every frame the key is held and cleared by the
movement code each frame after being consumed, so a dropped hold event is self-correcting
within one frame. This is why movement is expressed as a wish rather than as an event.

**Notes** — the camera's up and down actions are passed to the camera with their meanings
*swapped*, a sign convention the camera expects. Left and right are suppressed in
free-look mode, because free look rotates by mouse only.

## `IR_OnMouseMove` · `OnAxisMove`

**Contract** — relative pointer motion becomes camera rotation. The scale composes four
factors, and each one is a real decision:

```text
scale = (current field of view / the reference field of view)   # zoomed in turns slower
      * user sensitivity * user sensitivity scale / 50
      / look_factor                                             # the held item's inertia
```

Vertical motion is additionally scaled by three quarters, so that a given hand movement
turns less in pitch than in yaw. Inversion is applied per axis from settings.

**Invariants** — the field-of-view ratio is what makes a scoped weapon aim precisely with
the same mouse; without it, zooming would multiply the player's angular sensitivity by the
zoom factor. The look factor comes from the active item and is asserted non-zero, because
it is a divisor: a heavy weapon slows the turn.

**Notes** — the divisor of fifty is a units conversion between the settings' scale and
radians and has no derivation beyond feel.

## The gamepad handlers

**Contract** — the same four phases for a controller, with two actions that have no
keyboard equivalent and every other action *forwarded to the keyboard handler*. That
forwarding is the design: a gamepad button is a key.

- **look** — the right stick. Its scale composes the same factors as the mouse, but with
  an **adaptive intensity**: held at full deflection the turn rate ramps *up* by a step per
  frame to a configured maximum; at partial deflection it ramps down; released it ramps down
  five times as fast. This gives a gamepad both fine aim and fast turning from one stick.
  The intensity is a function-local static — process-global state — which is acceptable
  only because exactly one actor reads a gamepad.
- **move** — the left stick, quantized into the same direction bits the keyboard sets,
  with a dead zone of 0.3 in each axis, a sprint bit past 0.95 forward, and an
  acceleration bit when the stick's *magnitude* is below a half. That last rule is
  inverted from what one would guess and is what makes a half-pushed stick walk rather
  than run; the acceleration bit means "run" and is set when the stick is *not* pushed far.

**Notes** — the stick is sampled on press and on hold with identical code. A rebuild should
treat an analogue axis as a continuously-sampled value rather than as an event with phases.

## `IR_OnControllerAttitudeChange`

**Contract** — gyroscopic aiming: the controller's own orientation change nudges the view,
but only while aiming down the sights, unless a setting forces it always on. This is a
refinement of aim, not a replacement for the stick.

## `ActorUse`

**Contract** — the *use* gesture: one action that means six different things depending on
what the player is looking at. The order below is the priority, and it is the entire
contract.

```text
FUNCTION use()
  IF in a holder THEN send a "detach from holder" event; RETURN
     # routed as an EVENT, not a call: leaving a holder must not run inside input handling

  begin pickup mode (unless multi-item pickup is configured)
  IF the movement system is carrying a physics object THEN drop it

  IF looking at a usable object that is NOT an inventory item THEN use it
  IF looking at an inventory container that is script-openable THEN
    open the transfer screen, if it is not closed; RETURN
  IF the looked-at object is scriptable-only THEN stop here

  IF looking at a person THEN
    alive  -> try to start a conversation
    dead   -> open the transfer screen, but only once the body has been dead
              for three seconds and is not marked closed
  IF the modifier key is held AND the looked-at object's visual is in the
     grabbable list THEN grab it with the movement system
  ELSE IF the looked-at object is a holder THEN send an "attach to holder" event
```

**Invariants** — entering and leaving a holder are **events**, never direct calls, for the
same reason the engine defers level changes: the transition destroys and recreates the
actor's physics from inside the input handler that is still running. This is the chapter's
general rule and this is its most visible instance.

**Notes**

- The three-second delay before a corpse can be looted is not arbitrary: a creature's
  ragdoll is still settling, and opening the transfer screen against a moving body places
  dropped items unpredictably. The comment in the source calls the state "99.9% dead".
- The grabbable-object list is a configuration section keyed by *visual file name*, which
  is the only place in the game where behaviour is keyed by a model rather than by a class
  or a section.
- The modifier key is read as a **raw scancode**, bypassing the action binding. That is a
  violation of this chapter's own rule that nothing reads a key, and a rebuild should give
  it an action name.

## `use_Holder`

**Contract** — routes a holder transition to the right path by the holder's class: a car
goes through the vehicle path, a mounted gun or a generic holder through the general one.
On a successful entry the torch is switched off — you cannot hold a lamp and a turret. The
active item's animation is force-ended in both directions, because the item's own state
machine would otherwise be left mid-animation with no owner to advance it.

## `OnNextWeaponSlot` · `OnPrevWeaponSlot`

**Contract** — cycles the weapon slots in an authored order: knife, primary, secondary,
grenade, artefact. Finds the current slot's position in that order, scans forward (or
backward) for the first slot holding an item, and *synthesizes the corresponding key press*
rather than switching directly, so that the switch goes through exactly the same path a
direct slot key takes.

**Invariants** — the scan does not wrap. Reaching either end does nothing, which is why
scrolling repeatedly stops at the knife or at the artefact rather than cycling. An active
slot not in the table aborts the forward scan entirely but makes the backward scan start
from the end.

**Notes** — the artefact slot has its own action rather than a numbered one, so it is
special-cased. The numbered actions are derived by adding the table *position* to the first
weapon action, which silently requires the action enumeration and this table to stay in the
same order — a coupling a rebuild should replace with an explicit per-slot action.

## `GetLookFactor`

**Contract** — the divisor applied to every camera rotation. An external input handler
overrides it outright; otherwise it is the active item's control-inertia factor. This is
how a heavy weapon feels heavy.

## `set_input_external_handler`

**Contract** — installs or clears a filter over the action set. Installing one clears the
wishful movement state and synthesizes a fire release, so that a cutscene starting while
the player is running and shooting does not leave the actor running and shooting.

## `SwitchNightVision` · `SwitchTorch`

**Contract** — both walk the actor's attached items for the head-mounted lamp and toggle
it. Night vision additionally refuses while either weapon slot holds a zoomed weapon:
switching goggles on and off while looking through a scope would fight over the same
post-process effect.

## `HUDview`

**Contract** — whether the first-person weapon model should be drawn: the window has focus,
the first-person camera is active, and either the actor is not in a holder or the holder
permits both weapons and its own first-person view.

## `NoClipFly` (development builds only)

**Contract** — free-flight debugging. Movement actions translate the actor directly along
the camera's yaw, at a tenth of a metre per event, scaled by a quarter or by four with
modifier keys. The position is written into the movement system as well as into the object,
because the two are otherwise reconciled by physics that no longer runs. Compiled out of a
shipping build.
