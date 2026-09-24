# src/xrGame/UITeamPanels.cpp

> The whole multiplayer scoreboard: every team's panel, plus the rule that decides which panels are visible in which phase of the match.

**Needs** — [`UITeamPanels.h`](UITeamPanels.h.md) · [`UIPanelsClassFactory.h`](UIPanelsClassFactory.h.md) · [`UITeamState.h`](UITeamState.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`ui/UIStatsIcon.h`](ui/UIStatsIcon.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a container plus a phase-to-visibility table.

## Purpose

Holds one panel per authored team name and fans membership changes out to all of them —
letting each panel decide for itself whether a given player belongs to it. That
broadcast-and-filter shape is why adding a player costs a pass over every panel and why a
player who switches teams is handled without anyone computing a difference.

It also owns the layout document for the whole scoreboard, because the panels and their
rows keep building widgets from it long after initialization.

## State

```text
RECORD PanelContainer EXTENDS Window
  factory            : PanelFactory
  layout_document    : Document          # outlives every panel and row built from it
  panels             : map<text, TeamPanel>   # authored name -> panel
  players_dirty      : bool
  panels_dirty       : bool
```

**Invariant** — the document is a member, not a local: rows are laid out from it on
demand, so it must stay loaded for the container's whole life.

## `Init`

**Contract** — loads the named layout document, lays out the container from a named root
node, then builds in this order: every `frame` child (decoration — a frame line or a
static, chosen by a `class` attribute), then every `team` child (a panel, whose team
comes from its `tname` attribute through the factory). Decoration first so it sits behind
the panels. Finally it seeds membership from the game's current player map and applies
the phase visibility rule once, so the scoreboard is correct on the frame it appears.

An authored team name the factory does not recognize is an authoring error.

## Membership fan-out

**Contract** — `AddPlayer` and `RemovePlayer` call every panel; each panel ignores a
player that is not its own. `UpdatePlayer` calls every panel and, if no panel claims the
client, adds it everywhere — which, by the same filter, lands it in exactly the right
panel.

```text
FUNCTION update_player(container, client_id)
  found = false
  FOR EACH panel IN container.panels
    IF panel.update_player(client_id) THEN found = true
  IF NOT found THEN add_player(container, client_id)
```

A team switch surfaces here as "the old panel now answers false", which makes `found`
false and re-adds the player — so the switch needs no special case anywhere.

## Phase visibility

**Contract** — each authored panel is shown or hidden from the match phase alone. Panels
are never destroyed for being invisible.

```text
FUNCTION update_panels(container)
  phase = game.phase
  FOR EACH (name, panel) IN container.panels
    IF phase = PENDING THEN
      show = name IN {"greenteam_pending", "blueteam_pending", "spectatorsteam"}
    ELSE IF phase IN {IN_PROGRESS, PLAYER_SCORES, TEAM1_SCORES, TEAM2_SCORES} THEN
      show = name IN {"greenteam", "blueteam", "spectatorsteam"}
    ELSE
      show = false
    panel.visible = show
```

The spectators panel is visible in both phases; the two playing teams get a differently
laid-out panel before the match starts and during it. A phase outside both sets hides
everything.

## Deferred refresh

**Contract** — a row that notices a disconnect or a team change sets a dirty flag rather
than restructuring the container from inside its own update. The container consumes the
flags at the top of its next update: players first, then panels, then the normal child
update.

## `SetArtefactsCount`

**Contract** — forwards both teams' counts to every panel unchanged; the panels filter.
The comment in the source that this "could be an associative vector in future" is an
admission that the two-team assumption is baked in here.

## Teardown

**Contract** — releasing the container releases the shared stats-icon texture table.
That table is process-global and shared by every scoreboard icon, so it is freed when the
last scoreboard goes away; a rebuild should give it a lifetime of its own rather than
hanging it off this destructor.
