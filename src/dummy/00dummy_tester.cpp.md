# src/dummy/00dummy_tester.cpp

> A build target that exists only to compile one header in isolation, so that the symbols that header pulls in can be listed and the real dependency graph checked against the intended one.

**Needs** — whichever single header is under test; in the committed state, [`xrGame/stalker_movement_manager_smart_cover.h`](../xrGame/stalker_movement_manager_smart_cover.h.md)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T4: it is a build-hygiene probe, not program logic. It produces no runnable artifact.

## Purpose

The engine's modules are built as separate linkable units and the intent is that each one
depends on the layer below it and not sideways. Nothing enforces that. The way it got
checked was crude and effective: build an object file whose *entire* content is one
header, then list the symbols that object file references. Anything in the list that should
not be reachable from that header is a dependency leak — usually a header that pulls in a
whole subsystem because of one convenience declaration.

The decision recorded here is not the file. It is the **practice**: dependency direction in
this codebase is verified by inspecting what one compiled header drags in, and the header
named in the file is whichever one was last being investigated. A rebuild in a tier with a
real module system gets this enforcement for free and should delete the directory. A
rebuild in a tier without one should keep the practice and automate it, because the leaks
this finds are exactly the ones that make a module impossible to extract later.

## State

`Stateless.`

## Exported units

None. The target deliberately produces no code — the single allocation that would have
given it a reason to exist is commented out, and the file is excluded from every shipping
artifact. Its output is a symbol listing read by a person, not a program.

**Notes** — the file name begins with digits so that it sorts first in a directory listing;
that is cosmetic. The header under test is a working note left in place, not a decision: it
happens to be the movement manager for the creature behaviour that navigates authored cover
positions, and any other header would serve the same purpose.
