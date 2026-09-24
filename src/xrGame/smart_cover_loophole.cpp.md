# src/xrGame/smart_cover_loophole.cpp

> Parses one loophole out of its authored table, derives whether it is usable at all, and builds the inner graph of moves between its actions.

**Needs** — [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_action.h`](smart_cover_action.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`smart_cover_loophole.h`](smart_cover_loophole.h.md); callers name that, not this file.
**Tier floor** — T2: table parsing and graph construction at load time

## Purpose

Turns the authored description of one firing position into the record the AI reasons over.
Three things are decided here rather than authored:

1. **Usability.** A loophole with no actions is marked unusable, overriding whatever the
   author wrote in the usable field.
2. **Degenerate directions.** Any of the three direction vectors that is effectively zero
   is replaced by a fixed forward direction and reported, rather than propagating a
   not-a-number through every angle test in [`smart_cover.cpp`](smart_cover.cpp.md).
3. **The inner transition graph.** Which action can follow which, and with what clips.

## State

```text
RECORD loophole
  id                   : text
  fov_position         : vector    # cover-local; the creature's eye point at this loophole
  fov_direction        : vector    # normalized
  danger_fov_direction : vector    # normalized
  enter_direction      : vector    # normalized
  fov                  : real      # radians; authored in degrees, 0..360
  danger_fov           : real      # radians
  range                : real      # metres
  actions              : map<text, action>
  transitions          : graph
      vertex_id   : text                # action name, or a reserved enter/exit name
      edge_weight : real
      edge_data   : list<text>          # interchangeable clip names for this move
  usable, enterable, exitable : bool
```

**Invariants** —

- The three directions are unit length or exactly the fallback forward direction; never
  zero.
- `usable` is true if and only if `actions` is non-empty.
- Every transition edge carries at least one clip name, and no name twice.
- `fov` and `danger_fov` are zero on an unusable loophole, because parsing stops before
  them.

## Construction

**Contract** — takes the loophole's authored table. Reads identity and geometry, builds the
actions, derives usability, and — only if usable — reads the transition graph and the
three scalar limits. Hard-fails on a malformed table; reports and recovers from a
degenerate direction. Blocks on the script virtual machine.

```text
FUNCTION build_loophole(table) -> loophole
  REQUIRE table IS a table
  id = REQUIRED table.id
  read table.usable                      # read, then discarded; see Notes
  fov_position = REQUIRED table.fov_position
  FOR EACH of fov_direction, danger_fov_direction, enter_direction
    IF the field is absent (danger only)  -> report, use forward
    ELSE IF the vector is ~zero           -> report, use forward
    ELSE                                   normalize it
  FOR EACH (name, action_table) IN REQUIRED table.actions
    REQUIRE name IS text AND name IS NOT already present
    actions[name] = build_action(action_table)
  usable = actions is non-empty
  IF NOT usable  RETURN                  # an unusable loophole is never planned through
  fill_transitions(REQUIRED table.transitions)
  fov   = radians(REQUIRED table.fov,  0..360)
  danger_fov = radians(table.danger_fov, 0..360)   IF present
  range = REQUIRED table.range, non-negative
```

**Invariants** —

- The early return on an unusable loophole is what makes the transitions table and the
  three limits *optional in practice* for a loophole an author has emptied out. A rebuild
  that parses eagerly will fail on shipped data.
- The authored `usable` field is read and then unconditionally overwritten by the derived
  value. It is dead in the shipped engine; the author's intent is expressed by authoring
  actions or not.

**Notes** — the danger arc and its direction are the *later* addition, which is why they
alone have optional readers and a forward-direction default with only a soft report. The
original marks both with a note to check the placed cover's serialized version instead of
guessing; that version gate was never written. The consequence a rebuild inherits: a cover
authored before the danger arc existed behaves as if its danger arc points along the cover
object's local forward axis and is zero degrees wide, i.e. nothing is ever inside it.

## `add_action`

**Contract** — builds one action and files it under its name. A duplicate name is a hard
failure — two actions with the same name would make the inner transition graph ambiguous.

## `fill_transitions`

**Contract** — builds the loophole's inner graph. Each entry names a source action and a
destination action (empty meaning "outside this loophole's action set", i.e. arriving or
leaving), a weight, and a list of interchangeable clip names.

```text
FUNCTION fill_transitions(table)
  FOR EACH entry IN table
    REQUIRE entry IS a table
    from = vertex name of entry.action_from, read as an *incoming* endpoint
    to   = vertex name of entry.action_to,   read as an *outgoing* endpoint
    clips = []
    FOR EACH name IN REQUIRED entry.animations
      IF name IS NOT text THEN SKIP
      REQUIRE name NOT already in clips   ELSE FAIL WITH "duplicated animation"
      APPEND name
    REQUIRE clips is non-empty
    weight = REQUIRED entry.weight
    ensure vertices `from` and `to` exist
    add edge from -> to with that weight, carrying clips
```

**Invariants** — the same enter/exit reserved-name trick as the description's outer graph,
one level down. Here the reserved vertices mean "not yet performing any action at this
loophole" and "done at this loophole", so a creature's movement through a cover is a path
in the outer graph each of whose steps expands into a path in an inner one.

## `action_animations`

**Contract** — the clip list for a named purpose of a named action at this loophole. Fails
naming both the action and the loophole when the action is absent, and delegates to the
action for the purpose lookup, passing this loophole's name down so that failure can name
it too.

## `transition_animations`

**Contract** — the clip list on the inner edge from one action to another. Both endpoints
are passed through the reserved-name substitution, source as incoming and destination as
outgoing, so an empty name means arriving or leaving. Fails naming both endpoints when
there is no such edge: a planner that produced an impossible step is a bug, and silently
playing nothing would leave the creature standing.

## `exit_position`

**Contract** — writes the local position of the loophole's exit action, if it has one, and
leaves the caller's value alone otherwise. The action name is a literal, which makes "exit"
a reserved action name across all authored covers.

**Notes** — the silent no-op on a missing exit action is correct here and unusual for this
file: not every loophole is one a creature leaves the cover from, and the caller's default
is its own position, which is a sensible place to leave from.

## Destruction

**Contract** — destroys the actions and the inner graph's data. As with the description,
this is ownership bookkeeping; what survives is that the actions and the clip lists live
exactly as long as the loophole.
