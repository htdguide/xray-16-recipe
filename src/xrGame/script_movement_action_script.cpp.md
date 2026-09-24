# src/xrGame/script_movement_action_script.cpp

> Exports the movement channel to the script layer as `move`, with the six constant tables and the twenty constructor forms scripts address it by.

**Needs** — [`script_movement_action.h`](script_movement_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_params.h`](../xrAICore/Navigation/PatrolPath/patrol_path_params.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of a movement order. Everything here is frozen by
shipped scripts: the class name, the table names, the value names, and the fact that a
great many argument shapes all spell the same constructor.

## `script_register`

**Contract** — registers a class named `move` with six constant tables, twenty
constructors and eight methods.

```text
move.body                = { standing, crouch }
move.move                = { stand, walk, run }
move.path                = { line, dodge, criteria, curve, curve_criteria }
move.input               = { none, fwd, back, left, right, up, down,
                             handbrake, on, off }          # combinable bit flags
move.monster             = { walk_fwd, walk_bkwd, run_fwd, drag, jump, steal,
                             walk_with_leader, run_with_leader }
move.monster_speed_param = { default, force }

methods: body(state), move(type), path(type), object(game_object),
         patrol(path, path_name), position(vector), input(keys), completed()
```

**Notes**

**`curve` and `line` are the same value, and so are `curve_criteria` and `criteria`.** The
engine once distinguished a curved detail path from a straight one and no longer does; the
two extra names survive so that scripts written against the older engine still load. A
rebuild must keep both spellings mapping to one behaviour, and should not invent a
difference.

The twenty constructors are the cross product of goal kind and optional trailing arguments,
because the binding layer has no defaulted parameters: each arity is registered separately.
A rebuild whose script layer supports optional arguments collapses this to six.

`patrol` is exported through a small adapter rather than directly, because the script
passes the path name as plain text where the channel stores it interned. That is a binding
detail with no bearing on a rebuild.

`completed` is inherited from the shared action-channel base and re-exported here by name
so a script can ask a movement order, rather than the whole action, whether it is done.
