# The first sound-path target is an 8-bit PCM test tone

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

Casey (the primary persona, `call/0004`) judges the driver by hearing authentic Ad
Lib Gold sound on Windows 9x. plan/0001 asked which sound path to bring up first and
what authentic-sound acceptance means. The card can produce FM synthesis and digital
audio (PCM and ADPCM) as well as MIDI.

## Decision

The first sound-path target is an 8-bit PCM test tone through the WaveCyclic miniport
(the YMZ263 MMA digital audio).

- **Why PCM, not FM.** The FM voice is already available to Casey, so it is not the
  gap. The digital-audio path is what remains to prove.
- **Why 8-bit.** In the MMA, 8-bit PCM is the ISA-DMA path (data format
  `MMA_DATA_FMT_8BIT` with DMA enabled), the simplest transfer the hardware offers.
  16-bit runs the more involved PIO-with-dither path, so it comes later.
- **The acceptance.** A build passes this target when an 8-bit PCM test tone is
  audible from the card on the test hardware.

## Consequences

- Good: the first increment has a concrete, hearable pass condition that Casey can
  judge by ear.
- Neutral: later targets are sequenced after this one, not dropped.
- The next milestone (plan/0002) brings this target up on hardware.
