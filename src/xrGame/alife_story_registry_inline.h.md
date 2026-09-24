# src/xrGame/alife_story_registry_inline.h

> Lookup and removal in the story index, and the shared shape of "absent is usually a bug".

**Needs** — [`alife_story_registry.h`](alife_story_registry.h.md)
**Used by** — [`alife_story_registry.h`](alife_story_registry.h.md)
**Tier floor** — T2: a map lookup with a diagnosed absence

## Purpose

Two operations with real content and one accessor. Insertion is in
[`alife_story_registry.cpp`](alife_story_registry.cpp.md).

Both operations share one shape, and it is worth naming once:

```text
IF story_id is the invalid value -> the no-op answer (return, or nothing)
IF not present
  IF NOT tolerate_missing
    log the identifier, then FAIL
  RETURN the absent answer
```

The invalid identifier is checked **before** the lookup in both, so that the overwhelming
majority of entities — which carry no story identifier — cost nothing and never produce a
diagnostic. The logging before the failure is what makes a missing story entity
diagnosable at all: the identifier alone, printed by the assertion, is a number; the log
line is what a designer can match against the configuration.

## `object`

**Contract** — resolves a story identifier to the entity carrying it. Yields nothing for
the invalid identifier. Absence is a diagnosed fault unless the caller tolerates it, in
which case nothing is returned.

**Invariants** — the tolerant form is what scripts get (the script binding always passes
it), because a quest asking whether its story entity exists yet is a legitimate question.
Engine callers that hold a story identifier obtained from live data use the strict form.

## `remove`

**Contract** — drops the entry for a story identifier. The invalid identifier is a no-op.
Absence is a diagnosed fault unless tolerated.

## `objects`

**Contract** — the whole index, for callers that must sweep it.
