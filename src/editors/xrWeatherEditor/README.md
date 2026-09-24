# src/editors/xrWeatherEditor

## What this module is responsible for

The weather editor's **application half**: the windows, the menus, the timeline, the property grid, and — most of the directory by file count — the layer that turns the engine's description of an editable object into rows the grid can show.

It is one of the two modules the editor is split into. The other, [`xrWeatherEngine`](../xrWeatherEngine/README.md), is a real running engine with every weather record replaced by an editable one. This module owns the window and the message loop; that one owns the level, the renderer and the data. The two are separately built, in different implementation languages, and everything they may say to each other is [four headers](../../Include/editor/README.md).

## Where it sits and what it rests on

It rests on [`src/Include/editor`](../../Include/editor/README.md) for the contract, on [`xrSdkControls`](../xrSdkControls/README.md) for every widget, on a third-party property-bag library for the grid's row descriptions and a third-party docking library for the window layout, and on [`xrCore`](../../xrCore/README.md) and [`xrEngine`](../../xrEngine/README.md) for the engine types that appear in the contract. It is loaded at run time by the host, which finds exactly two symbols in it.

Nothing in the engine rests on this module. A rebuild that does not want the weather editor deletes chapter 29 entire.

## The load-bearing ideas

Seven ideas explain almost every file here. The twins are terse because these are named once.

**Control is inverted, and it costs.** The editor owns the message loop; the engine owns one "advance a frame" call and one "would you like this input event" call. The editor's idle handler pumps the engine in a tight loop until a message arrives. The consequence is structural and visible: **a modal dialog starves the engine**, so the rendered view freezes whenever a picker or a list editor is open. That is why [the colour picker is disabled](property_color_base.cpp.md) and why [the list editor repaints the view when it is dragged](property_collection_editor.cpp.md). A rebuild should invert this back — engine owns the loop, editor panels are drawn by the engine's own overlay — and every one of these symptoms disappears with it.

**A property is a binding, not a value.** The engine passes a getter and setter, or a direct reference to the field. The grid reads on paint and writes on edit. Nothing is copied, nothing is synchronized, there is no apply step, and editor/engine divergence is impossible to express.

**Every binding comes in a matrix.** Two binding styles (accessor pair, or direct field reference) multiplied by however many presentations a value type has (plain, range-clamped, named choices, indexed labels, live-regenerated lists). That product is why this directory holds forty-odd near-identical property classes. The duplication is forced by the boundary — a composed wrapper would need a template the two-language boundary forbids — and **a rebuild collapses it by composing a presentation around a binding instead of inheriting.** Doing so turns forty files into roughly eight.

**Presentation is split from data.** For every choice-valued property, the binding holds the value and the label list and parses a chosen label back; a separate *renderer* reads the label list off the binding and turns the stored value into a label. Both halves read the same list, which is why the list is public on every one of these bindings.

**Order is imposed by prefixing.** The grid sorts rows and categories alphabetically and offers no other control. The engine, however, describes an object's fields in a meaningful order. [`property_container`](property_container.cpp.md) works around this by prefixing older categories with a character that sorts before every printable one, so declaration order survives — and by prefixing duplicate names for the same reason. A rebuild whose grid accepts an explicit order deletes both rules.

**Strings and layouts cannot cross by value.** Text crossing the boundary is converted and copied, in one direction obliging the caller to free it and in the other not; a display name is written into a caller-supplied fixed buffer; two small numeric records are declared with an explicit layout on both sides because neither module may assume the other's rules. All of it is the boundary showing through, and all of it vanishes in a single-language rebuild — but the *ownership question* it encodes does not, and a rebuild that keeps the split must answer it the same way: **the module that allocated a thing is the module that frees it.**

**Release is not deterministic on one side.** Grid objects are collected at an unpredictable moment, possibly after the unmanaged half has been torn down. [`property_container`](property_container.cpp.md) guards against telling a destroyed holder that it went away, by checking that the editor root still exists. Any rebuild with two memory regimes meets this; the recipe's recommendation is to make container release deterministic and delete the class of bug.

## What the twins record as defects

Several, and two of them matter.

**Real numbers are printed with locale-independent rules and parsed with locale-dependent ones** ([`property_converter_float.cpp`](property_converter_float.cpp.md)). On a machine whose locale uses a comma as the decimal separator, the editor prints a value it then refuses to accept back. Every real-valued row is affected, which is most of the weather model.

