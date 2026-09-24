# src/xrGame/alife_monster_movement_manager.cpp

> The offline creature's movement brain: chooses between free travel to a destination and following an authored patrol path, and feeds whichever it chose into the detail mover each alife tick.

**Needs** — [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md) · [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md) · [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a two-way dispatch over owned sub-managers; no layout or device concern

## Purpose

An offline creature does not walk: it is *moved*, coarsely, by the alife simulation. This
file is the one-line policy that decides how. A creature is in exactly one of three
movement modes, and the mode determines who supplies the destination.

It is a separate file from either sub-manager because the composition is the decision: the
patrol manager never moves anything, and the detail mover never chooses anything. Keeping
the arbiter tiny is what lets a rebuild add a third mode without touching either.

## State

```text
RECORD MonsterMovementManager
  object     : ref MovementManagerHolder   # the offline creature this drives
  detail     : DetailPathManager           # owned; the actual mover
  patrol     : PatrolPathManager           # owned; supplies destinations from an authored path
  path_type  : PathType                    # no-path | game-path | patrol-path
```

Invariants: both sub-managers exist for the whole lifetime of the movement manager —
they are created with it and destroyed with it, never lazily. A rebuild that makes them
optional must then handle absence at every call site, which buys nothing; the cost of two
always-present records per offline creature is the deliberate trade.

The initial mode is **no-path**. A newly spawned offline creature stands still until
something — a smart terrain job, or a script — gives it a destination.

## `update`

**Contract** — called once per alife tick for the creature. Does not block, does not
allocate in the no-path case. Dispatches on the current mode:

```text
FUNCTION update()
  IF path_type == GAME_PATH
    detail.update()                 # destination was set directly by whoever set the mode
  ELSE IF path_type == PATROL_PATH
    patrol.update()                 # advance along the authored patrol, possibly picking a new point
    detail.target(patrol.target_game_vertex,
                  patrol.target_level_vertex,
                  patrol.target_position)
    detail.update()                 # then travel toward it
  ELSE IF path_type == NO_PATH
    # stand still
  ELSE
    FAIL WITH unreachable mode
```

**Invariants** — the ordering inside the patrol branch is load-bearing and is the whole
content of this function. The patrol manager is advanced **before** the destination is
read, so that a patrol point reached on this tick is replaced by the next one in the same
tick rather than one tick later; and the destination is pushed into the detail mover
**before** the mover runs, so that travel happens against the fresh destination. Swapping
either pair introduces a one-tick lag that is invisible on a short patrol and compounds
into visible desynchronization between a creature's recorded position and the patrol it is
supposedly walking.

The mode set is closed: an unrecognised mode is a programming error, not a condition to
recover from. A rebuild should treat it as one.

## `completed` / `actual`

**Contract** — both unconditionally report success. They exist because the movement
manager sits in a position where callers ask a mover "are you done" and "is your plan
still valid"; at this level the answer is always yes, because the manager itself holds no
plan — the plans live in the two sub-managers, and a caller that cares asks them directly.

**Notes** — these are not stubs awaiting implementation. They are the honest answer for a
component whose job is dispatch. A rebuild may delete them and let callers query the
sub-managers, at the cost of the uniform mover interface.

## `on_switch_online` / `on_switch_offline`

**Contract** — forwarded to the detail mover only, and deliberately not to the patrol
manager.

**Notes** — the asymmetry is the interesting part. Going online, a creature's coarse
graph position must be converted into a real position on the loaded level, and coming
offline the reverse; that is the detail mover's business, because it is the only part
holding a position. A patrol path is authored level data whose meaning does not change
with the creature's online state — the creature is at patrol point three whether or not
anyone can see it — so the patrol manager has nothing to do at the transition.
