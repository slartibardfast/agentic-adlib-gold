# 0009 Implement four-operator FM voices from the on-disk timbre bank

The driver plays General MIDI through 18 two-operator OPL3 voices. The YMF262 can pair two
2-operator channels into one 4-operator voice for richer timbres, and the manual documents
the whole register contract for it (`four-op-reference.md`). An earlier note deferred this for
want of 4-operator instrument data; that was wrong. A bank of 471 Ad Lib timbres ships in the
repository, 157 of them genuine 4-operator instruments, and the driver's note structure
already reserves four operators while `Opl3_CalcVolume` already computes all four 4-operator
algorithms. The gap is a handful of code paths the DDK sample left stubbed, plus a converter
from the bank's on-disk layout into the driver's structure. This milestone closes that gap.

## Who

- **Casey** hears fuller instruments: the 4-operator timbres the card was built for, not only
  the 2-operator subset the DDK sample shipped.
- **Piotr** gets the 4-operator path exercised by a spec obligation and an offline converter
  test, so a later change that breaks pairing fails a check rather than the ear.
- **Quill** gets the `fmsynth` spec extended to describe when a voice is 4-operator and how it
  is programmed.

## The rule for this milestone

The milestone follows the no-hollow-green discipline. The mechanically checkable parts are
tested offline: the bank converter round-trips known records under a userspace test, and the
per-voice register derivation stays a pure, tested helper. The parts that only the card can
prove, the audible correctness of each algorithm and the General-MIDI-to-timbre mapping, are
`attested` on the GoldLib and named as such, never silently passed. The reproducible build
stays byte-identical, and every added obligation is dispositioned in `fmsynth.obligations`.

## Build sequence

### Convert the timbre bank offline into the driver's structure {#convert-bank}

- verify: cc -Werror the converter test and run it
- inputs: software/adlib_gold/main/tools/bnkconv.c

Write a build-time converter (not kernel code) that reads `OPL3.BNK`, and for each timbre
emits an entry in the driver's note-structure layout: copy each operator's five register
bytes in order, split the connect byte into the two C-registers (first channel CNT in one,
second channel CNT in the other) and the feedback, and map the type byte to the 2-operator or
4-operator marker. A userspace test round-trips a known 4-operator record (for example
`Tuba1`) and asserts four populated operators. The converter emits a compiled-in C array plus
a General-MIDI-program-to-timbre mapping; the mapping is the milestone's one curation choice
and is attested by ear.

### Enable the paired voices at init {#enable-voices}

- depends: #convert-bank
- verify: attested operator

With the NEW bit already set, program connection-select (`AD_CONNECTION = 0x104`) with the
mask of pairs the driver will drive as 4-operator, and restore the same on resume. The mask is
derived in a pure helper so a test pins it.

### Add the four-operator voice model and allocator {#voice-allocator}

- depends: #enable-voices
- verify: cc -Werror the allocator test and run it

Model the six 4-operator voices over the existing slots (the three Array-0 pairs and the three
Array-1 pairs); the remaining channels stay 2-operator voices. When a patch is 4-operator,
allocate a free pair and mark both constituent slots busy; otherwise fall back to a 2-operator
slot. The pairing table and the allocation predicate are pure and tested.

### Program a four-operator note {#note-programming}

- depends: #voice-allocator
- verify: attested operator

Extend `Opl3_FMNote`/`Opl3_NoteOn` so a 4-operator voice writes all four operators at their
four offsets, writes the play registers to the first channel only, writes both C-registers
with matching output bits, and sets the mode to the two-CNT-bit value so `Opl3_CalcVolume`
reaches the right algorithm. The 2-operator path is unchanged. Audible correctness is attested
on the GoldLib.

### Extend note-off, volume and pan to four operators {#note-off-volume}

- depends: #note-programming
- verify: attested operator

`Opl3_NoteOff`, the volume update and the pan update iterate the voice slots and write one
C-register and two operators; extend them to write all four operators and both C-registers for
a 4-operator voice, with the output bits identical across the pair. Then the milestone is
complete: General MIDI plays through 4-operator timbres, attested on hardware.

## Depends on

Milestone `0006` (the 2-operator FM path this extends) and `0008` (the hardened driver it
builds on). The register contract and the data source are recorded in `four-op-reference.md`.