**An out-of-set choice is rendered as the first choice** ([`property_converter_integer_enum.cpp`](property_converter_integer_enum.cpp.md), [`property_converter_float_enum.cpp`](property_converter_float_enum.cpp.md)), so a value the data does not hold is shown, and touching the row commits it — while the indexed-label variant ([`property_converter_integer_values.cpp`](property_converter_integer_values.cpp.md)) does not bounds-check at all and faults instead.

The individual twins carry the rest: a display-name buffer that truncates silently, an append that returns the wrong position, a copy-out that indexes the source wrongly, a colour drag whose clamping is not reversible, and a multiple-selection path that shows the first object's rows rather than the intersection.

## The twins

### The module boundary

| File | Role |
|---|---|
| [`AssemblyInfo.cpp`](AssemblyInfo.cpp.md) | The editor library's identity as a loadable module. |
| [`engine_include.hpp`](engine_include.hpp.md) | Pulls in the engine-facing contract with the compilation mode the boundary requires. |
| [`entry_point.cpp`](entry_point.cpp.md) | The two exported symbols of the editor library, and the idle pump that runs a fixed-rate engine inside an event-driven application. |
| [`ide_impl.cpp`](ide_impl.cpp.md) | The editor's side of the engine contract: two window handles, the application loop, the per-frame repaint, and the factory for property holders. |
| [`ide_impl.hpp`](ide_impl.hpp.md) | Declares the editor's implementation of the engine-facing interface — the class the host talks to. |
| [`pch.cpp`](pch.cpp.md) | The translation unit that exists so the shared preamble has something to compile into. |
| [`pch.hpp`](pch.hpp.md) | The library's shared preamble, and the two text conversions every call across the language boundary needs. |
| [`resource.h`](resource.h.md) | Names one embedded image: the eyedropper cursor the three-dimensional view shows when a colour can be sampled. |

### The windows

