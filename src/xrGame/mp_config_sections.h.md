# src/xrGame/mp_config_sections.h

> Declares the two anti-cheat configuration reporters implemented in [`mp_config_sections.cpp`](mp_config_sections.cpp.md).

**Needs** — [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md)
**Used by** — [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) · [`configs_dumper.cpp`](configs_dumper.cpp.md) · [`configs_dumper.h`](configs_dumper.h.md) · [`mp_config_sections.cpp`](mp_config_sections.cpp.md) · [`mpactor_dump_impl.cpp`](mpactor_dump_impl.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the client-side half of the multiplayer anti-cheat check: a way for a server to ask
a client to prove that the configuration it is playing with is the shipped one. Substance is
in [`mp_config_sections.cpp`](mp_config_sections.cpp.md).

Exported units:

- `mp_config_sections` — walks a fixed list of gameplay-critical configuration sections and
  serializes them one at a time, so the transfer can be spread across frames.
- `start_dump` / `dump_one` — reset the walk, and emit the next section.
- `mp_active_params` — reports the *live, in-memory* values of one object's tuned
  parameters, which may differ from the configured ones if something has patched them.
- `dump` — write an object's live parameters into a report, under a derived section name.
- `load_to` — the verifier's side: copy a named configuration section out of the real
  configuration for comparison.
- `active_params_section` — the fixed section name under which the report records which
  derived section belongs to which key.
