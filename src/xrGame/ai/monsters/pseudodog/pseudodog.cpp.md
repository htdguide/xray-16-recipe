# src/xrGame/ai/monsters/pseudodog/pseudodog.cpp

> A leaping pack predator: the chapter's baseline creature with a full posture set, a corpse-dragging gait, a psi howl, and a leap that can rotate a quarter turn in mid-air.

**Needs** — [`pseudodog.h`](pseudodog.h.md) · [`pseudodog_state_manager.h`](pseudodog_state_manager.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../monster_sound_defs.h`](../monster_sound_defs.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../../../sound_player.h`](../../../sound_player.h.md)
**Used by** — [`pseudodog.h`](pseudodog.h.md)
**Tier floor** — T2: registers animation clips, gaits and jump data with the control layer at load

## Purpose

This is the chapter's clearest example of the pattern the whole chapter follows: almost nothing
here is behaviour. The file is a **declaration of data bindings** — which animation clip plays
for which action, which gait each clip moves at, which transitions exist between postures, and
which sound and jump parameters to load — and the behaviour comes from the shared creature base
reading those bindings.

What the pseudodog adds beyond the baseline is small and worth naming precisely: the dragging
gait and its clip, a psi-howl clip and its own sound bank, a threat-display clip that reuses
the howl, and a **rotating leap** — a jump the creature can enter from four different turn
offsets so it can commit to a leap and still land facing its target.

## State

```text
RECORD PseudoDog                    # on top of the shared creature base
  anger_hunger_threshold : real     # authored: "anger_hunger_threshold"
  anger_loud_threshold   : real     # authored: "anger_loud_threshold"
  became_angry_at        : int
  growling_since         : int
```

**Notes** — all four are dead. They are loaded and reset and nothing in the shipped creature
reads any of them, and the two named constants that go with them — a minimum time spent angry
and a maximum time spent growling — sit unused in
[`pseudodog_state_manager.cpp`](pseudodog_state_manager.cpp.md). Together they describe an
intended anger model: a dog becomes angry when hungry or when it hears something loud, stays
angry for a minimum period, and growls for at most a maximum one. None of it is implemented.
The two configuration keys are in the shipped data files, so a loader that rejects unknown keys
will fail on the original data.

## Construction and `reinit`

**Contract** — construction declares two capabilities to the control layer: this creature can
jump, and it can jump while rotating. `reinit` clears the dead anger timers and registers the
rotation data: **four named clips and a quarter turn**, meaning the creature has a leap clip for
each of four turn offsets and may rotate up to a quarter turn per leap.

**Notes** — the four clips are named by bare index in the shipped data. The quarter turn is the
creature's agility in the air and is the number that makes a pack able to converge on a moving
target.

## `Load` — the bindings

**Contract** — reads the creature's section: the two anger thresholds, then the whole animation
set, the posture transitions and the action bindings. Does not read any behaviour parameter;
everything behavioural comes from the base.

Four kinds of binding are declared, and the kinds matter more than the contents:

**Substitutions.** A clip is replaced by another while a condition holds. Two conditions are
used: *damaged* swaps walking and running for their limping variants, and *turning hard while
running* swaps the run for a banking left or right variant. Substitution is how a creature
changes its look without any state knowing.

**Acceleration chains.** Walking accelerates into running, and damaged walking into damaged
running. The chain tells the movement layer which clip to blend into as speed rises, so a dog
does not pop from a walk cycle to a run cycle.

**Clips with gaits.** Twenty-seven clips, each bound to an animation identifier, a name prefix
the loader expands into a numbered bank, a gait from
[`../monster_velocity_space.h`](../monster_velocity_space.h.md), and a posture (standing,
sitting or lying). The gait is what makes a clip's playback and the creature's actual movement
agree.

**Posture transitions.** Five, each naming the clip that bridges two postures: lying to
sleeping, sleeping to standing, sitting to lying, standing to sitting, sitting to standing.
Declared non-blocking, so the creature may be interrupted mid-transition.

**Action bindings.** Fourteen, mapping an abstract action the states request — idle, walk, run,
eat, sleep, rest, drag, attack, sneak, look around — onto a clip. Two are worth noting: *rest*
maps to sitting rather than standing, which is what makes idle dogs sit; and *sneak* maps to
the plain forward walk rather than to the sneak clip, so the sneak gait plays a walk cycle. The
latter is a data shortcut the shipped animation set forces.

**Notes** — the chapter's claim that creatures differ by data is literally true here: this
routine is the difference between a pseudodog and most other four-legged creatures.

## `reload`

**Contract** — loads two things beyond the base's own: the **psi-attack sound bank**, from the
key `sound_psy_attack`, registered under the species-extension tag declared in
[`pseudodog.h`](pseudodog.h.md), at a priority just below `high`, on the base channel, and
attached to the head bone so it emits from the right place on the model; and the **leap data**,
three clips naming the launch, the glide and the landing, with the running gait for both the
approach and the flight.

**Notes** — the sound is registered at `high + 3`, using the priority ladder's gaps exactly as
[`../monster_sound_defs.h`](../monster_sound_defs.h.md) intends: the howl outranks everything
ordinary but yields to the three most urgent things the creature can say.

The three leap clip names are misspelled consistently in the shipped data. The misspelling is
frozen by that data.

## `handle_special_animation_flags`

**Contract** — the hook the animation layer calls when the currently playing clip carries
authored special flags. Two are recognised: the psi-attack flag runs the howl clip as a
one-shot command sequence, and the threat flag sets the threat clip as the current animation.

**Notes** — this is how an *animation* triggers behaviour rather than the reverse. The flags
are authored into the clip data, so which frame of which clip fires the howl is a data
decision, not a code one. The two are handled differently on purpose: the psi attack is
sequenced, meaning it runs to completion and blocks other animation commands, while the threat
is just set and may be overridden immediately.

## `hit_entity_in_leap`

**Contract** — applies damage when a leap connects, reading the power, impulse and impulse
direction from the *named leap clip's* own authored attack parameters rather than from the
creature's section.

**Notes** — binding the damage to the animation, not to the creature, means the leap's force
is tuned by the animator next to the clip that delivers it. The clip is named literally here,
so a creature with a differently-named leap gets no damage at all — a silent coupling between
code and data that a rebuild should make explicit.

## `create_state_manager`

**Contract** — builds the pseudodog's brain. It exists as an overridable step, called from the
creature's construction, purely so that [`psy_dog.cpp`](psy_dog.cpp.md) can substitute a
different brain over the same body and the same bindings. That is the mechanism by which the
psi dog is "a pseudodog with one extra state".
