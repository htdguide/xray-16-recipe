# src/utils/mp_balancer/wpn_collection.hpp

> Declares the object that holds two generations of weapon configuration side by side and the rules for merging them.

**Needs** — [`pch.h`](pch.h.md) · [`xr_ini_ex.h`](xr_ini_ex.h.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`statistics_collector.cpp`](statistics_collector.cpp.md) · [`wpn_collection.cpp`](wpn_collection.cpp.md)

**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`wpn_collection.cpp`](wpn_collection.cpp.md). It is
one object because a merge run needs three things alive at once — the previous game's
configuration, the patch's configuration, and the operator's answers so far — and none of
them makes sense without the others.

The record shapes here carry one decision worth naming before the twin: the output is
grouped by **destination file**, not by weapon. A rule in the job description names a
section-name *prefix* and the file every section with that prefix goes to, so the tool's
result is a small set of files each holding many sections.

## Exported units

- `weapon_collection` — the whole merge run.
- `load_all_mp_weapons` — open both configuration generations and the job description,
  and enumerate the multiplayer items from the deathmatch price list.
- `load_settings` — read the job description's extraction rules.
- `get_extract_keys` — find the rule that governs a given section.
- `extract_all_params` — the merge itself.
- `try_extract_from_patch` — the patch's value for one key, if it has one.
- `copy_params_ex` — pull a named subset of an inherited section's keys down into a
  concrete section, asking the operator about each disagreement.
- `build_section` — assemble one output section: the inherited keys worth keeping, plus
  the section's own keys that are not merely restating an inherited value.
- `save_config_to_file`, `save_new_configs` — write the grouped result out.
- `tentity_extract_keys` — one rule: a section-name prefix and the set of inherited keys
  to pull down for it.
- `tnew_config_map` — the result, keyed by destination file name.
