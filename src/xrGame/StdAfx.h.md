# src/xrGame/StdAfx.h

> The game module's precompiled prelude: a build artifact, not a design decision.

**Needs** — _(nothing a rebuild must reproduce)_
**Used by** — [`StdAfx.cpp`](StdAfx.cpp.md)
**Tier floor** — T4: a build input

## Purpose

This file names roughly two hundred declarations that the game module's sources use so
often that resolving them once and reusing the result is cheaper than resolving them per
source file. Nothing in it is a decision about the game. It solves a compilation-time
problem that most rebuild targets do not have: in a language with a module system, or with
a compilation model that caches resolved declarations itself, the file has no counterpart
and should not be recreated.

Two things in it are worth carrying forward, and neither is the list:

**It is a map of the module's weight.** Each entry is annotated with how many of the
module's roughly eleven hundred sources need it, so the file doubles as a frequency census
of the game layer's own vocabulary. The heaviest entries — the object base and its
interfaces, the server entity records, the animation model, the level, the physics shell,
the script engine and the widget toolkit — are exactly the things the chapter's README
names as the ideas a reader needs before the twins make sense. A rebuilder wanting to know
what the game module is *made of* can read this file as a summary.

**It records a known structural problem.** The file's own leading note says the module
should be split into a core part, a user-interface part, an artificial-intelligence part
and a script part, each with its own prelude, and that it has not been. That is a
statement about the game module's internal coupling: today every source in it pays for the
widget toolkit, the planner and the script virtual machine whether it touches them or not.
A rebuild laying out modules for the first time should make that split, because the
coupling this file papers over is real and will otherwise be recreated.

It also pulls in the engine module's own prelude, which the file marks as a mistake: a
prelude is an internal build detail of the module that owns it, and depending on another
module's is depending on that module's build configuration rather than on its interface.

## State

`Stateless.`

## Notes

One entry is not a declaration at all: on one platform the file pulls in a vendor
networking header, so that the module's message definitions can name that library's types.
That is the only content with any semantic effect, and it belongs with the transport seam.