| File | Role |
|---|---|
| [`window_ide.cpp`](window_ide.cpp.md) | The application frame: builds the four panels, tracks the geometry worth restoring, and turns a window close into an engine quit. |
| [`window_ide.h`](window_ide.h.md) | Declares the application's main window: the frame that owns the dock, the four panels and the engine reference. |
| [`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) | Remembers the author's window layout between sessions, and puts it back. |
| [`window_levels.cpp`](window_levels.cpp.md) | The level browser panel: one grid showing which weather cycle each level plays. |
| [`window_levels.h`](window_levels.h.md) | Declares the level browser panel. |
| [`window_tree_values.cpp`](window_tree_values.cpp.md) | The modal tree chooser a text row opens when its list of admissible values is long and hierarchical. |
| [`window_tree_values.h`](window_tree_values.h.md) | Declares the modal tree chooser. |
| [`window_view.cpp`](window_view.cpp.md) | The engine's window inside the editor: where the frame is pumped, where input is claimed, and where two gestures edit a value by pointing at the world. |
| [`window_view.h`](window_view.h.md) | Declares the panel the engine renders into: a toolbar with two toggles over a bare surface. |
| [`window_weather.cpp`](window_weather.cpp.md) | The weather browser panel: one grid over the whole weather model's non-keyframe records. |
| [`window_weather.h`](window_weather.h.md) | Declares the weather browser panel. |
| [`window_weather_editor.cpp`](window_weather_editor.cpp.md) | The timeline: two scrubbers, two pickers, and the three keyframe grids — the editor's main working surface. |
| [`window_weather_editor.h`](window_weather_editor.h.md) | Declares the timeline panel: cycle and keyframe pickers, a day scrubber, a blend scrubber, and three property grids. |

### The document node and its grid face

| File | Role |
|---|---|
| [`property_container.cpp`](property_container.cpp.md) | One editable object as the grid sees it: an ordered set of described rows, each bound to a live accessor pair — plus the two disambiguation rules that let an object carry duplicate names and categories. |
| [`property_container.hpp`](property_container.hpp.md) | Declares the grid-facing face of an editable object, and attaches its renderer. |
| [`property_container_converter.cpp`](property_container_converter.cpp.md) | Makes a nested object expand into a sub-grid, and forces its rows back into the order the engine declared them. |
| [`property_container_converter.hpp`](property_container_converter.hpp.md) | Declares the nested-object renderer. |
| [`property_container_holder.hpp`](property_container_holder.hpp.md) | An empty marker: "this object owns a property container". |
| [`property_holder.cpp`](property_holder.cpp.md) | One document node as the editor sees it: the identity, the lifetime, and the rule that deleting a node from the grid deletes it from the document. |
| [`property_holder.hpp`](property_holder.hpp.md) | Declares the editor's implementation of the engine's property-holder interface — one document node, presentable as a grid of editable properties. |
| [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md) | Registers boolean properties on a node, choosing between a plain checkbox and a two-label choice. |
| [`property_holder_collection.cpp`](property_holder_collection.cpp.md) | Registers an ordered, user-editable list of child nodes as a single property. |
| [`property_holder_color.cpp`](property_holder_color.cpp.md) | Registers colour properties, giving each one a swatch in the grid row and a modal picker. |
| [`property_holder_container.cpp`](property_holder_container.cpp.md) | Registers one node as a nested property of another, which is how the document becomes a tree. |
| [`property_holder_float.cpp`](property_holder_float.cpp.md) | Registers real-valued properties, and fixes the step size a drag or a spin applies. |
| [`property_holder_include.hpp`](property_holder_include.hpp.md) | Declares the compilation boundary between the native engine interface and the managed user-interface layer, and the two tiny adapters every property value in the editor is built from. |
| [`property_holder_integer.cpp`](property_holder_integer.cpp.md) | Registers whole-number properties in five shapes, including the one where the stored number is an index into a list the engine rebuilds on demand. |
| [`property_holder_string.cpp`](property_holder_string.cpp.md) | Registers text properties, and picks which of three editing affordances — free text, a chooser, or a file browser — the user gets. |
| [`property_holder_vec3f.cpp`](property_holder_vec3f.cpp.md) | Registers three-component vector properties, which the grid shows as one row that expands into three. |
| [`property_property_container.cpp`](property_property_container.cpp.md) | The row that makes the document a tree: its value is another node's whole property set. |
| [`property_property_container.hpp`](property_property_container.hpp.md) | Declares the adapter that makes one node's value be another node's grid presentation. |

### Bindings — boolean

| File | Role |
|---|---|
| [`property_boolean.cpp`](property_boolean.cpp.md) | A yes/no grid cell bound to a pair of callables on the engine side. |
| [`property_boolean.hpp`](property_boolean.hpp.md) | Declares the callable-bound boolean property. |
| [`property_boolean_reference.cpp`](property_boolean_reference.cpp.md) | The same yes/no cell, bound straight to the field instead of to a pair of callables. |
| [`property_boolean_reference.hpp`](property_boolean_reference.hpp.md) | Declares the field-bound boolean property. |
| [`property_boolean_values_value.cpp`](property_boolean_values_value.cpp.md) | A boolean shown as two words of the author's choosing, rather than as a checkbox. |
| [`property_boolean_values_value.hpp`](property_boolean_values_value.hpp.md) | Declares the two-label boolean property. |
| [`property_boolean_values_value_reference.cpp`](property_boolean_values_value_reference.cpp.md) | The two-label boolean, bound to a field instead of to callables. |
| [`property_boolean_values_value_reference.hpp`](property_boolean_values_value_reference.hpp.md) | Declares the field-bound two-label boolean property. |

### Bindings — whole numbers

| File | Role |
|---|---|
| [`property_integer.cpp`](property_integer.cpp.md) | One grid row bound to a whole number in the engine by a getter and a setter. |
| [`property_integer.hpp`](property_integer.hpp.md) | Declares the accessor-bound whole-number property adapter. |
| [`property_integer_enum_value.cpp`](property_integer_enum_value.cpp.md) | A whole number that may only take one of an authored set of values, chosen by name. |
| [`property_integer_enum_value.hpp`](property_integer_enum_value.hpp.md) | Declares the accessor-bound whole-number adapter restricted to an authored set of named values. |
| [`property_integer_enum_value_reference.cpp`](property_integer_enum_value_reference.cpp.md) | The named-value whole-number row, bound by alias instead of by callbacks. |
| [`property_integer_enum_value_reference.hpp`](property_integer_enum_value_reference.hpp.md) | Declares the reference-bound whole-number adapter restricted to an authored set of named values. |
| [`property_integer_limited.cpp`](property_integer_limited.cpp.md) | A whole-number row that clamps to an authored range in both directions. |
| [`property_integer_limited.hpp`](property_integer_limited.hpp.md) | Declares the range-clamped accessor-bound whole-number adapter. |
| [`property_integer_limited_reference.cpp`](property_integer_limited_reference.cpp.md) | The range-clamped whole-number row, bound by alias instead of by callbacks. |
| [`property_integer_limited_reference.hpp`](property_integer_limited_reference.hpp.md) | Declares the range-clamped reference-bound whole-number adapter. |
| [`property_integer_reference.cpp`](property_integer_reference.cpp.md) | One grid row aliased straight onto a whole-number field the engine owns. |
| [`property_integer_reference.hpp`](property_integer_reference.hpp.md) | Declares the reference-bound whole-number property adapter. |
| [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md) | A whole number that is a position in a label list, with the list fixed when the row was built. |
| [`property_integer_values_value.hpp`](property_integer_values_value.hpp.md) | Declares the accessor-bound index-into-a-fixed-label-list adapter. |
| [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md) | The contract a converter relies on to ask a whole-number row for the label list it indexes into. |
| [`property_integer_values_value_getter.cpp`](property_integer_values_value_getter.cpp.md) | A whole number indexing a list that only exists while the editor is running, so the list is asked for every time. |
| [`property_integer_values_value_getter.hpp`](property_integer_values_value_getter.hpp.md) | Declares the accessor-bound index adapter whose label list is regenerated on every query. |
| [`property_integer_values_value_reference.cpp`](property_integer_values_value_reference.cpp.md) | The fixed-list index row, bound by alias instead of by callbacks. |
| [`property_integer_values_value_reference.hpp`](property_integer_values_value_reference.hpp.md) | Declares the reference-bound index-into-a-fixed-label-list adapter. |
| [`property_integer_values_value_reference_getter.cpp`](property_integer_values_value_reference_getter.cpp.md) | The live-list index row, with the index aliased onto an engine field. |
| [`property_integer_values_value_reference_getter.hpp`](property_integer_values_value_reference_getter.hpp.md) | Declares the reference-bound index adapter whose label list is regenerated on every query. |

### Bindings — reals

| File | Role |
|---|---|
| [`property_float.cpp`](property_float.cpp.md) | One grid row bound to a real number in the engine by a getter and a setter, plus the step a nudge applies. |
| [`property_float.hpp`](property_float.hpp.md) | Declares the accessor-bound real-number property adapter. |
| [`property_float_enum_value.cpp`](property_float_enum_value.cpp.md) | A real that may only take one of an authored set of magnitudes, chosen by name. |
| [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md) | Declares the accessor-bound real adapter restricted to an authored set of named magnitudes. |
| [`property_float_enum_value_reference.cpp`](property_float_enum_value_reference.cpp.md) | The named-magnitude real row, bound by alias instead of by callbacks. |
| [`property_float_enum_value_reference.hpp`](property_float_enum_value_reference.hpp.md) | Declares the reference-bound real adapter restricted to an authored set of named magnitudes. |
| [`property_float_limited.cpp`](property_float_limited.cpp.md) | A real row that clamps to an authored range on the way out as well as on the way in. |
| [`property_float_limited.hpp`](property_float_limited.hpp.md) | Declares the range-clamped accessor-bound real adapter. |
| [`property_float_limited_reference.cpp`](property_float_limited_reference.cpp.md) | The range-clamped real row, bound by alias instead of by callbacks. |
| [`property_float_limited_reference.hpp`](property_float_limited_reference.hpp.md) | Declares the range-clamped reference-bound real adapter. |
| [`property_float_reference.cpp`](property_float_reference.cpp.md) | One grid row aliased straight onto a real field the engine owns. |
| [`property_float_reference.hpp`](property_float_reference.hpp.md) | Declares the reference-bound real-number property adapter. |

### Bindings — text and file names

| File | Role |
|---|---|
| [`property_file_name_value.cpp`](property_file_name_value.cpp.md) | A text row that names a file, carrying the five settings its chooser needs. |
| [`property_file_name_value.hpp`](property_file_name_value.hpp.md) | Declares the accessor-bound text row that names a file. |
| [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md) | The contract a file-browsing row must satisfy so one chooser can serve every asset-reference property. |
| [`property_file_name_value_shared_str.cpp`](property_file_name_value_shared_str.cpp.md) | The file-naming row, bound to an interned-text slot. |
| [`property_file_name_value_shared_str.hpp`](property_file_name_value_shared_str.hpp.md) | Declares the interned-text row that names a file. |
| [`property_string.cpp`](property_string.cpp.md) | One grid row bound to a text value in the engine, with a copy made at every crossing in both directions. |
| [`property_string.hpp`](property_string.hpp.md) | Declares the accessor-bound text property adapter. |
| [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md) | A text row aliased onto an interned-text slot, with every read and write routed through the engine so the store's accounting stays correct. |
| [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) | Declares the text adapter bound to a slot in the engine's interned-text store. |
| [`property_string_values_value.cpp`](property_string_values_value.cpp.md) | A text row that also carries the list of values it admits, snapshotted when the row was built. |
| [`property_string_values_value.hpp`](property_string_values_value.hpp.md) | Declares the accessor-bound text adapter restricted to a fixed set of admissible values. |
| [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) | The contract a converter or a chooser relies on to ask a text row which values it may take. |
| [`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md) | A text row whose set of admissible values is asked for fresh every time, because it changes while the editor runs. |
| [`property_string_values_value_getter.hpp`](property_string_values_value_getter.hpp.md) | Declares the accessor-bound text adapter whose admissible set is regenerated on every query. |
| [`property_string_values_value_shared_str.cpp`](property_string_values_value_shared_str.cpp.md) | The fixed-choice text row, bound to an interned-text slot. |
| [`property_string_values_value_shared_str.hpp`](property_string_values_value_shared_str.hpp.md) | Declares the interned-text adapter restricted to a fixed set of admissible values. |
| [`property_string_values_value_shared_str_getter.cpp`](property_string_values_value_shared_str_getter.cpp.md) | The live-choice text row, bound to an interned-text slot. |
| [`property_string_values_value_shared_str_getter.hpp`](property_string_values_value_shared_str_getter.hpp.md) | Declares the interned-text adapter whose admissible set is regenerated on every query. |

