# src/xrGame/inventory_upgrade.cpp

> One purchasable upgrade: what it is made of in configuration, the three script hooks it owns, and the verdict it returns when asked whether it may be installed.

**Needs** — [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md) · [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`inventory_upgrade.h`](inventory_upgrade.h.md); callers name that, not this file.
**Tier floor** — T3: configuration parsing and script calls

## Purpose

A leaf of the upgrade forest. It answers three questions: *what does this cost and require*
(the precondition hook), *what does it do to the item* (the named configuration section
whose keys are merged in, plus the effect hook), and *what should the screen show* (name,
description, icon, up to four property names, a grid cell).

The file is not large, but it is where the mechanic's one genuinely awkward compatibility
problem lives: two of the three shipped games use the same numeric refusal codes from script
to mean different things.

## State

```text
RECORD Upgrade                       # extends UpgradeBase
  parent_group   : Group             # the group this upgrade belongs to
  section        : text              # configuration section merged into the item on install
  scheme_index   : (int, int)        # cell in the upgrade screen's grid
  name           : text              # localized
  description    : text              # localized
  icon           : text
  properties     : list<text>        # at most four, by name; looked up in the manager
  preconditions  : script hook -> int          # (parameter, section)
  effects        : script hook -> nothing      # (parameter, section, loading_flag)
  prerequisites  : script hook -> text         # (parameter, section)
  highlight      : bool
```

**Invariants**
- `section` names a configuration section that exists and is non-empty. An upgrade whose
  section is missing or empty is an authoring error, not a runtime condition.
- Each of the three script hooks resolves at construction or the game refuses to start. They
  are not optional and there is no fallback: the mechanic is defined by script, and a
  missing hook means the shipped data and the shipped scripts disagree.
- Every name in `properties` should resolve in the manager's property table; one that does
  not is reported but tolerated, and shows as blank on the screen.

## `construct`

**Contract** — reads every authored key for this upgrade, resolves the three script hooks,
and registers the groups this upgrade unlocks. Fails the process on a missing hook. Called
once, from the manager, while the forest is being built at startup.

```text
FUNCTION construct(upgrade_id, parent_group, manager)
  base_construct(upgrade_id, manager)
  self.parent_group = parent_group

  name        = localize(config upgrade_id."name")
  description = localize(config upgrade_id."description")
  icon        =          config upgrade_id."icon"
  section     =          config upgrade_id."section"     # must exist and be non-empty

  # each hook is bound together with the arguments it will always receive
  bind(preconditions, config upgrade_id."precondition_functor",
       args = (config upgrade_id."precondition_parameter", section))
  bind(effects,       config upgrade_id."effect_functor",
       args = (config upgrade_id."effect_parameter", section, 1))
  bind(prerequisites, config upgrade_id."prereq_functor",
       args = (config upgrade_id."prereq_params", section))

  # each is invoked once here, immediately, purely to prove it runs
  preconditions(); effects(); prerequisites()

  add_dependent_groups(config upgrade_id."effects", manager)
  known = config upgrade_id."known", default false

  FOR i IN 0 .. max_properties_count - 1
    properties[i] = comma_item(config upgrade_id."property", i)
  scheme_index = config upgrade_id."scheme_index"
  highlight = false
```

**Notes** — the three hooks being *called* during construction is deliberate and it is the
only validation that exists: a script function that resolves by name but throws when run
would otherwise not be discovered until a player opened the upgrade screen, deep into a
session. The effect hook is called with its loading flag set, which is what makes this
dry run harmless — the shipped effect scripts treat that flag as "you are replaying, do not
charge anything".

The key naming a node's dependent groups is `effects`, and the key naming its script effect
hook is `effect_functor`. They are unrelated. The overload is in the shipped data and cannot
be renamed.

## `can_install`

**Contract** — the full verdict for "may this upgrade go onto this item now". Combines three
sources in a fixed order: the base checks, the owning group's checks, and the script
precondition. Returns a `UpgradeStateResult`. Pure with respect to the item; the script hook
may do anything.

```text
FUNCTION can_install(item, loading) -> UpgradeStateResult
  IF loading
    RETURN result_ok          # this state was validated when it was first installed

  verdict = base_can_install(item, loading)
  IF verdict IS NOT result_ok
    RETURN verdict

  verdict = parent_group.can_install(item, self, loading)
  code    = preconditions()   # a small integer from script

  IF code IS 0
    RETURN verdict            # script agrees; the group's verdict stands
  IF code IS 1
    IF game IS clear_sky
      RETURN verdict IF verdict IS NOT result_ok ELSE result_e_precondition_money
    RETURN result_e_cant_do
  IF code IS 2
    RETURN verdict IF verdict IS NOT result_ok ELSE result_e_precondition_quest
  RETURN result_ok
```

**Notes** — this is the compatibility hazard. The script precondition returns a small
integer, and *Clear Sky*'s scripts mean "you cannot afford it" by 1 and "the story does not
permit it yet" by 2, while *Call of Pripyat*'s mean "no" by 1 and "some precondition failed"
by 2. The engine cannot ask the script which dialect it speaks, so it branches on which game
is running — one of the many places the three-way game identity flag decides behaviour. A
rebuild that is free to change the script surface should return a named refusal instead; a
rebuild that must run the shipped scripts has to keep this branch.

Note also that when the script refuses, the *group's* verdict wins if it was also a refusal.
The refusal the player is shown is therefore the structural one ("a sibling upgrade is
already installed") rather than the economic one ("you cannot afford it"), which is the more
useful message of the two.

## `can_add`

**Contract** — the base checks only, deliberately skipping both the group rules and the
script precondition. This is what the code that *records* an upgrade onto an item asks,
as opposed to what the screen asks before offering it: adding an upgrade the game itself
decided to grant must not be refused because the player has no money.

## `run_effects`

**Contract** — invokes the script effect hook, setting its loading flag first. The one piece
of state on the hook that varies between calls, and the reason the hook stores its arguments
rather than being a plain closure.

## `get_prerequisites`

**Contract** — returns the script-produced text listing what is still missing for this
upgrade. Called by the screen every time a tooltip is drawn, so the script side is expected
to be cheap.

## `fill_root_container`

**Contract** — registers this upgrade into the owning root's flat lookup table, then recurses
into the groups it unlocks. The root ends up holding every upgrade reachable from it, so
that "does this item have upgrade X" is a lookup rather than a tree walk.

## `highlight_up` / `highlight_down` / `set_highlight` / `check_scheme_index`

**Contract** — the upgrade screen's dependency highlight. Selecting a node lights the whole
chain it belongs to: `highlight_up` marks this node and everything it unlocks, recursing
forward through the dependent groups; `highlight_down` marks this node and asks the parent
group to continue backward through whatever unlocks it. Both directions set the same flag,
so a full selection is the union of the two walks. `check_scheme_index` matches a node
against a grid cell, which is how a click on the screen finds its upgrade.

**Notes** — neither walk has cycle protection. The forest is authored, and an authored cycle
would hang the screen. This is unrecovered: nothing validates acyclicity anywhere.
