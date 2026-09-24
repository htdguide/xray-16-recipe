# src/xrGame/controller_state_panic_inline.h

> An unfinished behaviour state: the psychic creature's panic, which selects nothing and does nothing.

**Needs** — [`ai/monsters/controller/controller_state_panic.h`](ai/monsters/controller/controller_state_panic.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: empty bodies

## Purpose

The psychic creature's behaviour is a tree of states, each of which picks among sub-states
each time it is re-evaluated. This file is the panic state's implementation, and it is
**empty**: construction adds no sub-states and the re-selection does nothing, so the state
holds whatever its base does and never changes it.

The file is honest evidence of an abandoned design. The commented-out constructor body shows
the intent — panic was to consist of a single "run" sub-state — and it was never wired up.

## State

`Stateless.` It adds nothing to the state it derives from.

## the panic state

**Contract** — constructs over a creature, adding no sub-states; re-selection is a no-op.

**Notes** — a rebuild has two honest options: omit the state entirely, or implement it as the
comment describes — a single flee-at-speed sub-state that is re-entered until the state's
owner switches away. Reproducing the empty version faithfully reproduces a creature that
enters panic and then stands still, which is what the shipped games do; if that behaviour is
visible in play it is a bug being preserved, and a rebuilder should decide deliberately.
