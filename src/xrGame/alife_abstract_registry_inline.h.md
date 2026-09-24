# src/xrGame/alife_abstract_registry_inline.h

> The generic alife registry's five operations.

**Needs** — [`alife_abstract_registry.h`](alife_abstract_registry.h.md)
**Used by** — [`alife_abstract_registry.h`](alife_abstract_registry.h.md)
**Tier floor** — T3: table operations.

## Purpose

Holds the definitions of the operations declared in
[`alife_abstract_registry.h`](alife_abstract_registry.h.md), which is where the contracts
are written. The split is an artefact of the original language's template rules; a rebuild
has one file.

## Operations

```text
FUNCTION add(key, record, tolerate_duplicate)
  IF key is present
    IF NOT tolerate_duplicate THEN FAIL WITH "already in this registry"
    RETURN                        # the existing record wins; the new one is dropped
  insert (key, record)

FUNCTION remove(key, tolerate_missing)
  IF key is absent
    IF NOT tolerate_missing THEN FAIL WITH "not in this registry"
    RETURN
  erase key

FUNCTION object(key, tolerate_missing) -> optional<record>
  IF key is absent
    IF NOT tolerate_missing THEN FAIL WITH "not in this registry"
    RETURN none
  RETURN the record
```

**Invariants** — When a duplicate add is tolerated, the *existing* record is kept and the
new one discarded, rather than the reverse. Callers that pass the tolerate flag rely on
that: it makes re-registration idempotent.

**Notes** — The failure is raised as a thrown condition rather than an assertion, so the
tolerate flag and the error are the same mechanism: the flag suppresses the throw. A
rebuild returning a result rather than throwing collapses the flag away entirely — the
caller either checks the result or does not.
