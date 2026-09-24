# src/xrEngine/ISheduled.cpp

> Default bounds, registration, and the two guards that catch a scheduled object outliving or re-entering its own update.

**Needs** — [`ISheduled.h`](ISheduled.h.md) · [`xrSheduler.h`](xrSheduler.h.md) · [`xr_object.h`](xr_object.h.md) · [`Engine.h`](Engine.h.md)
**Used by** — [`ISheduled.h`](ISheduled.h.md)
**Tier floor** — T2: registry bookkeeping and two assertions; nothing device- or layout-facing

## Purpose

Three decisions about a scheduled object's life: what rate bounds it starts with, what
must be true when it dies, and how a double update in one frame is detected. The file is
small and its name is wrong (it implements `ScheduledBase`, not `ISheduled`); merging it
into the header's twin would be defensible.

## State

`Stateless.` The record is the object's, declared in [`ISheduled.h`](ISheduled.h.md).

## Construction

**Contract** — a new scheduled object declares the general-purpose bounds and starts
unlocked. It is *not* registered — registration is an explicit later act, because objects
are constructed during a level load long before the scheduler should begin advancing them.

```text
FUNCTION on_create(self)
  self.shedule.t_min = 20
  self.shedule.t_max = 1000
  self.shedule.b_locked = false
```

## Destruction

**Contract** — a scheduled object must not be in the scheduler when it dies; the scheduler
holds a bare reference and would advance freed storage on the next pass. The engine both
asserts this and, as a belt-and-braces measure, unregisters anyway.

```text
FUNCTION on_destroy(self)
  ASSERT NOT scheduler.is_registered(self)   # with the object's name, to find the culprit
  scheduler.unregister(self)                 # defensive: see note
```

**Notes** — the assert and the unregister are enabled in *opposite* build configurations
in the original: the check runs in development builds, the silent unregister runs in
shipping ones. The source calls this out as a known wart. The load-bearing reading is:
the invariant is real and is violated somewhere in the game module that nobody found, so
the shipping build papers over it. A rebuild should keep the assertion in every build and
fix the call sites, because the papered-over path leaves the scheduler's own bookkeeping
inconsistent — see [`xrSheduler.cpp`](xrSheduler.cpp.md).

## `shedule_register` / `shedule_unregister`

**Contract** — pure delegation to the global update scheduler. Registering twice is
rejected by the scheduler, not here.

## `shedule_Update`

**Contract** — the base implementation does no work in a shipping build. In development
builds it is the *double-update detector*: it stamps the object with the frame it was last
advanced in and fails hard if that stamp already equals the current frame.

```text
FUNCTION shedule_update(self, dt_ms)
  IF self.dbg_last_update_frame == self.dbg_current_frame
    FAIL WITH "object advanced twice in one frame", naming the object
  self.dbg_last_update_frame = self.dbg_current_frame
```

**Notes** — this is a hard failure rather than a warning because a double update silently
doubles every rate in the object's logic — movement, damage over time, ammunition drain —
and produces a bug that reproduces only under specific scheduling pressure. Catching it at
the moment it happens is worth stopping the process.
