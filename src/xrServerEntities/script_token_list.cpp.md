# src/xrServerEntities/script_token_list.cpp

> Builds a name/number vocabulary in the terminated-array shape the configuration reader expects, and owns the names' storage.

**Needs** — [`script_token_list.h`](script_token_list.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md)
**Used by** — [`script_token_list.h`](script_token_list.h.md)
**Tier floor** — T2: the container's in-memory shape is dictated by a consumer that walks it looking for a terminator.

## Purpose

The configuration reader's token lookup was written against a C-style array terminated by an
entry with no name. Every vocabulary in the engine is such an array, declared statically.
This type builds one at run time from script, which means it must maintain that shape
exactly — including the terminator — rather than expose a list and convert on demand.

That single constraint explains everything unusual about this file.

## State

```text
RECORD ScriptTokenList
  entries : list<(name: optional<text>, id: int)>
```

**Invariants** — the last entry is always the **terminator**: no name, and a number of −1.
The list is therefore never empty, and its length is one greater than the number of real
tokens. Every real entry's name is a copy this type allocated and must free; the terminator
owns nothing.

Names and numbers are each unique across the real entries. Uniqueness is checked only in a
debug build.

## `add`

**Contract** — appends a pair. Requires that neither the name nor the number is already
present.

```text
FUNCTION add(list, name, id)
  REQUIRE no entry matches name, and none matches id
  # the terminator is overwritten in place and a fresh one appended:
  # this keeps the terminator last without an insert-before-end
  last = LAST ENTRY OF list.entries   # the terminator
  last.name = COPY OF name            # this list now owns the text
  last.id   = id
  APPEND (none, -1) TO list.entries
```

**Notes** — reusing the terminator slot rather than inserting before it is not a
micro-optimization; it is what keeps the terminator's address valid in the common case and
the code honest about the invariant. The important part for a rebuild is only that the
terminator stays last.

## `remove`

**Contract** — removes the entry with the given name, releasing its text. A name that is not
present is **reported to the log and otherwise ignored** — the caller is script code and a
stale name in a mod's cleanup path should not stop the game.

## `clear`

**Contract** — empties the list and re-establishes the terminator, so a cleared list is
still a valid vocabulary of zero tokens. Construction is a clear.

**Notes** — clearing does **not** release the names it drops. Everything is released only at
destruction, which walks every entry. A script that clears and refills a large vocabulary
repeatedly accumulates dead names for the list's lifetime. This is a genuine leak in the
original; a rebuild whose names are owned values has no way to reproduce it and should not
try.

## `id` / `name`

**Contract** — linear search for a name, or for a number, answering the other half.
**Both require a match**: a miss is checked only in a debug build, and in a shipping build
reads past the end of the list. The lookup skips the terminator implicitly, because the
terminator has no name and the name comparison is guarded on that.

**Notes** — linear search is correct here. These vocabularies are tens of entries and are
consulted at load time, not per frame.
