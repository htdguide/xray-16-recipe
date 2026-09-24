# src/editors/xrWeatherEngine

## What this module is responsible for

The weather editor's **engine half**: a real running engine whose weather system has been replaced, record by record, with one the property grid can edit. It owns the level, the renderer, the running clock and — the point of the whole chapter — the weather document.

It is the module that answers everything the editor application asks, and it is the module that writes the files the shipped game reads.

## Where it sits and what it rests on

It rests on [`xrEngine`](../../xrEngine/README.md) — this *is* the engine, with a substituted weather system — on [`xrCore`](../../xrCore/README.md) for the configuration format and the virtual filesystem, and on [`src/Include/editor`](../../Include/editor/README.md) for the contract with the other half. It loads [`xrWeatherEditor`](../xrWeatherEditor/README.md) at run time and finds two symbols in it.

## The load-bearing ideas

**The document and the running world are the same object.** There is no import, no model, no commit. The editor's weather manager *is* the engine's weather manager, with every authored record replaced by an editable subclass that inherits the engine's own record type. A keyframe the author edits is the keyframe the engine interpolates on the next frame; a cycle's keyframe list and the engine's cycle table are the same objects in the same order. This is the decision that removes every synchronization problem an editor normally has, and it is why the property layer next door can be a set of live bindings with no apply step.

**Substitute what, never when.** The editor overrides the base weather system's factory hooks — what kind of keyframe, what kind of ambient, what kind of thunderbolt — and leaves its load sequence untouched. That is what guarantees the editor and the game agree about the world: they run the same loader over the same files in the same order.

**The keyframe layout is frozen.** [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) is the file that defines what a weather keyframe is, and the engine reads what this tool writes. Renaming a field, changing an angle's unit or dropping one breaks every installed cycle file. Angles are degrees on disk and radians in memory; texture references are extensionless logical paths; a keyframe's *section name is the time of day it takes effect*, which is also its sort key.

**The model is a set of named records in separate files, joined by name.** A keyframe names an ambient, a sun and a thunderbolt collection; an ambient names sound channels and effects; a collection names thunderbolts. Every one of those references is authored as text and resolved on load, which is why so many of the small classes here exist: each is *one name, constrained to the set the model defines*, so the grid can offer a picker instead of a text box.

**Reading delegates, writing does not.** A keyframe writes all of its fields and reads only the three the base loader would resolve and discard. That asymmetry is deliberate and is the residue of the editor once having duplicated the loader.

**Undo is a reload; there is no history.** Reverting means re-reading configuration at one of four granularities — this keyframe, the target keyframe, this cycle, every cycle. Saving replaces a cycle file wholesale, so comments and original section order do not survive. That is tolerable where the editor reads and writes every field, and not tolerable where it does not — which is exactly why the suns file is deliberately never written back.

**A rebuild should put an undo stack here**, on the side that owns the data and performs the mutations. Neither half has one today, and the split is why.

## The twins

### The module and the engine contract

| File | Role |
|---|---|
| [`editor_environment_detail.cpp`](editor_environment_detail.cpp.md) | Sort names the way a person reads them, and turn a virtual path into one a file dialog can open. |
| [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) | Declares the two helpers every part of the weather model needs: a human-friendly sort order, and a real path for a file dialog. |
| [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) | Replaces the engine's weather system with one whose every record is editable, and gathers the name lists the grid's pickers browse. |
| [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) | Declares the editable weather system: the engine's own weather system with every authored record replaced by one the grid can edit. |
| [`editor_environment_manager_properties.cpp`](editor_environment_manager_properties.cpp.md) | A demonstration file, excluded from the build: one grid row of every kind the property surface supports. |
| [`engine_impl.cpp`](engine_impl.cpp.md) | The engine's answers to the editor: advance one frame, hand over input, and read and write the weather that is running right now. |
| [`engine_impl.hpp`](engine_impl.hpp.md) | Declares the one object that answers everything the editor application asks of a running engine. |
| [`ide.hpp`](ide.hpp.md) | One accessor: reach the editor application from anywhere inside the engine-side weather model. |
| [`pch.cpp`](pch.cpp.md) | Nothing. It exists so the build has one translation unit to compile the prelude from. |
| [`pch.hpp`](pch.hpp.md) | The module's compile-time prelude: it names the engine subsystems every file in the weather-editor engine layer reaches for. |
| [`xrWeatherEngine.hpp`](xrWeatherEngine.hpp.md) | Empty. The module's public name exists as a file so the build can refer to it; it declares nothing. |

