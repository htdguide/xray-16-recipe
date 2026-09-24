# src/xrGame/ai/stalker — the human NPC

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Read the [chapter opener](../README.md) first for the split between creatures and stalkers.

**The brain is not in this directory.** That is the first thing to understand and the easiest
thing to get wrong. This directory is the *body and the senses*: the entity that owns a
brain, feeds it, and supplies the primitive capabilities its actions call into. The brain
itself — the goals, the evaluators, the operators — lives one directory up, among the
`stalker_*` files of chapter 23, and its machinery is chapter 14.

So: `ai/stalker/` is the **substrate**, and `stalker_*` is the **policy**. A rebuilder who
implements this directory has a stalker that can see, aim, take cover, pick up items, talk
and be driven by script — and that stands still, because nothing is telling it what to want.

## How a stalker decides

The decision layer is a **tree of planners**, and the trick that makes it a tree is worth
naming: a sub-planner is simultaneously a planner and an *action in its parent*, and its goal
is literally the effect set it advertises upward. You never write a sub-goal; declaring "this
planner's effect is that there is no enemy" *is* the combat planner's goal.

The root has seven boolean world properties — dead, alive, enemy, danger, anomaly, items,
puzzle-solved — and six operators over them. The entire priority ordering *anomaly before
combat before danger before items before ordinary life* is encoded in preconditions. No
priority number is written anywhere.

Three decisions in that layer shape everything:

**The world model is entirely boolean.** A property is an identifier and a true/false. Every
continuous fact — distance, ammunition, cover quality, how long since the enemy was seen —
must be thresholded into a bool by an evaluator before the planner can reason about it. A
rebuild that gives the planner numbers gets a different, and much slower, system.

**The root goal is unreachable on purpose.** The goal property's evaluator is a constant
false. The NPC's behaviour is the side effect of a perpetual, unsatisfiable search: the
planner can never idle, and the cheapest route toward the impossible state is whatever branch
the current world permits. A rebuilder who "fixes" this by giving the root a reachable goal
gets an NPC that stops.

**Costs are all one, except one.** Nothing sets an action weight, so the search degenerates
to near-breadth-first over plan length and behaviour is driven by precondition structure
alone. The single exception is the combat retreat, which is registered with **no
preconditions at all** — applicable from any state — and given a prohibitive weight. That is
the whole cost model: unit costs, plus one expensive catch-all that is always available and
therefore always the last resort. It is also the only fallback: there is no default action
and no goal relaxation, so a stalker with no plan simply freezes until the world changes.

**There is a second planner.** The weapon in the hands is driven by its own independent
planner — draw, holster, strap, aim, fire, reload, throw — whose property space is rebuilt
every time an item enters or leaves the inventory. The stalker never calls "fire"; it sets a
goal on the weapon planner and lets that planner sequence the animations. This is why a
stalker reloading cannot be interrupted into an inconsistent pose.

## What this directory actually supplies

**Weapon choice and aiming.** The best weapon is scored by a data-driven effectiveness
function against the *currently selected enemy*, searched in three tiers: what I am carrying
that can kill, what I remember seeing in the world that can kill, and remembered weapon/ammo
pairs. Ties break on cost. That third tier is why a stalker will walk across a room to a
rifle it saw earlier.

Aim is a cone whose half-angle is a rank coefficient times one of eight configured dispersion
multipliers, selected by posture and whether the stalker is zoomed. The muzzle origin differs
by posture too — the eye when still, a head-derived offset when moving.

**Not shooting friends.** Before firing, five rays — centre plus small offsets in pitch and
yaw — accumulate material transparency and stop at the first entity. The result decides
whether the enemy is killable from here, whether a squadmate is in the way, and how far the
clear line extends. Suppression fire is gated separately: automatic weapons only, within a
short margin of the clear range, with limited vertical separation, and only if the enemy was
seen in the last ten seconds.

