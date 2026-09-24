# src/xrScriptEngine/BindingsDumper.cpp

> Writes out the entire script-visible surface — every namespace, class, base, constant, method,
> property and free function — as a declaration listing a person can read and a tool can diff.

**Needs** — [`BindingsDumper.hpp`](BindingsDumper.hpp.md) · [`script_space.hpp`](script_space.hpp.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — [`BindingsDumper.hpp`](BindingsDumper.hpp.md)

**Tier floor** — T2: it reads the binding layer's internal representation of a registered class
— its bases, its constants, its function signatures — which no portable interface exposes.

## Purpose

Criterion 10 says every shipped script must run unmodified, and that fixes "the entire exported
class surface with exact names and signatures". This file is the instrument that makes that
statement checkable: run the original with one command-line switch, run the rebuild with the
same switch, and diff the two listings. Nothing else in the repository produces a machine-
comparable description of the script surface.

It is also the only documentation of that surface that cannot go stale.

## State

```text
RECORD Dumper
  out          : stream                 # where the listing goes
  vm           : handle
  options      : (indent_width : int, omit_inherited : bool, omit_receiver : bool)
  indent_level : int
  pending      : (functions, classes, namespaces)   # one stack per kind, per namespace level
  operator_names : map<text,text>       # metamethod name -> its source spelling
```

Invariants: the three pending stacks are drained fully at the level that filled them, so a
nested namespace never inherits its parent's undrained entries; and the value stack is at the
same depth at the end of every level as at its start.

## Contract

**`Dump`** — Walks the globals table and writes a nested listing to a stream. Takes three
options: the indent width, whether to omit methods a class merely inherits, and whether to omit
the receiver argument from a method's signature. Disables the binding layer's decorated type
naming for the duration, so the printed types are the plain ones, and restores it afterwards.

**Invariants** — The walk must leave the value stack exactly as it found it. Every recursion
level asserts this, because the walk pushes and pops constantly and an imbalance here would
corrupt the VM of a *running game* — the dump happens after the engine is initialised.

## The traversal

```text
FUNCTION dump_namespace(table)
  # two passes: classify first, emit in a fixed order, so the listing is grouped
  FOR EACH key, value IN table
    CLASSIFY value AS function | class | nested namespace | unknown
  emit every free function
  emit every class
  FOR EACH nested namespace
    emit its header, indent, dump_namespace(it), outdent

FUNCTION dump_class(class)
  emit whether it originates in the engine or in script
  emit its name and its bases, comma-separated; an unnamed base prints as unknown
  indent
  FOR EACH member of the class's static table
    IF it is a class   THEN dump_class(it)          # nested classes
    IF it is a function THEN emit it as a static method
  FOR EACH named integer constant
    emit it
  FOR EACH member of the class's instance table
    emit it as a method or a property
  outdent
```

**Notes** — Classes are collected before being emitted so that free functions, classes and
nested namespaces appear as three blocks rather than interleaved by hash order. The collection
uses a stack, so within each block the order is the reverse of the traversal — which is
arbitrary, and is the reason a diff between two runs of the *same* build is stable but a diff
across a change in registration order is noisy.

## Rendering one function

**Contract** — A registered function knows how to render its own signature; the dumper's work is
deciding *which* signature to render and what to strip from it.

```text
FUNCTION render_method(class_name, function)
  name = function's registered name
  IF name is the constructor marker
    name = class_name ; strip the result type
  ELSE IF name is an operator marker
    name = the operator's source spelling ; strip the result type
  render the signature, unconcatenated, so individual arguments can be edited

  first_argument = the signature's first argument
  IF it matches "<class_name>[ const](pointer or reference)"
    it is the receiver of a method declared on this class
    IF stripping receivers THEN remove it
  ELSE IF it is the interpreter state AND there is more than one argument
    this is an operator: the interpreter state is an artifact, remove it
    IF the next argument is not this class and derived members are omitted THEN emit nothing
    the receiver is now the next argument
  ELSE IF it is the binding layer's generic argument placeholder
    this is a constructor: name the receiver after the class
  ELSE IF derived members are omitted
    emit nothing                       # inherited from a base; the base prints it
  IF stripping receivers THEN remove the receiver
  emit the assembled signature
```

**Invariants** — Whatever the branch, the pushed signature fragments are fully consumed: either
concatenated and printed, or discarded wholesale. Leaving fragments behind breaks the running
VM, which is why every early return pops its own fragments.

**Notes**

- Identifying "is this argument the receiver" by *pattern-matching the rendered type name*
  rather than by asking the binding layer is the weak point of this file. It is a consequence of
  the layer having no interface for "which argument is the receiver"; a rebuild whose binding
  layer records that directly deletes most of this function. The pattern allows an optional
  const qualifier and either pointer or reference spelling.
- Omitting inherited members is on by default when dumping, because a class hierarchy several
  levels deep otherwise repeats every base's surface at every level and the listing becomes
  unreadable. The base's own entry still carries them, and the printed base list tells the
  reader where to look.
- Operators are renamed from their metamethod names to their source spellings — addition,
  subtraction, multiplication, division, exponentiation, the four comparisons, equality and
  conversion to text — so that the listing reads as declarations rather than as metatable keys.
  That table is the complete set of operators the engine exports.

## Properties

**Contract** — A property appears as a native function carrying a reader and optionally a writer
as hidden values. The dumper renders the reader's result type as the property's type and emits
the name followed by which of the two accessors exist. A property with no reader is an error.

## Output form

The listing is deliberately shaped like a class declaration file — nested namespaces in braces,
classes with base lists, methods with argument types, constants as named integers — because the
audience is a modder reading it to find out what is callable, and that shape needs no
explanation. It is not intended to compile.
