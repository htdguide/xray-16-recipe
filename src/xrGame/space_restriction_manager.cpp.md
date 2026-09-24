# src/xrGame/space_restriction_manager.cpp

> Binds entities to restrictions: it keeps each entity's authored restriction lists, merges the level's defaults into them under a conflict rule, resolves the result to one shared restriction object, and re-derives everything when the defaults change.

**Needs** — [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`space_restriction_manager_inline.h`](space_restriction_manager_inline.h.md) · [`space_restriction.h`](space_restriction.h.md) · [`space_restriction_holder.h`](space_restriction_holder.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — reached through its declarations in [`space_restriction_manager.h`](space_restriction_manager.h.md); callers name that, not this file.
**Tier floor** — T2: list algebra over text plus a keyed cache with timed reclamation

## Purpose

The registry below knows about volumes; this layer knows about entities. It answers, for an
entity identifier, "where may you go" — and it does so through a shared object, so that the
dozens of creatures restricted to the same region pay for one border between them.

It is a separate layer from the registry because the two have different keys and different
lifetimes: the registry is keyed by restrictor name and lives as long as the level, this is
keyed by entity and lives as long as the entity. The composed restriction in between is
keyed by *both* lists at once and is reclaimed on a timer.

## State

```text
RECORD ClientRestriction                 # one per restricted entity
  restriction        : optional<SpaceRestriction>   # the resolved, shared object
  base_out           : text              # the entity's OWN permitted list, as authored
  base_in            : text              # the entity's OWN forbidden list, as authored

RECORD RestrictionManager
  clients            : map<entity id, ClientRestriction>
  space_restrictions : map<text, SpaceRestriction>   # key: normalized out + separator + normalized in
```

**Invariants**

- The *base* lists are kept separate from the merged ones, and every edit recomputes the
  merge from the base. Editing the merged list in place would fold the level defaults into
  the entity's own record, so that removing a restriction the entity never had would remove
  a default, and a later change to the defaults would compound rather than replace. This
  separation is the single most important decision in the file.
- The cache key joins the two normalized lists with a control character. It has to be a
  character no restrictor name can contain and that is not the comma already separating
  names within a list, or an entity permitted `a` and forbidden `b` would key the same as
  one permitted `a,b` and forbidden nothing — which is a completely different space.
- An entity's restriction may not be replaced while its border is stamped on the level
  graph. Asserted on every path that replaces one, because the clear that follows would
  remove a different mask than the one that was set.

## `restrict`

**Contract** — set an entity's authored restriction lists, merge the level defaults in, and
bind the entity to the resolved shared restriction. Creates the entity's record if absent.
Runs the reclamation pass. Hard-fails if the entity's current restriction is stamped.

```text
FUNCTION restrict(id, out_names, in_names)
  merged_out = out_names
  merged_in  = in_names

  # Conflict rule: an entity's OWN list of one sense cancels a default of the OPPOSITE
  # sense. A creature explicitly forbidden a region does not also inherit it as a
  # default permitted region, and vice versa. The entity's authored intent wins.
  defaults_out = default_out_restrictions MINUS merged_in
  defaults_in  = default_in_restrictions  MINUS merged_out

  merged_out = merged_out UNION defaults_out
  merged_in  = merged_in  UNION defaults_in

  REQUIRE the entity has no restriction, or it is not applied

  clients[id].restriction = restriction(merged_out, merged_in)
  clients[id].base_out    = out_names        # as authored, NOT merged
  clients[id].base_in     = in_names
  collect_garbage()
```

**Invariants** — the cancellation is asymmetric in the right direction: it removes from the
*defaults*, never from the entity's own lists. A default is a level-wide convenience and an
authored restriction is an instruction; the instruction wins.

Both list operations are set operations on text, so a name present in both the entity's
list and the defaults appears once.

## `restriction` (two lists) — the shared-object cache

**Contract** — resolve a pair of normalized lists to the one shared restriction object for
that pair, creating it on first ask. Returns nothing when both lists are empty.

```text
FUNCTION restriction(out_names, in_names) -> optional<SpaceRestriction>
  IF both lists are empty  RETURN none
  out_names = normalize(out_names)
  in_names  = normalize(in_names)
  key = out_names + separator + in_names     # separator: a control character
  IF key IS IN space_restrictions  RETURN space_restrictions[key]
  r = new SpaceRestriction(self, out_names, in_names)
  space_restrictions[key] = r
  RETURN r
```

**Notes** — nothing is resolved or computed here; the new object builds its border the first
time it is asked a question. That is what lets restrictions be assigned during level load,
before the restrictors they name exist.

## `add_restrictions`, `remove_restrictions`, `change_restrictions`

**Contract** — the three edit operations. Each recomputes the entity's lists **from its base
lists**, applies the edit, and re-runs `restrict`. Adding to an unrestricted entity is the
same as restricting it. Removing from one is a no-op. All three hard-fail if the current
restriction is stamped.

```text
FUNCTION change_restrictions(id, add_out, add_in, remove_out, remove_in)
  IF the entity has no restriction
    restrict(id, add_out, add_in)            # removal from nothing is nothing
    RETURN
  REQUIRE NOT applied

  new_out = base_out MINUS remove_out UNION add_out
  new_in  = base_in  MINUS remove_in  UNION add_in
  restrict(id, new_out, new_in)
```

**Invariants** — removal happens **before** addition. A call that both removes and adds the
same name ends with it present, which is the useful reading for a script that is replacing
one region with another and does not know whether they overlap.

`add_restrictions` and `remove_restrictions` are the same routine with one half empty; they
exist separately because the script surface exposes them separately.

## `unrestrict`

**Contract** — drop the entity's record entirely and run the reclamation pass. Hard-fails if
the entity was never restricted. Called when an entity leaves the level or goes offline.

## `on_default_restrictions_changed`

**Contract** — a default restrictor has spawned or despawned. Re-derive every restricted
entity's restriction from its base lists. Does not block; touches every client.

**Notes** — this is a full re-derivation rather than an incremental patch, and it is why the
base lists exist. It happens rarely — a default restrictor spawning is a level-load event
or a scripted world change — so the cost is paid where it does not matter, and the
alternative (working out which entities a default change affects) would have to know the
conflict rule in a second place.

## `accessible`, `accessible_nearest`, `in_restrictions`, `out_restrictions`, `base_in_restrictions`, `base_out_restrictions`, `add_border`, `remove_border`

**Contract** — per-entity forwarding to the entity's restriction. The accessibility queries
answer **true** for an unrestricted entity, and the list queries answer the empty text. The
nearest-position query instead hard-fails on an unrestricted entity, because asking for a
nearest legal position when everywhere is legal is a caller bug. `base_*` reports the
authored lists — the ones scripts set — while the unqualified forms report the merged ones
the pathfinder actually uses; both are exposed because a script that reads back what it
wrote wants the former and a debugger wants the latter.

## `collect_garbage`

**Contract** — destroy every shared restriction that has no entity bound to it and has had
none for five minutes of world clock. Called after every bind and unbind.

**Notes** — the same delay and the same rationale as the registry's collector: restriction
combinations churn as creatures take and finish jobs, and rebuilding a merged border is a
scan over the navigation mesh. Unlike the registry's collector there is no exemption here,
because nothing owns these objects except the entities bound to them.

## `restriction_presented`, `join_restrictions`, `difference_restrictions`

**Contract** — membership, union and difference over comma-joined name lists, all working on
text rather than on parsed sets. Union appends only names not already present, so the result
has no duplicates; difference keeps only names absent from the other list.

**Notes** — operating on text means every set operation is quadratic in the list length.
That is fine at the sizes involved — a handful of names — and it avoids maintaining a parsed
form alongside the text that must be the cache key anyway. A rebuild with a real set type
should still emit the canonical text for the key, or two entities with the same restrictions
will not share.

## `clear`

**Contract** — drop every client binding and every shared restriction, then clear the
registry below. Called on level unload.
