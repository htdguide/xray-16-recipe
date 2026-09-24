# src/xrGame/car_memory.h

> Declares the vehicle's vision client implemented in [`car_memory.cpp`](car_memory.cpp.md).

**Needs** — [`vision_client.h`](vision_client.h.md) · [`Car.h`](Car.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`car_memory.cpp`](car_memory.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`car_memory.cpp`](car_memory.cpp.md): a vision client
whose camera is set from outside rather than derived from a skeleton.

Exported units:

- **construction** — binds to a vehicle and fixes the re-evaluation period.
- **`reload`** — read the frustum from the vehicle's section.
- **`camera`** — hand the current camera to the vision system.
- **`set_camera`** — push a camera in.
- **`feel_vision_isRelevant`** — admit the actor and nothing else.
