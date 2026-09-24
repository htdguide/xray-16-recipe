# src/xrGame/ai/phantom/phantom.cpp

> An apparition on rails: born, flies at one target under a turn-rate limit, and ends on contact or on being shot — with every phase's length decided by its animation.

**Needs** — [`phantom.h`](phantom.h.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`phantom.h`](phantom.h.md)
**Tier floor** — T3: a state machine, a turn-rate-limited pursuit, and one damage event

## Purpose

Four decisions, and they are what a rebuild must reproduce.

**Phases end when their animation ends.** Birth, contact and shot each play a motion that
stops at its end and raises a completion callback; the callback requests the next state.
Nothing is timed. Re-pacing the apparition is an animation change, and the spawn code
asserts that those three motions really are non-looping — a model that loops them would
leave the phantom stuck forever.

**Transitions are deferred to the frame update.** A completion callback fires from inside
the animation system, where tearing down particles and sounds is not safe, so it only
records the wanted state; the change happens at a known point in the next frame.

**The phantom is invisible to the AI.** It strips itself from the AI-visible and
sound-reactive spatial categories at load, and re-strips the AI-visible flag on *every*
scheduled update — because something else keeps putting it back. Creatures never see a
phantom, never hear it, and never react to it. It is a player-facing effect only.

**It has one hit point.** Its health is set to a thousandth of a unit at spawn, so any hit
kills it, and the hit handler diverts a flying phantom into its shot animation. There is no
combat with a phantom; there is only shooting it before it reaches you.

## State

See [`phantom.h`](phantom.h.md).

## `Load`

**Contract** — reads flight speed, angular speed and the contact damage from the
configuration section, then for each of the four visible states reads a particle effect name
and, if one is named, creates its sound. Strips the AI-visible and sound-reactive spatial
categories. Allocates sounds; runs once.

## `net_Spawn`

**Contract** — when the spawn record carries no visual, picks one at random from the
section's visual list and *tells the server* through a change-visual event, so that the
choice is authoritative and survives a save. Then: requests the birth state, clears the
killer field (a guard against a crash with script-spawned phantoms), runs the base spawn,
aims itself at the current view entity, resolves four motions by name from the model,
asserts that three of them stop at their end, and performs the first state transition
immediately.

```text
FUNCTION net_Spawn(record)
  IF record has no visual
    visual = random entry from config.read(section, "visuals")
    record.set_visual(visual)
    send change-visual event to the server

  target_state = birth
  IF NOT base.net_Spawn(record) THEN RETURN failure

  target = current view entity
  health = 0.001                     # one hit kills

  # orient toward the target once, and seed the turn state from that orientation
  forward = normalize(target.position - position)
  up      = world up
  right   = cross(up, forward)
  (heading, pitch) = heading_and_pitch(forward)

  motion[birth]   = model.cycle("birth_0")
  motion[flying]  = model.cycle("fly_0")
  motion[contact] = model.cycle("contact_0")
  motion[shot]    = model.cycle("shoot_0")
  REQUIRE birth, contact and shot all stop at their end

  enter_state(target_state)
  visible = (current_state is past idle)
```

**Invariants** — the four motion names are literals. A model used for a phantom must carry
exactly them.

## `SwitchToState_internal` — the transition

**Contract** — performs one state change: runs the *outgoing* state's exit effect, then the
*incoming* state's entry effect, then records the new state. Idempotent on a no-op change.

```text
FUNCTION enter_state(new_state)
  IF new_state == current_state THEN RETURN
  transform = the phantom's transform translated to its centre
  update_action = none

  # exit effects, on the state being left
  SELECT current_state
    contact:
      play particles at the transform, auto-removed
      # the damage lands HERE, on leaving contact, not on entering it:
      # the phantom must finish its contact animation before it hurts anything
      IF distance(target.centre, own centre) < own radius
        deliver psychic damage of contact_hit to target
    shot:
      play particles at the transform, auto-removed
    otherwise: nothing

  # entry effects, on the state being entered
  SELECT new_state
    birth:
      play particles (auto-removed), play sound, play motion with a completion callback
    flying:
      update_action = fly
      play particles and KEEP the handle — this trail is owned and must be destroyed
      play sound looped, play motion with no callback (the flight loops)
    contact:
      update_action = drift
      play sound, play motion with a completion callback
    shot:
      update_action = drift
      play particles (auto-removed), play sound, play motion with a completion callback
    idle:
      update_action = die
      stop the OUTGOING state's sound and destroy the flight trail

  current_state = new_state
```

