# src/xrGame/script_watch_action_script.cpp

> Exports the look order to the script virtual machine as `look`, together with the script-facing names of the sight types.

**Needs** — [`script_watch_action.h`](script_watch_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Declares the look order to Lua. The interesting content is the *renaming*: the script-side
names of the sight types are not the engine's names, and the mapping is frozen by the
shipped scripts.

## State

`Stateless.`

## `CScriptWatchAction::script_register`

**Contract** — registers the class under the name `look`, with:

- a nested enumeration, also called `look`, mapping seven script names onto sight types:

  | script name | sight type |
  |---|---|
  | `path_dir` | look along the movement path |
  | `search` | sweep for the least-covered direction |
  | `danger` | face the least-covered direction (the engine calls this *cover*) |
  | `point` | look at a world position |
  | `fire_point` | look at a world position with the torso turned to it |
  | `cur_dir` | hold the current direction |
  | `direction` | look along a given direction vector |

  Only seven of the twelve sight types are reachable from script; the rest are set by
  engine code. Note `danger` and `fire_point`: the script vocabulary names the *intent*
  and the engine names the *mechanism*, and the two disagree. A rebuild must keep the
  script names.

- seven constructors: the default, by sight type, sight type with a direction, sight type
  with an object, sight type with an object and a bone name, and the two searchlight forms
  (target point with two rotation speeds; object with two rotation speeds).

- four methods — `object`, `direct`, `type`, `bone` — the four setters, and `completed`,
  inherited from the abstract action, which is how a script polls whether the look has
  finished.

**Notes** — the enumeration values are registered as plain integers rather than as the
typed enumeration, because the binding layer resolves the constructor overloads by
argument type and a typed enumeration there would make the sight-type and searchlight
constructors ambiguous.
