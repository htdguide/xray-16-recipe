# src/xrGame/ai/monsters/basemonster/base_monster_feel.cpp

> Perception in, damage out: what a creature hears and what it may see, and everything that happens on the player's screen when a creature hits him.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`ai_monster_effector.h`](../ai_monster_effector.h.md) · [`Actor.h`](../../../Actor.h.md) · [`ActorEffector.h`](../../../ActorEffector.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`sound_player.h`](../../../sound_player.h.md) · [`UIGameCustom.h`](../../../UIGameCustom.h.md) · [`script_game_object.h`](../../../script_game_object.h.md) · [Seam: Audio device](../../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: event-driven perception plus camera and screen effects with per-frame timing

## Purpose

Two halves that share a file because both are about the boundary between a creature and the
world it senses.

The **inbound** half is perception: which sounds reach a creature and what it does with
them, which objects are worth running a vision test against, and how a hit is classified
into a side and remembered. The important decisions here are all *filters* — perception is
event-driven, and the cost of a creature is dominated by how much it refuses to consider.

The **outbound** half is what a creature's blow does to the player: the damage event, a claw
mark drawn on the screen oriented to the blow's direction, a brief loss of movement control,
and a camera effector chosen from eight directional variants.

## `on_sound_heard`

**Contract** — the sound-perception entry, called by the senses layer for every sound event
that reaches this creature. Filters, classifies, and either records the sound or routes it
to the help-call path. Fires the scripted sound callback. Does nothing for a dead creature.

```text
FUNCTION on_sound_heard(source, type, user_data, position, power)
  IF not alive THEN RETURN
  IF source is myself THEN RETURN
  IF the sound carries AI user data THEN let the creature inspect it
  IF the type is the unknown marker THEN RETURN

  IF distance from my centre to the sound > settings.max_hear_dist THEN RETURN

  IF the source is not an entity AND the sound is an item being hidden THEN RETURN
      # a weapon dropped by a dying enemy must not read as a new threat

  IF the source is an entity that is not my enemy THEN
    sound_memory.consider_as_help_call(type, source's navigation vertex)
    RETURN

  IF the sound is a gunshot THEN power = 1        # a shot is always at full loudness
  IF the sound is a bullet impact within 2 units THEN
    hit_memory.record(source, front)              # near misses count as being attacked

  fire the scripted sound callback

  IF power >= settings.sound_threshold THEN
    sound_memory.record(source, type, position, power, now)
```

**Invariants** — a sound from a non-enemy never enters the sound memory. It is offered to
the help-call path instead, which is how a creature learns where its packmates are fighting
without treating them as threats.

Gunshot loudness is *overridden* to maximum rather than attenuated by distance, so a shot
anywhere within hearing range is as loud as a shot next door. That is what makes firing a
weapon the loudest thing a player can do.

**Notes** — the near-miss rule (a bullet impact within two units is recorded as a *frontal
hit*) is why creatures charge a player who is missing them. It is a deliberate cheat: the
creature has no way to know where the shot came from, so it records the impact's source
as the attacker and the side as front.

The unknown-sound marker is compared against an all-ones value with a comment in the
original acknowledging the magic number. A rebuild gives the type an explicit "none" member.

## `is_worth_seeing`

**Contract** — the vision-relevance filter, asked before any expensive visibility test.
Admits only entities that are marked visible to AI, and only when the creature is alive and
awake. A *friendly* living entity is rejected — but not before its enemies are copied into
this creature's own enemy list.

```text
FUNCTION is_worth_seeing(object) -> bool
  IF not alive THEN RETURN false
  IF the object is not an entity THEN RETURN false
  IF the object is not in the AI-visible set THEN RETURN false
  IF asleep THEN RETURN false                # a sleeping creature sees nothing

  IF the object is a living entity that is not my enemy THEN
    IF it is a creature AND enemy transfer is not suppressed THEN
      copy its enemies into my enemy memory
    RETURN false

  RETURN true
```

**Invariants** — the enemy transfer happens in the *filter*, as a side effect of rejecting a
friend. That is the mechanism by which alarm spreads through a pack by sight: seeing a
packmate is enough to inherit whatever it is angry at. It can be suppressed per creature,
which is how a creature that must stay calm near agitated packmates is expressed.

**Notes** — sleep is modelled purely as a vision filter here. A sleeping creature still
hears, and waking is driven by sound and by hits, never by sight.

## `on_hit_signal`

**Contract** — called when the creature is struck. Treats the hit as a loud gunshot from the
attacker's position, plays the damage sound, classifies the hit into one of four sides,
plays the matching reaction effect, records it, lowers morale, fires the scripted callback,
and — if the attacker was neutral — makes it an enemy.

```text
FUNCTION on_hit_signal(amount, local_direction, source, bone)
  IF not alive THEN RETURN
  on_sound_heard(source, gunshot, none, source.position, 1.0)
      # being hit tells me where the shooter is, exactly as if I had heard the shot
  play(take_damage sound)

  IF bone is not a real bone THEN RETURN

  yaw = heading of the hit direction in my local frame, normalized to [0, 2PI)
  side = front
  IF  45° <= yaw <=  135° THEN side = left
  ELSE IF 135° <= yaw <= 225° THEN side = back
  ELSE IF 225° <= yaw <= 315° THEN side = right

  animation.play_reaction_effect(side)
  hit_memory.record(source, side)
  morale.on_hit()
  fire the scripted hit callback
  IF the source is neutral toward me THEN enemy_memory.add(source)
```

