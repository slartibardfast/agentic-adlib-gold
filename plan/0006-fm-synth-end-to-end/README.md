# 0006 FM synth end-to-end under spec

The FM voice Casey hears today comes from a separate, stock synth. This milestone brings
up the driver's own OPL3 miniport instead (`call/0014`) and takes it end-to-end on the
target under spec. It is the FM Synth subsystem of the completion goal (`call/0010`).

## Who

- **Primary: Casey** (`cast/casey.md`) hears authentic OPL3 FM in games that drive the
  synth over MIDI.
- **Supporting:** Piotr (`cast/piotr.md`) owns the voice allocation and the register
  writes; Quill (`cast/quill.md`) keeps the FM spec and the SDK manual consistent.

## What the skeleton already provides

`fmsynth.cpp` is an OPL3 (YMF262) MIDI miniport: 18 two-operator voices, 16 MIDI
channels, a drum channel, and 256 patches, driven through the common object's
`WriteOPL3` with bank switching.

## Spec

`fmsynth.allium` in the adlib_gold repo, distilled from `fmsynth.cpp`, specifies the
note on and note off, the voice allocation across the 18 voices, the patch selection,
and the drum channel. The OPL3 inter-write timing is held by the `.tla` timing spec
(`call/0013`). Both run in that repo's CI, and are referenced here by pin.

## Acceptance

- A MIDI note routed to the FM synth sounds the expected OPL3 voice on the GoldLib
  recreation.
- The mixer's FM volume node (plan/0005) sets its level.

## Depends on

- plan/0003 (the build lane) and `call/0014` (our own synth). The mixer coupling in the
  acceptance depends on plan/0005.
