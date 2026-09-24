# src/utils/mp_configs_verifyer/mp_config_sections.h

> Declares which configuration sections a player's client must agree with the server about, and how that set is serialized in a fixed order.

**Needs** — [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)

**Used by** — [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) · [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) · [`mp_config_sections.cpp`](mp_config_sections.cpp.md)

**Tier floor** — T2: an ordered enumeration and a serialization order. It is only not T3 because the byte sequence it produces is hashed, so the order is part of a wire contract.

## Purpose

Declares the surface implemented in
[`mp_config_sections.cpp`](mp_config_sections.cpp.md). Two distinct ideas live here and
they are together because both answer "what goes into a dump":

- **`mp_config_sections`** — the *static* half: the fixed list of configuration sections
  whose contents both ends must agree about, and a stepper that serializes them one at a
  time into a growing byte buffer.
- **`mp_active_params`** — the *dynamic* half: the parameters of whatever the player is
  currently carrying, which are not known from configuration alone and are named by the
  dump itself.

## Exported units

- `mp_config_sections` — the static section set. Constructed by reading the shipped
  configuration; construction is the enumeration.
- `start_dump` — rewind the stepper to the first section.
- `dump_one` — serialize the next section into a byte sink and report whether more remain.
- `mp_active_params` — the dynamic half.
- `load_to` — copy a named section out of the authoritative configuration into a scratch
  file, which is how the verifier reconstructs the dynamic half from names the client
  supplied.
- `dump` — the client-side counterpart, which asks a live game object to write out its
  current parameters. Declared here and **not built into this tool**; see
  [`mp_config_sections.cpp`](mp_config_sections.cpp.md).
- `active_params_section` — the name of the dump section that lists the dynamic sections.
- `IAnticheatDumpable` — the interface a live object implements to be dumpable. Referenced
  only by the client-side half.
