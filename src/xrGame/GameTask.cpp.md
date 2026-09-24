# src/xrGame/GameTask.cpp

> One quest: an authored tree of objectives, each with completion and failure conditions expressed as information portions and script predicates, a map location, an encyclopedia article and a deadline.

**Needs** — [`GameTask.h`](GameTask.h.md) · [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`map_location.h`](map_location.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_spot.h`](map_spot.h.md) · [`encyclopedia_article.h`](encyclopedia_article.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_story_registry.h`](alife_story_registry.h.md) · [`ai_space.h`](ai_space.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: tree bookkeeping, predicate evaluation and a save record; no device or layout concern

## Purpose

A quest in this game is not a script — it is *data with hooks*. The authored definition
lives in an XML file and names, for each objective: the text to show, the icon, where on
the map to put a marker, which **information portions** must be held for the objective to
count as complete or failed, which script predicates say the same thing, and which
information portions and script calls to fire *when* it completes or fails.

That split — conditions that are *checked* versus effects that are *fired* — is the
central design decision here, and it is what makes quests declarative. A quest never
polls the world itself; it asks the player's information-portion set and a handful of
script predicates, each of which the world updates for its own reasons.

The second decision is that a task and its objectives are the **same type**. A task *is*
its own root objective and additionally holds a list of sub-objectives. The map marker,
the deadline and the state machinery are therefore written once. A rebuild is free to
model this as composition instead, but must keep the consequence: addressing objective
zero and addressing the task are the same operation.

## State

See [`GameTaskDefs.h`](GameTaskDefs.h.md) for the enumerations and identifiers.

```text
RECORD Objective
  index          : int (16-bit)   # 0 == the task itself
  state          : TaskState
  type           : TaskType
  title          : text           # localization key, never a literal
  description    : text
  article_id     : optional<text> # encyclopedia entry unlocked with this objective
  article_key    : optional<text>
  icon_texture   : optional<text>
  icon_rect      : rect           # in the atlas; stored as origin plus size, not as two corners
  map_location   : optional<text> # the *kind* of marker, not a position
  map_object_id  : int (16-bit)   # the entity the marker follows; "none" when unresolved
  map_hint       : optional<text>
  default_location_enabled : bool
  linked_marker  : optional<MapLocation>   # the live marker; not saved, recreated on load
  receive_time   : int            # game time the objective was handed out
  finish_time    : int            # game time it ended
  time_to_complete : int          # deadline; equal to receive_time means "no deadline"
  timer_finish   : int            # what the map marker's countdown shows

  complete_infos   : list<text>   # ALL must be held -> completed
  fail_infos       : list<text>   # ALL must be held -> failed
  infos_on_complete: list<text>   # granted when it completes
  infos_on_fail    : list<text>   # granted when it fails
  complete_predicates, fail_predicates,
  on_complete_calls, on_fail_calls : list<script function>

RECORD Task EXTENDS Objective
  id          : text              # the authored task name
  priority    : int               # sort order in the PDA; unset means the data forgot it
  read        : bool              # whether the player has opened it since it arrived
  objectives  : list<Objective>   # indices 1..n; index 0 is the task record itself
  active      : int (16-bit)      # which objective the map and PDA currently point at
```

Invariants that are enforced by scattered code and are easy to lose:

- An objective either has **both** a map-location kind and a story identifier naming the
  object to follow, or neither. The authored data is checked for this on load.
- The deadline is meaningful only on the root objective, and only when it differs from
  the receive time. Equality is the encoding of "no deadline".
- A marker exists only while its objective is in progress. Reaching a terminal state
  removes it, so a completed quest never leaves a stale map spot.
- `active` never points at the root objective while sub-objectives exist: selecting the
  root selects the first sub-objective instead. The root is a container, not a
  destination.

## `CGameTask::Load`

**Contract** — build a task from its authored definition, named by identifier, out of the
single quest XML document the manager holds open. Reads the task's title, priority, icon
and every objective in document order. Fails loudly if the identifier is absent or if a
named script predicate does not resolve — a missing predicate is a data error that must
surface at load rather than silently make an objective uncompletable.

**Invariants** — the first `objective` element in the document *is* the task's own root
record, not a sub-objective. Later elements become sub-objectives with indices from one
upward. Only the root carries the icon.

```text
FUNCTION load(task_id)
  node = quest document, the game_task element whose id attribute is task_id
  FAIL WITH "no such task" IF node is absent
  title    = node.title                  # a localization key
  priority = node.prio                   # -1 when the data omits it; see Notes

  FOR EACH objective element, index i IN node
    target = IF i == 0 THEN this task itself ELSE a new objective appended at index i
    target.description = element.text
    target.article_id  = element.article, target.article_key = element's key attribute

    IF i == 0 THEN resolve_icon(target, element)
    target.map_location = element.map_location_type
    target.map_hint     = that element's hint attribute
    target.default_location_enabled = NOT element.map_location_hidden
    IF element.object_story_id is present THEN
      target.map_object_id = entity_for_story_id(story_id_value(element.object_story_id))

    read list: infoportion_complete   -> complete_infos
    read list: infoportion_fail       -> fail_infos
    read list: infoportion_set_complete -> infos_on_complete
    read list: infoportion_set_fail     -> infos_on_fail
    resolve list: function_complete      -> complete_predicates
    resolve list: function_fail          -> fail_predicates
    resolve list: function_call_complete -> on_complete_calls
    resolve list: function_call_fail     -> on_fail_calls
```

**Notes** — the icon resolution has two paths, and the branch is on a *hard-coded texture
name*. If the icon names the shared task atlas, the rectangle is taken from that atlas's
declaration; any other name lets the authored element give explicit coordinates. This is
a compatibility wart: one game's data put icons in a shared atlas and another's did not.
A rebuild should keep both paths but recognise that the literal name is the only thing
distinguishing them.

The rectangle is stored as origin plus *size* rather than as two corners — the atlas
declaration supplies two corners and the second is immediately converted. Every consumer
assumes the size form.

## `story_id` and `storyId2GameId`

**Contract** — translate an authored *story identifier* — a symbolic name for a
plot-significant entity — first into its numeric code by reading a script-side table,
then into the entity identifier of the object that currently carries it. The second step
prefers the alife simulation's story registry and falls back to a linear scan of the
loaded level's objects when there is no alife simulation (multiplayer). Returns "none"
when no object carries the identifier.

**Notes** — the name-to-code table lives in the *script* layer, not in the engine, which
is why the lookup goes through the script virtual machine. That is a real dependency: a
rebuild cannot resolve a quest's target without the game's scripts loaded.

The fallback scan is linear over every object in the level. It runs only in multiplayer
and only when a task is loaded, so its cost never appears in a frame.

## `SGameTaskObjective::UpdateState`

**Contract** — evaluate the objective's conditions against the current world and report
what its state *should* be. Pure: it changes nothing. The manager calls it and applies
the result.

**Invariants** — the evaluation order is fixed and load-bearing: deadline, then failure
information portions, then failure predicates, then completion information portions, then
completion predicates. **Failure wins over completion.** A quest whose conditions are
simultaneously satisfiable in both directions fails, which is the conservative answer and
is what the shipped quest data relies on.

```text
FUNCTION update_state() -> TaskState
  IF this is the root objective AND a deadline is set AND game time is past it THEN
    RETURN fail
  IF all_held(fail_infos)        THEN RETURN fail
  IF all_true(fail_predicates)   THEN RETURN fail
  IF all_held(complete_infos)    THEN RETURN completed
  IF all_true(complete_predicates) THEN RETURN completed
  RETURN current state
```

**Notes** — both `all_held` and `all_true` are conjunctions over a list, and both are
*false for an empty list* — an objective with no completion conditions never completes on
its own and must be closed by script. The conjunction short-circuits on the first miss,
which also means a predicate list evaluates only as far as the first false one; a rebuild
must not "helpfully" evaluate them all, because the predicates are game script and may
have side effects.

Each predicate is called with the *task's* identifier, not the objective's. An objective
cannot tell a shared predicate which of its siblings is asking.

## `SGameTaskObjective::SetTaskState`

**Contract** — move an objective to a state and run the consequences. Entering a terminal
state removes the map marker, stamps the finish time from the game clock, grants the
state's information portions and calls its script functions. Always notifies the state
change callback, terminal or not.

**Invariants** — the marker is removed *before* the effects fire, so a script called on
completion sees a map without the finished quest's spot and may add its own. The finish
time comes from *game* time, not wall time, because it is shown to the player and must
survive time acceleration and sleeping.

## `CGameTask::SetTaskState`

**Contract** — the task-level override: set one objective's state, and when the objective
addressed is the root, cascade the same state onto **every sub-objective still in
progress**. Sub-objectives already in a terminal state are left alone.

**Invariants** — the cascade preserves history: an objective that already failed is not
retroactively completed when its parent task completes. This is what lets the PDA show
"completed, but you failed step two".

## `CGameTask::OnArrived`

**Contract** — hand the task to the player. Selects the root objective (which, per the
invariant, resolves to the first sub-objective when there are any), puts the task and
every sub-objective into progress, gives anything still typeless the storyline type,
marks the task unread, unlocks its encyclopedia articles and creates its map marker.

**Notes** — defaulting an unclassified task to storyline rather than to a side quest is a
deliberate bias: a quest whose data forgot to say what it is gets the most prominent
treatment, so the omission is visible rather than silent.

## `CGameTask::FillEncyclopedia`

**Contract** — for every sub-objective naming an encyclopedia article, add that article
to the player's encyclopedia registry if it is not already there, stamped with the
current game time and with the article's own kind. Idempotent.

**Notes** — the root objective's article is *not* added here; only sub-objectives are
walked. Nothing in the source explains whether that is deliberate or an oversight, and a
rebuild copying the behaviour reproduces whatever the shipped data expects.

## Map marker — `CreateMapLocation`, `RemoveMapLocations`, `ChangeMapLocation`, `LinkedMapLocation`

**Contract** — an objective's marker is created from two pieces: the *kind* of marker
(a named style from configuration) and the entity it follows. Creation does nothing when
either is missing. Creation has two modes: fresh, which asks the map manager for a new
marker and stamps it with the owning task's identifier; and on-load, which searches the
markers the map manager already restored for the one carrying that identifier and adopts
it.

**Invariants** — a marker is stamped with its owning *task* identifier, which is how the
on-load path re-establishes ownership without saving a pointer. Markers are serialized by
the map manager, not by the task; the task saves only the description of the marker it
wants. Two markers of the same kind on the same entity from different tasks are
distinguished only by that stamp.

The task's own marker accessor forwards to the *active* objective's marker unless the
active objective is the root — so the map highlights the step the player is on, not the
quest as a whole.

**Notes** — removal takes a flag meaning "the map manager is already removing this, do
not ask it again". That is a re-entrancy guard around the manager notifying the task of a
removal the task itself initiated. In a rebuild with a clearer ownership direction the
flag disappears.

Changing a marker is remove-then-create *and* forces the objective back into progress —
a quest that redirects the player necessarily un-completes the step.

## `ChangeStateCallback`

**Contract** — fire the player's task-state-change script callback. The task-level
override chooses between two *different signatures*: a task with sub-objectives reports
(task, objective, state); a task without reports (task, state).

**Notes** — the two signatures are frozen by the shipped scripts, which is why the
distinction exists at all. A rebuild must reproduce both arities rather than always
passing the objective.

## Save and load — `SGameTaskObjective::save`/`load`, `CGameTask::save`/`load`, `SScriptTaskHelper::save`/`load`, `SGameTaskKey::save`/`load`

**Contract** — a task persists its identifier, priority, root objective and then its
sub-objectives in order. An objective persists everything authored *except* its resolved
script predicates and its live map marker: predicates are saved as **names** and
re-resolved on load, and the marker is re-adopted from the map manager.

**Invariants** — script functions must be saved by name, because a resolved function
handle means nothing across a session; re-resolution on load is best-effort and only
warns, unlike the load-from-XML path which fails hard. The difference is deliberate: a
save must still open when a mod has removed a script the save refers to.

Loading a task re-resolves predicates and then adopts its marker, in that order, and only
after every sub-objective has been read — a sub-objective's marker lookup needs its
parent's identifier, which the parent link is patched in for before each element is read.

**Notes** — the script-name helper exists purely because scripts may *add* conditions to
a task at runtime. Conditions from XML are resolved immediately and never saved as names;
conditions added by script are kept as names precisely so they can be saved. A rebuild
can unify the two by always keeping the name beside the resolved handle.

## Script-facing accessors

**Contract** — the objective and task expose their title, description, type, icon,
priority, map hint, map location kind and map object identifier as settable properties,
plus eight "add a condition" calls (complete/fail × checked/fired × information
portion/function). A script builds a task by constructing one, setting properties and
appending objectives, then the manager hands it out.

**Invariants** — conditions added by script land in the *name* lists and do not take
effect until the names are resolved, which happens on load and on explicit commit. A
script that adds a predicate and expects it to fire on the next update must commit first.

Setting the icon by name re-derives the rectangle from the shared atlas and converts it
to the origin-plus-size form, so script-set icons must live in that atlas.
