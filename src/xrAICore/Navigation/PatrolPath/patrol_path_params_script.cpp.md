# src/xrAICore/Navigation/PatrolPath/patrol_path_params_script.cpp

> Publishes the patrol-path handle to scripts under the name `patrol`, together with the two enumerations that name how a path is entered and left.

**Needs** — [`patrol_path_params.h`](patrol_path_params.h.md) · [`patrol_path.h`](patrol_path.h.md) · [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a declarative registration into the script virtual machine.

## Purpose

Patrolling is configured almost entirely from script, so this boundary is wide and every name in
it is frozen by the shipped scripts. It is also where two native names are deliberately changed
on the way out, and a rebuild that reproduces the native names instead breaks every shipped
script that patrols.

## `patrol` (script class)

**Contract** — registers the handle with five constructors, taking the path name alone or with
each further setting appended in order: start type, then route type, then the random flag, then
the previous waypoint index. Scripts overwhelmingly use the one- and two-argument forms.

Registered methods, with the native routine each names:

| Script name | Native routine |
|---|---|
| `count` | `count` |
| `point` | position of a waypoint by index |
| `name` | authored name of a waypoint by index |
| `index` | index of the waypoint with a given name |
| `get_nearest` | index of the waypoint nearest a position |
| `level_vertex_id` | mesh vertex of a waypoint |
| `game_vertex_id` | cross-level vertex of a waypoint |
| `flag` | one authored flag bit |
| `flags` | the whole authored flag word |
| `terminal` | has this waypoint no outgoing links |

**Invariants** — `index` and `get_nearest` are the *same native name* distinguished by argument
type; the script surface separates them into two names because the script language resolves by
value, not by declared type. A rebuild binding a language that cannot overload must keep these
two names.

Registered enumeration values, which are the mapping a rebuild cannot guess:

| Script | Native meaning |
|---|---|
| `patrol.start` | join at the path's **first** waypoint |
| `patrol.stop` (start family) | join at the path's **last** waypoint |
| `patrol.nearest` | join at the waypoint nearest the creature |
| `patrol.custom` | join at a caller-named waypoint |
| `patrol.next` | join at the waypoint after the one last used |
| `patrol.dummy` (start family) | unspecified |
| `patrol.stop` (route family) | remain at the terminal waypoint |
| `patrol.continue` | keep going past the end |
| `patrol.dummy` (route family) | unspecified |

**Notes** — the two families are registered into one script namespace and **collide**: `stop` and
`dummy` appear in both, the second registration winning. Shipped scripts therefore use `stop`
with whichever meaning the surviving registration gives it, and that is the behaviour a rebuild
must reproduce rather than the tidier two-namespace version. Which registration survives depends
on the binding layer's ordering; this is the single most fragile line in the patrol surface and
should be verified against the original before relying on it.

The position accessor is bound through a small wrapper rather than directly, because the native
routine hands back a reference into the path and the script side needs a value it owns. The
wrapper also rejects a missing handle. That is a lifetime concern of the binding, not a decision
about patrolling.
