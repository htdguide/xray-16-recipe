# src/xrGame/AnselManager.h

> Declares photo mode — a camera, an infinite-lifetime camera effector and the manager that brackets a capture session — implemented in [`AnselManager.cpp`](AnselManager.cpp.md).

**Needs** — [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [`xrEngine/Effector.h`](../xrEngine/Effector.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrEngine/pure.h`](../xrEngine/pure.h.md)
**Used by** — [`AnselManager.cpp`](AnselManager.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the optional photography feature. Substance in
[`AnselManager.cpp`](AnselManager.cpp.md).

Exported units:

- `AnselCamera` — a bare camera with no behaviour, used as the photo session's own.
- `AnselCameraEffector` — the camera effector, created with an *infinite* lifetime, that
  hands the frame's camera description to the vendor tool and writes the result back.
- `AnselManager` — a game object that also subscribes to the per-frame signal. It loads the
  vendor library, describes the game to it, and owns the session's own clock — needed
  because the game is fully paused during a session and the engine's frame delta reads zero.
- `IsActive` — the process-wide query the rest of the engine uses to suppress anything that
  must not appear in a photograph.

**Notes** — the manager derives from the game-object base purely so it can be a camera
owner and a frame-signal subscriber; it is never spawned, never has a server object and
never appears in the world. A rebuild should make it a plain service.
