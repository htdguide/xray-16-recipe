# src/xrServerEntities/xrServer_Factory.cpp

> Turns a configuration section name into a new entity record.

**Needs** — [`object_factory.h`](object_factory.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two lookups.

## Purpose

The one entry point through which the level loader, the save loader and the script layer all
create records. It embodies the identity rule: **an entity is (class identifier, section)**.
The section is the caller's argument; the class identifier is read from that section's
`class` key; the registry maps the identifier to a constructor and the constructor is handed
the section back.

## `F_entity_Create`

**Contract** — given a section name, resolve its class key to a class identifier, ask the
class registry for a server-side record of that class built from that section, and return
it. A second form takes a flag that downgrades an unknown class from a hard failure to a
null result — used by tools that must open a level authored against a different
configuration set without dying.

**Notes** — a rebuild may collapse this into the registry itself. What must survive is that
the class tag is *data*, read per section, not a field of the record and not a compile-time
choice at the call site.
