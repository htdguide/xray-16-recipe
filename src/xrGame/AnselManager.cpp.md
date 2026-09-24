# src/xrGame/AnselManager.cpp

> Photo mode: hand the vehicle-maker's screenshot tool control of the camera's orientation while the world stands still, and take it back cleanly.

**Needs** — [`AnselManager.h`](AnselManager.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrEngine/CameraDefs.h`](../xrEngine/CameraDefs.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it loads a vendor library by name at run time and exchanges structures with it

## Purpose

An optional, platform- and vendor-specific feature: a graphics vendor's in-game photography
tool, which wants to own the camera for the duration of a capture while the game holds
still. Everything interesting here is about *handing over and taking back* — the tool is a
seam the engine reaches through a dynamically loaded library, and if the library is absent
the whole feature is simply not there.

A rebuild may omit this file entirely. What is worth carrying across is the shape: a
**camera effector of infinite lifetime** that intercepts the frame's camera description,
lets an outside party rewrite it, and writes the result back — plus the discipline of
pausing everything the tool must not see.

## State

```text
RECORD PhotoMode
  module      : optional<dynamic library>
  camera      : camera                # seeded from whatever camera was active
  effector    : camera effector       # infinite lifetime; removed explicitly
  timer       : clock                 # a clock that keeps running while the game is paused
  time_delta  : real                  # the fake frame delta fed to the camera, smoothed
```

Invariants: the effector and the per-frame registration are added together when a session
starts and removed together when it stops. The stored "suppress error text" flag is
captured on entry and restored on exit — a capture must not have a red diagnostic line
across it.

## `Load`, `Unload`, `IsActive`

**Contract** — load the vendor library by name, choosing the 32- or 64-bit variant to match
the process. Returns whether it loaded. `IsActive` reports whether a session is in
progress, and is read by the rest of the engine to suppress things that must not appear in
a photograph.

## `Init`

**Contract** — describe the game to the tool and install four callbacks. Returns whether the
tool accepted the configuration. Every failure is logged and non-fatal.

The configuration is a coordinate-system declaration plus a handful of limits:

```text
RECORD ToolConfiguration
  handedness         : right = +x, up = +y, forward = +z
  field of view      : measured vertically
  camera translation : NOT supported            # see Notes
  rotation speed     : 220 degrees per second
  translation speed  : 50 world units per second (unused while translation is off)
  capture latency    : zero, and zero settle    # the engine has no deferred frames to flush
  window handle      : the application window
```

**Notes** — **camera translation is deliberately disabled**, and the reason is written into
the file: a free-flying camera in a multiplayer-capable game is a cheat. Enabling it would
require collision against the level's geometry and a leash around the player's position.
Both are stated as prerequisites, neither is implemented. A rebuild that wants a flying
photo camera owes those two.

Zero capture latency is a statement about the renderer: there is no pipelined frame queue
to drain before the image is stable.

### The session-start callback

**Contract** — decide whether a session may begin, and if so put the game into the state a
photograph needs.

```text
FUNCTION on_session_start(session) -> allowed or disallowed
  IF no level is loaded THEN DISALLOW
  IF the main menu is open THEN close it
  allow up to 140 degrees of field of view
  suppress the paused-overlay text
  mark photo mode active and pause the game, in all three senses
  suppress error text, remembering the previous setting

  add the photo-mode camera effector to the level's camera manager
  register for the per-frame signal at the capture priority

  source = the actor's own camera, or the camera of whatever holds the actor,
           or in a non-single-player session the spectator's camera
  IF no source camera can be found THEN log and DISALLOW
  seed the photo camera's position, direction and up from the source
  copy its field of view and aspect
  ALLOW
```

**Invariants** — the seed is what makes photo mode start from exactly the view the player
had, rather than snapping. The vehicle case is explicit: an actor inside a holder is
looking through the holder's camera, not their own.

The pause is taken in all three of the engine's senses at once, so that the simulation
clocks, the scripted global clock and the sound all stop together. The non-pausing clock
keeps running, which is exactly why this file carries its **own** timer: the camera still
has to be advanced each frame, and the frame delta the rest of the engine reports is zero.

### `OnFrame`

**Contract** — advance the photo camera once per frame while a session runs, against a
locally measured delta rather than the engine's.

```text
FUNCTION on_frame()
  measure the real time since the last call; restart the local clock
  time_delta = 0.3 * time_delta + 0.7 * measured        # smoothed, favouring the new sample
  clamp it to (epsilon, 0.1 seconds)

  temporarily write time_delta into the device's frame delta
  drive the camera manager from the photo camera
  write zero back
```

**Notes** — the write-and-restore of the engine's global frame delta is a hack, and the
source says so: the camera manager derives its field-of-view interpolation from that value,
and there is no other way to feed it a delta while the game is paused. A rebuild should
pass the delta as an argument to the camera update; the *requirement* is that the camera
keeps moving smoothly while everything else is frozen.

The smoothing weights here are the reverse of the engine's own frame-delta smoothing (which
favours the old sample nine to one). A paused camera is not trying to hide a hitch; it is
trying to follow the user's input promptly.

## `AnselCameraEffector::ProcessCam`

**Contract** — the interception point. Runs as the highest-precedence camera effector every
frame, hands the frame's camera description to the tool, and writes back whatever the tool
returns. Always reports that the camera description should be applied.

```text
FUNCTION process(info)
  info.dont_apply = false
  build the tool's camera from info's basis vectors, converted to a quaternion
  copy across field of view, aspect, near and far planes, and the projection offsets
  ASK the tool to update that camera            # the user's input reaches us only here
  convert the returned rotation back to three basis vectors
  copy every field back into info
  RETURN applied
```

**Invariants** — the effector's lifetime is *infinite*, so it is never retired by the
effector stack's own expiry; it is removed explicitly when the session ends. That is the
correct choice for an effector whose end condition is external.

The position is deliberately neither read nor written — that is the translation ban, applied
at the one place it would otherwise leak.

**Notes** — the three basis vectors are held in storage that persists between calls but is
fully overwritten on entry each time, so the persistence buys nothing and costs
thread-safety. A rebuild makes them local.

## `AnselCamera`

**Contract** — a plain camera with no behaviour of its own, existing only to be a target the
photo session can seed and the frame loop can drive.
