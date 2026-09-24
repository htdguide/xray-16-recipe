# src/utils/mp_balancer/xr_ini_ex.h

> Declares the tool's own configuration reader — the engine's, plus the two things the engine throws away.

**Needs** — [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — [`pch.h`](pch.h.md) · [`statistics_collector.cpp`](statistics_collector.cpp.md) · [`wpn_collection.cpp`](wpn_collection.cpp.md) · [`wpn_collection.hpp`](wpn_collection.hpp.md) · [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md)

**Tier floor** — T3: a text format's in-memory shape.

## Purpose

Declares the surface implemented in [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md). It is a near-copy
of the engine's configuration reader, and the copy exists for exactly two reasons, both
visible in the record shapes here: an item carries the **comment** that followed it on its
line, and a section carries the **names of the sections it inherits from**. The engine
discards both the moment inheritance is flattened, because at run time nothing needs them.
A tool that *rewrites* configuration files needs both, or its output is unreadable to the
humans who maintain it.

The substitution is made in [`pch.h`](pch.h.md), so the whole tool gets this reader in
place of the engine's without naming it at each use site.

## Exported units

- `CInifileEx` — a loaded configuration file: an ordered set of named sections, each an
  ordered set of key/value items.
- `Sect` — one section: its name, its items, and the names of its parents.
- `Item` — one key/value pair plus the comment that followed it.
- `Create` / `Destroy` — construct and release; a rebuild needs neither.
- `Load` — parse a stream, resolving includes relative to a given folder.
- `save_as` — write the whole file back, to a named path or to an open sink.
- `r_section`, `section_exist`, `line_exist`, `line_count`, `sections` — structure queries.
- `r_string`, `r_string_wb` — a value as written, and a value with its surrounding quotes
  removed.
- `r_u8` … `r_s64`, `r_float`, `r_bool`, `r_color`, `r_fcolor`, `r_ivector2` … `r_fvector4`,
  `r_clsid`, `r_token` — the same value parsed as a number, a flag, a colour, a vector, a
  class identifier or a member of a named set.
- `r_line` — the *n*-th item of a section by position, which is how a tool walks a section
  it does not know the keys of.
- `w_string`, `w_u8` … `w_bool` — set a value, creating the section if needed.
- `remove_line` — delete one item.
- `set_override_names`, `save_at_end`, `fname` — write-mode policy and the path it came
  from.
- `IsBOOL` — the set of texts that count as true.
- `pSettingsEx` — a process-wide handle to one loaded file. Unused by this tool; see
  [`xr_ini_ex.cpp`](xr_ini_ex.cpp.md).