**Deliberate misses.** Grenades are thrown on a computed minimum-magnitude ballistic arc,
range-capped by difficulty, swept against geometry — and then multiplied by a small random
factor so a throw is never perfect. There is a separate routine that displaces the aim point
by several metres on purpose, retrying until it finds a reachable spot. A stalker that always
hit would be unplayable, so missing is a feature with its own code.

**Cover, cached with invalidation.** The best cover is *memoised*, not searched per tick. It
is invalidated by events: the enemy changing, restrictions changing, a danger location
appearing or vanishing within range, a squadmate taking the spot, the enemy closing inside
three metres, or the spot's own score degrading past a threshold. On a miss, the search runs
twice — a small radius first, then a large one — parameterised by an engagement band derived
from the weapon in hand, so a shotgun and a sniper rifle look for different places to stand.
Each of six cover evaluators carries its own **inertia in milliseconds**: hysteresis that
stops an NPC oscillating between two equally good positions. The chosen spot is published to
the squad so peers do not pick it.

**Perception as a filter, not a system.** The senses are inherited; this directory only
supplies what counts. Notably, the friend/foe test on vision is *disabled* — stalkers
register friendlies visually and the distinction is made downstream by the enemy manager.

**Item acquisition is asynchronous and server-authoritative.** Touching an item does not take
it. The stalker *requests* ownership and the actual transfer happens when the authoritative
event returns; if the inventory refuses on arrival, a compensating rejection is generated so
the two sides do not diverge. Declined items are remembered so the same object is not
re-evaluated on every touch.

**Head animation is sound-driven, not look-driven.** The head plays a talking variant while
dialogue or certain sounds are active and a neutral one otherwise; where the head actually
*points* is the sight manager's business, capped at a quarter turn from the travel direction.

**Script control is a bypass, not an extension.** Under script control the planner does not
run at all. A queue of script actions — go there, watch that, play this, use that weapon — is
translated directly into movement, sight, animation and weapon goals. This is the mechanism
behind the shipped level logic, and it means a rebuild must support two entirely separate
drive paths into the same body.

**A stalker is a squad member.** Group behaviour is not a blackboard of shared flags as it is
for creatures: the group has its own planner over fused group memory. A stalker's own
decisions therefore consult a *group* that is itself deciding.

## Where the configuration boundary falls

Numbers a designer or modder would tune per character live in the configuration section: the
eight dispersion multipliers, field of view and range, panic threshold, the voice scheme, the
named speed table, the bone names for weapon attachment and aiming, the torso turn limits,
and — unusually — the entire fire-queue table of burst sizes and intervals across weapon
classes and range bands. Rank coefficients for immunity, visibility and dispersion come from
a global section and are interpolated by the character's rank. Immunities and per-bone
protection come from the *model's* user data rather than from configuration, and the voice
prefix and critical-wound weights come from the character description.

Everything that shapes the AI's *character* is in code: every precondition and effect, every
cover inertia and search radius, the weapon-to-engagement-band table, the friendly-fire cone,
the grenade ranges, and the planner's own budget.

## What could not be recovered

Unusually many, because this is the oldest and most modified part of the chapter.

- **The shared fire-queue section never loads.** The guard that should fall back when the
  key is *empty* falls back whenever it is non-empty, so naming a shared section silently
  discards it. The feature has never worked.
- **The zoomed and unzoomed dispersion multipliers are swapped** in both the standing and
  crouching branches. A zoomed stalker gets the hip-fire spread.
- **The network export and import of a stalker disagree.** Two fields are read that are never
  written, so every remote stalker's health, team, squad and orientation are shifted. This
  matters only in multiplayer.
- **Rank does not affect perception.** The visibility coefficient is computed from
  configuration and never read anywhere, although the configuration keys exist and imply
  otherwise.
- **The "too far to kill" world property is permanently false.** Its body is stubbed out with
  an unconditional negative, so the plan branch it gates is unreachable. The disabled body
  holds the intended per-weapon ranges.
- **NPC recoil is cosmetic.** The conversion from the shot effector's angle back into the
  fire direction is stubbed at every call site, so recoil moves the camera and not the bullet.