### Bindings — colour and vector

| File | Role |
|---|---|
| [`property_color.cpp`](property_color.cpp.md) | The colour binding, reading and writing through the engine's callables. |
| [`property_color.hpp`](property_color.hpp.md) | Declares the callable-bound colour property. |
| [`property_color_base.cpp`](property_color_base.cpp.md) | A colour as three linked real rows plus one row that shows and parses all three — the editor's most-used property type, and the one composite binding in the directory. |
| [`property_color_base.hpp`](property_color_base.hpp.md) | Declares the composite colour binding, its three-channel accessor helper, and the boundary-safe colour record. |
| [`property_color_reference.cpp`](property_color_reference.cpp.md) | The colour binding, reading and writing a colour field directly. |
| [`property_color_reference.hpp`](property_color_reference.hpp.md) | Declares the field-bound colour property. |
| [`property_vec3f.cpp`](property_vec3f.cpp.md) | The vector row bound to the engine by a whole-vector getter and setter. |
| [`property_vec3f.hpp`](property_vec3f.hpp.md) | Declares the accessor-bound vector property adapter. |
| [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md) | A vector row is one value with three editable faces: the whole triple as text, and three component rows that each read-modify-write it. |
| [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md) | Declares the vector property adapter's shared half: the three-component presentation value, the nested component rows, and the abstract whole-vector access an implementor must supply. |
| [`property_vec3f_reference.cpp`](property_vec3f_reference.cpp.md) | A grid row bound directly to a three-component vector the engine owns — the smallest place the two-runtime boundary shows itself. |
| [`property_vec3f_reference.hpp`](property_vec3f_reference.hpp.md) | Declares the grid row for a three-component vector the engine owns, edited in place. |

