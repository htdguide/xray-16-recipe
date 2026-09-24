# src/xrUICore/ui_export_script.cpp

> Publishes the whole widget toolkit to the script layer — the class hierarchy, the message vocabulary, the named fonts and the texture registry — and this surface is frozen, because every shipped game screen is a Lua script that calls it.

**Needs** — [`Windows/UIWindow.h`](Windows/UIWindow.h.md) · [`Windows/UIFrameWindow.h`](Windows/UIFrameWindow.h.md) · [`Windows/UIFrameLineWnd.h`](Windows/UIFrameLineWnd.h.md) · [`Static/UIStatic.h`](Static/UIStatic.h.md) · [`ScrollView/UIScrollView.h`](ScrollView/UIScrollView.h.md) · [`TabControl/UITabControl.h`](TabControl/UITabControl.h.md) · [`TrackBar/UITrackBar.h`](TrackBar/UITrackBar.h.md) · [`SpinBox/UISpinNum.h`](SpinBox/UISpinNum.h.md) · [`SpinBox/UISpinText.h`](SpinBox/UISpinText.h.md) · [`PropertiesBox/UIPropertiesBox.h`](PropertiesBox/UIPropertiesBox.h.md) · [`XML/UITextureMaster.h`](XML/UITextureMaster.h.md) · [`ui_focus.h`](ui_focus.h.md) · [`ui_styles.h`](ui_styles.h.md) · [`UIMessages.h`](UIMessages.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: declarative registration. It touches no data and runs once at startup.

## Purpose

This file is a **compatibility contract**, not a feature. The tenth acceptance criterion in
§6 of the system requirements — every shipped Lua script loads and runs unmodified — is
decided here for the whole UI chapter: exact class names, exact method names, exact argument
shapes, exact enumeration values. A rebuild may reorganise everything behind this file and
must reproduce this file's surface byte for byte in naming.

Registration is *declarative and local*: each type declares its own export next to itself
rather than in one central table. This file is the exception — it collects the toolkit's
exports in one place — but the mechanism is the general one, and the game and server layers
use it the same way.

## State

`Stateless.` Every function here runs once against a fresh script machine and writes only
into it.

## `UIStyleManager::script_register`

**Contract** — exports the four document path prefixes as free functions, the style manager
class (enumerate styles, query and set the current one, force a UI reload), and a free
function returning the single instance.

**Notes** — the path prefixes are exported because a script that ships its own screen
documents has to build paths the same way the engine does. They are the script-side half of
the style mechanism in [`ui_styles.cpp`](ui_styles.cpp.md).

## `CUIFocusSystem::script_register`

**Contract** — exports the nine-value direction enumeration and the focus registry:
register/unregister, the three eligibility queries, the per-frame update, the locker, the
current focus, and the directional search. Plus a free function returning the live instance.

**Notes** — exporting the registry means a script-authored screen can take part in gamepad
navigation. Exporting `Update` also means a script can re-partition eligibility at a moment
of its choosing, which the engine's own screens rely on when they swap sub-screens without
tearing the tree down.

## `CUITextureMaster::script_register`

**Contract** — exports the texture-description record (its page name and its rectangle) and
four free functions for querying the registry: name to page filename, name to rectangle, and
two overloads each of the full lookup — with and without a fallback name, and in
return-value and out-parameter forms.

**Notes** — the overload set exists because the two failure policies differ: the
return-value form asserts the entry exists, the out-parameter form reports absence. A script
that probes for an optional icon must use the second. Overload resolution by argument type is
a hard requirement on the binding layer; see the seam.

## `CUIWindow::script_register`

**Contract** — the largest export and the base of everything. Three groups:

*Free functions* — pack a colour from components; fetch each named font by a stable
accessor; read and write the cursor position; place a window near the cursor inside a
rectangle (the tooltip fit).

*The base window class*, exported under the name `CUIWindowBase`, with: child attach (which
**transfers ownership to the script object graph**) and detach, the auto-delete flag,
hover/focus queries, geometry in three parallel spellings (rectangle, position, size), each
accepting either a packed value or loose components, enable/show, font, the last cursor
position, and the window name.

*A script-constructible subclass* exported as `CUIWindow`, which exists only so that a script
may construct a window without supplying a name — the shipped scripts do, and the engine's
own type requires one. The subclass supplies a fixed name.

*The message enumeration*, exported as a table of named values.

**Invariants**

- Attaching a child through the script binding transfers ownership; the script object graph,
  not the parent window, then governs the child's lifetime. This is the one place where the
  toolkit's auto-delete rule is overridden, and getting it wrong is a double free.
- The exported message table is a **subset** of the full vocabulary in
  [`UIMessages.h`](UIMessages.h.md) — only what shipped scripts reference. It also exports
  two *legacy aliases*, `STATIC_FOCUS_RECEIVED` and `STATIC_FOCUS_LOST`, pointing at the
  current focus-received and focus-lost values. Older games' scripts use the old spellings
  and must keep working.

**Notes** — the "script may omit the window name" subclass pattern recurs for every type
whose engine-side constructor gained a required name. The comment in the source states the
rule the whole file obeys: *we do not change game assets*. When the engine's own API has to
change, the script-visible shape is preserved with a shim.

## Per-widget registrations

**Contract** — the same treatment for each control, each exporting its own vocabulary on top
of the base window:

| Registration | Exposes |
|---|---|
| `CUIFrameWindow` | nine-slice frame; plus a name-free constructible subclass |
| `CUIFrameLineWnd` | three-part stretchable line; orientation |
| `CUIScrollView` | add/remove/clear children, scroll to begin/end/window, selection |
| `CUIListBox`, `CUIListBoxItem` | row list and its rows, including the message-chaining row |
| `CUIListWnd`, `CUIListItem` | the older list widget and its items |
| `CUIStatic` (and `CUILines`) | text and texture: content, font, colours, alignment, ellipsis, complex mode, adjust-to-text, texture rect and offset, heading |
| `CUIButton` | click behaviour on top of static |
| `CUITabControl` | tab set, active tab by id or index |
| `CUIComboBox` | drop-down list |
| `CUIProgressBar` | range and position |
| `CUIPropertiesBox` | context menu: add item, show at a point, clicked item |
| `CUIEditBox` | text entry |
| `CUIMessageBox` | modal prompt, including its host and password fields |
| `CUIOptionsManager` | the settings group operations, through a script-only facade |

**Notes**

- `CUIStatic` exports several *synonymous* names for one operation — `SetEllipsis` and
  `SetElipsis`, `SetTextureRect` and `SetOriginalRect`. The misspellings and the older names
  are in shipped scripts; both spellings must resolve. They are not deprecated aliases to be
  cleaned up, they are the contract.
- The options manager is exported as a **facade class the script constructs**, whose every
  method forwards to the single real manager. Scripts written against the original engine
  construct an options manager object; the engine has only one. The facade preserves the
  script's mental model at no cost.
- Several exports are hand-written adapters rather than direct method references, because the
  script-visible signature differs from the engine's: geometry setters that take four numbers
  where the engine takes a rectangle, colour setters that take four components where the
  engine takes a packed word, accessors that return a raw string where the engine returns an
  interned one. Each adapter marks a place where the engine's internal type was allowed to
  change while the script surface was held still.
- A few exports are marked in the source as extensions beyond the original engine's surface
  (the last-cursor-position accessors, some texture-info overloads). Those are *additive*:
  a shipped script never calls them, and a rebuild that omits them still passes the
  conformance criterion — but mods depend on them.
