# src/xrParticles/particle_actions_collection_io.cpp

> The frozen byte layout of an authored action: for each of the thirty-one kinds, which
> parameters appear in the stream, in what order, and which are derived rather than stored.

**Needs** — [`particle_actions_collection.h`](particle_actions_collection.h.md) · [`particle_core.h`](particle_core.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [`xrEngine/defines.h`](../xrEngine/defines.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: domains are copied out of the stream as raw memory images, so the
reader depends on the record's exact size and field offsets.

## Purpose

An effect ships with the game as a compiled action list, and this file is the only description
of that list's bytes. Nothing here is a decision the rebuild gets to make; it is a format to
be matched. It is split from the behaviour for exactly that reason — the behaviour can be
rewritten freely, the bytes cannot.

The separation also isolates the one place in the module that consults which of the three
games' data is loaded.

## State

Stateless. The layout it defines is:

```text
# The action list blob, as it appears inside a compiled effect definition.
# All integers are 32-bit little-endian; all reals are 32-bit little-endian.
action_list := count : int32, then `count` actions

action := kind   : int32          # ActionKind, selects which action to construct
          flags  : int32          # the action's flag word
          kind   : int32          # the SAME kind code again
          payload                 # kind-specific, per the table below
```

**Invariants** — the kind code appears twice, before and after the flag word. The reader
consumes the first copy to decide which action to build and then the action's own common
header consumes the flag word and the second copy. A rebuild that reads it once will
desynchronize on the first action.

The flag word carries the "parameters rotate with the emitter" bit; the source action
additionally claims its three top bits for single-size, silent and vertex-B-tracking. A
reader must not mask the word down to the bits it recognizes.

## Per-kind payloads

`domain` below means the domain record copied verbatim — 68 bytes, see
[`particle_core.cpp`](particle_core.cpp.md) — *including its derived fields*. The reader does
not re-derive the cached normals, reciprocals, plane offsets or orthonormal frames; whatever
the authoring tool computed is what the simulation uses.

| Kind | Payload, in order |
|---|---|
| `Avoid` | domain (position), real look_ahead, real magnitude, real epsilon |
| `Bounce` | domain (position), real one_minus_friction, real resilience, real cutoff_sqr |
| `CopyVertexB` | int32 copy_pos, read as a boolean |
| `Damping` | vec3 damping, real vlow_sqr, real vhigh_sqr |
| `Explosion` | vec3 center, real velocity, real magnitude, real stdev, real age, real epsilon |
| `Follow` | real magnitude, real epsilon, real max_radius |
| `Gravitate` | real magnitude, real epsilon, real max_radius |
| `Gravity` | vec3 direction |
| `Jet` | vec3 center, domain (acceleration), real magnitude, real epsilon, real max_radius |
| `KillOld` | real age_limit, int32 kill_less_than |
| `MatchVelocity` | real magnitude, real epsilon, real max_radius |
| `Move` | *(empty)* |
| `OrbitLine` | vec3 point, vec3 axis, real magnitude, real epsilon, real max_radius |
| `OrbitPoint` | vec3 center, real magnitude, real epsilon, real max_radius |
| `RandomAccel` | domain |
| `RandomDisplace` | domain |
| `RandomVelocity` | domain |
| `Restore` | real time_left |
| `Scatter` | vec3 center, real magnitude, real epsilon, real max_radius |
| `Sink` | int32 kill_inside, domain (position) |
| `SinkVelocity` | int32 kill_inside, domain (velocity) |
| `Source` | domain position, domain velocity, domain rotation, domain size, domain color, real alpha, real particle_rate, real age, real age_sigma, vec3 inherited_velocity, real inheritance_factor |
| `SpeedLimit` | real min_speed, real max_speed |
| `TargetColor` | vec3 color, real alpha, real scale, **then** real time_from, real time_to |
| `TargetSize` | vec3 size, vec3 scale |
| `TargetRotate`, `TargetRotateD` | vec3 rotation, real scale |
| `TargetVelocity`, `TargetVelocityD` | vec3 velocity, real scale |
| `Vortex` | vec3 center, vec3 axis, real magnitude, real epsilon, real max_radius |
| `Turbulence` | real frequency, int32 octaves, real magnitude, real epsilon, vec3 offset |

Two entries in that table need their own paragraph.

**`TargetColor` is version-dependent.** In data from the first of the three games, the record
ends after the scale and the two time fields are absent; the action keeps its constructed
defaults of 0 and 1, which is the whole-lifetime window that engine always applied. In the
later games both fields are present. The reader decides by consulting a process-wide flag set
at startup from which game's installation was detected — the only place in the chapter that
does. The decision cannot be made from the stream itself, because an action is not
length-prefixed: a reader that guessed wrong would consume the next action's kind code as a
time bound and desynchronize the rest of the list. A rebuild must pass the data generation
into the loader.

**`Source` stores the inherited velocity it will immediately overwrite.** That field is
recomputed from the emitter's motion on every placement change, so the stored value is dead on
arrival. It occupies its bytes and must be skipped correctly.

## After loading

Every action with a local/world parameter pair reads *one* copy from the stream and assigns it
to both. The stored value is therefore the local-space one, and it is valid as a world-space
value only until the emitter is first placed.

```text
# the pattern, repeated for every spatial action
read the world-space field from the stream
local_copy = world_copy
```

**Notes** — this is why an effect looks correct for exactly one frame if its placement is never
applied: the authored local parameters double as world parameters at the origin.

## `load` / `save` for each action

**Contract** — read (or write) that action's payload after the common header, as tabulated.
Neither validates: the stream is trusted, lengths are not checked, and a truncated or
mismatched stream yields garbage parameters rather than an error. Loading allocates nothing
beyond the action itself.

**Notes** — **the writers are not the inverse of the readers.** A writer emits the flag word
and one kind code; a reader expects a kind code, the flag word, and a second kind code. Saving
a list and loading it back therefore does not round-trip. This has never surfaced because
nothing in the engine saves: authored files are produced by an editor that is not part of this
codebase and that writes the leading selector itself, and the engine only reads. A rebuild
should write the reader first, derive the writer from it, and treat the asymmetry above as
what it is — a format documented only by its reader.