**Invariants**

- The damage is delivered on *leaving* contact and gated on still being within the
  phantom's own radius of the target, so a player who backs away during the contact
  animation takes nothing. That is the phantom's only counterplay besides shooting it.
- Entering the idle state stops the sound of the state being left, not of the state being
  entered — correct, since idle has no sound, but inconsistent with every other branch,
  which uses the incoming state. A rebuild should say "stop whatever was playing".

## The three per-frame actions

**Contract** — exactly one is bound at a time, by the transition.

- **fly** — advance toward the target, then check for contact: when the phantom's centre
  and the target's centre are within the sum of their radii, request the contact state and
  *hit itself* with a large firearm hit, which — because it has a thousandth of a hit point
  — kills it. The phantom therefore always dies from reaching its target; the contact state
  is its death throe.
- **drift** — advance toward the target and keep the trail and sound following, with no
  contact test. Used during both endings so the apparition keeps moving while it dissolves.
- **die** — destroy the object. The idle state is the terminus.

## `UpdatePosition` — the pursuit

**Contract** — turn-rate-limited homing. Each frame the phantom computes the heading and
pitch to the target, interpolates its own heading and pitch toward them at the configured
angular speed, and then moves forward along its *own* facing at the configured flight speed.

```text
FUNCTION update_position(target_position)
  (wanted_heading, wanted_pitch) = heading_and_pitch(target_position - position)
  heading = angle_lerp(heading, wanted_heading, angular_speed, frame_delta)
  pitch   = angle_lerp(pitch,   wanted_pitch,   angular_speed, frame_delta)

  facing = unit vector at (heading, pitch)
  rotate own transform to the new heading
  position = position + facing * speed * frame_delta
```

**Invariants** — it moves along where it is *looking*, not toward the target, so a phantom
that has to turn sharply overshoots and arcs back. That arc is the whole reason it reads as
a drifting apparition rather than a homing missile, and it comes entirely from the ratio of
flight speed to angular speed. Both are authored per phantom section; the defaults in code
(four metres per second, 1.7 radians per second) are overwritten at load.

## `Hit`

**Contract** — a hit on a *flying* phantom requests the shot state. The hit is then applied
normally, which kills it. A hit in any other state is applied but changes no state.

## `shedule_Update`

**Contract** — re-strips the AI-visible spatial category, runs the base update, and advances
the model's animation tracks. The phantom advances its own animation because it is not
driven by the creature animation machinery.

## `PsyHit`

**Contract** — constructs and sends a psychic-type hit event naming the phantom as both the
attacker and the weapon, with a fixed upward direction, no impulse and no bone. Sent as an
event rather than applied directly so that the authoritative side owns the damage.

## `save` / `load`

**Contract** — **both are empty.** The disabled bodies would have written and restored the
current state. The consequence: a phantom does not survive a save. Reloading a game with a
phantom in flight brings it back in its birth state, at the position the base entity
serialisation restored, aimed at whoever the view entity now is. The spawn code carries a
comment claiming the initial state is overridden on load, which was true of the disabled
version and is not true now.

## `net_Export` / `net_Import`

**Contract** — export writes health, three placeholder numbers, a timestamp, a flag byte,
four angles and the team, squad and group bytes; import reads the same shape back. The
angles are written as full-precision floats where the surrounding code's disabled
alternatives quantised them, and the *heading is written twice* while the bank is never
written at all, so a phantom's roll is not transmitted. The shape matches between the two
sides, so this is self-consistent — but it is also unreachable, since the shipped build has
no working transport. See [Seam: Networking transport](../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).

## Notes

**The target is re-resolved defensively** in three places as "the current view entity" when
the stored one is missing. A script can point a phantom at something else through the enemy
setter, but any of those three re-resolutions will silently snap it back to the player. A
rebuild should decide whether the script's choice is authoritative and then honour it
consistently.
