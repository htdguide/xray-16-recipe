# src/xrGame/MainMenu.cpp

> The main menu: a dialogue holder that takes over input and rendering, suspends the running level while it is up, and restores everything it changed on the way out.

**Needs** — [`MainMenu.h`](MainMenu.h.md) · [`UIDialogHolder.h`](UIDialogHolder.h.md) · [`ui/UIDialogWnd.h`](ui/UIDialogWnd.h.md) · [`ui/UIMessageBoxEx.h`](ui/UIMessageBoxEx.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`ui/UICDkey.h`](ui/UICDkey.h.md) · [`DemoInfo.h`](DemoInfo.h.md) · [`DemoInfo_Loader.h`](DemoInfo_Loader.h.md) · [`ai_space.h`](ai_space.h.md) · [`game_type.h`](game_type.h.md) · [`account_manager.h`](account_manager.h.md) · [`login_manager.h`](login_manager.h.md) · [`profile_store.h`](profile_store.h.md) · [`CdkeyDecode/cdkeydecode.h`](CdkeyDecode/cdkeydecode.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrEngine/IGame_Level.h`](../xrEngine/IGame_Level.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrEngine/xr_input.h`](../xrEngine/xr_input.h.md) · [`xrUICore/Cursor/UICursor.h`](../xrUICore/Cursor/UICursor.h.md) · [`xrUICore/Buttons/UIBtnHint.h`](../xrUICore/Buttons/UIBtnHint.h.md) · [`xrCore/os_clipboard.h`](../xrCore/os_clipboard.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`MainMenu.h`](MainMenu.h.md)
**Tier floor** — T2: a modal overlay whose substance is ordering and state restoration; the only tier pressure is the device-reset and screenshot timing

## Purpose

The main menu is not a screen the engine switches to; it is an overlay that suspends what is
underneath. Bringing it up means taking input capture, pausing the device, removing the
running level from the frame and render lists, and inserting the menu's own rendering into
them. Taking it down means putting every one of those back exactly as it was — including
things the user might have changed while the menu was up.

That save-and-restore is the file's substance. Almost every field on the object is either a
flag recording something to put back, or a piece of deferral machinery that exists because
an action cannot safely happen at the moment it is requested.

## State

```text
RECORD MainMenuState
  start_dialog        : dialog          # the root menu window, created from a class identifier
  active              : bool            # the menu owns input and rendering
  restore_console     : bool            # the console was visible before we hid it
  restore_pause       : bool            # the device was ALREADY paused before we paused it
  restore_pause_str   : bool            # the "paused" banner was enabled
  restore_cursor      : bool            # the cursor was visible
  need_change_capture : bool            # take or release input capture on the next frame
  game_save_screenshot: bool            # a save-game thumbnail is pending
  need_vid_restart    : bool            # the graphics device must be reset on the way out
  need_ui_restart     : bool            # the whole menu must be rebuilt, but not right now
  screenshot_name     : text
  screenshot_frame    : int             # the frame the thumbnail must be taken on
  deactivated_frame   : int             # when the menu last closed; guards teardown
  pending_error       : ErrorKind       # one deferred message box
  error_start_time    : int             # rate-limits the session-terminate box
  pp_draw_windows     : list<window>    # windows drawn INTO the post-process pass
  demo_info_loader    : loader          # lazily created
  account / login / profile / matchmaking clients
```

**Invariants**

- `restore_pause` records whether the device was *already* paused. Restoration unpauses only
  if it was not: a menu opened over an already-paused game must leave it paused.
- The menu is registered on the frame list at the lowest priority, so it runs after
  everything it might suspend; it is registered on the render list at a fixed slot behind
  the console, the cursor and the tutorial overlay, which is the drawing order those four
  must have.
- `screenshot_frame` is compared against the current frame with a one-frame tolerance in both
  directions, and activation is refused within that window. A thumbnail must capture the
  *game*, so the menu must not be drawn in the frames around it.

## `Activate`

