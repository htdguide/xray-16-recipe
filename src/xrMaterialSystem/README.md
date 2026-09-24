# Chapter 8 — `src/xrMaterialSystem`

> Surface materials and the pairwise interaction table.

Five files and about six hundred lines, and almost every other chapter reads them. This
module answers one question — *what is this surface, and what happens when this other
surface meets it* — and then physics, sound, the renderer, the AI, the damage model and
the weapon code all ask it. It is the smallest module in the engine with the widest reach,
and the reason it is a module at all rather than a header in the core is that it is the
one piece of shared vocabulary that the authoring tool, the game and the renderer must
agree on exactly.

## Where it sits

It rests on the core library only: the virtual filesystem and the chunked-container reader
from chapter 6, the containers from chapter 2, the vector types from chapter 3. It reaches
the sound device and the renderer exclusively through the interfaces declared in chapter 4
— it asks for a sound handle and for a decal array and never learns what fills either.
That discipline is what lets a module this widely depended-on sit this early in the build
order.

It has one coupling that is not a link edge and is easy to miss: a collision triangle in
chapter 7 carries a 16-bit surface number, and that number is an index into this module's
material list. The two modules never reference each other, and the contract between them
is enforced at level load by the game, which rewrites every triangle's number from the
library's authored identifier to the library's load-time index. A rebuild that changes how
materials are ordered in memory has changed the meaning of every triangle in every level.

## The load-bearing ideas

**A material is a surface, not a look.** It has no textures and no shaders. It carries
friction and bounce for the solver, a penetration threshold for bullets, a transparency
number for the AI's line of sight, a rate at which standing on it costs health, and a bit
set of yes/no facts — passable, climbable, liquid, dynamic. Visual material description is
a different system entirely, and in this engine's vocabulary the word *shader* usually
means that other thing.

**The pairwise table is the design.** Anything that needs two surfaces to decide — which
footstep sound, which impact sound, which particle burst, which decal — lives not on
either material but on the *pair*. The library authors only the combinations that need
something, so the pair list is sparse; the engine expands it once at load into a dense
square table indexed by both material indices, so that the lookup inside a physics contact
callback is one array read. The table is symmetric: a pair authored as (A, B) answers a
query for (B, A), and the engine therefore cannot express a direction. Where direction
matters it is smuggled in as a material: a bullet is a material (`objects\bullet`), a
knife is a material (`objects\knife`), and "bullet hits concrete" is a pair like any
other. Most cells of the table are empty and every caller must have a fallback.

**Identifier versus index.** A material has an authored identifier, stable in the file and
never reused, and a load-time index, which is its position in the loaded list. The file
speaks identifiers; everything in a frame speaks indices, because an index fits in the
14 bits a collision triangle can spare (see [`xrCDB/xrCDB.cpp`](../xrCDB/xrCDB.cpp.md),
which owns that packing) and because it addresses the pair table directly. Fourteen bits
caps a level at 16384 distinct static materials.
Getting these two confused is the standard way to corrupt a level's surfaces, and the
engine defends against it with a checksum: the cached collision model records which
library it was remapped against and is discarded if that library changed.

**The library is the game detector.** The library file has carried the same version number
since 2002 across three games that changed it substantially. So the engine infers the
generation from the *shape* of the data — whether the material records carry a density
chunk, whether they carry a multiplayer penetration chunk — and that inference then
selects rule variants nowhere near this module: how a cartridge's armour-piercing value is
read, which sign convention an artefact's damage immunities use, what a bone's default hit
fraction is, and which of three incoming-damage formulas a stalker uses. A rebuild that
wants to run all three games needs this inference; a rebuild targeting one game can
hard-code the answer.

**The file is frozen and the write side is absent.** `gamemtl.xr` ships with the game and
the reading side must accept it byte for byte, including three burned chunk identifiers
and two burned flag bits that must stay dead so the live ones keep their positions. The
writing side lives in the authoring tool and is not in this repository.

## Who reads what

Grepped from the source; each row names a concrete consumer, not a possibility.

