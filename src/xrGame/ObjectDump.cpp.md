# src/xrGame/ObjectDump.cpp

> Formats an object's identity, registration flags, recent position history and visual bounds into readable text, for a developer staring at a bug.

**Needs** — [`ObjectDump.h`](ObjectDump.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md)
**Used by** — [`ObjectDump.h`](ObjectDump.h.md)
**Tier floor** — T3: string formatting

## Purpose

When an object misbehaves — updates twice in a frame, renders while destroyed, teleports — the
question is always the same: what state is it actually in? This file answers it in one string.
It is compiled only into non-shipping builds and a rebuild may omit it entirely, but the *set
of fields it prints* is worth recording, because that set is the engine's own answer to "what
about an object can be wrong".

## State

`Stateless.`

## `dbg_object_full_dump_string` · `dbg_object_full_capped_dump_string`

**Contract** — the whole picture: identity, then registration properties, then position
history, then visual bounds. The capped form is the same with a leading label. Tolerates a
missing object at every level.

## `dbg_object_base_dump_string`

**Contract** — the object's instance name, its configuration section and its visual model name,
or a note that the object reference is empty.

**Notes** — those three answer "which object is this" completely: the instance name identifies
it in the level, the section supplies every tuned number, and the visual is the model. An
entity is (class identifier, section), and this prints the half that is data.

## `dbg_object_props_dump_string`

**Contract** — the object's registration and lifecycle state. Prints every flag the object
keeps about its own participation in the engine, plus the frame counters that say when it was
last updated.

```text
# the flags, and what a wrong value means
net_ID          the entity identifier
active_counter  how many times the object has been made active without being deactivated
enabled         participates in the scheduler
visible         participates in rendering
destroy         marked for removal
net_local       this side is the authority for it
net_ready       its spawn has completed
net_sv_update   the server wants an update from it
crow            it is in the "always updated, whatever the distance" set
pre_destroy     removal has begun but not completed

# the counters
dbg_update_cl        how many times the per-frame update ran
frame_update_cl      the frame the per-frame update last ran
frame_as_crow        the frame it was last updated as an always-updated object
current frame, current global time
```

**Invariants** — comparing the last-update frame against the current frame is the whole point:
an object whose update frame equals the current frame *and* which is about to be updated again
is the double-update bug the engine asserts against, and an object whose update frame is far
behind is one the scheduler has starved.

## `dbg_object_poses_dump_string`

**Contract** — the object's current transform followed by its recorded position history: each
saved position with the time it was recorded.

**Notes** — the position stack is a short ring of recent positions every object keeps. Its
purpose is exactly this: when an object is found somewhere impossible, the history says
whether it moved there or was placed there.

## `get_string` for values

**Contract** — formats a boolean, a three-component vector, a four-by-four transform (one row
per line, including the fourth column that is usually ignored) and an axis-aligned box as its
two corners.

**Notes** — printing the transform's fourth column matters: a transform that has picked up a
non-unit fourth row is corrupted, and the corruption is invisible if only the upper three-by-three
is shown.