### Bindings — editable lists

| File | Role |
|---|---|
| [`property_collection.cpp`](property_collection.cpp.md) | The editable list, bound to one engine collection that will not move. |
| [`property_collection.hpp`](property_collection.hpp.md) | Declares the directly bound editable list. |
| [`property_collection_base.cpp`](property_collection_base.cpp.md) | An editable list of engine objects, presented to the grid as an ordinary indexed collection — add, remove, reorder — with every operation forwarded straight to the engine. |
| [`property_collection_base.hpp`](property_collection_base.hpp.md) | Declares the editable-list binding and attaches the two presentation pieces every collection row uses. |
| [`property_collection_converter.cpp`](property_collection_converter.cpp.md) | What a collection row shows when it is not open: an ellipsis, because a list has no one-line value. |
| [`property_collection_converter.hpp`](property_collection_converter.hpp.md) | Declares the collection row's text rendering. |
| [`property_collection_editor.cpp`](property_collection_editor.cpp.md) | The add/remove/reorder dialog for a collection row: what a new element is, what each element is called, and the one repaint that keeps the rendered view honest while the dialog is open. |
| [`property_collection_editor.hpp`](property_collection_editor.hpp.md) | Declares the collection row's dialog. |
| [`property_collection_enumerator.cpp`](property_collection_enumerator.cpp.md) | Walks an engine collection for the grid, one element at a time, holding only a position. |
| [`property_collection_enumerator.hpp`](property_collection_enumerator.hpp.md) | Declares the collection walk. |
| [`property_collection_getter.cpp`](property_collection_getter.cpp.md) | The editable list, where *which* collection is a question asked fresh on every access. |
| [`property_collection_getter.hpp`](property_collection_getter.hpp.md) | Declares the deferred editable list. |

