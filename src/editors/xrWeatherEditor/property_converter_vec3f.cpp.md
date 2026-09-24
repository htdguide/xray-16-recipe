# src/editors/xrWeatherEditor/property_converter_vec3f.cpp

> Makes a vector row readable as one line of three numbers and writable the same way, and forces its child rows into axis order.

**Needs** — [`property_converter_vec3f.hpp`](property_converter_vec3f.hpp.md) · [`property_vec3f.hpp`](property_vec3f.hpp.md) · [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_converter_float.hpp`](property_converter_float.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure presentation; the vector crosses by value through the row's own interface

## Purpose

The presentation half of a vector property. [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md) builds the three component rows; this file decides what the *parent* row shows, what a user may type into it, and in what order the children appear.

## State

Stateless — one instance serves every vector row.

## `GetProperties`

**Contract** — Returns the vector's child rows sorted into `x`, `y`, `z` order. The set must contain exactly three.

**Notes** — Sorting by an explicit name list is the point. The grid's default ordering for child rows is alphabetical, which for these three happens to be right and for any other axis naming — `pitch`, `yaw`, `roll`; `r`, `g`, `b` — would be wrong. Stating the order explicitly means the presentation does not depend on an accident of spelling. The exact-count requirement is how a component set that has drifted out of step with this ordering is caught at once rather than silently mis-ordered.

## `GetPropertiesSupported`

**Contract** — Always true: a vector row always expands.

## `ConvertTo`

**Contract** — Two destinations. To text: reads the row's raw vector and formats the three components separated by single spaces, each through the same real-number formatter every scalar real row uses. To the presentation vector: reads the raw vector and repackages it field by field. Any other destination falls through to the grid's default.

```text
FUNCTION ConvertTo(row_container, destination)
  vector = row_container.owner.get_value_raw()        # the engine's current value
  IF destination IS text THEN
    RETURN format_real(vector.x) + " " +
           format_real(vector.y) + " " +
           format_real(vector.z)
  IF destination IS presentation_vector THEN
    RETURN presentation_vector(vector.x, vector.y, vector.z)
  RETURN default
```

**Notes** — Formatting through the shared real formatter, rather than the runtime's default, is what keeps a vector's components displayed to the same precision as the scalar reals beside them in the grid — and, more importantly, keeps the text this converter *writes* parseable by the text it *reads*.

## `CanConvertTo`

**Contract** — Admits the presentation vector; explicitly **refuses** text. This contradicts `ConvertTo`, which formats text perfectly well, and the contradiction is load-bearing: the grid consults this answer to decide whether the parent row is editable as a line of text, and the intended answer is no — the user edits the three children or opens them. The text path remains implemented because other parts of the grid format a value for display without asking first. A rebuild should separate "can this be displayed as text" from "can this be edited as text" rather than reproduce the inconsistency.

## `CanConvertFrom` · `ConvertFrom`

**Contract** — Accepts text. Parses three reals separated by single spaces: everything before the first space, everything between the first and second, everything after the second. Fails with a message naming the offending text when any of the three does not parse or the separators are missing.

```text
FUNCTION ConvertFrom(text) -> result<presentation_vector, Error>
  # exactly two single-space separators; no tolerance for padding or commas
  split text at first space  -> x_text, rest
  split rest at first space  -> y_text, z_text
  IF any part fails to parse as real THEN
    FAIL WITH "cannot convert <text> to a vector"
  RETURN presentation_vector(x, y, z)
```

**Invariants** — The format this parses is exactly the format `ConvertTo` produces. The two must be changed together; nothing enforces it.

**Notes** — The parser is deliberately unforgiving about the separator, and the original is aware the path is nearly unreachable given the refusal above. What a rebuild should take from it is the round-trip requirement, not the strictness: a value the tool displays must be a value the tool can read back, or copy-and-paste between rows silently corrupts the document.