**Contract** — opens or closes the menu. Idempotent against the current state, and refuses
entirely while a save-game thumbnail is pending or within a frame of one. Never activates on
a dedicated server. Rebuilds the root dialogue from scratch on every activation.

```text
FUNCTION Activate(on)
  IF already in that state THEN RETURN
  IF a save thumbnail is pending THEN RETURN
  IF the current frame is within one of screenshot_frame THEN RETURN
  IF dedicated server AND on THEN RETURN

  IF on THEN
    pause the device (input only; the world keeps its own pause state)
    active = true; need_change_capture = true
    restore_cursor = cursor is visible
    IF NOT ReloadUI() THEN RETURN            # no menu class: abort the activation
    restore_console = console is visible
    IF single player THEN restore_pause = device is already paused
    hide the console
    IF single player THEN
      restore_pause_str = the paused banner is on; turn it off
      IF the game was NOT already paused THEN pause the world too
    END IF
    IF a level is loaded THEN
      IF single player THEN remove the level from the FRAME list   # stop simulating
      remove the level from the RENDER list                        # stop drawing
      reset the post-process camera state
    END IF
    add this menu to the render list behind console, cursor and tutorial
  ELSE
    deactivated_frame = current frame
    active = false; need_change_capture = true
    remove this menu from the render list
    # Releasing input capture steals console focus, so hide and re-show it around the call.
    hide the console if visible; release input capture; show it again if it was visible
    hide the root dialogue; tear down the holder's internals
    IF a level is loaded THEN
      IF single player THEN add the level back to the FRAME list
      add the level back to the RENDER list
    END IF
    IF restore_console THEN show the console
    IF single player THEN
      IF the game was not already paused THEN unpause the world
      restore the paused banner
    END IF
    IF restore_cursor THEN show the cursor
    unpause the device
    IF need_vid_restart THEN
      clear the flag
      reset the graphics device            # shipping build
      # non-shipping builds only precache instead, because a full reset here is
      # a second reset in one transition and its necessity was never established
    END IF
  END IF
```

**Notes** — multiplayer is excluded from every pause and frame-list change. The level must keep
running: the server is elsewhere and stopping the client's simulation while the menu is open
would desynchronize it. Single player alone gets the true suspension, and that asymmetry is
the single most important thing on this page.

The console hide/show dance around releasing input capture is not cosmetic. Capture release
reassigns the foreground input receiver, and the console re-registers itself when shown, so
showing it after the release is what puts it back on top.

## `ReloadUI`

**Contract** — destroys the existing root dialogue and builds a new one from the registered
menu class identifier. Returns false if the class is not registered, in which case it also
clears the activation flags so the caller's activation unwinds. The new dialogue is marked as
working while paused, which is what lets it update at all with the device paused.

**Notes** — the menu is rebuilt on every activation rather than kept. That is deliberate: the
menu's layout comes from data and its content from the script layer, both of which can change
while a game is running.

## `DestroyInternal`

**Contract** — releases the root dialogue, but only once the menu has been closed for at least
a few frames, unless forced. The delay exists because the dialogue can still be referenced by
the frame that closed it.

**Notes** — the frame comparison in the original is written in a way that makes the
non-forced branch always true, so the guard does not actually delay anything. A rebuild
should implement the intent — "not while a frame that saw it is still in flight" — rather than
copy the expression.

## `OnFrame`

**Contract** — the menu's per-frame work: applies a pending input-capture change, runs the
dialogue holder's frame, completes a pending save-game thumbnail, pumps the matchmaking
client, and raises any deferred error dialogue or menu rebuild. Runs every frame whether or
not the menu is active, because the pending work outlives deactivation.