| Property | Read by | For |
|---|---|---|
| `friction` | physics contact callback; character slope movement | contact friction is the **product** of both surfaces' values |
| `spring`, `damping` | physics contact callback | multiplied together and against world constants into the solver's soft-contact terms |
| `bounce_start_velocity`, `bouncing` | physics contact callback | bounce is enabled only when **both** surfaces set `bounceable`; the threshold is the max of the two, the restitution the min |
| `flotation_factor` | physics contact body effector | drag applied to a body inside a slow-down surface, scaled by the square of the deficit from 1 |
| `bounce_damage_factor` | collision damage receiver; character impact damage | damage from a collision is split between the two surfaces in proportion to their factors |
| `shoot_factor` | bullet trace; explosion trace | the armour a bullet's penetration value must exceed; the surplus becomes the fraction of speed that survives |
| `shoot_factor_mp` | *(loaded, no reader in this tree)* | the multiplayer rule set's penetration value |
| `density_factor` | bullet trace | energy lost per metre while the bullet is inside a solid |
| `injurious_speed` | actor condition; character movement | health lost per second while standing on it |
| `vis_transparency` | AI visual memory; lens flare occlusion; crosshair target test | how much a surface between two points blocks sight |
| `snd_occlusion_factor` | this module only | seeds the acoustics fallback; no other reader |
| `acoustics` | *(no reader in this tree)* | three-band absorption, scattering and transmission for a reverb model |
| `passable` | physics contact; grass placement; foot IK; actor senses and camera | the contact is discarded, the grass blade is not grown, the foot does not step on it |
| `actor_obstacle` | character collision; vehicles; foot IK | passable to everything except a character's body |
| `climable` | character collision; foot IK | may be climbed |
| `liquid` | physics contact | slow-down is applied as buoyancy rather than drag |
| `slow_down` | physics contact | gates the drag effector at all |
| `bounceable` | physics contact | both surfaces must agree before a contact bounces |
| `dynamic` | level load; model pool | partitions the library: dynamic materials belong to model bones, static ones to level geometry |
| `suppress_shadows`, `suppress_wallmarks` | level load | baked into each collision triangle so the renderer need not consult the library per triangle |
| `bloodmark` | wound decals on creatures | a hit on this surface stamps blood rather than a generic mark |
| `no_ricochet` | bullet trace | bullets never deflect |
| `injurious` | character movement | tracks which surface is currently hurting the character |
| `breakable`, `skidmark`, `shootable`, `transparent` | nobody | authored, loaded, never read |
| pair `step_sounds` | footstep player; animation-driven step manager | one of several recordings per surface pair, never the same one twice in a row |
| pair `breaking_sounds` | footstep player | reused as the *running* footstep variant when the pair has one |
| pair `collide_sounds` | physics collision sound player; bullet impact; poltergeist ability | impact sound, picked uniformly at random |
| pair `collide_particles` | physics collision; bullet impact; footsteps | impact burst, resolved by name at spawn time |
| pair `collide_marks` | bullet impact; physics scrape | the decal stamped at the contact point |
| library generation | ammunition load; bone protection; stalker damage; artefact immunities and their UI | selects which game's rules apply |
| library checksum | level load | invalidates a cached collision model when the library changed |

Six material names are hard-coded in the engine and must exist in any library it loads:
`default` (the fallback, and the remap target for an unrecognized triangle),
`default_object` (which must be marked dynamic), `objects\bullet`, `objects\knife`,
`objects\small_box` (the default for physics geometry) and `materials\earth_slide`.

## The files

| Twin | Role |
|---|---|
| [`GameMtlLib.h`](GameMtlLib.h.md) | The substance: material and pair records, the flag vocabulary, the frozen chunk identifiers, the generation ladder, and the library's whole query surface |
| [`GameMtlLib.cpp`](GameMtlLib.cpp.md) | Reads the material half of the file field by field, infers the generation, supplies acoustics the shipped files lack, builds the dense pair table |
| [`GameMtlLib_Engine.cpp`](GameMtlLib_Engine.cpp.md) | Reads the pair half and turns its comma-separated name lists into live sounds, particle names and decals — the only part that touches a device |
| [`stdafx.h`](stdafx.h.md) | Compile-time prelude; incidental |
| [`stdafx.cpp`](stdafx.cpp.md) | Anchor for the prelude; incidental |

The directory also holds a build description and two project files for one toolchain.
They are build-system artifacts, not program content, and have no twins; what they say
that matters is stated above — the module depends on the core library and on the shared
interfaces, and on nothing else.
