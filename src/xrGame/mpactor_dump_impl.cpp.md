# src/xrGame/mpactor_dump_impl.cpp

> Declares which of the multiplayer actor's numbers are worth cheating with: sixteen movement and weapon-dispersion values, reported as they stand in memory.

**Needs** — [`actor_mp_client.h`](actor_mp_client.h.md) · [`mp_config_sections.h`](mp_config_sections.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field serialization

## Purpose

One method, in its own file so that the actor's anti-cheat exposure is written down in one
place rather than scattered through the actor's many implementation files. It answers the
anti-cheat system's question "what are you actually running with" for the player character.

## State

`Stateless.`

## `DumpActiveParams`

**Contract** — writes the multiplayer actor's live movement and dispersion parameters into a
report document under a given section name, as named real values. Reads only; no side
effects beyond the write.

The reported set, which is the load-bearing content of this file:

```text
  # locomotion
  walk acceleration              # the base from which every factor below scales
  jump speed
  run factor, run-backward factor, walk-backward factor
  crouch factor, climb factor, sprint factor
  walk strafe factor, run strafe factor

  # weapon dispersion as the actor affects it
  base dispersion, aimed dispersion
  dispersion per unit velocity
  dispersion per unit acceleration
  dispersion while crouching
  dispersion while crouching and not accelerating
```

**Invariants** — this list is the complete set of actor-side numbers a server will check.
Anything the actor holds that is not here can be tampered with undetected. Speed and
accuracy are the two things worth cheating in a shooter, which is exactly what the two
groups cover; health and damage are checked through the actor's *configuration* sections
instead (see [`mp_config_sections.cpp`](mp_config_sections.cpp.md)), because those are read
fresh rather than cached in fields.

**Notes** — the keys are written under the field names the engine uses internally, so the
server's comparison is against the configuration keys of the same names. A rebuild that
renames these fields must keep the reported *keys* stable, or a rebuilt client and an
original server can never agree.

Every value is reported as a real number, with no quantization. The comparison downstream
is therefore exact, which means a rebuild whose parsing or arithmetic produces a
bit-different value from the same configuration text will be indistinguishable from a
cheat.

The dispersion group distinguishes crouching-with-acceleration from
crouching-without-acceleration but has no equivalent standing pair. Standing accuracy is
derived from the base value and the velocity and acceleration factors; crouching is the
only posture given its own two-point curve, because it is the posture competitive play
spends its aimed shots in.