```text
FUNCTION OnFrame()
  IF need_change_capture THEN
    clear it; take input capture IF active ELSE release it
  END IF
  run the dialogue holder's frame

  IF a save thumbnail is pending AND the current frame is past screenshot_frame THEN
    clear the flag
    take the thumbnail
    # The level was temporarily put back on the lists so it could be drawn for the shot;
    # take it off again if the menu is still up.
    IF a level is loaded AND active THEN remove it from the frame and render lists
    IF restore_console THEN show the console
  END IF

  IF active OR a download is in progress THEN
    pump the matchmaking client and turn its status into an error dialogue
    hide the "connecting to master server" box unless that is the current status
  END IF

  IF active THEN
    raise any pending error dialogue
    IF need_ui_restart THEN clear it; ReloadUI(); tell the new root it was reloaded
  END IF
```

## `Screenshot`

**Contract** — takes a screenshot. An ordinary screenshot goes straight to the renderer. A
save-game thumbnail is *deferred by one frame*: the level is put back onto the frame and
render lists, the console is hidden, and the capture happens next frame from `OnFrame`.

**Invariants** — the deferral is required. A save-game thumbnail must show the world, not the
menu, and the world is not on the render list while the menu is up. Putting it back, letting
one frame render, capturing, and taking it off again is the whole mechanism — and it is why
`Activate` refuses to run in the frames around `screenshot_frame`.

## `OnRender` · `OnRenderPPUI_query` · `OnRenderPPUI_main` · `OnRenderPPUI_PP`

**Contract** — the menu draws in one of two places. Normally it draws directly, after the
renderer's menu background pass. When the post-process menu path is enabled, the dialogues
are drawn *inside* the post-processing pass instead, so that the menu is affected by the
same blur and colour grading as the scene behind it. `OnRenderPPUI_query` answers which of
the two is in use; the other three each draw nothing when the thumbnail is pending.

`OnRenderPPUI_PP` is a separate, always-post-processed layer for windows explicitly
registered with `RegisterPPDraw` — used by the parts of the menu that must be blurred even
when the rest is not.

**Notes** — `CanSkipSceneRendering` tells the device it may skip drawing the world entirely
while the menu is up, which is what makes the menu cheap. It returns false while a thumbnail
is pending, for the same reason as everything else in that group.

## Input handling

**Contract** — every input entry point returns immediately unless the menu is active, then
forwards to the dialogue holder. Three actions are consumed before the holder sees them:
screenshot, show console, and enter the editor. Mouse buttons and gamepad buttons are
forwarded into the keyboard path, joining the same binding space; gamepad axes go to the
holder with their analogue state.

**Notes** — one deliberate escape hatch: holding both a platform modifier and the alt key
releases the window's input grab and makes the window draggable, restoring it on the next key
release. This exists so a player whose mouse is captured by a fullscreen menu can get it back
without quitting.

The menu declares that it ignores pause, which is what lets it run at all with the device
paused.

## `SetErrorDialog` · `CheckForErrorDlg`

**Contract** — an error is *requested* by name and shown at a safe point in the frame, not at
the moment it is detected. One pending error at a time; a second request overwrites the first.

**Invariants** — the error kinds and the message-box template names form a **positional
table**: the enumerator's numeric value indexes the array of template names. Adding a kind
without adding its template at the same index silently shows the wrong dialogue. A rebuild
should pair them in one table rather than in two parallel ones.

```text
ENUM ErrorKind      # each maps positionally to one message-box template name
  InvalidPassword · InvalidHost · SessionFull · ServerReject
  CDKeyInUse · CDKeyDisabled · CDKeyInvalid · DifferentVersion
  MatchmakingServiceFailed · MasterServerConnectFailed
  NoNewPatch · ConnectingToMasterServer · SessionTerminated
  LoadingError · DownloadMultiplayerMap
```

Templates that fail to load are dropped from the list, which shifts every later index — a
real hazard the original accepts and a rebuild should not.

## `OnSessionTerminate`

**Contract** — shows the "kicked by server" box with the server's reason. Rate-limited: a
second termination within eight seconds of the first is swallowed, because a disconnect
typically produces several notifications.

**Notes** — a reason beginning with a marker character is treated as a *localization key* and
translated whole; anything else is appended to a generic prefix. That convention lets a
server send either a translatable code or free text over the same field.

