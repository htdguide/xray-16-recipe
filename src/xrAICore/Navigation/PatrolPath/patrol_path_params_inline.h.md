# src/xrAICore/Navigation/PatrolPath/patrol_path_params_inline.h

> Empty — the patrol-path handle has no inline surface left.

**Needs** — [`patrol_path_params.h`](patrol_path_params.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: there is nothing here.

## Purpose

The file contains nothing but its include guard. Every sibling type in this directory pairs a
declaration with an inline implementation file, and this one keeps the slot without using it —
the handle's routines all went into
[`patrol_path_params.cpp`](patrol_path_params.cpp.md) instead.

## State

Stateless.

**Notes** — a rebuild should not create this file. It is recorded here only because the mirror is
complete, and because its emptiness is the answer to "did I miss something in the patrol-path
handle" — no.
