# 18. Radio station audio and scheduling

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-22; confidence: inferred unless stated (observed on one install).*

Xbox Series format notes. For the inputs, see the [music catalogue](17-other.md#music-catalogue)
and [AWC container](../structure/03-audio-awc.md). A rotating station has
pools of individual clips, including music, host voice, and breaks. A
premixed station can resolve to a single long audio payload.

## Resolve a station to audio

1. In `music.dat`, look up the station by name hash. The object table is a
   `u32` count followed by `(u32 name_hash, u32 kind)` pairs. The array index
   is the object ID. The fix-up table is a `u32` count followed by the file
   offsets of object-ID fields. Use it to follow references from the station
   to its category lists and entries. Per-type `(name_hash, offset)` indexes
   point to records whose tag starts at `offset + 8`.
2. Category lists carry their numeric category at tag `+0x08`. In a
   `0xAAAAAA0F` entry, tag `+0x0C` can refer to a kind-2 sound and
   `+0x10` can refer to a title object (`0xFFFFFFFF` is unset). Music pools
   use category `0xB2`. Host clips occur in `0x2820`, and voice clips also
   occur in `0x243C`. The station category and AWC chunk tag `0xB2` are
   independent fields.
3. Use the kind-2 object's name hash in the sound index to get the offset
   into the sound bodies. In the observed `08 0B 00 01 02` record form,
   read `u32 duration_ms` at record start `+0x0F`. At `+0x1A` the selector
   is `0x00` or `0x11`. The two following `u32` values start at `+0x1B`
   for `0x00` or at `+0x1E` for `0x11`.
4. Match the **second** `u32` masked with `0x1FFFFFFF` to an AWC stream ID.
   Check the decoded duration against `duration_ms` (within 100 ms in
   the tested join). Keep zero matches and ambiguous matches visible.
   The matched stream can carry markers, with playable data on a sibling
   stream in the same AWC. Keep both stream indices in the result.

For one Xbox Series revision, the following are zlib chain starts in
`audio.rpf`. Reassemble them using the [RPF9 page rule](../structure/01-rpf9-container.md#payload).
Offsets inside the assembled `music.dat` are independent of archive offsets.

| Payload | Archive offset | Check in the assembled payload |
|---|---:|---|
| `music.dat` (confirmed: zlib start; the payload has the catalogue magic at `0x08`) | `0x74E000` | Object table at `0x77C437`, fix-ups at `0x83C15B` |
| Sound index | `0xD268E0` | `(u32 name_hash, u32 body_offset)` pairs |
| Sound bodies | `0xB02000` | Sound record at `body_offset + 8` |
| Parameter records and index | `0x37000` | Kind-256 name hashes resolve to curve records |

For a premix, decode the long payload once. Use times from the markers
named by hash `0xA6D93246` in chunk `0xBD` as chapter times. The wave-ID
and duration match identifies the mix even when a catalogue set-group ID
gives only the station object.

## Deterministic `GAME.DAT` / `SOUNDS.DAT` (`audMetadataFile` `0x045A88EA`)

On Xbox Series 1.0.2.1 the file groups before the first `ADAT` bank in `audio.rpf` can be identified
by grouping consecutive 64 KiB inflated chunks until a short chunk ends the group. An
`audMetadataFile` payload has magic `0x045A88EA` at `+0x08`, after `u32 version` and `u32 meta_size`.
A zlib-only carve identifies the groups but omits stored chunks. In this build, `SOUNDS.DAT` is
missing three stored 64 KiB chunks from section 2 when carved that way.

- **Container layout:** metadata body (`meta_size` bytes from `+0x08`), `u32 s1_size` + section 1,
  `u32 s2_size` + section 2 including its stored chunks, `u32 obj_count` +
  `(u32 hash, u32 offset)` pairs sorted by `hash & 0xFF`, `u32 pre_ref_count` + `u32[]`, then
  `u32, u32, u32 ref_count` + `(u32 target_hash, u32)` reference table.
- **`GAME.DAT`** is identified by `joaat("RADIO_STATIONS_01")` (`type 0x0E`). Walking
  `RADIO_STATIONS_01`..`16` (`count` byte at `+0x05`, `u32` ref indices from `+0x08`) enumerates
  `RadioStationSettings` (`type 0x10`) objects:
  - `+0x01` `u32 flags`: `(flags & 0x30) == 0x10` marks static world emitters (not on the player
    wheel); `(flags & 0x0C) == 0x04` marks hidden player wheel stations (23 player wheel stations
    total on the measured build).
  - `+0x08` `u32 name_hash`, `+0x0C` `u32 wheel_position`, `+0x3C` `u32 tracklist_count` followed by
    12-byte tracklist entries starting at `+0x40` (first `u32` is the tracklist ref index).
- **`RadioStationTrackList` (`type 0x11`):** external `ObjectList` (`type 0x02`, `u32 count`
  at `+0x08`, `u32` refs from `+0x0C`) ref at `+0x44`. If that field is `0xFFFFFFFF`, use the
  `u16` inline count at `+0x48` and `u32` refs from `+0x4C`.
  Some tracklists have no station reference (probably looked up by hash at run time; inferred).
- **`RadioTrackSettings` (`type 0x0F`):** for the observed `u32 sound_ref` at `+0x0C`, bit 19 is
  clear, selecting the reference table. Use `sound_ref & 0x7FFFF` as a zero-based index into
  `GAME.DAT`'s reference table and read the entry's first `u32` (`target_hash`). Find that hash in
  `SOUNDS.DAT`'s object table and use its paired offset from the start of the metadata body (`+0x08`
  in the file). The first byte of the target object is `0x08` (`StreamingSound`).
- **`SOUNDS.DAT` `StreamingSound` (`type 0x08`) -> `SimpleSound` (`type 0x0C`):** in the
  observed layout, `u32 duration_ms` immediately precedes `0x0C <child_count> <u32 child_refs...>`.
  In six checked three-stem mixes, three `SimpleSound` objects immediately follow in metadata offset order.
  Each holds `(u32 bank_hash, u32 wave_hash)` at `+0x02` (`wave_hash & 0x1FFFFFFF` = AWC stream ID).

The object tables contain 86,715 entries in `GAME.DAT` and 241,999 in `SOUNDS.DAT`. The radio type
names follow GTA V's. The radio type IDs and field offsets here were checked
against the Xbox build's metadata.

## Track and station text lookup

A track's title and its artist or show are text labels `joaat("<N>s")` and `joaat("<N>a")` for
track number N (numbers above 1 seen). In this build's English data, all 1,703 entries in `trackid` are radio label hashes with
zero overlap against other eligible tables. Ten titles absent from `trackid` resolve in `GLOBAL`.
There are 77 empty title entries with nonempty show or host labels. All 23 wheel station display
labels resolve in `GLOBAL` via `RadioStationSettings.name_hash`.

## Build a rotating station timeline

| Record | Observed layout relative to its tag |
|---|---|
| Station tail | Pairs of category-label (`0xAAAAAA1C`) and `0xAAAAAA17` references, followed by a start category reference |
| `0xAAAAAA17` | Count at `+0x08`, then pairs `(next category ref, mix ref)` from `+0x0C` |
| `0xAAAAAA1E` | Default rule ref at `+0x08`, then a count and pairs `(current category ref, rule ref)` |
| `0xAAAAAA1F` | Rule count in the byte at `+0x05`, then rows `(next category ref, factor count, factor refs...)` from `+0x08` |
| Mix records | `0xAAAAAA16` holds transition parameters. In the weighted `0xAAAAA803` mix form, the count byte is at `+0x06` and `(mix ref, f32 weight)` pairs start at `+0x0C` |

1. Start at the station's start category. For the next category, use the
   current category's rule rows, or its default rule when no override
   exists. Consider categories with resolved audio. In the reconstruction,
   multiply a row's factors and sample by the resulting weight. Factor
   `0xAAAAA821` has a constant `f32` at `+0x08`. Factors `0xAAAAA823`
   and `0xAAAAA824` have category and kind-256 curve references at `+0x08`
   and `+0x0C`. The reconstruction evaluates these against consecutive
   plays and elapsed time since the category last ended, respectively.
   Use a fixed seed for repeatable clip selection. When a speech entry has
   a verified title reference, the preview can cue that song next.
2. Look up the mix on the edge between the chosen categories. A working
   interpretation of `0xAAAAAA16` is `(out offset, out fade, in offset,
   in fade)` in milliseconds at `+0x0C`, `+0x14`, `+0x1C`, and `+0x20`.
   The signed offsets are relative to the outgoing clip's end. A negative
   incoming offset overlaps the clips. Record each clip's source, start
   sample, length, incoming fade, and outgoing fade window.
3. Decode the selected clips to a common PCM rate and layout (48 kHz
   stereo works for this reconstruction). Place them at their start
   samples, apply fade envelopes, and sum overlapping samples with
   headroom. Derive chapters from the same timeline. Check each clip's
   identity and duration and log any transition that needed a fallback.

A quick check of the overlap arithmetic: two 3-second clips joined with a
500 ms overlap give 5.5 seconds.

## Open questions

- Rotation timelines generated this way are simulations. Runtime flags,
  the game's exact factor and mix behavior, and the context for generic
  host lines remain unverified.


---

← [17. Other formats](17-other.md) · [Index](README.md) · [19. Strand hair, cloth and head deltas](19-ped-physics-resources.md) →
