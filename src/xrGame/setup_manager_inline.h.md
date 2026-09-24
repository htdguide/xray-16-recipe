# src/xrGame/setup_manager_inline.h

> The weighted-random action selector: holds one action running until it finishes, then draws its successor by weight from the applicable others.

**Needs** — [`setup_manager.h`](setup_manager.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — [`setup_manager.h`](setup_manager.h.md)
**Tier floor** — T2: a selection step on the per-tick path

## Purpose

Carries the bodies for [`setup_manager.h`](setup_manager.h.md). The idea it encodes is
small and used in several places: an object has a handful of competing behaviours, each
with a weight and an applicability test; exactly one runs at a time; when it says it is
finished, the next is drawn at random with probability proportional to weight, *excluding
the one that just ran*.

The exclusion is the point. Without it a high-weight action would re-draw itself
repeatedly and the object would appear stuck doing one thing; with it, variety is
guaranteed at the cost of never repeating an action twice in a row.

## State

```text
RECORD SetupManager
  actions            : list<(id, Action)>   # ordered by insertion; owns every action
  object             : DrivenObject         # never none
  current_action_id  : id
  previous_action_id : id                   # what ran on the previous tick
  actuality          : bool                 # false = the selection must be redrawn
```

**Invariants** —

- The registry holds at most one action per identifier; adding a duplicate is a
  programming error.
- `current_action_id` names a present action whenever the registry is non-empty. The
  first action added becomes the current one, so there is never a window where an object
  with actions has no selection.
- `actuality` false forces a redraw on the next update even if the current action has not
  completed. That is how an outside caller forces a change of behaviour: adding an action,
  clearing, or reinitializing all clear it.
- Cleared state sets both identifiers to the type's all-ones value — a sentinel that
  cannot collide with a real identifier, which is why the identifier type must be one
  where that value is unused.

## `update`

**Contract** — the per-tick step. Does nothing at all when no actions are registered,
which is what lets an object own a manager it never populates.

```text
FUNCTION update()
  IF actions IS empty THEN RETURN
  select_action()
  IF previous_action_id != current_action_id
    current_action().initialize()      # only on a change, so a continuing action
                                       # is not restarted every tick
  previous_action_id = current_action_id
  current_action().execute()
```

**Invariants** — initialization is driven by the *comparison with the previous tick*, not
by the selection step, so an action that the selector re-confirms is not re-initialized.
Execute always runs, on the tick of the change too.

## `select_action`

**Contract** — the algorithm. Redraws only when the selection is stale or the running
action has finished; otherwise leaves everything alone.

```text
FUNCTION select_action()
  IF actuality AND NOT current_action().completed() THEN RETURN
  actuality = true

  IF actions has exactly one entry
    IF current_action_id != that entry's id
      that entry.initialize()          # see the note below
    current_action_id = that entry's id
    RETURN

  total = 0
  FOR EACH (id, action) IN actions
    IF id != current_action_id AND action.applicable()
      total = total + action.weight()
  REQUIRE total is non-zero            # at least one other action must be available

  draw = uniform random in [0, total)
  running = 0
  FOR EACH (id, action) IN actions
    IF id == current_action_id OR NOT action.applicable() THEN CONTINUE
    running = running + action.weight()
    IF running > draw
      IF current_action_id names a present action
        current_action().finalize()
      current_action_id = id
      action.initialize()
      BREAK
```

**Invariants** —

- The single-action case is special-cased, and it *has* to be: the general path excludes
  the current action, so with one action the total weight would be zero and the draw would
  be undefined. With one action the manager simply keeps it forever, re-selecting it once.
- The predecessor is finalized *before* the successor's identifier is installed, and the
  successor is initialized immediately after. So exactly one action is live at any moment,
  and the transition is finalize-then-initialize, never the reverse.
- The requirement that the total weight be non-zero means a populated manager must always
  have at least one applicable action other than the current one. An object whose actions
  are all conditionally applicable can deadlock here, and the assertion is what surfaces
  that as a bug in the action set rather than as frozen behaviour.

**Notes** — the single-action branch initializes the action and then `update` sees the
identifier changed and initializes it *again*. That double initialization is visible only
on the first tick after a clear, and actions in this engine tolerate it; a rebuild should
initialize in one place, in `update`, and delete the call here.

The draw is over weights of *applicable* actions only, and applicability is evaluated
twice — once to total and once to walk. A rebuild may collect the applicable set once.

## Construction, destruction, `clear` and `reinit`

**Contract** — construction demands a driven object and never a null one. Destruction
clears, which deletes every registered action: the manager owns them from the moment they
are added. `clear` also resets both identifiers to the sentinel and marks the selection
stale. `reinit` is `clear` plus the same staleness — the distinction is that `reinit` is
the lifecycle hook an object calls when it respawns, and `clear` the internal one.

## `add_action`

**Contract** — registers an action under an identifier, transfers ownership, tells the
action which object it drives, and marks the selection stale. If this is the first action,
it becomes the current selection immediately. Registering an identifier twice is a
programming error.

## Lookup

**Contract** — lookup by identifier is a linear scan of the registry and asserts the
identifier is present. Linear is correct here: these registries hold single digits of
actions and are walked in full by the selector anyway.
