# src/xrScriptEngine/ScriptExporter.cpp

> Collects every module's self-declared script registration into one dependency-ordered pass,
> so that a base class is always exported to the VM before anything derived from it.

**Needs** — [`ScriptExporter.hpp`](ScriptExporter.hpp.md) · [`xrCommon/xr_unordered_map.h`](../xrCommon/xr_unordered_map.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — [`ScriptExporter.hpp`](ScriptExporter.hpp.md)

**Tier floor** — T2: a graph sort over records that register themselves before the program's
entry point runs. What keeps it off T3 is the registration timing, not the algorithm.

## Purpose

Roughly two hundred and fifty classes across the engine export themselves to script. Listing
them in one central table would mean that every module's public surface is described somewhere
other than where the class lives, and that adding a class means editing a file in another
module. Instead each class carries its own registration function and its own list of the classes
it must follow, and this file turns that scattered declaration into a single ordered traversal.

The ordering is not a convenience. The binding layer must know a base class before it can be
told that another class derives from it; registering a derived class first produces a surface
where the inherited methods are missing, which shows up only as a script failing at run time
with a nil method. Criterion 10 makes that unacceptable.

## State

```text
RECORD ExportNode
  next          : optional<ExportNode>   # intrusive chain; head is a process-wide single value
  register      : function(vm)           # this class's own registration
  dependencies  : function() -> list<ExportNode>   # the nodes that must run first

# process-wide
head  : optional<ExportNode>
count : int
```

**Invariants**

- Every node links itself into the chain during static initialisation, before the program's
  entry point runs. The order in which they do so is *unspecified*, which is exactly why the
  sort exists.
- The chain is sorted at most once per process; the sort is idempotent but not re-run when
  modules are exported into a second VM, because the dependency relation does not change.
- After the sort, walking the chain from the head visits every node after all of its
  dependencies.

## `node` construction and destruction

**Contract** — Construction pushes the node onto the head of the chain and increments the count.
Destruction unlinks it, searching the chain when it is not the head. Neither allocates. A node's
lifetime is the program's, so destruction only matters when the engine is a dynamically loaded
module that can be unloaded.

## `export_all`

**Contract** — Called once per VM, from the script engine's initialisation. Sorts the chain on
first use, then walks it and calls each registration function with the VM. Does nothing when no
node exists. Reports the node count to the log unconditionally, which is the only cheap way to
notice that a module failed to link and took its exports with it.

## `sort`

**Contract** — Orders the chain so dependencies precede dependents. A cycle is fatal: it means
two classes each claim the other as a base, which cannot be satisfied in any order.

```text
FUNCTION sort()
  mark all nodes not_visited

  FUNCTION visit(n)
    IF mark(n) == visiting  THEN FAIL WITH "cyclic dependency in script export"
    IF mark(n) == done      THEN RETURN
    mark(n) = visiting
    FOR EACH d IN n.dependencies()
      visit(d)
    mark(n) = done
    append n to ordered

  FOR EACH n IN all nodes
    visit(n)

  relink the chain so that walking from the head yields `ordered` front to back
```

**Notes**

- A node reached as a dependency but never registered itself — possible when a base class lives
  in a module that was not linked in — is visited harmlessly and contributes nothing; the
  traversal tolerates it rather than failing, because the failure would be a link-time
  configuration problem reported at the wrong moment.
- The outer loop iterates an unordered container, so **two independent nodes may be exported in
  a different relative order between runs**. That is invisible as long as no two modules export
  the same name into the same namespace. Nothing enforces it. A rebuild that wants a
  reproducible surface should iterate the registration chain in a defined order instead — and
  should, because the bindings dump is the tool criterion 10 is checked with, and a dump that
  reorders between runs cannot be diffed.
