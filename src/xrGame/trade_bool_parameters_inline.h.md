# src/xrGame/trade_bool_parameters_inline.h

> The three operations of the refusal set.

**Needs** — [`trade_bool_parameters.h`](trade_bool_parameters.h.md)
**Used by** — [`trade_bool_parameters.h`](trade_bool_parameters.h.md)
**Tier floor** — T3: a list and a linear search.

## Purpose

The body of [`trade_bool_parameters.h`](trade_bool_parameters.h.md). A rebuild uses whatever
set its language offers and deletes this file.

## `clear` / `disable(section)` / `disabled(section)`

**Contract** — empty the set; append a section, which must not already be present; and report
membership by linear search.

**Notes** — the duplicate check on insertion is a diagnostic-build assertion rather than a
tolerated case, and it is the right call: a duplicate means the same configuration section
was processed twice, which is a loading bug that would otherwise go unnoticed because the
membership answer is unchanged.
