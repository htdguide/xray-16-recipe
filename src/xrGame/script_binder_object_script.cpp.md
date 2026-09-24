# src/xrGame/script_binder_object_script.cpp

> Exports the binder base class to the script layer under the name `object_binder`.

**Needs** — [`script_binder_object.h`](script_binder_object.h.md) · [`script_binder_object_wrapper.h`](script_binder_object_wrapper.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of the binder. The names here are frozen by every
shipped script (conformance criterion 10), and they are *not* the engine-side names — the
mapping is part of the contract.

## `script_register`

**Contract** — registers a class named `object_binder`, constructible from one game object,
with a read/write property `object`, and these methods, each paired with the base
implementation so a subclass can call up the chain:

```text
reinit            -> reinit
reload            -> reload
net_spawn         -> net_spawn
net_destroy       -> net_destroy
net_import        -> net_import
net_export        -> net_export
update            -> shedule_update        # renamed: scripts say "update"
save              -> save
load              -> load
net_save_relevant -> net_save_relevant
net_Relcase       -> net_relcase           # capitalization is inconsistent and frozen
```

**Notes**

Each registration names two functions: the virtual one, which dispatches to the script
override, and a static one that invokes the base implementation directly. The second is
what makes `base:method()` work from a subclass without infinite recursion — a rebuild
whose binding layer expresses "call the parent implementation" some other way may drop the
pairing, but must keep the capability.

`net_Relcase` and `update` differ in spelling from their engine names. Those spellings are
in shipped scripts and cannot be tidied.
