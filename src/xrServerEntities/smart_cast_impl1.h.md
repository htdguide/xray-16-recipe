# src/xrServerEntities/smart_cast_impl1.h

> Searching the cast table at build time: find a chain of facet calls from source to target, or give up and fall back to the general type query.

**Needs** — [`smart_cast.h`](smart_cast.h.md) · [`smart_cast_impl0.h`](smart_cast_impl0.h.md) · [`smart_cast_stats.cpp`](smart_cast_stats.cpp.md)
**Used by** — [`smart_cast.h`](smart_cast.h.md) · [`smart_cast_impl0.h`](smart_cast_impl0.h.md) · [`smart_cast_stats.cpp`](smart_cast_stats.cpp.md)
**Tier floor** — T1: a whole-program search over a type table, resolved before the program runs.

## Purpose

[`smart_cast.h`](smart_cast.h.md) holds the table;
[`smart_cast_impl0.h`](smart_cast_impl0.h.md) builds it. This file **searches** it: given a
source type and a target type at a call site, decide at build time which of four answers
this particular cast is, and emit only that one.

Everything here is a compile-time computation. At run time the result is either nothing, a
pointer move, one virtual call, or a general type query — never a search.

## The four answers

```text
FUNCTION choose_cast(Source, Target) -> plan
  IF Target is Source, or Target is a base of Source
    RETURN direct                 # the relationship is static; no check exists to make
  chain = shortest_chain(Source, Target, max_length)
  IF chain is present
    RETURN facet_calls(chain)     # one virtual call per link
  RETURN general_query            # and, in a debug build, record that this happened
```

**Invariants** — the maximum chain length is **one**, in every build. The search below is
written for arbitrary lengths and then clamped, so in practice "chain" always means a single
registered (source, target) pair. This is the most consequential line in the family and it
is not explained anywhere: the generality was built, measured or not, and then turned off.
The plausible reason is build time, since the search is quadratic in the table's size.

**A cast of nothing answers nothing**, checked before anything else. This is the one runtime
branch the scheme always pays.

## The search

Three questions compose, each answered over the table's list-of-groups shape:

- **does this source have a group at all** — a walk looking for a group whose source type is
  the given one *or a base of it*, which is what lets a derived class use its base's
  registered casts;
- **does that group offer this target** — a walk over the group's targets looking for an
  exact match;
- **is there a longer way round** — for a chain of more than one hop, recurse through each
  reachable intermediate with the budget reduced by one, taking the first success.

**Invariants** — the intermediate search must not revisit a type, or a table with a cycle —
and the table has several, since most facet pairs are registered in both directions — would
not terminate. The recursion terminates because the length budget decreases, not because
cycles are detected; with the budget clamped to one, the question never arises. A rebuild
that raises the budget must add cycle detection first.

## Casting to an untyped pointer

**Contract** — a cast whose target is "any object at all" resolves to the most-derived
object's address. There is no facet method for that and no chain can produce it, so it
always takes the general query. It is used where the engine needs an identity for an object
whose type it does not care about.

## The constness rule

**Contract** — a cast from a read-only source to a writable target is **rejected at build
time**. Constness is stripped for the duration of the internal machinery and restored on the
result, which is what lets one implementation serve both.

## The debug cross-check

**Contract** — in a debug build every cast performs **both** the fast answer and the general
query and asserts they agree, naming both types and both addresses in the failure.

**Invariants** — this is the only mechanism that can catch a class whose facet method lies —
returns itself for a facet it does not have, or nothing for one it does. Since the table's
correctness rests entirely on hand-written facet methods in seventy-odd classes, a rebuild
that keeps the scheme must keep this check, and a rebuild whose type query is cheap should
keep the query and delete the scheme.

## The statistics hook

**Contract** — in a debug build, every cast that falls through to the general query records
the (source, target) pair. Optionally, every cast records itself regardless of which path it
took. See [`smart_cast_stats.cpp`](smart_cast_stats.cpp.md) for what is done with that.

**Notes** — this is how the table was populated in the first place: run the game, list the
casts that were not answered by the table, add the frequent ones. The "record everything"
mode is off by default because it is expensive enough to change what the profile looks like.

## Notes

**The type-safety preconditions** are checked at build time on every call: the target must
be a pointer or reference to a polymorphic type (or to nothing in particular), and the
source must be polymorphic. These reproduce what the language's own query would have
enforced, and they exist because the scheme bypasses it. They are pure incidental
machinery — a rebuild gets them from its own type system.

**A reference-taking form** exists alongside the pointer form, differing only in that it
dereferences the result. It cannot express failure, so a failed cast through it is undefined
rather than diagnosable. Every call site is expected to have established the relationship
another way.

**A build-time note in the source marks the fallback path as "not optimized"** and can be
turned on to list every such call site during a build. It is off, and the list it produces
is long.