- **The "can I advance to better cover" question is a no-op.** Its guard is disabled with a
  literal false, so the flag it sets is written in three places and read nowhere, and the two
  combat actions that ask it always get the same answer.
- **The medicine and food accessors both return nothing**, behind a note saying they are
  unfinished. The script surface that depends on them therefore always answers nothing.
- **Both fire branches of the script-entity path are commented out**, leaving only the
  out-of-ammo case with any effect.
- **A grenade reaction interval is written as a product that evaluates to zero**, so the
  guard it was meant to impose never fires. An "infinite" danger interval is expressed as
  about seventeen hours rather than as a sentinel.
- **A universal maximum engagement distance of a hundred and seventy units** appears three
  times as a bare literal with no derivation.
- A death-pose coin flip is written as though it were one chance in three and is in fact one
  in two, because the range is exclusive of its upper bound.
- Several smaller defects — a format-string type mismatch inside an assertion that would
  crash exactly when it fired, a missing null check on the secondary-fire path, an
  uninitialised vector read whose results are unused, and a redundant clamp — are recorded in
  the individual twins.

## Twins

| Twin | Role |
|---|---|
| [`ai_stalker.cpp`](ai_stalker.cpp.md) | The stalker's lifecycle and its update loop: two brains per tick, thirty vocalisations and a hundred-odd fire-queue numbers read from configuration, and a rank that scales three things at once. |
| [`ai_stalker.h`](ai_stalker.h.md) | Declares the stalker: the human brain, and the widest single class in the game. |
| [`ai_stalker_cover.cpp`](ai_stalker_cover.cpp.md) | Cover selection and its cache: a two-radius search whose acceptable distance band is chosen by the weapon in hand, and an invalidation set that is most of the behaviour. |
| [`ai_stalker_debug.cpp`](ai_stalker_debug.cpp.md) | Renders a stalker's inner state for a developer — memory contents, the active plan, the sight target, visibility accumulation, throw trajectories and aim rays — and compiles to nothing outside a development build. |
| [`ai_stalker_events.cpp`](ai_stalker_events.cpp.md) | Negotiates item ownership with the authoritative side: requests a pickup on touch, completes or rejects it when the event returns, and drops items back into the world at the hand's position. |
| [`ai_stalker_feel.cpp`](ai_stalker_feel.cpp.md) | Filters what reaches a stalker's senses — which objects are worth seeing, which touches are worth reacting to — and answers whether a given navigation cell falls inside its current view. |
| [`ai_stalker_fire.cpp`](ai_stalker_fire.cpp.md) | Everything about a stalker and its weapon: how wide it shoots, where the round starts, whether a squadmate is in the way, which gun it wants, how it throws a grenade, and how it is crippled rather than killed. |
| [`ai_stalker_impl.h`](ai_stalker_impl.h.md) | Two inline helpers that need heavy includes: reaching the squad's shared brain, and the recoil-to-aim conversion that is switched off. |
| [`ai_stalker_inline.h`](ai_stalker_inline.h.md) | The stalker's accessors: four sub-manager handles, a frame-stamp helper, and a hundred and eight one-line fire-queue readers. |
| [`ai_stalker_misc.cpp`](ai_stalker_misc.cpp.md) | What a stalker considers worth picking up, worth fighting, and worth shouting about — plus the squad telepathy that lets one stalker inherit another's enemy. |
| [`ai_stalker_script.cpp`](ai_stalker_script.cpp.md) | Publishes the stalker's decision vocabulary to the script layer: every world property, every operator and every voice line gets a scriptable name. |
| [`ai_stalker_script_entity.cpp`](ai_stalker_script_entity.cpp.md) | Lets a script drive a stalker directly, translating queued script actions — go there, watch that, play this, use that weapon — into movement, sight, animation and weapon goals, with the planner switched off. |
| [`ai_stalker_space.h`](ai_stalker_space.h.md) | The stalker's vocabulary: thirty vocalisations, the bit algebra that decides which may interrupt which, and the weapon-class table that reconciles three games' worth of data. |