### Renderers

| File | Role |
|---|---|
| [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md) | Turns a boolean into one of two author-supplied words, and offers exactly those two words as the dropdown. |
| [`property_converter_boolean_values.hpp`](property_converter_boolean_values.hpp.md) | Declares the two-label boolean renderer. |
| [`property_converter_color.cpp`](property_converter_color.cpp.md) | A colour's one-line form: three space-separated reals that the author can read, type and paste. |
| [`property_converter_color.hpp`](property_converter_color.hpp.md) | Declares the colour row's text rendering and sub-row ordering. |
| [`property_converter_float.cpp`](property_converter_float.cpp.md) | Every real number in the editor is shown to three decimal places — and that decision is frozen into the file the engine reads. |
| [`property_converter_float.hpp`](property_converter_float.hpp.md) | Declares the real-number text rendering. |
| [`property_converter_float_enum.cpp`](property_converter_float_enum.cpp.md) | Renders a real as the label of the nearest declared choice — an enumeration whose underlying values happen to be real numbers. |
| [`property_converter_float_enum.hpp`](property_converter_float_enum.hpp.md) | Declares the named-real renderer. |
| [`property_converter_integer_enum.cpp`](property_converter_integer_enum.cpp.md) | Renders a whole number as the label of its declared choice — the ordinary enumeration case. |
| [`property_converter_integer_enum.hpp`](property_converter_integer_enum.hpp.md) | Declares the named-integer renderer. |
| [`property_converter_integer_values.cpp`](property_converter_integer_values.cpp.md) | Renders an integer as a label *indexed by* the integer — the case where the stored value is the position in the choice list. |
| [`property_converter_integer_values.hpp`](property_converter_integer_values.hpp.md) | Declares the indexed-label integer renderer. |
| [`property_converter_string_values.cpp`](property_converter_string_values.cpp.md) | Hands the grid the list of values a text row admits, and declares the list closed. |
| [`property_converter_string_values.hpp`](property_converter_string_values.hpp.md) | Declares the converter that turns a text row's admissible set into a drop-down, and the variant that forbids typing. |
| [`property_converter_tree_values.cpp`](property_converter_tree_values.cpp.md) | One answer: no, this row cannot be typed into. |
| [`property_converter_tree_values.hpp`](property_converter_tree_values.hpp.md) | Declares the converter whose only job is to refuse typed input. |
| [`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md) | Makes a vector row readable as one line of three numbers and writable the same way, and forces its child rows into axis order. |
| [`property_converter_vec3f.hpp`](property_converter_vec3f.hpp.md) | Declares the converter that gives a vector row its text form and its three ordered child rows. |

### Cell editors

| File | Role |
|---|---|
| [`property_editor_color.cpp`](property_editor_color.cpp.md) | Paints a colour row's swatch and opens a picker for it, converting between the engine's unbounded reals and the picker's bytes at both ends. |
| [`property_editor_color.hpp`](property_editor_color.hpp.md) | Declares the colour row's swatch painter and modal picker. |
| [`property_editor_file_name.cpp`](property_editor_file_name.cpp.md) | Turns a file the artist picked into a path relative to the asset root, which is the only form the document may hold. |
| [`property_editor_file_name.hpp`](property_editor_file_name.hpp.md) | Declares the file chooser a text row opens to pick an asset. |
| [`property_editor_tree_values.cpp`](property_editor_tree_values.cpp.md) | Opens a long list of admissible values as a browsable tree, and stores what the artist picked. |
| [`property_editor_tree_values.hpp`](property_editor_tree_values.hpp.md) | Declares the modal tree chooser a text row opens to pick from a long, hierarchical list. |


The directory also carries a build description and its filter list, a package manifest, six per-window resource tables, a resource script and five small toolbar images. None holds a decision and none has a twin.
