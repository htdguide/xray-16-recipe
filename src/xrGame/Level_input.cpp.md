# src/xrGame/Level_input.cpp

> The level's input dispatch: the fixed chain every key, mouse movement, gamepad event and text character walks through on its way from the window to the controlled entity, and the handful of actions the level consumes itself.

**Needs** — [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`HudItem.h`](HudItem.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`ui/UIDialogWnd.h`](ui/UIDialogWnd.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`saved_game_wrapper.h`](saved_game_wrapper.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrEngine/xr_input.h`](../xrEngine/xr_input.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a dispatch chain over already-decoded events; no layout or budget constraints

## Purpose

Input arrives as raw device events. Several consumers want each one and they must be
offered it in a fixed order, because whoever takes an event decides what the game does
with the rest of the frame. This file is that order, written once for every event kind.
It is a separate file from the rest of the level because the order is the whole content:
the level class itself has nothing to do with input beyond owning the receiver role.

The engine also asks this layer for a *binding*: keys are never handled by identity but by
the abstract action they are bound to, so that key bindings are user data and the game
logic never names a key. The exceptions are the debug and developer paths at the end of
the press handler, which do name scancodes and which are the reason the shipping build
compiles them out.

## State

`Stateless.` The one piece of module-level state is a global flag that suppresses all
input — set by the code that plays cutscenes and demos — and it is a gate, not data.

```text
RECORD InputGates                  # module-level switches this file obeys
  all_input_disabled : bool        # hard mute: nothing but nothing is dispatched
  pause_blocked      : bool        # the pause action is refused (scripted sequences)
```

## The dispatch chain

Every event handler in this file is the same chain with a different payload. Stating it
once is the point of the file; a rebuild that scatters these checks across handlers will
get one of them wrong and produce a bug that only shows up while paused or in a dialogue.

```text
FUNCTION dispatch(event)
  # 1. Precache gate. During a level's warm-up frames there is no world to steer.
  IF device is precaching THEN RETURN                       # press handler only

  # 2. Hard mute. Set while a cutscene owns the screen.
  IF all_input_disabled THEN RETURN

  # 3. Script observation. The actor's script callbacks see the RAW event, before any
  #    consumer, and cannot veto it. Scripts observe input; they do not intercept it.
  IF actor exists THEN fire actor script callback for this event kind

  # 4. The user interface gets first refusal and CAN veto. A dialogue, the inventory
  #    screen or the console-driven menu returning "handled" ends dispatch here.
  IF game ui handles(event) THEN RETURN

  # 5. Pause gate. A paused device stops the world, so gameplay input is dropped —
  #    but the UI above has already had its turn, which is why menus work while paused.
  #    Demo playback is exempt, because a demo advances while the device is paused.
  #    A developer no-clip flag also lifts the gate in non-shipping builds.
  IF device paused AND NOT demo playing THEN RETURN

  # 6. The game mode (multiplayer rules, spectator control) may claim the action.
  IF game rules handle(action) THEN RETURN                  # press/release only

  # 7. The script layer's own level-input hook may claim it, by action AND raw key.
  IF script hook "level_input.on_key_press" returns true THEN RETURN   # press only

  # 8. Console key bindings: a key may be bound directly to a console command.
  IF console bindings execute(key) THEN RETURN              # press only

  # 9. Whatever is left reaches the controlled entity as an ACTION, never as a key.
  target = controlled_entity()
  IF target implements the input-receiver role THEN target.receive(action, payload)
```

**Invariants**

- Steps 3 and 4 are not interchangeable. The script callback fires even for events the
  user interface will swallow, because scripts that watch for a key while a dialogue is
  open depend on it.
- Step 9 passes the *bound action*, not the key — except for mouse motion, wheel and text
  input, which have no binding.
- The pause gate sits *after* the user interface and *before* everything else. That single
  placement is what makes "paused" mean "the world stops but the menus work".

### `controlled_entity`

```text
FUNCTION controlled_entity() -> object or none
  IF no game rules are active THEN RETURN none
  IF single player THEN RETURN the current entity          # normally the actor
  ELSE RETURN the current CONTROL entity                   # spectating splits these
```

In multiplayer the entity the camera follows and the entity the player steers are
different things while spectating; input follows the control entity and the camera follows
the view entity. Single player collapses both onto one.

## `IR_OnKeyboardPress`

**Contract** — the full chain for a key-down, plus the actions the level consumes itself.
Does not block. May execute console commands, take a screenshot, pause the device, enter
the editor, or trigger a save or load, any of which can take arbitrarily long.

**Invariants** — the actions handled before the mute gate (pause, editor) work even when
all input is disabled, deliberately: a scripted sequence that mutes input must still be
escapable.

```text
FUNCTION IR_OnKeyboardPress(key)
  IF device is precaching THEN RETURN
  action = binding_for(key)
  IF NOT all_input_disabled AND actor exists THEN fire actor key-press callback(key)

  # --- consumed before the mute gate ---
  IF action == PAUSE THEN
    IF editor is open THEN RETURN
    IF NOT pause_blocked AND (single player OR demo playing) THEN toggle device pause
    RETURN                    # multiplayer cannot pause: the world is elsewhere
  END IF
  IF action == EDITOR THEN advance the editor to its next state; RETURN

  IF all_input_disabled THEN RETURN

  # --- consumed by the level, ahead of the UI ---
  IF action == SCREENSHOT THEN ask the renderer for a screenshot; RETURN
  IF action == CONSOLE    THEN show the console; RETURN
  IF action == QUIT THEN
    IF a modal dialogue is open and the device is not paused THEN
      offer it the key first (multiplayer and the main menu need the special case),
      otherwise close that dialogue
    ELSE open the main menu through the console
    RETURN
  END IF
  IF action == ALIFE_COMMAND THEN call the script entry point that starts a simulated
     faction attack; RETURN   # a hook for the alife-level strategic layer

  IF level not ready OR no game ui THEN RETURN
  ... steps 4 through 9 of the chain ...

  # --- single-player quick save/load, after the script hook so scripts can veto ---
  IF action == QUICK_SAVE AND single player THEN run the console save; RETURN
  IF action == QUICK_LOAD AND single player THEN
    name = user_name + " - quicksave"
    IF that saved game is not valid THEN RETURN      # refuse rather than fail mid-load
    run the console load of it
    RETURN
  END IF
```

**Notes** — the developer block that follows (compiled out of the shipping build) is not
part of the contract but two of its behaviours are worth recording because they describe
invariants the rest of the engine relies on:

- *Reload configuration and scripts* marks the configuration and script search paths as
  needing a rescan, then sends a reload-game message rather than reloading in place. The
  level is rebuilt from its spawn data; nothing is patched live.
- *Take control of the next living entity* walks the object registry to the next entity
  that is alive, switches the view to it, and **re-registers both the old and the new
  object with the scheduler**. That re-registration is the load-bearing part: the
  scheduler's update rate depends on distance from the controlled entity, so changing
  which object that is invalidates both objects' scheduling. It also moves the first-person
  item display from one inventory to the other and re-enters the active item's current
  state so the held weapon's model is rebuilt for the new owner.

The quick-load path validating the save *before* issuing the load, rather than letting the
load fail, exists because a failed load leaves the level half-unloaded.

## `IR_OnKeyboardRelease`

**Contract** — the chain for a key-up, minus the level-consumed actions and minus the
console bindings (a binding fires on press only). Requires the level to be ready.

**Notes** — the pause gate applies to releases too, which creates a real hazard: a key held
down when the game pauses never delivers its release, so the controlled entity can be left
holding a movement action. `IR_OnActivate` is the compensation.

## `IR_OnKeyboardHold`

**Contract** — the chain for a key still held, dispatched every frame the key is down. Same
order, same gates. Demo playback is exempt from the pause gate.

## `IR_OnTextInput`

**Contract** — a composed Unicode character, offered to the user interface and then to the
controlled entity. No binding lookup: text is text. Requires the level ready and input not
muted.

**Notes** — separate from key events because the windowing seam delivers composed
characters separately from scancodes, and must: a key binding is by scancode so that it
survives a keyboard-layout change, while typed text must respect the layout.

## `IR_OnMouseMove`

**Contract** — relative motion in device units. Fires the actor's mouse-move script
callback, offers the delta to the user interface, then delivers it to the controlled
entity, which is what turns the view. Exempt from the pause gate during demo playback so a
recorded demo's free camera still works.

## `IR_OnMouseWheel`

**Contract** — wheel deltas on both axes. Same chain. Note the script callback receives
the two axes in the opposite order to the handler's own parameters — vertical first —
which is a frozen part of the script surface, not a mistake to tidy.

## `IR_OnMousePress` · `IR_OnMouseRelease` · `IR_OnMouseHold`

**Contract** — pure delegations to the keyboard handlers. Mouse buttons occupy the same
identifier space as keys and are bound the same way, so a binding can name a mouse button
wherever it can name a key. A rebuild should keep one identifier space for all digital
inputs for exactly this reason.

## `IR_OnControllerPress` · `IR_OnControllerRelease` · `IR_OnControllerHold`

**Contract** — gamepad events. A gamepad *button* identifier is forwarded straight into the
keyboard handler, joining the same binding space. Anything that is not a button — an axis
crossing a threshold, reported with its analogue state — runs the chain in its own right,
carrying the axis state through to the controlled entity so that analogue movement is not
flattened to on/off.

## `IR_OnControllerAttitudeChange`

**Contract** — the controller's orientation sensor, delivered as a three-axis change. Has
no user-interface stage: it goes from the script callback straight past the pause gate to
the controlled entity. Nothing in the shipped user interface consumes motion control.

## `IR_OnActivate`

**Contract** — called when the window regains focus. Marks the game's user interface as
foremost, then **synthesizes a press for every movement-class key that is currently
physically held**.

**Invariants** — after this runs, the controlled entity's held-action set matches the
physical key state.

```text
FUNCTION IR_OnActivate()
  mark game ui foremost
  IF no input device THEN RETURN
  FOR EACH key IN all keyboard keys
    IF key is physically down THEN
      action = binding_for(key)
      IF action IS ONE OF { forward, back, strafe left, strafe right, turn left,
                            turn right, up, down, crouch, accelerate,
                            lean left, lean right, fire } THEN
        IR_OnKeyboardPress(key)
      END IF
    END IF
  END FOR
```

**Notes** — the whitelist is not arbitrary and is the file's sharpest decision. Only
*continuous* actions are re-synthesized: these are the ones whose effect is "while held",
so a missed press leaves the entity stuck in the wrong state. A discrete action —
reloading, using, switching a weapon — must *not* be re-fired, because replaying it on
focus gain would perform it a second time. A rebuild that re-presses everything will
reload the player's weapon every time they alt-tab back.

## `IR_OnDeactivate`

**Contract** — called when the window loses focus. Clears the foremost mark on the game's
user interface. Nothing else: the held keys are deliberately left as they are, and
`IR_OnActivate` reconciles on the way back.
