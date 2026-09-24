# src/xrGame/script_game_object_impl.h

> Turns "the script is holding a destroyed object" from a crash into a named script error.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrScriptEngine/script_engine.hpp`](../xrScriptEngine/script_engine.hpp.md)
**Used by** — [`raypick.h`](raypick.h.md) · [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_game_object.h`](script_game_object.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md) · [`script_game_object_smart_covers.cpp`](script_game_object_smart_covers.cpp.md) · [`script_game_object_trader.cpp`](script_game_object_trader.cpp.md) · [`script_game_object_use.cpp`](script_game_object_use.cpp.md) · [`script_game_object_use2.cpp`](script_game_object_use2.cpp.md) · [`script_sound.cpp`](script_sound.cpp.md)
**Tier floor** — T2: a validity check on a handle

## Purpose

Every one of the several hundred methods on the script-visible facade begins by reaching
for the client object behind it. This file is that reach, and it is a separate file because
it must be *inlined into every implementation file* of the facade — which is also why it is
included by each of them individually rather than by the facade's own header.

The decision it encodes: scripts routinely keep a game object handle across a frame in
which the entity dies. Dereferencing it is undefined; instead the debug build detects it
and reports which handle, as a script error the modder can act on.

## State

`Stateless.`

## `object`

**Contract** — returns the client object this facade fronts. In a development build it
first checks the two-way link — the facade points at the object *and* the object points
back at this facade — and, on failure, logs a script error naming the stale handle and
aborts. In a shipping build it returns the pointer unchecked, so the cost is zero in
release.

**Invariants**

- The two-way link is the validity test. A one-way test would pass for a handle whose
  object was destroyed and whose memory was reused by another object, which is the common
  case.
- The check is wrapped so that a fault *during the check itself* — reading a freed
  object's back-pointer — is caught and turned into the same script error, rather than
  becoming the crash the check exists to prevent.

**Notes**

The wrapping is a known wart in the original, commented as such. A rebuild should get the
same guarantee from a handle that can be validated without dereferencing — a generation
counter beside the entity identifier, say — which makes the check cheap enough to keep in
the shipping build and removes the need to trap a fault.
