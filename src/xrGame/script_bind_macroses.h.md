# src/xrGame/script_bind_macroses.h

> The generator for the game object's script facade: every exported method is a downcast to a concrete class, an error if the cast fails, and a forwarded call.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_game_object.h`](script_game_object.h.md)
**Tier floor** — T3: a code-generation convenience; the *pattern* it generates is the real content

## Purpose

The [game object](../../GLOSSARY.md) is one script-visible handle standing in front of a deep
class hierarchy — actors, stalkers, monsters, weapons, artefacts, cars, doors. A script calls
`object:condition()` on whatever it is holding, and only some of those things *have* a
condition. This file holds the rule for what happens in every one of those cases, expressed
once as a generator rather than typed out several hundred times.

The generator itself is incidental — a rebuild in a language with reflection, generics or
macros will express it differently, or write it out. **What must survive is the uniform
behaviour of every generated method**, described below, because the shipped scripts depend on
it: they call methods on objects that may not support them and expect to keep running.

## State

`Stateless.` It generates method bodies.

## The generated shape

Every exported method of the game object facade has exactly this body:

```text
FUNCTION <exported name>(args...) -> <result type>
  concrete = downcast(this.client_object TO <the class that has this method>)
  IF concrete IS none THEN
    log script error "<class> : cannot access class member <name>!"
    RETURN <the declared fallback>        # or return nothing, for a void method
  RETURN <result type>(concrete.<real method>(converted args...))
```

**Invariants**

- **A wrong-type call is a logged error and a fallback value, never a failure.** This is the
  load-bearing decision in the file, and it is a decision about the whole script layer: a
  script calling a weapon method on a door keeps running. Shipped scripts rely on this —
  generic handlers walk heterogeneous object lists and try methods speculatively. A rebuild
  that raises instead will not run the shipped scripts, which is conformance criterion 10.
- **The fallback value is declared per method, not derived.** Each binding names what a
  failed cast answers, which is how a method can answer zero, or an empty handle, or nothing
  at all, according to what a script can sensibly do with it. A rebuild must keep the
  fallback a per-method choice rather than defaulting every type to its zero.
- **The error message names the class the cast wanted and the method asked for**, not the
  class the object actually is. That is the wrong half for diagnosis — knowing what it *is*
  is what tells a modder their mistake — and a rebuild should report both.
- The result and each argument are **converted explicitly** at the boundary rather than
  passed through. That is what lets the script surface expose simpler types than the engine
  uses: an enumeration as an integer, a handle as an identifier.

## The generated families

Six shapes are generated, and the set is worth naming because it bounds what the facade can
express:

```text
member read          -> value           # forwards a field, converted
call, no args        -> nothing
call, no args        -> value
call, one arg        -> value
call, one arg        -> nothing
call, two args       -> nothing
call, three args     -> nothing
```

**Notes**

- There is no shape for a call with two or three arguments that *returns* a value, and none
  for four or more arguments at all. That is a real ceiling on the facade: a method needing
  more is written out by hand elsewhere or is not exported. A rebuild without the ceiling
  should not be surprised to find the shipped surface shaped around it.
- The conversion of each argument and of the result is specified independently of the
  declared script-side type, so a binding can accept an integer from script and pass an
  enumeration to the engine. This separation is the reason the generator takes as many
  parameters as it does.
