# src/xrEngine/ISheduled.h

> What an object must answer to be advanced by the update scheduler, and the bounds it declares on how often.

**Needs** — [`ISheduled.cpp`](ISheduled.cpp.md) · [`xrSheduler.h`](xrSheduler.h.md) · [`Engine.h`](Engine.h.md)
**Used by** — [`ISheduled.cpp`](ISheduled.cpp.md) · [`PS_instance.h`](PS_instance.h.md) · [`xrSheduler.cpp`](xrSheduler.cpp.md) · [`xrSheduler.h`](xrSheduler.h.md) · [`xr_object.h`](xr_object.h.md) · [`alife_update_manager.h`](../xrGame/alife_update_manager.h.md) · [`autosave_manager.cpp`](../xrGame/autosave_manager.cpp.md) · [`autosave_manager.h`](../xrGame/autosave_manager.h.md) · [`base_client_classes_wrappers.h`](../xrGame/base_client_classes_wrappers.h.md) · [`configs_dumper.cpp`](../xrGame/configs_dumper.cpp.md) · [`configs_dumper.h`](../xrGame/configs_dumper.h.md) · [`vision_client.h`](../xrGame/vision_client.h.md)
**Tier floor** — T1: the per-object record is packed into a single word because the scheduler walks thousands of them inside a frame budget

## Purpose

The second of the three facets a game object wears. An object that needs logic run over
time registers here; the scheduler then calls it back with the elapsed milliseconds,
choosing *when* based on the bounds the object declares and on how much time is left in
the frame's update budget. The object does not get to decide its own rate.

The interface is substantive — it is the contract the update scheduler is written
against. The shared filling is in [`ISheduled.cpp`](ISheduled.cpp.md); the scheduler
itself is in [`xrSheduler.cpp`](xrSheduler.cpp.md).

**Spelling.** The project spells it `shedule`. That spelling is in the registered script
surface and in every call site, so the recipe keeps it where it names an identifier and
uses *schedule* in prose.

## State

One record per scheduled object. The two bounds are packed into fourteen bits each, which
caps a declared period at 16383 ms — about sixteen seconds — and that cap is real: an
object that wants to run more rarely than that must count its own skips.

```text
RECORD SchedulerData
  t_min    : int (14-bit)   # do not call me more often than this, in ms. typical 20
  t_max    : int (14-bit)   # do not leave me un-called longer than this, in ms. typical 1000
  b_rt     : bool           # this object must be advanced in real time, never deferred
  b_locked : bool           # currently inside its own update; re-entry forbidden

# invariant: t_min <= t_max
# invariant: b_locked is true only between the scheduler entering and leaving this object
```

Debug builds add two frame stamps used only to catch a double update in one frame.

## `ISheduled`

**Contract** — what an implementor owes the scheduler.

```text
INTERFACE Sheduled
  get_scheduler_data() -> SchedulerData     # by reference; the scheduler writes b_locked
  shedule_scale()  -> real                  # 0..1 importance; see note
  shedule_update(dt_ms : int)               # advance me by this much elapsed time
  shedule_name()   -> text                  # for diagnostics only
  shedule_needed() -> bool                  # false = skip me entirely this pass
```

`shedule_scale` is the object's own statement of how much it matters right now — in
practice distance from the camera, normalized, so a creature across the level returns near
1 and one in front of the player returns near 0. The scheduler interpolates the object's
actual period between `t_min` and `t_max` using it, so importance directly buys update
frequency. Returning a constant is legal and gives a fixed-rate object.

`dt_ms` is the time actually elapsed since this object's *previous* update, not the frame
delta — an object updated every 800 ms receives 800, and its logic must integrate over
that, which is the single most commonly violated invariant in code built on this
interface.

**Invariants** — `shedule_update` is called at most once per frame for a given object;
re-entering an object that is already locked is a programming error.

## `ScheduledBase`

**Contract** — the shared filling: owns a `SchedulerData`, defaults it, and offers
register/unregister against the global scheduler. Its lifetime decisions are in
[`ISheduled.cpp`](ISheduled.cpp.md).

**Notes** — the default bounds (20 ms floor, 1000 ms ceiling) are the game's general-purpose
pair: 20 ms is one update per simulation step at the engine's nominal rate, so an important
object is effectively per-step; 1000 ms is slow enough that a distant object costs nothing
and fast enough that a returning player does not see it snap.
