# GTA6 Asset Info

Technical notes on the asset formats of **GTA VI (Xbox Series build)**:
how the archives, textures, models and map data are laid out, what is
understood, and what is still unknown.

Everything here was worked out from the data itself: headers, statistics
over many samples, and checking decoded output by eye. Nothing has been
confirmed against game code or official documentation. This repository holds
documentation only: no game files, no extracted assets, no keys. Read
[LEGAL.md](LEGAL.md).

## Formats

| Area | Pages |
|------|-------|
| Containers | [RPF9 archives](formats/01-rpf9.md) · [Resources and pages](formats/02-resources.md) |
| Textures | [Texture dictionaries](formats/03-texture-dictionaries.md) · [Xbox tiling](formats/04-xbox-tiling.md) · [`ttpf` stores](formats/05-ttpf.md) · [BCPack](formats/06-bcpack.md) |
| Models | [Drawables, fragments and vehicles](formats/07-models.md) |
| Map data | [Meta resources](formats/08-meta.md) · [Placements and batches](formats/09-placements.md) · [Interiors](formats/10-interiors.md) |
| World | [Collision](formats/13-collision.md) · [Navmesh](formats/14-navmesh.md) · [Lights](formats/15-lights.md) · [Road graph](formats/16-roads.md) |
| Materials | [Vehicle paint and tables](formats/11-vehicle-paint.md) · [Material classes](formats/12-materials.md) |
| Other | [PSO, RBF, BVH4, audio, animation, body shape, video, UI, world height map](formats/17-other.md) · [Radio stations](formats/18-radio.md) |
| Byte-level notes | [Index](structure/00-INDEX.md) · [RPF9 container](structure/01-rpf9-container.md) · [Resources and textures](structure/02-resources-textures.md) · [Audio (`ADAT`)](structure/03-audio-awc.md) · [PSO and object graph](structure/04-pso-objectgraph.md) · [`ttpf` stores](structure/05-ttpf-stores.md) · [Models and shaders](structure/06-models-shaders.md) |

The [format index](formats/README.md) has the conventions. Every page gives
each field a confidence level and ends with its open questions. The
[structure/](structure/00-INDEX.md) pages give the same formats as raw byte
facts, each with the test behind it.

## State of knowledge

As of **2026-11-21**, from one retail Xbox Series install. Other builds may
differ.

| Area | Understood | Not yet |
|------|-----------|---------|
| RPF9 archives | header layout; payload stored as zlib streams | the table of contents is encrypted, so there are no file names or paths |
| Resources | system/graphics pages, pointers, names inside resources | some unnamed resources |
| Textures | dictionary textures; Xbox tiling for BC formats; `ttpf` stores with AMD Gfx10 layouts (BC and all uncompressed formats); BCPack lane decoding; all six codecs decode (every one of the 357 K store textures readable at full size; BC3–BC7 with fitted tables); embedded tint palettes | **BCPack** byte-exact reconstruction (fitted BC3–BC7 tables); where the real mip count is stored |
| Models | vertex streams, meshlet index buffers, LODs, skeletons, skinning (incl. u16 bone indices; ped clothing with 8 influences through a bone palette to the shared ped skeleton), shaders' texture references; shader names (all) and most parameter names | texture names (64-bit hash), constant buffers |
| Vehicles | parts, damage and mod-kit parts, wheel mounts; model info, spawn configurations (body colour, rim, tyre), weighted paint-colour lists, handling records | how the wheel scale sizes the shared wheels, secondary colours, liveries, many handling fields |
| Peds | clothing skinning (8 influences, bone palette, shared 596-bone skeleton); head faces through material presets (893 of 1,657 heads); body-shape presets, the control → rig-input map and the pose-rig format, which runs; strand hair and fur (20,010 grooms, [§19](formats/19-ped-physics-resources.md)), cloth particles and edges, head identity deltas | head skinning, faces of the other heads, which groom belongs to which ped, two rig ops, lower-LOD rigs, rig input names |
| Map data | self-describing meta resources; entities, LOD hierarchy and LOD parents (99 %), archetypes, grass and prop batches, interiors with rooms and entity sets | prefab placement, some hashed fields |
| Collision | bounds geometry and primitives, surface ids and their names (ELF hash; 257 of 398 ids named, 84 % of triangles, plus `DEFAULT` for the default id, inferred); inferred surface categories for 98.3 % of area | names of the other surface ids (unreadable part of the material XML) |
| Navmesh | tiles, polygons, road flag | most other flags |
| Lights | position, colour, type, intensity, range, cones, hour mask | a few fields |
| Road graph | nodes, links, lanes, speed limits, street names, 34 of 51 node flags, junctions, traffic-light phases | some link flags |
| Video | 515 Bink 2 files, stored uncompressed and carved by header; frame index; per-track audio packet framing; all 515 decode with a public Bink 2 decoder; 276 more in three content packs; minigame movies identified | 720p artifacts not checked frame by frame; original names |
| UI graphics | 75 named dictionaries (loading screens, HUD, gauges); the UI texture store (10,013 sprite atlases, 90 % complete) | BCPack parts, menu layouts, a minimap tile set (none found) |
| Text | the localised text tables (14,482 in `data.rpf`, 4.21 M strings, one table per language), key lookup, variant groups, subtitle labels and markup | the language of each table, most label names, variant selection |
| Audio | `ADAT` container, chunk tags, Opus framing, codec ids (PCM16, float32 PCM, XMA2, Opus); music streams with 1–8 Opus sub-streams (stems) | a few header fields |
| PSO data | big-endian section headers, little-endian block table (v22 and unversioned script layouts), the `LSCH` schema (joaat field names), `LSNS` framing; 2,225 of 2,281 files in `data.rpf` decode; one serialised object graph (30,683 records) | 56 files in the older layout, `LSNS` record meanings, enums, most struct names |
| Shaders | cache layout, shader names, native RDNA2 code spans inside DXBC wrappers | decoding to a readable form, runtime shader selection and bindings |

## Biggest open problems

- **BCPack**, the Xbox Series hardware texture codec. The entropy stream
  decodes for all six codecs, and all six decode to BC blocks (BC3, BC4,
  BC5, BC6H and BC7 with fitted history tables, so some blocks speckle).
  Byte-exact reconstruction is not established for any codec.
  [Status and how to resume](formats/06-bcpack.md#60-status).
- **Collision surface names**: the id is `18 + elf(name) % 32749`, but part
  of the material XML does not inflate, so 140 of 398 ids have no name yet.
  [§13.2](formats/13-collision.md#132-surface-ids).
- **The RPF9 TOC** is encrypted.
- **Vehicle data**: colours, wheel choice and handling are decoded, but how
  the wheel scale sizes the shared wheel models, the secondary colours and
  the liveries are not.

## Conventions

- Offsets are hexadecimal, little-endian unless stated.
- Hashes are hex. Where a field or enum name is known, it is the
  case-sensitive joaat ("one-at-a-time") hash of that name.
- The game is RAGE Z-up, right-handed, metres. glTF output conventions are
  Y-up: `gltf(x, y, z) = rage(x, z, −y)`.
- Confidence: **confirmed** (holds over many samples, output checked by
  eye), **inferred** (consistent with the data, not proven), **unknown**.

## Tool notes

The [tool/](tool/README.md) folder documents the extractor these notes were
worked out with: its commands and the files it writes.

## Licence

GPL-3.0; see [LICENSE](LICENSE). This covers these notes only, not anything
from the game.