### The editable-list adapter

| File | Role |
|---|---|
| [`property_collection.hpp`](property_collection.hpp.md) | The adapter that makes any list of editable objects look to the property grid like an add/remove/reorder collection. |
| [`property_collection_forward.hpp`](property_collection_forward.hpp.md) | Announces that the editable-list adapter exists, so headers can hold a reference to one without pulling in its implementation. |
| [`property_collection_inline.hpp`](property_collection_inline.hpp.md) | How an editable list behaves: what the grid may do to it, who owns the elements, and how a new element gets a name nothing else has. |

### Weather cycles and keyframes — what this editor exists to author

| File | Role |
|---|---|
| [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) | Owns every weather cycle: discovers them as files, exposes them to the timeline, and routes clipboard and reload commands to the right one. |
| [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) | Declares the owner of every weather cycle: the set of files this editor exists to author. |
| [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) | One keyframe: the frozen set of fields the engine interpolates, the grid page that authors them, and the blend that reports what the engine is actually using. |
| [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md) | Declares one keyframe: the record the engine interpolates, plus everything needed to edit it live. |
| [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md) | One weather cycle: its file, its ordered keyframes, and the rule that a keyframe's name is the time of day it takes effect. |
| [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md) | Declares one weather cycle: a named, ordered set of keyframes that is also one file on disk. |

### Ambients: the named background sound and effect bundles a keyframe selects

| File | Role |
|---|---|
| [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) | One ambient record: which sound channels play continuously, which effects fire occasionally, and how often. |
| [`editor_environment_ambients_ambient.hpp`](editor_environment_ambients_ambient.hpp.md) | Declares one ambient record: a name, a firing period, a list of sound channels and a list of effects. |
| [`editor_environment_ambients_effect_id.cpp`](editor_environment_ambients_effect_id.cpp.md) | One entry in an ambient's effect list: nothing but a name, constrained to the effects the model actually defines. |
| [`editor_environment_ambients_effect_id.hpp`](editor_environment_ambients_effect_id.hpp.md) | Declares one entry in an ambient's effect list: a name chosen from the effects the model defines. |
| [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) | Owns the ambient records: the named bundles of background sound channels and occasional effects that a keyframe selects by name. |
| [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md) | Declares the owner of the ambient records — the named bundles of background sound and occasional effects a keyframe points at. |
| [`editor_environment_ambients_sound_id.cpp`](editor_environment_ambients_sound_id.cpp.md) | One entry in an ambient's sound-channel list: a name, constrained to the channels the model defines. |
| [`editor_environment_ambients_sound_id.hpp`](editor_environment_ambients_sound_id.hpp.md) | Declares one entry in an ambient's sound-channel list: a name chosen from the channels the model defines. |
| [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) | One effect record: what fires, where, for how long, and how hard it blows. |
| [`editor_environment_effects_effect.hpp`](editor_environment_effects_effect.hpp.md) | Declares one effect record: a particle system, a sound, an offset, a life time and a wind blast. |
| [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md) | Owns the effect records: the occasional bursts — a particle system, a sound, a gust of wind — that an ambient fires. |
| [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md) | Declares the owner of the effect records — the one-shot particle-and-sound bursts an ambient fires occasionally. |
| [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) | One sound channel: a pool of interchangeable sound files, how far they carry, and the two intervals that decide when the next one starts. |
| [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md) | Declares one sound channel: a pool of sound files, a distance range, and the four numbers that pace how often one plays. |
| [`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md) | Owns the sound-channel records: the continuous background layers an ambient mixes together. |
| [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md) | Declares the owner of the sound-channel records — the continuous background layers an ambient mixes. |
| [`editor_environment_sound_channels_source.cpp`](editor_environment_sound_channels_source.cpp.md) | One sound file in a channel's pool: a path the author picks with a file dialog. |
| [`editor_environment_sound_channels_source.hpp`](editor_environment_sound_channels_source.hpp.md) | Declares one sound file in a channel's pool: a browsable path and nothing else. |

### Suns and lens flares

| File | Role |
|---|---|
| [`editor_environment_suns_blend.cpp`](editor_environment_suns_blend.cpp.md) | Three numbers controlling how a sun's flare fades as the sun enters and leaves the view — in code nothing currently reaches. |
| [`editor_environment_suns_blend.hpp`](editor_environment_suns_blend.hpp.md) | Declares how a sun's flare fades in and out as the sun enters and leaves view. |
| [`editor_environment_suns_flare.cpp`](editor_environment_suns_flare.cpp.md) | One flare in a sun's series: where on the line it sits, how big, how bright, and what it looks like. |
| [`editor_environment_suns_flare.hpp`](editor_environment_suns_flare.hpp.md) | Declares one flare in a sun's series: a texture, an opacity, a position along the line, and a radius. |
| [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md) | A sun's flare series, decoded from four parallel lists into one editable list — in code nothing currently reaches. |
| [`editor_environment_suns_flares.hpp`](editor_environment_suns_flares.hpp.md) | Declares the flare series of a sun: a switch, a shared material, and an ordered list of flares along the line from the sun to the screen centre. |
| [`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md) | The soft halo around a sun — in code nothing currently reaches, with one abandoned experiment left in it. |
| [`editor_environment_suns_gradient.hpp`](editor_environment_suns_gradient.hpp.md) | Declares the soft halo drawn around a sun: whether it is drawn, how big, how bright, and with which material and texture. |
| [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) | Owns the sun records, and is the one sub-manager the editor deliberately refuses to save. |
| [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) | Declares the owner of the sun records — the named lens-flare setups a keyframe selects. |
| [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) | One sun record — six of its fields, which is why the suns file is never written back. |
| [`editor_environment_suns_sun.hpp`](editor_environment_suns_sun.hpp.md) | Declares one sun record: whether the disc is drawn, how big, and with which material and texture. |

