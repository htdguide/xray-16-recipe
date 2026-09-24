# src/xrGame/Spectator.h

> Declares the bodiless viewer entity implemented in [`Spectator.cpp`](Spectator.cpp.md), and names its five camera modes.

**Needs** — [`Entity.h`](Entity.h.md) · [`Actor_Flags.h`](Actor_Flags.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`xrEngine/IInputReceiver.h`](../xrEngine/IInputReceiver.h.md)
**Used by** — [`GamePersistent.cpp`](GamePersistent.cpp.md) · [`HUDManager.cpp`](HUDManager.cpp.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`ShootingObject.cpp`](ShootingObject.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md) · [`UIGameDM.cpp`](UIGameDM.cpp.md) · [`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_cl_mp.cpp`](game_cl_mp.cpp.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: a declaration and one enumeration

## Purpose

Declares `CSpectator`, a game object that is also an input receiver: an entity with no
body whose only content is a camera. Substance is in [`Spectator.cpp`](Spectator.cpp.md).

The one thing it defines outright is the set of camera modes, which is frozen because the
game modes' permission tables and the localization strings are both keyed by it:

```text
ENUM CameraMode
  free_fly        # fly anywhere, look anywhere
  first_eye       # through the watched player's own eyes
  look_at         # orbiting the watched player, level with his body
  free_look       # following the watched player at a free angle
  fixed_look_at   # pinned where the viewer died, looking where he was looking
```

Exported units:

- `CSpectator` — the viewer.
- `IR_OnMouseMove`, `IR_OnKeyboardPress`, `IR_OnKeyboardRelease`, `IR_OnKeyboardHold` —
  the controls.
- `UpdateCL`, `shedule_Update` — the per-frame and scheduled work.
- `net_Spawn`, `net_Destroy`, `net_Relcase` — the entity lifecycle, plus the notification
  that lets it drop a reference to a departing watch target.
- `On_SetEntity`, `On_LostEntity` — becoming and ceasing to be the view entity.
- `GetActiveCam`, `cam_Active`, `cam_FirstEye` — the active mode and two cameras.
- `GetSpectatorString` — the localized status line.
- `Center`, `Radius` — the spatial extent: a point.
- `cast_game_object`, `cast_input_receiver` — the two capability answers it gives.

## Notes

The class pulls in the senses' touch interface and does not implement it. A leftover; a
spectator senses nothing.

The status line is written into a caller-supplied fixed buffer of about a kilobyte. That
is the format's only bound and nothing checks it; a rebuild should return a string.
