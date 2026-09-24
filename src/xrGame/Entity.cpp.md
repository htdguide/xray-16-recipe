# src/xrGame/Entity.cpp

> The thing that can be damaged and can die: health, team membership, the damage path, and the once-only bookkeeping of who killed it and when.

**Needs** — [`Entity.h`](Entity.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`damage_manager.h`](damage_manager.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`monster_community.h`](monster_community.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: scalar bookkeeping and a membership registry

## Purpose

The layer between "an object with a physical shell" and "a creature with a brain". It adds
exactly three things, and each is small but load-bearing:

**Health, and the fact that death is a threshold on it.** There is no separate alive flag —
alive means health above zero, everywhere, and there is no way for the two to disagree.

**Team, squad and group.** Three integers that place the entity in a three-level hierarchy,
and a *registration* in that hierarchy which must be maintained exactly: registered on
spawn while alive, unregistered on death and on destruction, and moved atomically when the
affiliations change. Getting it wrong leaves the hierarchy holding a dead pointer, which is
the conformance invariant about registries.

**The killer, recorded once.** When something dies, the entity that killed it is recorded,
a death event is broadcast, and the record is *sticky*: a second attacker cannot overwrite
it. Three minutes later the record is cleared, because holding a killer identifier
indefinitely keeps the killer's entity alive in the alife simulation for no reason.

## State

```text
RECORD Entity
  condition         : entity condition   # owns health; see EntityCondition.cpp
  id_Team, id_Squad, id_Group : int      # the three-level affiliation
  registered_member : bool               # currently in the hierarchy registry
  killer_id         : optional<entity id>
  level_death_time  : int                # real milliseconds, for timeouts
  game_death_time   : int                # in-world time, for the alife simulation
  body_remove_time  : int                # how long the corpse stays, default 600000
  morale            : real
  food              : real
```

Invariants, and these are the file's whole substance:

- `registered_member` must exactly mirror the hierarchy registry's contents. It is set at
  spawn only when alive, cleared on death, cleared on destruction, and asserted before
  every team change.
- **the killer is written at most once per death.** A second kill attempt on an entity
  that already has a killer is ignored, and a self-kill is only recorded if no killer is
  set.
- a spawn record naming a killer while reporting positive health is a hard failure — the
  two can never be consistent.
- the two death timestamps are deliberately different clocks: one real, one in-world. The
  timeouts below use the real one; the alife simulation uses the in-world one.

## `Hit` — the damage path

**Contract** — the ordering here is the contract, and it is documented in the source with
an explicit "must be last".

```text
FUNCTION hit(blow)
  # 1. express the blow's direction in the entity's own frame, inverted
  #    (the impulse points INTO the body)
  local_dir = -(inverse(transform) applied to blow.direction)

  # 2. physical response first, so the body is already moving when the
  #    animation reacts
  IF blow carries an impulse THEN apply_impulse(blow.impulse, world_dir, local_dir)

  # 3. health
  lost = apply_to_condition(blow.damage)

  # 4. the reaction — flinch animation, sound, AI notification — is given the
  #    health ACTUALLY lost, not the damage dealt, so armour is visible in it
  IF the blow names a bone THEN hit_signal(lost, local_dir, attacker, bone)

  # 5. death, exactly once, and only on the authoritative side
  IF we own this entity AND health <= 0 AND not already dying AND no killer recorded THEN
    kill(attacker)

  # 6. the base's own handling, LAST
  base.hit(blow)
```

**Invariants** — the reaction receives lost health rather than raw damage. A shot that
armour absorbs entirely produces no flinch, which is how the player reads that their
armour worked.

## `CalcCondition`

**Contract** — applies damage to health, on the authoritative side and only while alive,
and returns the damage. Health is floored at -1000 rather than at zero, so that overkill is
representable — the amount by which a blow exceeded the remaining health is what decides
dismemberment elsewhere.

**Notes** — the floor is a magic constant with no discoverable derivation. It only has to
be below any plausible overkill.

## `KillEntity`

**Contract** — records the killer and broadcasts the death. The single most guarded routine
in the file, because it must fire exactly once per death and because scripts are allowed to
veto it for the player.

```text
FUNCTION kill(killer_id, bypass_actor_check = false)
  IF single player AND this is the player THEN
    eject the player from any vehicle
    IF the script veto is compiled in AND not bypassed THEN
      fire the "actor before death" script callback and RETURN
      # a script may then call back in with the bypass set, or not — and the
      # player survives

  IF killer_id is not this entity THEN
    (a second killer arriving is a diagnosable error, asserted in debug)
  ELSE
    IF a killer is already recorded THEN RETURN      # self-kill loses to any other

  killer_id = the killer
  stamp both death times

  IF not already being destroyed THEN
    broadcast a death event naming the killer,
    on the authoritative side only, reliably and ordered
```

**Invariants** — death is *always* announced as an event and applied when the event returns,
never applied inline. That is what makes death happen at the same moment on every client,
and it is why `Die` is reached through the event handler rather than called here.

**Notes** — the script veto is a build option, added after the original. It hands scripts
the ability to prevent the player's death entirely — the common use is a quest-critical
rescue — and requires the script to re-enter with the bypass flag if it decides the player
should die after all.

## `Die`

**Contract** — applying death. Stamps the death time if it was not already stamped, marks
the entity ready to be saved, forces health negative, and **unregisters from the
hierarchy**. The unregistration is the reason this exists as a separate step from the
health reaching zero.

**Invariants** — health is set to exactly -1 rather than left at whatever the killing blow
produced. So a saved dead entity always reloads with the same health, and the overkill
amount does not survive the save.

## `OnEvent`

**Contract** — handles the death event: reads the killer's identifier, resolves it, logs the
kill in multiplayer, and applies death. The resolution is allowed to fail — the killer may
already be gone — and death proceeds regardless.

## `net_Spawn`

**Contract** — brings the entity up from its authoritative record. The order matters:
everything that decides *whether the entity is alive* happens before the hierarchy
registration, and the registration happens before the base spawns.

```text
FUNCTION spawn(record)
  clear both death times and the killer

  IF the record is a creature record THEN
    health    = record.health
    REQUIRE not (the record names a killer AND health > 0)
    killer    = record.killer, unless it names this entity itself
    team, squad, group = the record's

    IF this is a monster THEN
      look up its species' community and, if that community has a team,
      OVERRIDE the record's team with it
  ELSE
    # the only non-creature entities are vehicles, traders and the helicopter
    REQUIRE the record is one of those three
    health = 1; team = squad = group = 0

  IF alive AND single player THEN
    register in the hierarchy and increment its alive count

  IF not alive THEN
    level death time = now; game death time = the record's

  base.spawn(record)

  # damage configuration comes from the MODEL, not the section
  IF the model carries a damage block THEN load the per-bone damage scaling from it
  load the model's particle attachment points
```

**Invariants**

- the monster community override means a monster's team is a property of its *species*,
  not of its spawn record. Two records of the same species cannot be on different teams,
  which is exactly the intent: mutants are hostile to everyone by species.
- the hierarchy registration is single-player only. Multiplayer teams are the game mode's
  business.

## `net_Destroy`

**Contract** — unregisters from the hierarchy if still registered, then the base, then marks
ready to save. Unregistering before the base tears down is required: the hierarchy holds a
pointer that must not outlive the object.

## `ChangeTeam`

**Contract** — moves the entity between affiliations atomically: notify, unregister from the
old triple, assign, register into the new triple, notify. A no-op when nothing changes.
Changing a dead entity's team is a hard failure.

**Invariants** — unregister-then-register, never register-then-unregister, and the
affiliation fields are written between them. Any other order leaves the entity in two
groups or none.

## `shedule_Update`

**Contract** — one timeout. Three minutes after a dead entity's death, its killer record is
cleared and a "no killer" assignment is broadcast.

**Notes** — the purpose is to let the alife simulation release the killer. An entity that
is nobody's killer can be forgotten; one that is somebody's killer has to be kept because
the corpse still names it. Three minutes is long enough that nothing gameplay-relevant
still refers to the kill.

## `Load` / `reload` / `reinit`

**Contract** — the section supplies the three affiliation defaults (all -1, meaning
unaffiliated) and the corpse removal timeout (ten minutes by default). The entity is
created *invisible* and something else must make it visible. Damage scaling is re-read on
reload.

**Notes** — morale is assigned a compiled-in 66, with a note in the source asking for it to
be explained. It is not.

## `_construct` / `create_entity_condition`

**Contract** — construction builds the damage manager and then the condition object through
an overridable factory. The factory has an unusual shape: passing nothing creates a *simple*
condition (health only); passing an existing condition adopts it, downcast to the full
condition type. Subclasses use the second form to install a richer condition —
see [`EntityCondition.cpp`](EntityCondition.cpp.md).

**Notes** — the downcast on the adopting path is unchecked and would be wrong for anything
but the full condition type. The factory is really two functions sharing a signature.

## `IsFocused` / `IsMyCamera`

**Contract** — two questions that are usually but not always the same: is this the entity
the player is *controlling*, and is this the entity the player is *looking through*. They
differ while the player is in a vehicle or watching a scripted sequence, and code that
confuses them puts the camera in the wrong place.

## `set_death_time` / `AlreadyDie` / `killer_id` / `GetGameDeathTime` / `GetLevelDeathTime`

**Contract** — stamp both clocks; ask whether the entity has already died; read the killer
and either timestamp.

## `set_ready_to_save`

**Contract** — empty in the base; an extension point for subclasses that must flush
something before a save can capture them.

## Could not recover

- The morale value of 66 is compiled in, with the source's own note asking why.
- The health floor of -1000 has no stated derivation.
- The three-minute killer-forgetting timeout has no stated derivation.
- `in_solid_state` — "hits pass through this entity rather than hitting it" — is declared
  true here and its purpose is only legible from its overriders.