### Thunderbolts

| File | Role |
|---|---|
| [`editor_environment_thunderbolts_collection.cpp`](editor_environment_thunderbolts_collection.cpp.md) | One named set of thunderbolts — stored as a section whose keys are the members and whose values are nothing. |
| [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md) | Declares one named set of thunderbolts: what a keyframe points at when it says a storm is happening. |
| [`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md) | One glow on a thunderbolt, addressed by a key prefix so the same record serves both ends of the bolt. |
| [`editor_environment_thunderbolts_gradient.hpp`](editor_environment_thunderbolts_gradient.hpp.md) | Declares one glow on a thunderbolt: a material, a texture, an opacity and a radius range. |
| [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) | Owns the thunderbolts, the named sets a keyframe draws from, and the eight world-wide parameters that place a strike in the sky. |
| [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) | Declares the owner of two related files — the individual thunderbolts and the named sets a keyframe draws from — plus the storm parameters shared by the whole world. |
| [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md) | One thunderbolt: the mesh that is drawn, the sound that follows, the light curve that flashes, and the two glows at its ends. |
| [`editor_environment_thunderbolts_thunderbolt.hpp`](editor_environment_thunderbolts_thunderbolt.hpp.md) | Declares one thunderbolt: a mesh, a sound, a light curve, and the two glowing gradients drawn at its ends. |
| [`editor_environment_thunderbolts_thunderbolt_id.cpp`](editor_environment_thunderbolts_thunderbolt_id.cpp.md) | One member of a thunderbolt set: a name, constrained to the thunderbolts the model defines. |
| [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md) | Declares one member of a thunderbolt set: a name chosen from the thunderbolts the model defines. |

### Levels

| File | Role |
|---|---|
| [`editor_environment_levels_manager.cpp`](editor_environment_levels_manager.cpp.md) | Which weather cycle each level plays: read from the two level catalogues, shown as one row per level. |
| [`editor_environment_levels_manager.hpp`](editor_environment_levels_manager.hpp.md) | Declares the level-to-weather assignment: which cycle each shipped level plays. |

The directory also carries a build description, its filter list and a package manifest. None holds a decision and none has a twin.

## What the twins record as gaps

One sub-manager is deliberately not saved (the suns file, because the editor models only part of it and a wholesale rewrite would lose the rest), several sun and thunderbolt sub-records are authored by code nothing currently reaches, and one file is a demonstration of every grid row type that is excluded from the build. Each is flagged in its own twin.
