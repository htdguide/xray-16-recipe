# src/xrGame/script_object_action_script.cpp

> Exports the object channel to the script layer as `object`, with the frozen table of item-handling orders.

**Needs** — [`script_object_action.h`](script_object_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of an item-handling order. The order names are frozen and
are matched by the animation sets authored for every creature that can hold a weapon.

## `script_register`

**Contract** — registers a class named `object` with one constant table, five constructors
and four methods.

```text
object.state = { idle, show, hide, take, drop, strap,
                 aim1, aim2, reload, reload1, reload2,
                 fire1, fire2, switch1, switch2,
                 activate, deactivate, use, turn_on, turn_off, dummy }

object()
object(game_object, state)
object(game_object, state, burst_size)
object(state)
object(bone_name, state)

methods: action(state), object(bone_name), object(game_object), completed()
```

**Notes**

`reload` and `reload1` are the **same value** — the unnumbered spelling is an alias for the
primary firing mode, kept because scripts predating the two-mode weapons use it. There is
no unnumbered alias for aim, fire or switch, so the inconsistency is real and a rebuild
must reproduce it rather than tidy it.

`strap` is the script spelling of the *strapped* state — weapon slung on the back rather
than holstered or in hand. It is a third carry position, not a synonym for hide.

`dummy` is the one-past-the-end marker exported as if it were an order, giving scripts an
upper bound. Nothing may be ordered with it.

`object` is one script name dispatching on argument type — text selects the bone form,
a game object the object form. A rebuild whose binding layer cannot overload must emulate
it; shipped scripts use both spellings under the one name.
