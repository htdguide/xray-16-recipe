# src/dummy — the dependency probe

Not a chapter of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order): it ships
nothing and nothing depends on it. One source file, six lines, all of them comment.

## What this module is responsible for

Verifying, by hand, that a module depends on the layer below it and not sideways.

The engine is built as a dozen separately linkable modules, and the intended dependency
direction is the build order in the preface. Nothing checks it. The method recorded here
is to compile a target whose entire content is **one header**, then list the symbols the
resulting object file references. Every symbol in that list is something the header drags
in. Anything that should not be reachable from that header is a leak — nearly always a
header that names a whole subsystem to get at one convenience declaration, and nearly
always the reason a module later turns out to be impossible to extract.

## Where it sits

Nowhere in the runtime. It links against the core only, is excluded from every shipping
artifact, and is built on demand by a person who is investigating a specific header.

## The load-bearing idea

**Dependency direction in this project is checked by a human reading a symbol dump, not by
a tool.** That is worth knowing for two reasons. First, it means the direction is only as
clean as the last time somebody looked — a rebuilder should not assume the module graph in
the preface is enforced anywhere. Second, it tells a rebuilder what to replace it with: a
tier with a real module system enforces this at compile time and should delete this
directory outright; a tier without one should keep the practice and automate it.

## The twins

| File | Role |
|---|---|
| [`00dummy_tester.cpp`](00dummy_tester.cpp.md) | The probe: compiles one named header alone so its pulled-in symbols can be listed |