**Invariants** — the four quadrants are 90 degrees each centred on the axes, with front
spanning the wrap point. The boundaries are inclusive on both sides, so a hit exactly on a
boundary falls in the later branch; that is harmless.

**Notes** — routing the hit through the sound path is the mechanism that makes a sniped
creature run toward the sniper: it has no other way to locate an attacker it cannot see.

Turning a neutral attacker into an enemy — but not a hostile one, which is already an
enemy, and not a friendly one — is the whole of the "you shot me, now we have a problem"
rule.

## `hit_entity`

**Contract** — the creature's outgoing melee damage. Issues a damage event against its
current enemy only, and when that enemy is the player, adds the screen feedback. Does
nothing without a living creature, a valid target, or a current enemy.

```text
FUNCTION hit_entity(target, damage, impulse, local_dir, type, draw_marks)
  IF not alive OR target is gone OR I have no enemy THEN RETURN
  IF target is not my current enemy THEN RETURN

  world_dir = my transform applied to local_dir, normalized
  send a damage event: source = me, weapon = me, direction = world_dir,
                       power = damage, bone = the target's root bone,
                       impulse, type

  IF the target is the player AND draw_marks THEN
    draw_claw_marks(world_dir)
    lock the player's acceleration for min(damage, 1) * 1000 ms
    IF no big-creature-hit camera effector is already running THEN
      start_directional_camera_effector(local_dir, damage)

  morale.on_attack_success()
  record the time of this successful attack
```

**Invariants** — a creature can only damage the entity it currently considers its enemy.
There is no friendly fire between creatures and no incidental damage to bystanders from
melee.

The damage always lands on the target's **root** bone with a zero offset, so creature melee
never has a hit location. Only the direction matters.

**Notes** — the acceleration lock is the mechanic that makes being mauled feel like being
mauled: the player's movement input is damped for up to a second, scaled by the damage, so
a strong blow briefly takes control away. The one-second ceiling is a named constant with no
derivation.

## `draw_claw_marks` (private step)

**Contract** — adds a screen overlay of claw marks, rotated to match the blow's direction
relative to the camera and offset from centre along that direction.

```text
FUNCTION draw_claw_marks(world_dir)
  overlay = add the claw-mark screen element
      # the two older games keep it up for a fixed 3 seconds; the newest uses
      # the element's own configured lifetime
  angle = (heading of the reversed hit direction) - (camera heading)
  overlay.rotation = angle
  overlay.position = centre offset by (500 * sin angle, 400 * cos angle)
```

**Notes** — the offsets of 400 and 500 pixels are in the UI's own coordinate space and place
the marks toward the edge of the screen on the side the blow came from. The difference
between them is the screen's aspect. Neither is derived.

The three-second lifetime applies only in the two older games' compatibility modes; the
newest passes a sentinel meaning "use the element's configured lifetime". That is a genuine
per-game behaviour difference, not a refactoring artefact.

## `start_directional_camera_effector` (private step)

**Contract** — chooses one of eight camera effectors by the angle between the camera's
facing and the blow's direction, and whether the blow came from above or below, then starts
it scaled by the damage. Does nothing if such an effector is already running.

```text
FUNCTION start_directional_camera_effector(dir, damage)
  angle = |angular difference between camera heading and dir heading|   # in [0, PI]
  from_above = the cross product of the two points upward

  IF angle <= 22.5°                    THEN variant = 2       # dead ahead
  ELSE IF angle <=  67.5°              THEN variant = from_above ? 5 : 7
  ELSE IF angle <= 112.5°              THEN variant = from_above ? 3 : 1
  ELSE IF angle <= 157.5°              THEN variant = from_above ? 4 : 6
  ELSE                                      variant = 0       # from directly behind

  start the effector named "<creature's actor_hit_effect>_<variant>" at strength `damage`
```

**Invariants** — eight variants over five angular bands: the two extreme bands (ahead and
behind) have no up/down distinction, and the three middle bands each split into two. That is
why the numbering is 0 through 7 with an irregular assignment — the numbers are section-name
suffixes, and the mapping is authored in data, not derived.

**Notes** — the section prefix comes from the creature's own configuration, so each creature
kind shakes the camera its own way. The guard against starting a second one means a creature
being mauled by three dogs produces one shake, not three.

## `attack_effector`

**Contract** — applies the creature's configured attack effector to the player: a camera
shake and a post-process wash, both from the settings block. Does nothing when the player is
not the current view entity.

## `deal_psy_damage` / `deal_wound_damage`

**Contract** — two direct damage events a creature can issue outside melee. The psi variant
targets no bone, has no impulse, and is typed as telepathic; the wound variant targets the
root bone with a direction and an impulse. Both name the creature as both source and weapon.

**Notes** — the psi hit's direction is straight up, which is arbitrary and unread: telepathic
damage has no direction. It is there because the damage event requires one.

## critical wounds

**Contract** — two hooks. The suitability check refuses a critical wound while the sequencer
channel is unavailable or the creature is not in a standing animation. The start hook picks
the head, torso or legs animation by the recorded wound type and hands it to the
custom-ability manager.

**Invariants** — requiring a standing animation is what stops a creature entering a
dramatic wound animation from a lie or a sit, where the transition would visibly teleport it.
