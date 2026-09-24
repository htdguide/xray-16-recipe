# src/xrGame/inventory_upgrade_property.cpp

> One row of the upgrade screen's parameter display: an icon, a label, and a script function that turns a raw item parameter into the string shown beside it.

**Needs** — [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: reads configuration and calls into the script virtual machine

## Purpose

An upgrade changes numbers; the player needs to be told what changed in words. A property
is the authored answer: it names a family of item parameters ("accuracy", "handling"),
carries the icon and label the screen draws for that family, and delegates the actual
"0.87 → *Good*" conversion to a script function so that the wording can be changed without
touching the engine.

It is a separate file from the upgrade node because properties are declared once globally
and referenced by name from many upgrades, rather than owned by any one of them.

## State

```text
RECORD Property
  id             : text          # the configuration section, also the key in the manager
  name           : text          # already localized at construction
  icon           : text
  color          : int (32-bit, packed RGBA)   # default opaque white
  script_call    : bound script function taking (parameter_value, property_id) -> text
  parameters     : list<text>    # the item parameter names this property covers
```

**Invariant** — the display name is localized *once, at construction*, not per draw. The
language cannot change without restarting, so this is safe and it keeps the string-table
lookup off the screen's redraw path.

**Invariant** — the bound script function's second argument is permanently the property's
own identity, set at construction and never changed; only the first argument varies per
call. One script function can therefore serve many properties and branch on which it was
called for.

## `construct`

**Contract** — reads the descriptor from the configuration section named by the property
identifier and binds its script function. A missing section is a hard failure; so is a
`functor` key naming a script function the virtual machine cannot resolve. The colour is
optional and defaults to opaque white; the name, icon, functor and parameter list are
required. Allocates; blocks on the script virtual machine.

**The function is called once immediately after binding, with an empty parameter**, purely
to prove it runs. The result is discarded. This turns a broken script — a syntax error, a
wrong arity, a nil global — into a failure at load time with the property's name in the
message, instead of a failure mid-frame when the player opens the upgrade screen. Keep this
smoke call; it is the only thing that makes the binding's failure mode legible.

```text
FUNCTION construct(property_id, manager)
  id = property_id
  FAIL WITH "no such property section" IF the section is absent

  name  = localize(section.name)
  icon  = section.icon
  color = section.color IF PRESENT ELSE opaque white

  script_call = bind(section.functor)
  FAIL WITH "cannot resolve functor" IF the binding failed
  script_call.second_argument = id
  script_call("")                     # smoke test; result discarded

  parameters = split(section.params)  # comma-separated item parameter names
```

## `run_functor`

**Contract** — calls the bound script function with one item parameter name and returns
the string it produced. Reports failure — leaving the output empty — when the script
returns nothing or the empty string, which is the script's way of saying "this property has
nothing to say about this parameter". Not an error; the screen simply omits the row. The
result is copied into a caller-supplied fixed buffer, so a script returning something
longer than the buffer is truncated rather than rejected; a rebuild should return an owned
string and drop the limit.

```text
FUNCTION run_functor(parameter) -> optional<text>
  script_call.first_argument = parameter
  out = script_call()
  IF out is none OR empty THEN RETURN none
  RETURN out
```