## `OnLoadError` · `OnPatchCheck` · `CancelDownload`

**Contract** — compose and raise the corresponding error box. The patch check shows "no new
patch" on failure and does nothing on success when a download is already running.

## `Show_CTMS_Dialog` · `Hide_CTMS_Dialog`

**Contract** — show and hide the "connecting to master server" box, each a no-op if the box is
already in the requested state or was never created.

## `OnDeviceReset` · `SetNeedVidRestart` · `OnUIReset`

**Contract** — three deferrals. A device reset while the menu is up over a live level arms a
video restart to be performed on deactivation, not now. A user-interface reset arms a full
menu rebuild for the next frame.

**Invariants** — the user-interface reset *must* be deferred: it can arrive while a script is
executing, and destroying the menu out from under a running script destroys the objects the
script is calling through. This is the clearest instance of a rule that recurs across the
whole game layer — a script callback may request a teardown but may never perform one.

## `RegisterPPDraw` · `UnregisterPPDraw`

**Contract** — add or remove a window from the always-post-processed draw list. Registration
removes first, so a window cannot appear twice.

## `SwitchToMultiplayerMenu`

**Contract** — dispatches a fixed command pair into the root dialogue that selects the
multiplayer branch. The two numbers are an opaque contract with the menu's own script; they
are not otherwise meaningful.

## `AddHyphens` · `DelHyphens`

**Contract** — insert and remove the grouping hyphens in a product key, converting between the
compact stored form and the four-character-group display form. Both write into a shared static
buffer, so the result is only valid until the next call.

```text
FUNCTION AddHyphens(key) -> text
  # A key is four groups of four characters: "XXXX-XXXX-XXXX-XXXX".
  place a hyphen after every fourth output position
  copy each input character to position i + floor(i / 4)
  terminate

FUNCTION DelHyphens(display) -> text
  groups = min(floor(length / 4), 3)      # at most three hyphens
  copy from position i + floor(i / 4) back to position i
  terminate at length - groups
```

**Notes** — the shape (four groups of four, three hyphens) is frozen by the printed keys the
retail games shipped with. The shared static buffer is a genuine hazard: two calls in one
expression return the same pointer.

## `IsCDKeyIsValid` · `ValidateCDKey`

**Contract** — checks the installed product key against the four game identifiers the engine
knows, returning true if any accepts it. An absent key counts as valid — the retail games are
playable without one. On platforms with no key store, always true. `ValidateCDKey` wraps it
and raises the invalid-key dialogue on failure.

**Notes** — this is part of the dead matchmaking seam and gates nothing but the master-server
browser. A rebuild should make it return true unconditionally.

## `GetPlayerName` · `GetCDKeyFromRegistry` · `GetGSVer`

**Contract** — the player's display name (from the logged-in matchmaking profile if there is
one, otherwise from the platform key store), the stored product key, and the version string
the master server is told. Each caches into a shared string on the object and returns it.

## `GetDemoInfo`

**Contract** — reads the summary block out of a demo file, creating the loader on first use.
Returns the cached summary; see [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) for the
fixed-size block this reads.

## `FillDebugTree`

**Contract** — contributes the menu and its dialogue tree to the debug overlay's object
inspector. Compiled out of the shipping build.

## Construction and destruction

**Contract** — the constructor subscribes to the script-engine reset event, creates the button
hint overlays, the matchmaking client and its account, login and profile managers, builds one
message box per error template (dropping any that fail to load), wires the map-download box's
two buttons, and registers the menu on the frame list at the lowest priority. The destructor
unwinds all of it and unsubscribes.

**Invariants** — the script-reset subscription's handler branches on whether the menu is
active: an *active* menu is reloaded in place, an inactive one is destroyed outright. Both
paths exist because a script reset invalidates every script-backed menu element, and a menu
that is on screen must be replaced rather than left holding stale handles.

Nothing user-interface-related is created on a dedicated server, which has no device.
