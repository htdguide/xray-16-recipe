# src/xrGame/profile_store_script.cpp

> Exports the profile store and the two achievement enumerations to the script layer.

**Needs** — [`profile_store.h`](profile_store.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration only

## Purpose

Names the profile store's script surface, which the shipped multiplayer profile screen
calls into.

## `script_register`

**Contract** — registers the class `profile_store` with:

| Script member | Role |
|---|---|
| `load_current_profile(progress_cb, complete_cb)` | asynchronous load; callbacks may be Lua functions |
| `stop_loading()` | cancel; inert |
| `get_awards()` | returns an **iterator** over award key/value pairs, not a table |
| `get_best_scores()` | returns an iterator over best-score key/value pairs |

plus two nested enumerations scripts address by name:

- `enum_awards_t` — only `at_award_massacre` (the first) and `at_awards_count` (the
  sentinel) are exported. Scripts walk the range between them numerically rather than
  naming each of the thirty awards.
- `enum_best_score_type` — all seven kinds plus the sentinel.

**Invariants** — both getters hand out iterators over the engine's own containers, so the
script must consume them before the store changes. Since the store never changes, this
costs nothing in practice, but a rebuild that revives the service must copy instead.

**Notes** — two of the exported best-score values are wrong in the original:
`bst_bleed_kills_in_row` is bound to the *backstabs* ordinal and
`bst_explosive_kills_in_row` to the *head shots* ordinal. Scripts reading those two names
therefore address the wrong counter. Since every counter is permanently zero this is
invisible, but a rebuild that restores real profiles must decide deliberately whether to
reproduce the aliasing or correct it — the shipped scripts were written against the buggy
binding.
