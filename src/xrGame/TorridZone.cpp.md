# src/xrGame/TorridZone.cpp

> The burner anomaly that moves: the same damaging zone as its static parent, driven along an authored path so that it drifts through the level.

**Needs** — [`TorridZone.h`](TorridZone.h.md) · [`MosquitoBald.h`](MosquitoBald.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`xrEngine/ObjectAnimator.h`](../xrEngine/ObjectAnimator.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — reached through its declarations in [`TorridZone.h`](TorridZone.h.md); callers name that, not this file.
**Tier floor** — T3: a transform driven from a clip, plus sound repositioning

## Purpose

Every other anomaly in the game is a fixed volume. This one wanders: its whole
contribution over its stationary parent is that its transform comes from an authored
motion clip rather than from its spawn position, so the same damaging volume tracks a path
the level designer drew.

That one change has three consequences, and they are the file:

- the transform must be taken from the clip each update and the zone must be re-indexed
  spatially, because a zone that moves without telling the spatial database is felt in the
  wrong place;
- every sound the zone owns must follow it, because a zone's sounds are placed at its
  centre and the centre is now moving;
- the clip must start, stop and restart with the zone's enabled state.

## State

```text
RECORD TorridZone EXTENDS MosquitoBaldZone
  motion : ObjectAnimator      # an authored transform clip, owned; looped
```

**Invariants** — the clip is the sole source of the zone's transform while it is enabled.
Nothing else may move the zone, or the next update will snap it back.

## `net_Spawn`

**Contract** — brings the zone up, then reads the name of its motion clip from its server
record, loads it, and starts it looping. The server record must be of the moving-zone kind;
anything else is a data error and is asserted, because a moving zone with no path has no
defined position at all.

## `UpdateWorkload`

**Contract** — the zone's own per-interval work, plus the movement: advance the clip by the
elapsed interval, adopt the clip's transform as the zone's, and announce that the zone has
moved so that the spatial index, the felt-object set and anything tracking the zone's
position are all updated.

```text
FUNCTION UpdateWorkload(elapsed_ms)
  base.UpdateWorkload(elapsed_ms)
  motion.advance(elapsed_ms / 1000)        # the clip is in seconds, the scheduler in milliseconds
  transform <- motion.transform
  announce_moved()
```

**Invariants** — the base's work runs *before* the move, so that damage for this interval
is applied at the position the zone occupied during it rather than at the one it is about
to occupy. At the zone's speeds the difference is small, but the ordering is the honest
one and costs nothing.

## `shedule_Update`

**Contract** — the scheduled work, plus re-anchoring all four of the zone's sounds — the
idle hum, the discharge, the hit and the entry cue — at the zone's current centre. Each is
moved only if it is actually playing.

**Notes** — the sounds are repositioned on the *scheduled* update, which runs less often
than the movement does. A fast-moving zone's sound therefore lags its damage slightly.
That is a deliberate cost: positional audio updates are not free and the ear cannot
resolve the difference at these speeds.

## `Enable` / `Disable`

**Contract** — when the zone is switched on, the clip is restarted from its beginning and
looped; when it is switched off, the clip stops and the zone stays where it was. Both
defer to the parent first and only act if the parent actually changed state, so a
redundant enable does not jump the zone back to the start of its path.

## `light_in_slow_mode`

**Contract** — false: this zone's light is updated every frame rather than at the reduced
rate the generic zone uses. A stationary zone's light can be updated lazily because it
does not move; this one's must keep up with its transform or the glow trails behind the
anomaly.

## `AlwaysTheCrow`

**Contract** — true: this object is always kept in the engine's every-frame update set,
regardless of distance or visibility. The generic zone opts in only under conditions; a
moving zone must always run, because its position is produced by its own update and a
skipped frame leaves it stale — and because an entity elsewhere in the level may be
standing where the zone is about to arrive.

## Notes

The class names its parent as the generic zone rather than as the burner zone it actually
derives from, so every deferral in this file skips a level of the hierarchy. As shipped
nothing breaks, because the burner zone overrides none of the five methods this file
defers through. It is a live trap: adding an override to the burner zone silently loses it
for the moving one. A rebuild should name the immediate parent.
