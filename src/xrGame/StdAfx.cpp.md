# src/xrGame/StdAfx.cpp

> The one source file that exists to make the precompiled prelude compile. Nothing else.

**Needs** — [`StdAfx.h`](StdAfx.h.md)
**Used by** — reached through its declarations in [`StdAfx.h`](StdAfx.h.md); callers name that, not this file.
**Tier floor** — T4: a build input

## Purpose

Some compilers build a precompiled prelude by compiling exactly one source file that
includes it and nothing else. This is that file. It has no content, declares nothing and
runs nothing.

It also suppresses one diagnostic — the one a compiler emits when a generated type name
grows past its internal limit, which the module's heavily nested container types do
routinely. The name length has no effect on behaviour.

A rebuild has no counterpart to this file.

## State

`Stateless.`
