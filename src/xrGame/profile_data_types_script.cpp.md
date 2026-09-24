# src/xrGame/profile_data_types_script.cpp

> Exports the profile record shapes to the script layer.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [`profile_data_types_script.h`](profile_data_types_script.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration only

## Purpose

Makes the profile records readable from Lua so the profile screen can render them. Nothing
here decides anything; it names what scripts may touch.

## `script_register`

**Contract** — registers three read-only record shapes:

| Script name | Fields (all read-only) |
|---|---|
| `award_data` | `m_count`, `m_last_reward_date` |
| `award_pair_t` | `first` (the award kind), `second` (an `award_data`) |
| `best_scores_pair_t` | `first` (the score kind), `second` (the value) |

The two pair types exist because the award and best-score sets are handed to Lua as
iterators over key/value pairs rather than as tables; the pair is the element type that
iteration yields.

It also registers the completion-callback type under the script name
`store_operation_cb`, so a Lua function can be passed where an engine callback is expected.

**Invariants** — every field is exported read-only. A script may display a profile; it may
never author one, because the authority was the account service.
