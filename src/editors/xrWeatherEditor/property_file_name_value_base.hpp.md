# src/editors/xrWeatherEditor/property_file_name_value_base.hpp

> The contract a file-browsing row must satisfy so one chooser can serve every asset-reference property.

**Needs** — _(none)_
**Used by** — [`property_editor_file_name.cpp`](property_editor_file_name.cpp.md) · [`property_editor_file_name.hpp`](property_editor_file_name.hpp.md) · [`property_file_name_value.cpp`](property_file_name_value.cpp.md) · [`property_file_name_value.hpp`](property_file_name_value.hpp.md) · [`property_file_name_value_shared_str.cpp`](property_file_name_value_shared_str.cpp.md) · [`property_file_name_value_shared_str.hpp`](property_file_name_value_shared_str.hpp.md)
**Tier floor** — T3: a pure presentation-side capability declaration

## Purpose

A text row that names a file on disk needs five things the chooser cannot guess: what extension to assume, what to filter the listing by, where to start, what to title the dialog, and whether the chosen name keeps its extension. This interface is that list. It exists so [`property_editor_file_name.cpp`](property_editor_file_name.cpp.md) can drive any such row without knowing how its value is bound.

A substantive interface: what it demands of an implementor is the contract.

## State

Stateless.

## `default_extension`

**Contract** — The extension the chooser assumes when the user types a bare name, and the one stripped from the result when stripping is asked for. Includes the separating dot. Compared case-insensitively — the chooser lowercases before matching.

## `filter`

**Contract** — The listing filter, as the pairs of human label and pattern the platform's file chooser expects.

## `initial_directory`

**Contract** — Where the chooser opens, and — this is the load-bearing half — the **prefix the stored value is relative to**. The value the document holds is a path relative to this directory, so the chooser prepends it going in and strips it coming out.

## `title`

**Contract** — The caption on the chooser window.

## `remove_extension`

**Contract** — Whether the chosen name is stored without its extension.

**Notes** — Extension stripping is a data-format decision wearing a user-interface hat. Some engine subsystems are handed a name and append the extension themselves — a texture name is stored bare and resolved against several candidate extensions — while others store the full file name. The row knows which, because the engine told it at registration; the chooser must not assume.

**Invariants** — `initial_directory` and `default_extension` must agree with whatever the engine will do with the stored value. Nothing checks this. A row that strips an extension the engine does not re-append produces a reference that resolves to nothing, and the failure shows up at load time in the game, not in the editor.
