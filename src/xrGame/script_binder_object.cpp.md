# src/xrGame/script_binder_object.cpp

> The base class a script author subclasses to attach their own state and lifecycle to an existing game object.

**Needs** — [`script_binder_object.h`](script_binder_object.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`script_binder_object.h`](script_binder_object.h.md); callers name that, not this file.
**Tier floor** — T2: pure dispatch and policy; nothing here touches a device or a byte layout

## Purpose

A **binder** is a script-authored object permanently attached to one game object, receiving
the same lifecycle events the engine sends the object itself. This file supplies the base
of that hierarchy: every callback exists, every callback does nothing, and the object
handle is public. Scripts override the hooks they care about and inherit no-ops for the
rest, so adding a new lifecycle hook to the engine never breaks a shipped script.

The split from the wrapper ([`script_binder_object_wrapper.cpp`](script_binder_object_wrapper.cpp.md))
is load-bearing: this class is the *default behaviour*, the wrapper is the *dispatch to
the script*. A script calling the base implementation must reach the no-ops here, not
recurse back into its own override.

## State

```text
RECORD ScriptBinderObject
  object : GameObject          # the script-visible facade this binder is attached to;
                               # public, because scripts read `self.object` constantly
```

The binder does not own the object. The object outlives the binder in the normal teardown
order, and the binder must never free it.

## `construct(object)`

**Contract** — binds this instance to one game object facade for its whole life. There is
no rebind.

## The lifecycle hooks

**Contract** — eleven virtual hooks, all no-ops at this level, that mirror the entity
lifecycle exactly. Their *order* is the contract, because a script that saves state in one
and reads it in another depends on it:

```text
reinit()                      # reset to a just-constructed state, before any spawn data
reload(section)               # re-read tuning from the named configuration section
net_spawn(server_record) -> bool   # bring online from the authoritative record;
                                   # false aborts the spawn and the object is discarded
shedule_update(time_delta)    # periodic update, at the scheduler's degraded rate
net_import(packet)            # apply a net update from the authoritative side
net_export(packet)            # produce a net update for the authoritative side
save(packet)                  # append this binder's state to the save stream
load(reader)                  # restore it, reading exactly what save wrote
net_save_relevant() -> bool   # does this binder have any state worth saving at all
net_relcase(object)           # a game object is being destroyed; drop every reference
                              # you hold to it before it becomes invalid
net_destroy()                 # go offline; release everything acquired at spawn
```

**Invariants**

- `net_spawn` and `net_destroy` are paired and may run many times over one binder's life —
  an alife entity crosses online/offline repeatedly.
- `load` must consume exactly the bytes `save` produced, and `save` runs only when
  `net_save_relevant` answered true, so the save stream carries no record for a binder
  that declined.
- `net_relcase` fires *before* the named object's memory is released. Ignoring it leaves a
  dangling reference, which is one of the runtime invariants in
  [conformance](../../SYSTEM-REQUIREMENTS.md#6-conformance).

**Notes**

`net_spawn` returning true by default is deliberate: a script that overrides nothing must
not be able to veto its object's spawn by accident. `net_save_relevant` returning false by
default is the matching decision on the other side — a binder that stored nothing writes
nothing.

The destructor logs the bound object's name in debug builds only; the reason is that
binder teardown ordering bugs are otherwise invisible, since a binder's death is a
consequence of its object's death rather than of anything the script did.
