# src/xrGame/actor_defs.h

> The player character's shared vocabulary: the movement command bit set, the camera modes, the context-action kinds, and the three record shapes the network and prediction paths pass around.

**Needs** — [`PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [`xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`Actor.h`](Actor.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`EffectorBobbing.cpp`](EffectorBobbing.cpp.md) · [`HudItem.cpp`](HudItem.cpp.md) · [`HudItem.h`](HudItem.h.md) · [`actor_anim_defs.h`](actor_anim_defs.h.md) · [`player_hud.h`](player_hud.h.md) · [`UIDragDropReferenceList.cpp`](ui/UIDragDropReferenceList.cpp.md) · [`UIHudStatesWnd.h`](ui/UIHudStatesWnd.h.md)
**Tier floor** — T1: the network records are wire-adjacent and their float widths are relied upon

## Purpose

Declarations that more than one part of the player implementation needs and that none of
them owns: the movement state, which is a bit set and not an enumeration; the camera
modes; the two constants describing the player's physical size and lean speed; and the
records that carry player state between the client's input, the network and the
interpolator.

The single most load-bearing thing here is the movement bit set. The player's entire
locomotion state — direction, posture, sprint, jump, fall, land, lean — is *one integer of
flags*, and it is the same integer in the input handler, the animation selector, the
physics mover and the network packet. Every part of the player reads it.

## State

```text
ENUM MoveCommand                   # bit positions in one 32-bit flag word
  forward, backward, strafe_left, strafe_right     # bits 0..3
  crouch, accelerate, turn, jump                   # bits 4..7
  fall, landing, landing_hard, climb               # bits 8..11
  sprint, lean_left, lean_right                    # bits 12..14

  any_move   = forward | backward | strafe_left | strafe_right
  any_action = any_move | jump | fall | landing | landing_hard
  any_state  = crouch | accelerate | climb | sprint
  lean       = lean_left | lean_right
```

The three composite masks are the load-bearing part. `any_move` asks "am I going
anywhere", `any_state` asks "what posture am I in", and `any_action` is the set that
animation and network treat as transient. Note that turning is in *none* of them:
turning on the spot is deliberately neither a move nor a state, because it must not
interrupt an idle.

```text
ENUM CameraMode      first_person, look_at, free_look, fixed_look_at
ENUM ContextAction   none, pick_up, talk, open_door, search_corpse
CONSTANT actor_height        = 1.75   # metres; the player's capsule height
CONSTANT actor_lookout_speed = 2.0    # lean in and out at this rate
CONSTANT death_sound_variants = 4
CONSTANT damage_reaction_count = 12   # must equal the flinch motion array in actor_anim_defs
```

```text
RECORD InputSample                 # one frame of the local player's intent
  timestamp    : int
  wished_state : int (32-bit flags)   # the movement bit set the player asked for
  camera_mode  : int (8-bit)
  camera_yaw, camera_pitch, camera_roll : real
  # ordered by timestamp: the client keeps a queue of these for reconciliation

RECORD StateUpdate                 # a player's authoritative state at a server tick
  timestamp    : int                  # server game clock, not the client's
  model_yaw    : real                 # the body's facing
  torso        : rotation             # the torso's aim, in world space, independent of the body
  position, acceleration, velocity : vector   # world space
  movement_state : int (32-bit flags)
  weapon       : int                  # which weapon is in hand
  health       : real

RECORD PhysicsStateUpdate          # the rigid-body half, sent separately
  timestamp    : int
  state        : physics sync state

RECORD InterpolationSample         # what the interpolator blends between
  position, velocity : vector
  model_yaw  : real
  torso      : rotation
```

Invariants:

- The torso rotation is carried in **world** coordinates, not relative to the body. The
  player can aim in a direction the body is not facing, and expressing the aim relatively
  would make it depend on a body yaw that arrives in the same packet and may be
  interpolated differently.
- A state update carries position, velocity *and* acceleration. The third is what lets the
  receiver extrapolate for a frame when the next update is late, rather than freezing.
- Input samples are ordered by timestamp and compared against a time directly, because the
  client's reconciliation replays the queue from the last acknowledged tick.
- Separating the physics half into its own record is what lets a player be
  network-updated without a physics update, which is the normal case: a player standing on
  the ground has no interesting rigid-body state.

## `lerp`

**Contract** — blends two state updates by a fraction, producing the pose a remote player
is drawn at between two received updates. Interpolates position, velocity, body yaw and
torso aim; the flag word and the weapon are *not* blended — they are taken from one
endpoint, because a bit set has no midpoint.

## Sleep result

**Contract** — the answer to "may the player sleep here" is a *string-table key*, not an
enumeration: an empty result means yes, and any other value names the reason why not, in
the player's language. The original had an enumeration and replaced it, which moved the
reasons from the engine into data — a rebuild should start where this one ended up.

## Quick-use slots

**Contract** — four named inventory slots the player binds items to for one-key use. The
binding is by section name and is a user setting, persisted with the console variables.

**Notes**

- The player's height is a constant here rather than a configuration key, and the
  animation set, the camera height and the collision capsule are all tuned around it.
  Changing it is not a supported edit.
- The context-action enumeration exists to drive the dynamic hint shown under the
  crosshair; it is a presentation classification of what the player is looking at, not a
  capability check.
