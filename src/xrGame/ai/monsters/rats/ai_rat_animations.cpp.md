# src/xrGame/ai/monsters/rats/ai_rat_animations.cpp

> The rat's clip table and the rule that picks a clip from nothing but speed, turn angle, and whether the rat is biting — the animation layer for a creature with no action table.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`../../../movement_manager.h`](../../../movement_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: resolves named clips against a skeleton and plays one per change

## Purpose

Every other creature in chapter 24 declares an action-to-animation table and the shared base
picks the clip. The rat, being outside that base, chooses directly from its own physical state
each time it is asked. The result is a selector with no notion of "what the rat is doing" at
all — it reads speed, turn and one flag, and that is the entire mapping from behaviour to
motion.

## `load_animations`

**Contract** — resolves eleven clip names against the rat's skeleton into handles, and starts
the first idle clip playing. Called once on spawn. Fails hard on a missing clip.

```text
RECORD RatClips
  death[2]        # two variants
  attack[3]       # three registered; only the third is ever played (see below)
  idle[2]         # variant 0 = milling, variant 1 = standing still
  walk, run       # each a directional set, resolved as a group
  run_attack      # running with a bite
  turn_left, turn_right
```

**Notes** — the walk and run entries resolve a whole *directional* set from one prefix (forward,
back, left, right), while every other entry is a single clip. That asymmetry is the model
format's, not a decision here.

**Two of the three attack clips are never played.** The selector below always chooses the
third. There is no recoverable reason for registering three; the likeliest reading is that
variation was intended and the random draw never written.

## `SelectAnimation`

**Contract** — chooses and, if it differs from what is playing, starts the rat's clip. Called
every frame from the position update and once directly on death. Its three declared arguments —
view direction, movement direction, speed — are all ignored; it reads the rat's own fields
instead.

```text
FUNCTION SelectAnimation()
  IF dead
    IF the currently playing clip is already one of the death clips
      keep it                                  # a death clip is never restarted
    ELSE IF the currently playing clip is the standing idle
      clip = death[0]                          # a rat killed standing dies the first way
    ELSE
      clip = death[uniform_integer(0 .. 2)]    # otherwise, at random
  ELSE IF biting
    clip = attack[2]
  ELSE IF turn_needed <= 15 degrees            # facing close enough to the target heading
    IF speed < 0.2
      clip = idle[standing ? 1 : 0]            # the standing budget decides which idle
    ELSE IF speed == attack_speed
      clip = run_attack
    ELSE IF speed == max_speed
      clip = run.forward
    ELSE
      clip = walk.forward
  ELSE
    clip = turn_left OR turn_right, by which way is shorter

  IF clip != currently_playing
    play(clip)
```

**Invariants**

- **A death clip, once started, is never replaced.** The first branch exists for exactly that:
  the routine runs every frame on a corpse too, and without the check the rat would restart its
  death animation forever.
- **A rat killed while standing still always dies the same way.** The standing idle is the
  variant played by a rat in the standing budget, so this is a deliberate pairing of poses — a
  frozen rat has a matching death. Any other pose picks at random.
- **The clip is chosen by comparing the speed against the authored values for equality.** This
  is the same brittle contract as the pitch-rate lookup in [`ai_rat.cpp`](ai_rat.cpp.md): it
  only works because [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) assigns speed from a
  fixed set of four and never interpolates. A rebuild that eases speed silently falls through
  to the walk clip.
- **Turning outranks moving.** Any turn wider than fifteen degrees plays a turn-in-place clip
  regardless of speed, which is consistent with the steering in
  [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) stopping the rat dead to turn.
- **Only the forward directional variants are ever used.** The rat never plays a strafe or a
  backward walk; it turns to face wherever it is going first.

**Notes** — the fifteen-degree threshold is written as a sixth of a right angle halved, and is
the same magnitude the steering uses to decide it must stop and turn. Keeping the two in step
is what stops the rat from sliding while playing a turn clip; a rebuild must change both or
neither.

The three ignored arguments are a leftover from the shared base's signature. Two call sites
pass real values and one passes a constant; none of it is read.
