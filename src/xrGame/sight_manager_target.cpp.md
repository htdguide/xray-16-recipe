# src/xrGame/sight_manager_target.cpp

> Computes what angle a creature should want: aim solves against a point, the direction of travel, and the search over the level graph for the yaw that exposes the least of the map.

**Needs** — [`sight_manager.h`](sight_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai/stalker/ai_stalker_space.h`](ai/stalker/ai_stalker_space.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`Actor.h`](Actor.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a bounded search over navigation vertices, run per creature per look order

## Purpose

The other half of the aiming manager: [`sight_manager.cpp`](sight_manager.cpp.md) decides
*how to get to* an angle, this file decides *which angle to want*. Three distinct
questions live here, and they are unrelated to each other except that all three write the
same two numbers.

The third one — the cover search — is the interesting one, and it is the reason a creature
in this game looks like it is watching the right part of the room. It does not reason about
threats at all; it asks the level's navigation graph which facing *sees the most open
ground*, and looks there.

## State

Stateless. Every function writes into the creature's head and body angle pairs, which are
owned by the movement manager.

## `SetPointLookAngles`

**Contract** — given a target point, an eye point, and optionally the entity being looked
at, writes the yaw and pitch that face the eye point at the target. If the aim refinement
below accepts the entity it may substitute both points; if it declines, the originals are
used unchanged. Both angles are negated on the way out — the movement layer's angle sign
convention is the opposite of the vector decomposition's.

```text
FUNCTION set_point_look_angles(target_point, eye_point, object) -> (yaw, pitch)
  from = eye_point
  to   = target_point
  IF NOT refine_aim_point(from, to, object)
    from = eye_point ; to = target_point       # refinement may have half-written; restore
  (yaw, pitch) = heading and pitch of (to - from)
  RETURN (-yaw, -pitch)
```

**Invariants** — the restore after a declined refinement is not defensive noise: the
refinement writes the eye point *before* it can decide to decline, so the caller's values
must be put back.

## `SetFirePointLookAngles`

**Contract** — identical to the above, except that a degenerate direction (target and eye
coincident) is replaced by a fixed forward direction instead of producing undefined
angles. This is the variant used by torso-look orders.

**Invariants** — the substitution matters because a torso order drives the creature's
whole body; undefined angles there would spin the model, while a head-only order with a
bad angle merely looks wrong for a frame. That is why only this variant guards.

## `aim_target` (aim point refinement)

**Contract** — given an entity, decides whether the creature should aim at something more
specific than its geometric centre, and if so rewrites the aim point (and possibly the
creature's own eye point). Returns whether it took over.

```text
FUNCTION refine_aim_point(my_position, aim_point, object) -> bool
  IF object IS none                              RETURN false
  IF I have a configured aim bone
    aim_point = that bone's position on `object`
    RETURN true
  IF object IS the player                        -> aim at its head bone; RETURN true
  IF object IS a living creature of my own kind  -> aim at its head bone; RETURN true
  IF object does not want to be aimed at by centre  RETURN false
  my_position = my centre, shifted 20 cm forward in my own frame
  RETURN true
```

**Invariants** —

- The creature's *own* configured aim bone takes priority over everything, including the
  player. It is a per-creature-section setting: a creature may be authored to aim at, say,
  the chest.
- The head-bone rule applies only to the player and to *living* creatures of the stalker
  kind. A corpse and a non-humanoid are aimed at by centre, because the bone name is a
  humanoid rig's.
- The bone is named by a literal string in the original. That is a frozen coupling to the
  shipped humanoid skeleton; a rebuild must keep the same rig or move the name into
  configuration.
- Objects can opt out of centre-aiming entirely, in which case the caller's own point is
  kept.

**Notes** — the 20 cm forward shift of the *shooter's* eye point is annotated in the
original as a workaround for the player model being animated with a horizontal offset: the
model's centre does not sit over its origin, so a solve from the centre aims off to one
side. What survives is the requirement, not the hack — the eye point must be the point the
weapon actually fires from in the *posed* model, and if your rig has no such offset you do
not need the correction. Note also that the shift is applied at the *centre's* height, not
the eye's, which keeps the vertical solve honest.

## `SetDirectionLook`

**Contract** — face the direction of travel. Asks for the path heading; on success writes
it to the head target with the yaw negated and the **pitch forced to zero**; on failure
holds the current head angles. Either way the body target is then set equal to the head
target, so this order always turns the whole creature.

**Invariants** — pitch is zeroed, not negated. A creature walking downhill should not look
at its own feet; the path heading's pitch is a property of the terrain, not of where the
creature wants to look.

## `SetLessCoverLook` — the exposure search

**Contract** — chooses the head yaw that maximizes the area of the level graph visible
from the creature's current navigation vertex within its field of view. Two forms: the
short one first faces the direction of travel, returns immediately if the creature has no
path at all, and otherwise runs the long form with the default head-turn window (an eighth
of a turn either side); the long form takes the window explicitly, which torso-look orders
widen to a half turn.

```text
FUNCTION less_cover_look(vertex, max_head_turn, difference_look)
  (range, fov) = my current sight range and field of view
  half_fov = fov / 2                              # configured in degrees, used in radians
  best_angle = head.target.yaw
  next_vertex = none

  IF difference_look AND my path has a point beyond the current one
    FOR EACH neighbour OF vertex
      IF the path's next travel point lies inside that neighbour
        next_vertex = it ; BREAK

  IF next_vertex IS none                          # plain search
    FOR yaw FROM body.target.yaw - max_head_turn
            TO   body.target.yaw + max_head_turn
            STEP max_head_turn / 18
      area = visible area from `vertex` facing yaw within half_fov
      keep yaw with the greatest area
  ELSE                                            # difference search
    FOR yaw over the same window, STEP 2*max_head_turn / 60
      gain = area from next_vertex - area from vertex
      keep yaw with the greatest gain, breaking ties toward the body's own yaw
      also track the best plain area as a fallback
    IF the best gain is not positive
      best_angle = the plain-area winner

  head.target.yaw   = normalize_signed(best_angle)
  head.target.pitch = 0
```

**Invariants** —

- The search window is centred on the **body's target** yaw, not the head's. The creature
  is choosing where to look *relative to where its feet will be*, which is what keeps the
  chosen angle inside the twist limit the turning-in-place rule enforces in
  [`sight_manager.cpp`](sight_manager.cpp.md).
- The plain search takes 37 samples across the window and the difference search 61. The
  difference search is finer because it is comparing two nearly equal areas and a coarse
  sample misses the crossover.
- Pitch is always zeroed. The exposure measure is a horizontal one — it is computed from
  the navigation graph, which is a ground surface.
- Tie-breaking in the difference search prefers the angle *closest to the body yaw*. Without
  it, a symmetric corridor makes the creature pick an arbitrary side and flip between them
  as the path advances.

**Notes** —

- The "difference look" mode is the one used while walking: it scores each candidate yaw
  by how much *more* the creature would see from the next step than from here, which makes
  a creature walking a corridor look ahead into the rooms it is about to be able to see,
  rather than at the wall it is already facing. This is the single most characteristic
  piece of the series' stalker behaviour and it is entirely a property of the shipped
  navigation data.
- The fallback threshold in the original compares the square root of the best gain against
  zero, a form that is trivially "is the gain positive" but written as a scaled comparison
  with the scale multiplied by zero. The intent readable from the shape is that a *minimum*
  gain was meant to be required before preferring the difference answer, and the minimum
  was disabled rather than deleted. A rebuild should implement the plain condition; if it
  reintroduces a threshold it will change walking gaze behaviour.
- The field of view arrives in degrees from the creature's configuration and is halved
  after conversion, so the search sees a half-angle. Getting this wrong by a factor of two
  makes the exposure measure useless rather than merely different.

## `GetDirectionAngles`

**Contract** — the heading and pitch of the creature's direction of travel. Prefers the
vector from the current travel point to the next one on the detailed path; when there is
no such pair, falls back to the difference of the last two recorded positions. Returns
whether it produced an answer at all.

**Invariants** — the path is the better source because it is where the creature is *going*,
including around a corner it has not turned yet. The position history is where it has
*been*, which lags by a frame and is noisy when standing still.

## `GetDirectionAnglesByPrevPositions`

**Contract** — heading and pitch from the last two recorded positions. Declines when there
are fewer than two samples, and declines when the displacement is below a threshold rather
than returning a direction derived from numerical noise.

**Invariants** — the "essentially stationary" rejection is what makes the caller hold the
current head angles instead of jittering. A rebuild that returns a normalized
near-zero vector here will make standing creatures' heads twitch every frame.
