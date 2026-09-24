# src/xrGame/script_game_object4.cpp

> The creature sound player, the wounded and sight states, inventory boxes, bone-attached particles, the absolute health write, and the flat table of class predicates.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`sight_control_action.h`](sight_control_action.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`stalker_sound_data.h`](stalker_sound_data.h.md) · [`InventoryBox.h`](InventoryBox.h.md) · [`ZoneCampfire.h`](ZoneCampfire.h.md) · [`PhysicObject.h`](PhysicObject.h.md) · [`Artefact.h`](Artefact.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation, plus one bone-visibility check

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across.
Four unrelated subjects share it, and the only one with real structure is the creature
**sound player** — the mechanism by which a creature's vocalizations are declared once and
then requested by kind rather than by file.

## State

`Stateless.`

## The creature sound player

**Contract** — a creature does not play sounds by name. It **registers a sound family**
once, under a caller-chosen identifier, and afterwards asks for that identifier; the player
picks one of the family's variants, respects priorities between families, and places the
emitter on a named bone.

```text
FUNCTION add_sound(prefix, max_count, ai_kind, priority, mask, id, bone_name) -> int
  # prefix     : the authored sound name stem; the player appends an index
  # max_count  : how many numbered variants exist under that stem
  # ai_kind    : the perception attribute listeners will read
  # priority   : which family wins when two want to play at once
  # mask       : the group bits this family belongs to, for muting whole groups
  # id         : the caller's own identifier for this family, used by every later call
  # bone_name  : where on the skeleton the emitter sits
```

**Invariants**

- The identifier is chosen by the **caller**, not returned by the engine. Two scripts
  registering the same identifier on one creature collide silently, which is a real hazard
  with several mods loaded.
- The shorter form defaults the bone to the creature's head bone, spelled as the literal
  bone name shared by every humanoid skeleton in the shipped data. That literal is a hard
  dependency on the models: a rebuild keeps it, because the alternative is a per-model
  configuration entry that does not exist in the data.
- `set_sound_mask` mutes and unmutes whole groups at once, and is where the *"is this
  creature alive"* check lives — stated in the original as an assertion with the comment
  "stalkers talk after death?". The check is debug-only, so in a shipping build a corpse
  can be given a sound mask. A rebuild should refuse it outright.
- `play` takes up to five timing bounds — earliest and latest start, earliest and latest
  stop, and an explicit instance identifier. The bounds are *randomization windows*, not
  delays: the player picks within them, which is what keeps a squad from grunting in
  unison. The same reasoning as the smart-cover dwell times.
- `add_combat_sound` is the same registration restricted to stalkers, with a stalker-
  specific data block attached that lets the sound carry combat context. It exists because
  a combat vocalization must know who it is about.
- `active_sound_count(only_playing)` counts registered families, or only the ones currently
  sounding. Its fallback is −1, per the facade's convention.

**Notes**

This indirection — register a family, then request by identifier — is what makes a
creature's voice moddable without touching the engine, and it is the same shape as the
monster vocalization roles in
[`script_sound_action_script.cpp`](script_sound_action_script.cpp.md). A rebuild that lets
scripts play creature sounds by filename loses the priority arbitration and the muting
groups along with it.

## `is_body_turning`

**Contract** — whether the creature is still rotating toward its intended facing. For a
stalker the answer covers **both** head and body; for any other creature only the body.

```text
FUNCTION is_body_turning -> bool
  IF not a creature THEN log script error; RETURN false
  IF not a stalker THEN
    RETURN body.target_yaw differs from body.current_yaw
  RETURN head.target_yaw differs from head.current_yaw
      OR body.target_yaw differs from body.current_yaw
```

**Invariants** — only the yaw is compared; pitch is ignored deliberately, since a creature
looking up or down is not "turning". The comparison is against zero difference with the
engine's own tolerance, so a creature within tolerance of its target reads as settled.

## Wounded and sight state

**Contract** — `wounded` reads and writes a stalker's wounded state, the incapacitated
condition distinct from being dead or merely hurt. `critically_wounded` reads the
irrecoverable variant on any creature. `sight_params` returns a snapshot of what the
creature is currently looking at: the sight kind, the object it tracks if any, and the
point in space.

**Invariants** — the sight snapshot's failure value sets every field to an out-of-band
marker — no object, the largest representable coordinates, and the "no such kind" sight
type — rather than a zero vector. A script that ignores the failure gets a position it
cannot mistake for a real one, which is the opposite convention from the rest of the facade
and the better one.

## Inventory box

**Contract** — five predicates and writers over a container: whether it is empty, whether
it is closed and why, and whether its contents may be taken.

**Invariants** — all of them answer **false on the wrong kind of object and log nothing**.
A script asking a crate-shaped thing that is not a crate gets a plausible answer with no
diagnostic. The writers double as success flags — they answer true when they applied — which
is the only place in the facade where a mutator reports whether it worked.

The closed state carries a **reason** string, which is what the player is shown when they
try to open it. A locked container with no reason gives no feedback at all.

## Bone-attached particles

**Contract** — `start_particles(effect, bone)` and `stop_particles(effect, bone)` play an
effect at a named bone of the object's skinned model.

```text
FUNCTION start_particles(effect_name, bone_name)
  IF the object cannot play particles THEN RETURN          # silently
  REQUIRE the object has a skinned model
  bone = resolve bone_name on that skeleton
  REQUIRE bone resolved
  IF bone is currently visible THEN
    play effect at bone, oriented up, with the reserved slot identifier
  ELSE
    log script error "bone is not visible now"
```

**Invariants**

- An **invisible bone refuses the effect**. Bones are hidden when a body part is severed or
  a model variant does not include it, and an effect attached to a hidden bone would
  render at the skeleton's origin.
- A bone name that does not resolve **fails hard** rather than logging, unlike almost
  everything else in the facade. The justification is that a typo'd bone name is an
  authoring error that will never work, whereas a hidden bone is a transient state.
- Both calls use one fixed slot identifier, so a script can only have **one** bone-attached
  effect at a time per object; starting a second silently replaces the first. The number
  chosen is arbitrary and has no discoverable meaning beyond being outside the range the
  engine uses for its own effects.

## `set_health_ex`

**Contract** — writes health **absolutely**, where the facade's `health` writer adds a
delta. The value is clamped to the range from slightly below zero to one.

**Invariants** — the lower bound is deliberately *below* zero, so that a script can write a
health that is unambiguously dead rather than one that rounds to exactly zero and may read
as alive. This method exists because the delta convention on the condition axes is a frozen
trap; see
[`script_game_object.cpp`](script_game_object.cpp.md#condition-axes).

A wrong-kind object is ignored silently.

## Class predicates

**Contract** — twenty-two yes-or-no questions of the form "is this object an X", one per
entity kind a script commonly needs to distinguish: living entity, inventory item,
inventory owner, actor, creature, weapon, outfit, scope, silencer, grenade launcher,
magazined weapon, under-barrel weapon, restrictor, stalker, anomaly, monster, trader,
held item, artefact, ammunition, inventory box.

**Notes**

These exist because the facade is one flat type: a script cannot ask what kind of thing it
holds any other way, and before these existed it had to call a kind-specific method and
watch for the error. They are the correct fix and a rebuild should offer the same set — but
as *one* query taking a kind, not twenty-two methods, unless the shipped names must be
kept, which they must.

A further dozen predicates are present but disabled in the source — car, helicopter,
holder, medkit, antirad, food, explosive, script zone, projector, missile, grenade, bottle,
torch, physics-shell holder. Their absence is not a decision about the model; they were
commented out to avoid the includes, and a rebuild should simply provide the whole set.

Three accessors in the same spirit return the object *as* a kind rather than answering a
question — campfire, artefact and physics object — and each answers nothing rather than
logging when the object is something else.
