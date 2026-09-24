# src/xrServerEntities/PropertiesListTypes.h

> The editor's property model: a typed, editable handle onto a field of a live entity record, with multi-selection, revert, and per-type display and validation.

**Needs** — [`gametype_chooser.h`](gametype_chooser.h.md) · [`xrEngine/WaveForm.h`](../xrEngine/WaveForm.h.md) · [`xrCore/xr_shortcut.h`](../xrCore/xr_shortcut.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md)
**Used by** — [`ItemListTypes.h`](ItemListTypes.h.md) · [`PropertiesListHelper.h`](PropertiesListHelper.h.md) · [`script_properties_list_helper_script.cpp`](script_properties_list_helper_script.cpp.md) · [`xrEProps.h`](xrEProps.h.md)
**Tier floor** — T2: a widget data model with type erasure; no byte layout, no device.

## Purpose

Entity records are edited by hand in the level editor, and this file is how. A **property**
is a named row; a **value** is a typed pointer *into a record's actual field*, so editing
the row writes the record directly. A property holds several values — one per selected
entity — which is what makes editing twenty doors at once work, and what makes the "(mixed)"
display and the revert-to-original behaviour necessary.

This is the surface every entity record's `fill_properties` is written against, so it is the
reason that method exists on every class in this chapter. It is compiled out of the shipping
build entirely.

## State

```text
RECORD Value                        # one editable binding
  owner   : Property
  target  : pointer to a field of one record
  initial : a copy of that field, taken when the binding was made
  on_change, on_before_edit, on_after_edit : callbacks

RECORD Property
  key      : text                   # the displayed path; backslashes make a tree
  kind     : PropertyKind
  values   : list<Value>            # one per selected entity
  flags    : disabled, has_checkbox, checked, mixed, draw_thumbnail, sorted
  colours, rectangle, callbacks     # presentation
```

**Invariants**

- A value's *target* is a raw pointer into a live record. Nothing here owns the record, and
  the whole property list is rebuilt whenever the selection changes — a stale property list
  points at freed records.
- The **mixed** flag is recomputed on every append, every apply and on demand: it is set
  when any two of the property's values disagree. A mixed property displays "(mixed)" and
  editing it writes one value to all of them.
- The **initial** copy is taken at binding time and is what revert restores. It is a copy of
  the field, not of the record, so reverting one property does not disturb the others.
- Applying a value returns whether anything actually changed, and the change callback fires
  only for the values that did. That is what keeps an editor's dirty flag honest when the
  user re-types the value that was already there.

## `PropertyKind`

```text
ENUM PropertyKind
  undefined,
  caption, shortcut, button, choose,
  numeric,        # all integer widths and float share one kind
  boolean, flag, vector, token, rtoken, rlist,
  colour, float_colour, vector_colour,
  shared_text, string_text,
  wave, canvas, time,
  fixed_text, fixed_list, token_with_names, texture, game_type
```

**Notes** — the kind is what the editor's renderer switches on; the *width* of a numeric
property is carried by the value, not the kind, which is why one `numeric` covers every
integer size and the float. A rebuild can merge more of these (the three colour kinds differ
only in how they present the same data) or fewer; what must survive is that the kind decides
the editor widget and the value decides the field.

## the value families

Each family is a Value specialized to one field type, and each contributes one decision:

- **caption** — display-only; equality against another caption is by text.
- **canvas** — a host-drawn region; equality is delegated to a callback because the host is
  the only thing that knows whether two drawings are the same.
- **button** — no field at all; the "value" is a list of labels and the payload is a click.
- **shortcut** — a key binding; equality is the binding's own similarity test, because
  modifiers compare structurally.
- **shared text / string text / fixed text** — three string representations (interned,
  owned, and a caller-supplied fixed buffer). The fixed-buffer one exists for records that
  still hold character arrays and is marked obsolete in place.
- **choose** — a shared-text value plus a *mode* and a *path*, which together tell the
  editor what kind of asset picker to open (a visual, a sound, a level, a section). This is
  how a record field that names a game asset gets a browse button instead of a text box.
- **numeric** — a value plus minimum, maximum, step and decimal places. Clamping happens on
  apply, so a record can never be edited out of range.
- **vector** — the same for a three-component value, clamped per component.
- **flag** — a bit mask value plus the mask this row edits and the two labels for its states.
  One property edits one bit of one field.
- **token / token-with-names** — an integer constrained to a named set. The distinction
  between the two is where the names come from: a token table shared by reference, or a
  table owned by the property.
- **rtoken** — the same idea with interned names, used where the set is data-driven.
- **rlist / fixed list** — a string constrained to a list of allowed strings. Both carry the
  rule that **two selections with different allowed lists cannot be edited together**: the
  equality test detects it and disables the property rather than offering a list that is
  wrong for half the selection. That is the one genuinely subtle behaviour in this file.
- **wave** — an authored curve, compared by its own similarity test.
- **game type** — the mode mask from [`gametype_chooser.h`](gametype_chooser.h.md),
  compared as a whole.

## Notes

**The whole file is packed to byte alignment.** That is a relic of these types once crossing
a compiled-module boundary into a tool built by a different compiler, and it is meaningless
now; a rebuild ignores it.

**Equality is declared per type, by operator, at namespace scope** for the types that do not
have one — vectors, colours, shortcuts, waves, mode masks. The important part is that
equality is *approximate* for the floating-point ones (a similarity test, not a bit
comparison), because otherwise two entities that a designer placed at the same position
would show as "(mixed)" over a rounding difference.

**Property keys are paths.** A backslash in the key makes a tree node in the editor, which
is why every record's property fill starts by composing a prefix. See the key-composition
helpers in [`xrEProps.h`](xrEProps.h.md).
