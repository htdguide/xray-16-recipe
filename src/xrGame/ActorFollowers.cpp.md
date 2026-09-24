# src/xrGame/ActorFollowers.cpp

> Dead code: an abandoned squad-of-followers feature, commented out in its entirety.

**Needs** — _(none: the file's whole body is disabled)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: nothing compiles from this file

## Purpose

The file contains no live code. What is commented out is a feature that was designed and
then withdrawn: the actor could acquire *followers* — other characters registered by
entity identifier — shown on a dedicated screen panel, and could broadcast a command to
all of them at once, which each follower's inventory-owner side would react to.

A rebuild should not implement it. It is recorded here only so that a reader who finds the
file is not left wondering what was removed, and because the shape is worth knowing: the
follower set was a list of entity identifiers, never of object references, resolved
through the level's identifier lookup at the moment a command was sent. That is the
correct pattern for any set of remembered entities in this engine — an identifier survives
a follower going offline, unloading with its level, or being destroyed; a pointer does
not.

## State

`Stateless.`

## Notes

The disabled code also shows one live convention: a user-interface panel owned by a
gameplay object registers itself with the game's dialogue-render list on construction and
must remove itself before destruction. That ordering requirement is real and applies
everywhere in the chapter — see the chapter opener's teardown-order rule.
