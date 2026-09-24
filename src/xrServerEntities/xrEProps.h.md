# src/xrServerEntities/xrEProps.h

> The property-factory interface every record's editor contribution is written against, plus the path helpers that build a property's key.

**Needs** — [`PropertiesListTypes.h`](PropertiesListTypes.h.md) · [`ItemListTypes.h`](ItemListTypes.h.md) · [`gametype_chooser.h`](gametype_chooser.h.md)
**Used by** — [`PropertiesListHelper.h`](PropertiesListHelper.h.md) · [`gametype_chooser.cpp`](gametype_chooser.cpp.md) · [`script_properties_list_helper.h`](script_properties_list_helper.h.md) · [`script_properties_list_helper_script.cpp`](script_properties_list_helper_script.cpp.md) · [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md)
**Tier floor** — T2: an interface plus string assembly; the handles it manufactures point at record fields, which is what keeps it out of T3.

## Purpose

Every record in this directory contributes rows to a property list so the level editor can
edit its fields in place. It does so by calling a **factory** — one call per field, naming
the field's key, its address and its constraints — and this file is that factory's
interface.

It lives in this directory, not in an editor module, for a reason worth stating once: the
records are compiled into the game *and* into the editor, so the vocabulary they call has to
be shared. The interface is declared here and implemented on the other side of a dynamic
library boundary, which is resolved at run time by name — see
[`script_properties_list_helper_script.cpp`](script_properties_list_helper_script.cpp.md).
In a game build the implementation is never loaded and the whole surface is compiled out.

## Property keys are paths

A property's key is a **delimiter-joined path** — `"General"`, `"Visual\Model"` — and the
list renders it as a tree by splitting on the delimiter. Rather than have every record
concatenate by hand, the file supplies a small helper that joins one, two or three prefix
segments to a leaf name.

```text
FUNCTION prepare_key(prefixes..., leaf) -> text
  REQUIRE leaf is present          # a property with no leaf name is a programming error
  result = ""
  FOR EACH p IN prefixes
    IF p is non-empty
      result = result + p + DELIMITER    # an empty prefix contributes nothing, not a bare delimiter
  RETURN result + leaf
```

**Invariants** — an empty prefix must collapse away entirely. That is the whole subtlety:
records build their keys with a group name that is sometimes absent, and a bare leading
delimiter would create an unnamed tree node. The leaf is required and its absence is a hard
failure, because a keyless property cannot be found again.

The result is interned, because a property list of a few hundred rows re-keys itself on
every selection change and the same few dozen paths recur.

## `IPropHelper` — the factory

**Contract** — manufactures one property handle per call, appends it to a caller-supplied
row list, and answers it so the caller can attach behaviour. Every creator takes the row
list, the key, and **the address of the field in the record**; the handle writes through
that address, which is what makes editing in place possible and what forbids moving a record
while its property list is open.

The creators, by family:

- **Scalars** — the signed and unsigned integers at 8, 16 and 32 bits, and a float, each
  with a minimum, a maximum and a step. The bounds are the field's real domain, not a UI
  hint: the editor clamps to them.
- **Boolean**, at machine-word width (see
  [`script_value_wrapper.h`](script_value_wrapper.h.md) for why the width matters).
- **Composites** — a three-component vector with bounds and decimal places, a packed colour,
  a four-channel float colour, and a vector edited as a colour.
- **Text** — an interned string, a fixed-capacity character buffer (marked obsolete but
  still used), and a name field that validates against its owner's siblings for uniqueness.
- **Bit fields** at 8, 16 and 32 bits: a mask selects which bit this row edits, and two
  optional captions name the clear and set states, so one flag word yields several rows.
- **Enumerations** — a value constrained to a vocabulary, at 8, 16 and 32 bits, in both the
  build-time-vocabulary and run-time-vocabulary forms (see
  [`script_token_list.h`](script_token_list.h.md) and
  [`script_rtoken_list.h`](script_rtoken_list.h.md)), plus a plain list of interned strings.
- **Chooser** — a value picked from a browser over the game's own data, parameterized by
  *what kind* of data: a sound, a reverb preset, a library object, a shader of either
  compiler, a particle effect or system, a texture, an entity class, a spawnable item, a
  light animation, a visual model, a skeleton's animations or bones, a material, a game
  animation, a motion — or a caller-supplied fill. This enumeration is the editor's map of
  the game's asset kinds and is the most useful thing in the file.
- **Domain-specific** — an angle (stored in radians, shown in degrees), a three-component
  angle, a time of day bounded to a day's seconds, a sound waveform, a keyboard shortcut,
  and the game-mode set from [`gametype_chooser.h`](gametype_chooser.h.md).
- **Decoration** — a caption row (a label with no value), a canvas row of a given height,
  and a button row with a click handler.

**Invariants** — a property handle outlives neither the record whose field it addresses nor
the row list it was appended to. `FindItem` retrieves a previously created row by key and
optionally by type, which is how a record adjusts a row it created earlier — for example
disabling one field because another changed.

## the shared edit behaviours

**Contract** — the interface also publishes a small set of *ready-made* before-edit,
after-edit and draw handlers for the three cases every record needs identically: a vector, a
float, and an interned name. A record attaches one of these rather than writing its own, so
that a vector reads the same way in every property list in the editor.

The before/after pair is the validation protocol: before-edit transforms the stored value
into the form the user sees, after-edit transforms it back and **answers whether to accept**
— rejecting leaves the field untouched. Draw renders the value as text for the collapsed
row.

## `IListHelper` — the item list

**Contract** — the much smaller sibling for the editor's *object list* (a tree of folders
and objects, not a property grid). Creates a row with a key, a kind (folder or object), row
flags and an opaque payload; finds one by key; and validates a rename.

**Notes** — an item's kind distinguishes a folder from an object, with a third value meaning
"invalid", used as the answer to a failed classification rather than as a stored state.

## Notes

**The event types are declared as delegates over a small fixed set of signatures** — item
focused, list closed, item renamed, item removed, removal finished, modified. These are
plain callbacks; the mechanism is incidental, but the *set* is not: it is the editor's
complete notification vocabulary for a list, and a rebuild needs all six.

**A block of this file compiles only under the original editor's compiler** and reaches into
that framework's window types for a "keep this window on screen" helper. It is dead in every
build this project produces and carries no decision.

**The factory is reached through a free function**, not passed in. That is a global, and it
is the reason a record can contribute properties without being handed an editor — but it
also means the first call must succeed in loading the implementation or the process fails
hard. A rebuild that passes the factory down explicitly loses nothing.
