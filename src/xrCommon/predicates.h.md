# src/xrCommon/predicates.h

> The two text orderings every sorted structure in the engine is built from: exact, and ASCII case-folded.

**Needs** — [`xrCore/xrstring.h`](../xrCore/xrstring.h.md)
**Used by** — [`xr_map.h`](xr_map.h.md) · [`xr_set.h`](xr_set.h.md) · [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md)
**Tier floor** — T3: the decision is about which characters compare equal, not about machines.

## Purpose

Ordered tables and collections keyed by raw text need an ordering supplied explicitly,
because the default ordering on a text *pointer* compares addresses. This file supplies the
only two orderings the engine uses, and by supplying exactly two it makes the choice at
every sorted-structure declaration a binary one: does this table distinguish case, or not?

That question has a right answer per site and the wrong answer is a bug that only shows up
with the shipped game data, which references files and sections with inconsistent
capitalisation.

## `pred_str` — exact ordering

**Contract** — orders two texts by comparing their characters as unsigned byte values, left
to right, the shorter being smaller when it is a prefix of the longer. Total, strict, and
transitive. Used where the key set is engine-generated and case is already normalised: the
virtual filesystem's root table, and the sorted name lists that get written back out.

## `pred_stri` — case-folded ordering

**Contract** — the same ordering, after folding each byte's case before comparing.

**Invariants**

```text
# The fold is ASCII-only and locale-independent, by requirement.
#   Exactly the 26 unaccented Latin letters fold; every other byte compares as
#   itself. This is not laziness. Game text ships in several single-byte codepages
#   depending on localization, so a locale-aware fold would make the set of
#   matching resource names depend on the user's locale, and a level would load
#   different files on a Turkish system than on an English one.
#   A rebuild MUST NOT reach for its language's ordinary case-insensitive compare
#   unless that compare is documented as ASCII-only.

# Folding both operands must produce the same answer as folding neither when both
# are already lowercase.
#   Callers rely on being able to pre-normalize a key and then look it up in a
#   case-folded structure. Several do exactly this to avoid the per-comparison
#   fold cost in hot lookups.
```

**Notes**

Both orderings are stateless and carry no data, which is what lets a sorted structure name
one as a type parameter rather than hold one as a field. A rebuild passes a comparison
function; nothing is lost.

Note what is *not* here: there is no ordering for the engine's interned string type. That
type orders by the identity of its interned record — which is stable within a run and
meaningless across runs — and a sorted structure keyed by it is therefore ordered
arbitrarily but consistently. Where content order is actually wanted, callers order by the
underlying text with the predicates on this page. This trap is described at
[`xrCore/xrstring.h`](../xrCore/xrstring.h.md).
