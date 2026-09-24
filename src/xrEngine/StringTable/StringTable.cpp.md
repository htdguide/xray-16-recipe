# src/xrEngine/StringTable/StringTable.cpp

> Every piece of player-visible text, keyed by identifier, in the language the installation is configured for.

**Needs** — [`StringTable.h`](StringTable.h.md) · [`xr_level_controller.h`](../xr_level_controller.h.md) · [`xrCore/XML/XMLDocument.hpp`](../../xrCore/XML/XMLDocument.hpp.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`StringTable.h`](StringTable.h.md)
**Tier floor** — T3: XML parsing into a map, built once in parallel at startup

## Purpose

No player-visible string is written in code. Everything — menu labels, item descriptions,
subtitles, task text — is an identifier that this table resolves against per-language XML
files shipped with the game. This file owns that resolution, the discovery of which
languages an installation has, the selection of one, and one substitution that happens
while loading rather than at display time.

**Frozen.** The XML shape and the identifiers are the shipped game's.

## State

One process-wide table. Its payload is held separately from the object so that changing
language means dropping and rebuilding the payload, while every holder of the table itself
stays valid.

```text
RECORD TableData
  language     : text                  # e.g. "rus", "eng"; also the directory name
  font_prefix  : text                  # suffix appended to font names for this language
  currency     : text                  # the money symbol for this language
  strings      : map<text, text>       # identifier -> translated text

process-wide:
  data           : optional<TableData>     # none until Init
  languages      : list<(name, index)>     # the languages this installation actually has
  language_id    : int                     # index into languages; "unset" until chosen
  language_in_ltx: text                    # what the configuration said last time

# invariant: data is none or fully built; there is no partially-loaded state visible
# invariant: languages always ends with a terminator entry, because the interface layer
#            consumes it as a menu option list
```

A lock guards insertion into the map, and only that: the files are parsed in parallel and
each parsed entry is inserted under the lock. Reads after startup are unsynchronized
because the table is immutable once built.

## `Init`

**Contract** — builds the table, or returns immediately if one already exists. Discovers
the available languages, selects one, then loads *every* XML file in that language's
directory in parallel. Finally resolves the currency symbol. Idempotent; not thread-safe
against itself.

```text
FUNCTION init(table)
  IF data EXISTS
    RETURN
  data = new TableData
  fill_language_list()
  select_language()
  files = every "*.xml" under game_config / "text" / data.language
  FOR EACH file IN files IN PARALLEL
    load(file)
  IF NOT translate("st_currency", data.currency)
     AND NOT translate("ui_st_money_descr", data.currency)
     AND NOT translate("ui_st_money_regional", data.currency)
    data.currency = config value [gameplay]/currency, defaulting to "RU"
```

**Notes** — the currency symbol is looked up under three identifiers in turn. Only the first
is this engine's; the other two are the identifiers two widely used modification branches
chose for the same thing. The fallback chain exists so those mods' string tables work
unmodified, which is a stated goal of the project. The order is "ours first", so a data set
that defines several is unambiguous.

Loading in parallel is safe because each file's entries are independent and insertion is
locked. The *outcome* is not fully deterministic, though: when two files define the same
identifier, which one wins depends on the order the parallel loads happen to finish. The
engine detects and warns about duplicates in development builds precisely because the
result is order-dependent. A rebuild that wants determinism must either load serially or
resolve duplicates by a stated rule.

## `FillLanguageToken`

**Contract** — discovers which languages this installation can actually offer by listing
the subdirectories of the text directory and rejecting the ones that are not languages.
Three rejections, each for a concrete reason:

```text
FUNCTION fill_language_list()
  languages = subdirectories of game_config / "text"
  FOR EACH name IN languages
    IF name == "map_desc"                           # level descriptions, not a language
      SKIP
    files = the files directly in that directory
    IF files IS empty                               # an empty directory is not a language
      SKIP
    IF files IS exactly one file named "openxray.xml"   # only this engine's own additions
      SKIP
    append (name, next index)
  append the terminator entry
```

**Notes** — the third rejection is the interesting one. This engine ships its own small
string file of engine-added text into *every* language directory. A directory containing
only that file means the installation has no translation for that language — offering it in
the menu would give the player an all-English game labelled as, say, Polish. The source
notes that the check must be an *else-if* chained after the empty test, since otherwise an
empty directory would be examined for a file it does not have.

Indices are assigned only to accepted languages and increment as they are accepted, so an
index is a position in the *offered* list, not in the directory listing. That is what the
interface layer's language menu indexes by.

A missing text directory entirely is warned about in development builds but is not fatal —
the engine runs with every string rendering as its own identifier, which is ugly but
diagnosable.

## `SetLanguage`

**Contract** — decides which language the table will load. There are two sources that can
change it: the configuration file (which mods edit) and the console (which the player uses),
and this routine's whole job is telling them apart.

