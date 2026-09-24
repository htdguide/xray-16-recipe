# src/xrGame/anticheat_dumpable_object.h

> The interface an object implements to have its live tuning values dumped for server-side comparison against the shipped configuration.

**Needs** — [`configs_dumper.h`](configs_dumper.h.md) · [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md)
**Used by** — [`ShootingObject.h`](ShootingObject.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`actor_mp_client.h`](actor_mp_client.h.md) · [`configs_dumper.cpp`](configs_dumper.cpp.md) · [`mp_config_sections.cpp`](mp_config_sections.cpp.md) · [`mp_config_sections.h`](mp_config_sections.h.md)
**Tier floor** — T3: two method declarations

## Purpose

Multiplayer's cheat problem is not primarily code injection; it is a player editing the
text configuration that gives a weapon its damage, its recoil or its magazine size, and
then joining a server. The defence is to make each client dump the values it is *actually
using* into a configuration image, hash it, and have the server compare that against its
own — see [`configs_dumper.cpp`](configs_dumper.cpp.md) for the dump and
[`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) for the comparison.

This header is the one thing those two need from every participating class: the ability to
write its live values out as a named configuration section. It is deliberately the smallest
possible interface, because it must be implemented by a long tail of weapon, ammunition and
outfit classes that otherwise have nothing in common.

## State

`Stateless.` A pure interface.

## `IAnticheatDumpable`

**Contract** — an implementor must be able to write its effective parameters into a
configuration image under a caller-chosen section name, and may name the configuration
section it was built from.

```text
INTERFACE IAnticheatDumpable
  FUNCTION dump_active_params(section_name : text, destination : config image)
      # REQUIRED. Write the values this object is running with -- not the values
      # its section declares. The two differ once anything has modified them,
      # and the difference is exactly what is being detected.

  FUNCTION anticheat_section_name() -> text
      # OPTIONAL, defaults to empty. The configuration section this object
      # was built from. Empty means "not a participant"; the dumper skips it.
```

**Invariants** — the dump must be deterministic: the same object in the same state on two
machines must produce byte-identical output, because the whole scheme compares hashes. That
rules out iterating an unordered container, writing a floating-point value with a
locale-dependent or shortest-round-trip formatter, or including anything that varies with
machine state. This is the single most important constraint in the file and it is written
down nowhere in the original.

**Notes** — the default empty name is what lets the dumper walk every object of a class
family and skip the ones that do not opt in, without a separate registry.
