# src/xrGame/actor_mp_client.cpp

> The multiplayer player object: the player character with camera freedom removed, death forced, remote-view smoothing applied, and one extra server-driven event.

**Needs** — [`actor_mp_client.h`](actor_mp_client.h.md) · [`Actor.h`](Actor.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`eatable_item.h`](eatable_item.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Level.h`](Level.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-event logic on the client object

## Purpose

A networked player is the same player character as in single player with four rules laid
on top, each of which exists because something that is a *choice* in single player is a
*fairness or consistency* question in a match.

The import and export halves are separate files;
[`actor_mp_client_export.cpp`](actor_mp_client_export.cpp.md) and
[`actor_mp_client_import.cpp`](actor_mp_client_import.cpp.md).

## State

Adds to the player: the wire-state holder, and a saved camera-smoothing setting with the
value to use while spectating.

## `cam_Set`

**Contract** — refuses every camera mode except first person in a release build; a debug
build allows all of them. Deactivating and activating is otherwise the ordinary camera
switch, handing the outgoing camera to the incoming one so it can inherit the orientation.

**Invariants** — third-person and free-look cameras see around corners the first-person
camera cannot. Permitting them in a match is an advantage, so they are compiled out rather
than merely discouraged. This is a rule a rebuild must keep on the *engine* side: anything
configurable is exploitable.

## `Die`

**Contract** — forces health to exactly zero before running the ordinary death, rather
than letting death happen at whatever fraction the damage left.

**Invariants** — death is decided by the server and announced; every client must agree the
player is dead, and a client whose local health arithmetic left a sliver would otherwise
keep simulating a living player. Setting the value rather than trusting it is the general
pattern for server-decided facts.

## `OnEvent`

**Contract** — intercepts one event kind, *use a consumable*, and passes everything else
to the player base.

## `use_booster`

**Contract** — the server has told this client that the player consumed an item. Resolves
the item by its network identifier, verifies it is consumable, and applies its effect.
Ignored entirely when running on the server, which applied the effect when it decided it.
A missing or wrong-typed identifier is logged and dropped rather than fatal.

**Invariants** — consumables are the one inventory effect that *must* be
server-announced rather than locally predicted: their effect is a change to the condition
values the server is authoritative over, and predicting it would desynchronize.

## `On_SetEntity` / `On_LostEntity`

**Contract** — called when the camera starts and stops following this player. On taking
the view, the global camera-smoothing setting is saved and, **if this is not the player
this client controls**, replaced with a fixed smoothing value. On losing the view it is
restored.

**Invariants** — a spectated player's motion arrives in discrete network updates, so the
camera following them needs smoothing that a locally controlled player does not. Applying
it to the local player would add input latency, which is why the substitution is
conditional. The smoothing value is a hard-coded constant with no derivation beyond
looking right.

**Notes** — the setting being changed is a *global* console variable, saved and restored
around the spectate. That is a global mutation used as a scoped override, and it leaks if
the view is lost without the matching call. A rebuild should make the camera's smoothing a
property of the camera.
