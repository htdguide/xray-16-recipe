# src/xrGame/WeaponShotgun.cpp

> A manually cycled shotgun: one shell per trigger pull, and a reload the player can abort mid-way because it loads a shell at a time.

**Needs** — [`WeaponShotgun.h`](WeaponShotgun.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Inventory.h`](Inventory.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`weaponBM16.cpp`](weaponBM16.cpp.md)
**Tier floor** — T2: a sub-state machine nested inside the weapon state machine.

## Purpose

A tube-fed shotgun cannot use the one-shot reload that every other weapon uses. Its
magazine fills one shell at a time, each shell has its own animation and sound, and the
player may stop at any point and fire what is loaded. This file is that behaviour: a
three-phase sub-state machine living inside the weapon's single *reload* state.

Its second job follows from the first. Because shells are loaded individually, a
shotgun's magazine is genuinely **heterogeneous** — slug on top of buckshot on top of
slug — and that mix must survive the network. So this is the only weapon in the game
whose replication carries the magazine's *contents*, not just its count.

## State

```text
RECORD Shotgun EXTENDS CustomPistol
  tri_state_reload : bool        # authored; false makes it reload like any other weapon
  open_sound, add_cartridge_sound, close_sound : SoundHandle
```

The sub-state itself lives on the weapon base (`m_sub_state`), because the state-change
network message carries it.

**Invariant** — every sub-state transition goes back through the *same* reload state, so
the weapon never leaves `eReload` until the sequence finishes. The state machine sees one
long reload; the sub-state machine sees three phases.

## The three-phase reload

```text
ENUM ReloadPhase: Begin, InProcess, End
```

- **Begin** — the weapon is opened: breech-open sound and animation.
- **InProcess** — one shell is pushed in: a shell sound and animation, repeated.
- **End** — the weapon is closed: breech-close sound and animation, and pending is
  cleared.

### `TriStateReload` — entering the sequence

**Contract** — refuses outright if the magazine is already full or the inventory holds
no acceptable shell; otherwise leaves zoom (through the weapon base's reload, which is
only that) and enters the reload state at phase *Begin*.

```text
FUNCTION tri_state_reload(shotgun)
  RETURN IF the magazine is full OR no acceptable shell is available
  weapon_base.reload()                 # leaves zoom; loads nothing
  shotgun.sub_state = Begin
  switch to reload
```

**Notes** — it calls the *weapon base's* reload, deliberately skipping the magazined
weapon's, which would have run the ordinary whole-magazine transfer.

### `OnStateSwitch` — dispatching a phase

**Contract** — every entry into the reload state runs one phase. The guard runs first and
is the sequence's exit condition.

```text
FUNCTION on_state_switch(shotgun, target, previous)
  IF NOT tri_state_reload OR target <> reload THEN
    base.on_state_switch(target, previous); RETURN

  weapon_base.on_state_switch(target, previous)      # skip the magazined weapon's table

  IF the magazine is full OR no acceptable shell is available THEN
    switch2_end_reload(shotgun)                      # close up and stop
    shotgun.sub_state = End
    RETURN

  MATCH shotgun.sub_state
    Begin     -> IF a shell is available THEN switch2_start_reload(shotgun)
    InProcess -> IF a shell is available THEN switch2_add_cartridge(shotgun)
    End       -> switch2_end_reload(shotgun)
```

The magazined weapon's transition table is bypassed because it would play the ordinary
reload sound and animation over the top of the shell sequence.

### `OnAnimationEnd` — advancing a phase

**Contract** — each phase's animation ending advances the sub-state and re-enters the
reload state. This is what makes the sequence loop.

```text
FUNCTION on_animation_end(shotgun, state)
  IF NOT tri_state_reload OR state <> reload THEN
    base.on_animation_end(state); RETURN

  MATCH shotgun.sub_state
    Begin ->
      shotgun.sub_state = InProcess
      switch to reload                       # re-enter: plays the first shell
    InProcess ->
      IF add_cartridge(shotgun, 1) <> 0 THEN  # non-zero = the shell could not be taken
        shotgun.sub_state = End
      switch to reload                        # re-enter: another shell, or close up
    End ->
      shotgun.sub_state = Begin               # rearm for next time
      switch to idle
```

**Invariants** — the shell is transferred when the *shell animation ends*, matching the
whole-magazine rule elsewhere: nothing moves until the animation that depicts it moving
has finished. The loop's exit is the guard in `OnStateSwitch`, not this procedure —
which is why the magazine-full case is checked on entry rather than after each shell.

### Aborting the reload

**Contract** — pressing fire during the *InProcess* phase loads one final shell and jumps
to the *End* phase, so the weapon closes and becomes fireable rather than continuing to
fill.

```text
IF tri_state_reload AND state = reload AND the command is fire pressed
   AND sub_state = InProcess THEN
  add_cartridge(shotgun, 1)
  shotgun.sub_state = End
  consume the command
```

**Notes** — the abort loads a shell it did not animate, which is deliberate: the player
who taps fire mid-reload expects the shell that was being pushed in to be there.

## `HaveCartridgeInInventory` — and its side effect

**Contract** — answers whether at least `cnt` acceptable shells exist, and **switches the
weapon's ammo type** if the current type runs out but another is available.

```text
FUNCTION have_cartridge(shotgun, cnt) -> bool
  IF ammunition is unlimited THEN RETURN true
  IF there is no inventory THEN RETURN false
  available = count of the current ammo type
  IF available < cnt THEN
    FOR EACH other type i IN ammo_types
      available = available + count of type i
      IF available >= cnt THEN
        shotgun.ammo_type = i          # <- the side effect
        BREAK
  RETURN available >= cnt
```

**Invariants** — this is why a shotgun fed a mix of slug and buckshot ends up with a
mixed tube: as each type runs out the loader silently moves to the next, shell by shell.
The predicate mutating the weapon is a design smell but is the mechanism; a rebuild
splitting it into a query and an explicit switch must keep the switch at the same point
in the sequence or the mix changes.

**Notes** — the running total is wrong when the current type is partially stocked: the
first branch's `available` already holds the current type's count and the loop adds
others on top, so the answer is "do I have enough *in total*" while the type chosen is
the first one that crossed the threshold — which may itself hold fewer than `cnt`. With
`cnt` always 1 in practice, this never bites.

## `AddCartridge` — one shell

**Contract** — pushes up to `cnt` shells into the magazine and returns how many it could
**not** push. Clears any jam and adopts a pending ammo-type change on the way in.

```text
FUNCTION add_cartridge(shotgun, cnt) -> int
  clear the jam
  IF a pending ammo type was requested THEN adopt it; clear the request
  RETURN cnt IF no acceptable shell is available      # via have_cartridge, which may
                                                      # have switched the type
  box = the inventory's first box of the current type
  IF the prototype cartridge is of the wrong type THEN reload it from the section
  WHILE cnt > 0
    IF ammunition is not unlimited THEN
      take one round from the box; BREAK if the box is empty
    cnt = cnt - 1
    ammo_elapsed = ammo_elapsed + 1
    stamp the current ammo type onto the round and push it onto the magazine
  IF the box is now empty AND we are authoritative THEN mark it for disposal
  RETURN cnt
```

**Invariants** — `ammo_elapsed` and the magazine length stay equal, asserted before and
after the loop. The return value's polarity is the sequence's stop condition: non-zero
means "could not take one", which drives the sub-state to *End*.

## `switch2_Fire`

**Contract** — the semi-automatic arming, plus clearing the firing flag a second time.
Redundant — the base already clears it — and harmless.

## Replication of a mixed tube

**Contract** — this weapon's network payload carries, after the magazined weapon's: the
magazine's length as a byte, then one byte per shell naming its ammo type index.

```text
export: shell count (8 bits)
        for each shell, its ammo type index (8 bits), from the bottom of the tube up
```

On import, each received type is compared against the type of the shell at that position
and, if it differs, that shell is **reloaded from the named section in place** — so the
receiving client's tube ends up holding the same mix, with the right ballistics, without
the magazine ever being rebuilt. Received entries past the local magazine's length are
discarded, because the count itself came from the base's payload and is authoritative.

**Invariants** — this is the only replication in the weapon system that is not a scalar.
Its cost is one byte per shell per update for a weapon in hand, which is affordable only
because a tube holds a handful of shells.

## Sounds and animations

**Contract** — three extra sounds (open, add shell, close), loaded only when the weapon
is authored as tri-state, and three extra animations, each probing a newer and an older
name. All three animations assert the weapon is in the reload state when they play.

The shell and close sounds are tagged with the **shooting** sound class rather than the
recharging one, so the AI hears a reloading shotgun as if it were firing. Whether that is
intended is not recoverable.
