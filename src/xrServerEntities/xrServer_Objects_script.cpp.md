# src/xrServerEntities/xrServer_Objects_script.cpp

> Exports the abstract record and its four structural facets to scripts, with the serialization methods overridable so a script class can be a record.

**Needs** — [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [`script_ini_file.h`](script_ini_file.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The root of the record export. Everything a script can do with *any* record begins here: read
its identity, move it, learn its class, read its per-entity configuration, and — the part
that matters — override any of its four serializations.

The exported surface is frozen by conformance criterion 10; every name below appears in
shipped scripts.

## `cse_abstract`

**Contract** — the abstract record. Read-only: the entity identity, the parent's identity,
the script version the record was created against. Read/write: position and orientation.
Methods: the section name, the display name, the class number, the per-entity configuration,
and the four serializations.

**Invariants** — **position and orientation are directly writable from script**, with no
notification to anything. A script moving a record that is currently online moves the record
and not the live object; reconciling them is the caller's problem. This is the sharpest edge
in the whole script surface.

**The class number exposed here is the registry's small integer, not the eight-character
tag** — see the chapter opener. A script comparing against `clsid.<name>` is comparing
against this.

**The per-entity configuration is handed out as the script configuration type**, so a script
reads an entity's authored overrides with the same surface it reads any configuration file —
see [`script_ini_file.h`](script_ini_file.h.md).

**Notes** — the section name and the display name are exported through free functions rather
than methods, because the record's own accessors have shapes the binding layer cannot take
directly. No behavioural difference.

**The constructor is not exported.** A script cannot build a bare abstract record; it must
register a class (see [`object_item_script.cpp`](object_item_script.cpp.md)) and let the
factory build it. That is the guard rail keeping a script from creating a record the registry
does not know about.

## Overriding a serialization

**Contract** — each of the four serializations is exported in the two-sided form: the native
implementation, and a **fallback** used when the engine calls the method on a script subclass
that did not override it.

**Invariants** — **the fallback logs a "pure virtual method called" message and does
nothing.** The record is left half-serialized and the stream misaligned; every record after
it in the same save reads garbage. The line that would have called the native implementation
instead is present and commented out in all four cases.

That is the correct failure for a *pure* method — there is no native implementation to fall
back to at this level — but it makes "a script class that forgot to implement save write"
into a corrupted save rather than an error. A rebuild should fail the save, loudly, at the
point of the missing override.

## `script_server_object_version`

**Contract** — a free function answering the script-facing record format version. A mod
compares it to decide which fields a record it did not write will have.

## `iserializable` / `ipure_server_object` / `cpure_server_object`

**Contract** — the three levels below the abstract record, registered as opaque types with
no members. They exist so the binding layer knows the inheritance chain; nothing calls them
from script.

## `cse_shape` / `cse_visual` / `cse_motion`

**Contract** — the three structural facets, registered as opaque types. A script can hold one
and pass it back, but the facets' own surfaces — the shape's volumes, the visual's model
name, the motion's path — are deliberately **not** exported.

**Notes** — the constructors are present and commented out in each. A script may not
construct a bare facet; facets exist only as part of a record.

## `cse_spectator` / `cse_temporary`

**Contract** — the two records that are not alife objects at all: the multiplayer spectator
camera and the transient record. Both exported at the abstract level, so a script class may
subclass either and override its serialization.
