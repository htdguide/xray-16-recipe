# src/xrGame/account_manager_script.cpp

> Exports the account manager and its four callback shapes to the script virtual machine, so the account screens can be written in Lua.

**Needs** — [`account_manager.h`](account_manager.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

The account screens are script, not engine, so the account manager's whole asynchronous
surface has to cross into Lua: submit an operation with a script function as its callback,
ask whether it is still in flight, cancel it, and read back the results.

## State

`Stateless.`

## `account_manager::script_register`

**Contract** — registers the account manager type into the script virtual machine under
its own name, exporting: nickname suggestion (submit, cancel, read results), profile
creation and deletion, account profile listing (submit, in-flight query, cancel, read
results), email search (submit, in-flight query, cancel), and all three field validators
together with the string-table key of the last validation failure. Names and signatures
are frozen by conformance criterion 10. Runs once at script-engine bring-up.

**Invariants** — the two result collections are exported as *iterators*, not as tables:
the binding walks the engine's own collection and yields elements, so no copy is made and
the script sees the current contents at the moment it iterates. A script that holds the
iterator across a resubmission is iterating a collection the manager clears — the export
choice buys speed and pays for it with that hazard.

**Notes** — the manager is exported with no constructor. Scripts never create one; they
reach the single instance the main menu owns.

## Callback-type registration

**Contract** — four callback types are also registered, one per operation shape, each
under its own script name: account operation (success, description), account profiles
(count, description), found email (found, nickname) and nickname suggestions (count,
description). A script constructs one of these around a Lua function and hands it to the
manager, which then calls back into script when the service answers.

**Notes** — each type is a delegate that can hold *either* an engine method or a script
function. That duality is what lets the same manager serve the console (engine callbacks)
and the menu (script callbacks) with one code path; a rebuild needs the same union, however
it spells it.
