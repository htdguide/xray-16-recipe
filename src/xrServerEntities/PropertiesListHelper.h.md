# src/xrServerEntities/PropertiesListHelper.h

> The concrete factory for editor properties: one call per property kind, each binding a field of a record to a typed, range-limited editor row.

**Needs** — [`PropertiesListTypes.h`](PropertiesListTypes.h.md) · [`xrEProps.h`](xrEProps.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

This is the implementation side of the property-helper interface declared in
[`xrEProps.h`](xrEProps.h.md) — the thing every entity record's `fill_properties` calls.
The interface exists so that the records, which are compiled into both the game and the
tools, can build property rows without knowing what an editor is; this class is the filling
the tools supply.

It is a header with no companion source in this directory: the bodies live in the editor
module. What is contract here is the *shape* of the surface, which is where the substance
already is — in [`PropertiesListTypes.h`](PropertiesListTypes.h.md).

## The surface

- `find(items, key, kind)` — locate an existing row, so a record can amend a property its
  base class already added.
- one `create…` per kind: caption, canvas, button, choose, each signed and unsigned integer
  width, float, boolean, vector, the three flag widths, the three token widths, the three
  interned-token widths, list, colour in three forms, text in three forms, wave, time,
  shortcut, angle, three-component angle, name, game type — plus four marked obsolete that
  take fixed character buffers.
- the predefined edit hooks: a vector and a float edit that interpret their input in
  **degrees while storing radians**, and a name edit that validates and normalizes an
  entity's name. These are shared because more than one record needs the same conversion and
  the same validation, and duplicating them would let two records disagree about what a
  legal name is.

## Notes

**Angles are the reason for the hooks.** Records store orientation in radians because that
is what the maths layer uses; designers type degrees. The conversion is attached to the
property as a before/after-edit pair rather than being done by the record, so that the
record's field stays canonical and only the editor sees degrees.

**The integer widths are enumerated rather than generic** — a separate call for each of the
six — because each carries its own default minimum, maximum and step. The defaults are the
same shape everywhere (`0` to `100`, step `1`), which is a convenience default and not a
constraint on any record.
