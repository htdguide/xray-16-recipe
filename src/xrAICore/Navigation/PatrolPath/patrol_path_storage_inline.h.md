# src/xrAICore/Navigation/PatrolPath/patrol_path_storage_inline.h

> Looking a patrol path up by name, with the caller choosing whether an absent path is a failure or an answer.

**Needs** — [`patrol_path_storage.h`](patrol_path_storage.h.md)
**Used by** — [`patrol_path_storage.h`](patrol_path_storage.h.md)
**Tier floor** — T2: a binary search over interned names.

## Purpose

The one interesting decision in the registry's read surface: whether "there is no such path" is
an error. Both answers are needed. A creature whose spawn record names a patrol path that does
not exist is a content bug and should stop the game at the point of the mistake; a caller
*testing* whether a path exists needs the same lookup to answer quietly.

## `path(name, tolerate_absence)`

**Contract** — looks the name up. When present, yields the path. When absent and the caller did
not opt in to tolerating it, logs the missing name and fails hard. When absent and the caller
did opt in, yields nothing.

**Notes** — the flag defaults to "do not tolerate", so the strict behaviour is what a caller gets
by not thinking about it. That is the right default for this data: a missing patrol path is
almost always a typo in a spawn record or a script, and failing at the lookup names the offending
string, while returning nothing turns it into a crash several frames later with no context.

The missing name is logged *and* raised, so the name survives even when the failure is caught
further out.

## `patrol_paths`

**Contract** — the whole registry, for callers that enumerate.

## Construction

**Contract** — an empty registry. There is nothing to do; the loaders fill it.
