# src/xrGame/actor_input_handler.h

> The interface something must satisfy to take over, filter or scale the player's input.

**Needs** — [`Actor.h`](Actor.h.md) · [`actor_input_handler.cpp`](actor_input_handler.cpp.md)
**Used by** — [`ActorInput.cpp`](ActorInput.cpp.md) · [`actor_input_handler.cpp`](actor_input_handler.cpp.md) · [`controlled_actor.cpp`](ai/monsters/controlled_actor.cpp.md) · [`controlled_actor.h`](ai/monsters/controlled_actor.h.md)
**Tier floor** — T3: an interface declaration

## Purpose

An abstract base declaring what the player asks of whatever currently holds its input.
Two of its four questions are the contract; the other two are lifecycle, implemented in
[`actor_input_handler.cpp`](actor_input_handler.cpp.md).

## State

Holds the player it is installed on. Nothing else; an implementor adds its own.

## `authorized`

**Contract** — given a command identifier, answer whether the player is allowed to act on
it right now. Called for **every** player command, every frame it is pressed, so it must
be cheap. The default answer is yes, which makes a handler that only wants to scale the
mouse trivially writable.

**Invariants** — this is a *veto*, not a translation: a handler cannot substitute one
command for another, only suppress. Anything that wants to drive the player rather than
constrain them does so by other means. A rebuild tempted to widen this into a
transformation should note that the player's command handling assumes the command it
receives is the one the human pressed.

## `mouse_scale_factor`

**Contract** — a multiplier applied to look input. The default is one. This is how aiming
down a scope slows the crosshair, and how a vehicle makes the view heavier.

## `install` / `install(player)` / `release` / `reinit`

**Contract** — attach to the player (found automatically, or given), detach, and forget
without detaching. See [`actor_input_handler.cpp`](actor_input_handler.cpp.md).

**Notes** — there is exactly one handler slot on the player, so handlers do not compose:
a scope and a tutorial cannot both be installed. Every shipped use is exclusive by
construction, but the constraint is undeclared and a rebuild adding a second claimant will
discover it the hard way.
