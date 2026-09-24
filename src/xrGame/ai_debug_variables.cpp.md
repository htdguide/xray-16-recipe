# src/xrGame/ai_debug_variables.cpp

> A global scratch table of named numbers, so a probe deep in the AI can publish a value that a console command or another subsystem can read back without a plumbing change.

**Needs** — [`ai_debug_variables.h`](ai_debug_variables.h.md)
**Used by** — [`ai_debug_variables.h`](ai_debug_variables.h.md)
**Tier floor** — T3: a keyed table.

## Purpose

Diagnosing AI behaviour means watching a number produced five call levels below anything
that can display it. Rather than thread the value out, a probe writes it into this table
by name and whatever wants it reads it by name. The table is process-global and has no
lifetime management, which is acceptable precisely because it is diagnostic: nothing in
the game's behaviour may depend on a value being present.

## State

```text
RECORD DebugVar
  kind  : ENUM { text, real }
  text  : text (fixed capacity, 1024)   # only meaningful when kind is text
  value : real                          # only meaningful when kind is real

# One process-global table: map<text, DebugVar>, keyed by variable name.
# Invariant: the table is never cleared; entries live for the process.
# Invariant: the text variant can be read only by the display path — there is no
#   setter for it, so in practice every entry is a real.
```

## `set_var`

**Contract** — Stores a real under a name, replacing any previous entry including one of
the other kind. No allocation bound, no size limit on the table. Not thread-safe: the
table is a plain map with no lock, so a probe on a worker thread and a console read on the
main thread race. The engine gets away with it because the debug console runs on the main
thread and the probes that use this are on it too; a rebuild adding threaded probes must
add the lock.

## `get_var`

**Contract** — Three readers over the same lookup: as a real, as a non-negative integer
(truncated toward zero from the real), and as a truth value (non-zero is true). Each
answers whether the name exists *and* holds a real; a name holding text reads as absent.
The output is untouched on a miss, so the caller's default survives.

## `show_var`

**Contract** — Prints one variable's name and value to the engine log, formatting by kind.
Silent when the name is absent — this is a console convenience, not a query.

**Notes** — The text kind exists with a setter missing, and the fixed 1024-byte capacity
suggests it was intended for printf-style debug strings. A rebuild should either add the
setter or drop the kind; today it is dead weight that only `show_var` can observe.