```text
FUNCTION select_language()
  declared        = config [string_table]/language
  declared_prefix = config [string_table]/font_prefix

  IF a language was already chosen AND declared == what the config said last time
    # the config has not changed, so the console's choice stands
    data.language = languages[language_id].name
    IF data.language == declared
      data.font_prefix = declared_prefix
    ELSE
      data.font_prefix = the known prefix for this language, or none
  ELSE
    # first run, or the config changed under us: the config wins
    data.language     = declared
    data.font_prefix  = declared_prefix
    language_id       = index of declared IN languages
    FAIL WITH "check localization.ltx" IF declared is not an offered language

  remember declared AS what the config said last time
```

**Notes** — the remembered configuration value is the whole mechanism. Before a console
command existed to switch language, mods changed it by editing the configuration file; the
engine must keep honouring that. So: if the configuration says something different from
what it said last time, the configuration has been edited and wins; otherwise the player's
console choice wins. A rebuild that stores the player's choice in its own settings can
delete this dance, but not while it must accept a mod's edited configuration.

The **font prefix** exists because the shipped fonts cover different character sets. When
the player switches away from the configured language, the configured prefix no longer
applies, so a small built-in table maps language to prefix: French, German, Italian and
Spanish take the Western-European set; Polish and Czech take the Central-European set;
everything else (including Russian and English) takes none, since the default fonts cover
them. A language not in the table gets no prefix, which is correct for the two that ship.

Naming a language the installation does not have is fatal, with the configuration file
named in the message — silently falling back would hide a broken mod install.

## `Load`

**Contract** — parses one XML file into the map. Each `string` element carries an `id`
attribute and a `text` child; an entry without text is skipped with a warning rather than
stored as empty, because an empty translation displays as nothing and is indistinguishable
from a missing widget. The text is passed through the action substitution before storage.

```text
FUNCTION load(file)
  doc = parse XML at game_config / "text" / language / file
  FOR EACH "string" element i IN doc.root
    id   = attribute "id" of element i
    text = child "text" of element i
    IF text IS absent
      warn and CONTINUE
    value = parse_line(text)
    LOCK map DURING
      warn IF id already present
      map[id] = value
```

## `ParseLine`

**Contract** — performs one substitution while loading: a marker naming a *game action*
becomes a single byte encoding that action's identifier, so the display layer can replace
it with whatever key the player has currently bound. Markers naming an unknown action are
left alone.

```text
FUNCTION parse_line(text) -> text
  strip every occurrence of the action-marker byte from the input  # see note
  pos = 0
  WHILE text contains "$$ACTION_" at or after pos
    start = that position
    end   = position of the next "$$" after the name
    name  = the text between them
    action = lookup(name)
    IF action EXISTS
      replace the whole marker WITH (action-marker byte, action id as one byte)
      pos = start + 2
    ELSE
      pos = end + 2            # leave it in place and move past it
  RETURN text
```

**Notes** — this is an in-band escape: one reserved byte introduces a one-byte action code
inside otherwise ordinary text, so "press $$ACTION_JUMP$$ to jump" becomes a string the
display layer can render with the player's actual key in it, and re-render for free when
the binding changes. The reserved byte is stripped from incoming text first so localization
data cannot forge one — the source asserts on it rather than silently accepting, since a
forged code would name an arbitrary action.

The action identifier is written as a single byte, which caps the action count at 255; the
source pins that with a compile-time check. Substituting at load time rather than at display
time means the cost is paid once per string instead of once per frame, and it is why the
table must be rebuilt when key bindings change — it is not, in the original, which is a
latent bug: a binding changed after startup is not reflected in already-loaded strings.
(The display layer re-resolves the code, so in practice it is; the substitution stores the
*action*, not the key.)

## `translate`

**Contract** — three forms over the same map, all returning immediately when no table is
loaded.

```text
FUNCTION translate(id) -> text
  IF no table                RETURN id
  IF id IS in the map        RETURN map[id]
  RETURN id                                  # untranslated text displays as its own key

FUNCTION translate(id, fallback_id) -> text
  IF id resolves             RETURN it
  IF fallback_id resolves    RETURN it
  RETURN id

FUNCTION translate(id, out value) -> bool    # the only form that reports a miss
```

**Notes** — returning the identifier itself for a miss is deliberate: a missing string shows
up on screen as `ui_st_something_missing`, which is immediately recognizable and searchable,
rather than as blank space. Every caller relies on it, so a rebuild must not return empty.

## `ReloadLanguage` / `rescan` / `Destroy`

**Contract** — `ReloadLanguage` rebuilds the table when the selected index no longer matches
the loaded language, and does nothing otherwise; this is what the language console command
calls. `rescan` rebuilds only when nothing is loaded. `Destroy` drops the payload and
releases the language list's names, which are separately allocated because the list is
handed to the interface layer as a menu option array.
