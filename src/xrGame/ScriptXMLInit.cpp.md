# src/xrGame/ScriptXMLInit.cpp

> The script layer's window factory: a Lua-visible object that holds one parsed UI layout document and builds a typed widget from any node in it, attaching it to a parent that then owns it.

**Needs** — [`ScriptXMLInit.h`](ScriptXMLInit.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`ui/ServerList.h`](ui/ServerList.h.md) · [`ui/UIMapList.h`](ui/UIMapList.h.md) · [`ui/UIMapInfo.h`](ui/UIMapInfo.h.md) · [`ui/UIKeyBinding.h`](ui/UIKeyBinding.h.md) · [`ui/UICDkey.h`](ui/UICDkey.h.md) · [`ui/UIMMShniaga.h`](ui/UIMMShniaga.h.md) · [`ui/UISleepStatic.h`](ui/UISleepStatic.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [`xrUICore/XML/UITextureMaster.h`](../xrUICore/XML/UITextureMaster.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — reached through its declarations in [`ScriptXMLInit.h`](ScriptXMLInit.h.md); callers name that, not this file.
**Tier floor** — T3: document navigation and object construction

## Purpose

Every screen in the game is built by a Lua script, and a screen is a tree of widgets whose
geometry, textures, fonts, colours and text come from an XML document. Neither half can do
this alone: the script knows *which* widgets a screen has and what they do; the document
knows what each one looks like. This class is the join — script code holds one of these,
points it at a document, and then asks it for one widget per node.

The design decision that makes the whole thing work is stated once in the attachment step
and applies to every factory here: **the parent owns the widget, not the script**. A
widget handed to a parent is marked for automatic destruction with that parent, so a Lua
script can create a hundred widgets, drop every reference, and leak nothing — the screen's
teardown takes them all. A widget created with *no* parent is the script's to keep alive,
and that is the only case where a script holds a real reference.

## State

```text
RECORD ScriptXmlInit
  document : UILayoutDocument     # one parsed XML file; every factory reads nodes from it
```

**Invariants** — the document must be parsed before any factory is called; a factory
against an unparsed document finds no node and produces a widget with default geometry
(zero size, top-left), which is the usual cause of "my window is invisible" in script.

## `ParseFile`

**Contract** — loads and parses a UI layout document by file name, replacing whatever the
object held. The file is resolved through a three-step search: the configuration root,
then the localized UI directory, then the default UI directory. The fallback chain is what
lets a localization override only the screens it needs to change and inherit the rest, and
it is why the file name alone identifies a document.

## `ParseShTexInfo`

**Contract** — registers a texture-atlas description document with the global texture
registry, so that later widgets can name a sprite by identifier instead of by file and
rectangle. Global side effect; nothing about this object is touched.

## The widget factories

**Contract** — every factory has the same shape and the same contract. Given a node path
inside the loaded document and a parent widget, it constructs a widget of one specific
type, applies the node's attributes to it, attaches it to the parent, and returns it. The
returned widget is valid as long as its parent lives. A missing node does not fail: the
widget is created with whatever defaults its initializer applies.

```text
FUNCTION InitX(node_path, parent) -> Widget
  widget <- new widget of type X
  apply the document node at node_path, occurrence 0, to widget
  attach(widget, parent)
  RETURN widget

FUNCTION attach(child, parent)
  IF parent IS none THEN RETURN          # caller keeps ownership; no auto-destruction
  child.owned_by_parent <- true          # destroyed with the parent
  IF parent IS a scroll view THEN parent.add_scrolled_window(child)
  ELSE parent.attach_child(child)
```

**Invariants**

- Attaching to a scroll view is not the same as attaching to an ordinary window. A scroll
  view owns its children's layout — it stacks them and computes the scrollable extent — so
  a child must enter through its own insertion path or it will be positioned as if the
  view were a plain container and the scrollbar will be wrong.
- Every factory reads occurrence zero of the named node. A document with two nodes at the
  same path is a data error nothing detects; the second is unreachable through this class.

The types offered, and what each initializer is:

| Factory | Produces | Initialized by |
|---|---|---|
| `InitWindow` | applies a node to an existing window rather than creating one | the central initializer, at a caller-chosen occurrence index |
| `InitStatic` | a static image/text panel | the central initializer |
| `InitTextWnd` | a static configured for text: it reads and owns its string | the central initializer, with the text options on |
| `InitAnimStatic` | a static playing a sprite-sheet animation | the central initializer |
| `InitSleepStatic` | the sleep-screen static | the central initializer |
| `InitFrame` / `InitFrameLine` | a nine-slice frame and a three-slice line, both named by their node path | the central initializer |
| `InitEditBox` | a text field | the central initializer |
| `InitCheck` | a checkbox | the central initializer |
| `InitSpinNum` / `InitSpinFlt` / `InitSpinText` | integer, real and enumerated spinners | one shared spinner initializer |
| `InitComboBox` | a drop-down | the central initializer |
| `Init3tButton` | the three-state (normal, highlighted, pressed) button | the central initializer |
| `InitTab` | a tab strip | the central initializer |
| `InitTrackBar` | a slider | the central initializer |
| `InitProgressBar` | a progress bar | the central initializer |
| `InitScrollView` | a scrolling container | the central initializer |
| `InitListWnd` / `InitListBox` | the two list widgets | the central initializer |
| `InitHint` | a tooltip | the widget reads the document itself |
| `InitServerList` / `InitMapList` / `InitKeyBinding` / `InitMMShniaga` | the four composite screens whose internal structure is too rich for the central initializer | each reads the document itself |

Three factories deviate and the deviations are the content of the file:

**`InitMapInfo`** — the map-info panel needs its own geometry before it can build its
contents, so the node is applied first and the widget is then told to lay itself out
inside the rectangle it just received. A rebuild whose widgets lay out lazily on first
draw needs no such two-step.

**`InitCDkey`** — the serial-key field wires up its own validation and formatting
callbacks after the node is applied, then loads the current stored key into itself. It is
the only factory that reads persistent state.

**`InitMPPlayerName`** — a text field subclass that enforces the multiplayer name rules;
otherwise an ordinary edit box.

## `script_register`

**Contract** — declares the factory to Lua as the type `CScriptXmlInit`, default
constructible, with one method per factory above. Runs once at script-engine bring-up.
The *script* names are frozen by conformance criterion 10 and are not all the same as the
internal ones:

- `InitLabel` and `InitStatic` are two names for the same factory. Shipped scripts use
  both, so both must exist.
- `InitList` is the list-window factory.
- `InitButton` exists only as a script-side factory — it builds a plain button, which has
  no method of its own on the class. A rebuild may implement it either way; what is frozen
  is the script name.
- `InitAutoStaticGroup` also exists only script-side, and it is the odd one out: it does
  not return a widget at all. It walks a node that declares a *group* of statics and
  attaches every one of them to the window the script passes in. It is how a screen gets
  its dozen decorative labels in one call.

**Notes** — the class exposes its document to callers so that a screen written partly in
script and partly in the engine can share one parsed file. That shared mutable document is
the only coupling between a script-built screen and an engine-built one, and a rebuild
should pass the parsed document explicitly rather than reaching for it.
