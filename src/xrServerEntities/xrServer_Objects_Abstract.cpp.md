# src/xrServerEntities/xrServer_Objects_Abstract.cpp

> How a record's model reference and its animation clip name are normalized, stored and serialized.

**Needs** — [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) · [`xrMessages.h`](xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it defines two on-disk field groups.

## Purpose

The two string fields every visible record carries — the model it draws as, and the
animation clip it starts in — together with the normalization they go through before being
stored. Both are interned names resolved against the virtual filesystem later, so the
normalization rule is what decides whether a record written by one tool is found by
another.

It is separated from the record definitions themselves because both fields are mixed into
many otherwise unrelated records.


## State

```text
RECORD VisualMixin
  visual_name       : text     # normalized: extension stripped, lowercased
  startup_animation : text     # not serialized here; owners write it themselves
  flags             : int (8-bit)   # bit 0: the model obstructs navigation

RECORD MotionMixin
  motion_name       : text
```

**Invariants**

- **The model path is normalized on every assignment**, not on load: the extension is cut and
  the whole string lowercased. The virtual filesystem matches case-insensitively but stores
  case-sensitively, so records authored on a case-insensitive system reference models with
  inconsistent case; normalizing at the record is what makes them resolve. A rebuild that
  skips this will fail to find perhaps a third of the shipped models.
- **The startup animation defaults to a marker string** (`$editor`) rather than empty, so a
  tool can tell "never set" from "deliberately none".
- **The visual's flag byte only exists from spawn version 104.** Reading an older record
  leaves the flags zero. This is the general shape of every version gate in the chapter.

## `visual_read` / `visual_write`

**Contract** — reads the model path, then the flag byte if the record's version is above 104;
writes both unconditionally. Note the asymmetry: the writer has no version parameter, because
the engine only ever writes the current layout.

## `motion_read` / `motion_write`

**Contract** — one zero-terminated string, no version gate. The motion mixin has never
changed shape.

## `set_visual`

**Contract** — assigns and normalizes as above.

## Editor change notification

**Contract** — assigning a model, an animation or a motion through the editor marks the
owning record's corresponding change flag, so the tool knows to reload the model or restart
the clip. The mixin reaches its owner by asking itself for the entity interface — a
consequence of the mixin not knowing which record it was mixed into. A rebuild that composes
rather than inherits passes the owner in.
