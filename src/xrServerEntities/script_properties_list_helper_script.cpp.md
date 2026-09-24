# src/xrServerEntities/script_properties_list_helper_script.cpp

> Exports the editor's property factory to scripts, and resolves the factory's implementation out of a separately loaded module on first use.

**Needs** — [`script_properties_list_helper.h`](script_properties_list_helper.h.md) · [`xrEProps.h`](xrEProps.h.md) · [`PropertiesListTypes.h`](PropertiesListTypes.h.md) · [`script_token_list.h`](script_token_list.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it resolves a symbol out of a dynamically loaded module by name.

## Purpose

Two jobs that happen to share a file because they share a trigger.

The first is the **late binding of the property factory**. Records call the factory through
a free function; the factory itself lives in a separate module that only the editor ships.
This file resolves it — load the module, find the entry point by name, cache both — on the
first call and never again.

The second is the **script export** of the script-facing property factory, so that a record
declared in script can contribute editor rows for fields that live in a script table rather
than at a record address.

## the factory's late binding

**Contract** — answers the property factory. The first call loads the module and looks up
its entry point; every later call returns the cached result. A failure to find either is
**logged and then asserted**, so a game build that reaches this function dies rather than
proceeding without an editor.

```text
FUNCTION property_factory() -> Factory
  IF first call
    first call = FALSE
    module = LOAD MODULE "xrEPropsB"
    IF module not loaded
      LOG "cannot find library"        # and fall through to the failure below
    ELSE
      entry = FIND SYMBOL "PHelper" IN module
      IF entry is none
        LOG "cannot find entry point"
      ELSE
        script_factory = NEW ScriptPropertyFactory   # only now does the script side exist
  REQUIRE entry is present, "cannot find entry point or library"
  RETURN entry()
```

**Invariants** — the once-only flag is checked without synchronization. The first call
happens during single-threaded startup in every build that reaches it, which is what makes
that safe; a rebuild whose editor tooling starts concurrently must guard it.

**Notes** — the script-facing factory is constructed **only if the module loaded**, and
everything below keys off that. The consequence is visible to scripts: in a game build the
accessor answers nothing, and the export says so explicitly rather than failing.

**The failure message names the same string twice** where it means to name the function and
the library — a copy-paste defect in the diagnostic, harmless but worth not reproducing.

## the exported value types

Every property-handle type from [`PropertiesListTypes.h`](PropertiesListTypes.h.md) is
registered as an opaque script type with no methods: the base handle, the row vector,
caption, canvas, button, chooser, the signed and unsigned integers at each width, float,
boolean, vector, colour, text, the three bit-field widths, the three enumeration widths, and
the string list. A script never manipulates a handle; it only receives one from a create
call and holds it. Registering them at all is what lets the binding layer type-check those
returns.

**Notes** — the float handle is registered under the **same script name as the 32-bit
unsigned handle**. That is a defect: the later registration wins, so one of the two types is
unreachable by name from script. Nothing in shipped scripts names either, which is why it
has survived. A rebuild should give it its own name.

**The interned-vocabulary handles are registered but commented out**, along with their three
create calls — see [`script_rtoken_list.h`](script_rtoken_list.h.md) for the type they would
have exposed. Scripts use the run-time string list instead.

## the chooser kinds

A separate script enumeration publishes the asset kinds a chooser row can browse: custom,
sound source, reverb preset, library object, engine shader, compiler shader, particle effect,
particle system, texture, entity class, spawnable item, light animation, visual model,
skeleton animations, skeleton bones, material, game animation, game motion. This is the
editor's map of the game's asset kinds, and a script row that wants a texture picker names
it here.

## the exported factory

**Contract** — one method per property kind, each taking the row list, the key, **the script
table** and **the field name within it** where a native call would take a field address —
that substitution is the entire difference and the reason this type exists. Each kind is
exported in several arities, from "just the field" up to "field plus bounds plus step plus
decimal places", because the binding layer resolves overloads by argument count and a script
should not have to pass defaults.

The kinds: caption, canvas, button, chooser (four arities), the integers at 16 and 32 bits
signed and 8, 16 and 32 unsigned (four arities each), float (five), boolean, vector (five),
the three bit-field widths (four each, with the mask always required), the three
enumeration widths, a string list, three colour forms, text, time (three), angle (five) and
three-component angle (five).

Plus the three shared edit behaviours from [`xrEProps.h`](xrEProps.h.md) — vector, float and
name — exposed so a script row can attach the same validation a native row gets. The float
and name variants are declared to write their edited value back through an output parameter
rather than a reference, because script has no references.

**Notes** — **the 8-bit signed creator is commented out** while the other widths are
present. No reason is recorded and none is discoverable; the field it would have edited
exists in records.

## `properties_helper`

**Contract** — a free function answering the script-facing factory. In a build where the
editor module never loaded it **logs a script error and answers nothing**, which is a
deliberate softening: a mod that asks for the property factory outside the editor gets a
diagnosable failure in its own log rather than a dead process.
