# Full duplex is out of scope on the Gold 1000 class target

- Status: superseded by call/0031
- Scope: software
- Date: 2026-07-08

## Context and Problem Statement

`plan/0010` designed full duplex over the card's second DMA select (reg 14h): mono render on
one MMA channel and mono capture on the other, each over its own system DMA line. The YMZ263
documents this mode (its overview names simultaneous record and playback via a separate DMA
per channel, and P/R direction is per-channel outside interleave), so the chip is not the
obstacle. The obstacle is the pool of grantable lines. The Gold 1000 selects only DMA 1, 2
and 3; DMA 0 is a Gold 2000 capability (SDK 3-7 and 7-9). On the target machine DMA 2 belongs
to the floppy controller and DMA 3 to the parallel port, so exactly one line is free, and the
operator attests DMA 0 is not available on the GoldLib, a Gold 1000 equivalent. Two
simultaneous audio DMA channels therefore cannot exist on this target, and full duplex is
structurally impossible there, however the driver is written.

Even where a second line exists, the card's duplex was always degraded: stereo playback
consumes both MMA channels through interleave (`call/0017`), so duplex is mono-plus-mono;
each channel's ADC is wired to one side of the stereo input, so the capture direction hears
one input side; and each channel's single antialiasing filter serves either its ADC or its
DAC, never both at once (SDK 7-8).

## Decision

Full duplex is out of scope for this driver. The half-duplex behaviour the driver already
enforces is the permanent contract: `NewStream` refuses a second direction while one is open
(`STATUS_INVALID_DEVICE_REQUEST`), so Windows sees a well-behaved half-duplex device and no
code change is required. `plan/0010` is withdrawn; its build sequence (never started, no
receipts) is removed from the task graph and preserved in git history for a hypothetical
Gold 2000 port, which is the only hardware class where the design could be attested.

## Consequences

- The completion goal (`call/0010`) is unaffected: it names PCM, mixer, and FM synth
  end-to-end, not simultaneous directions.
- The `NewStream` rejection comment still cites its interim origin (`plan/0008`); its wording
  moves to cite this decision on the next driver change, not in a rebuild of its own.
- A future Gold 2000 target would revisit this by reinstating the withdrawn design from
  history, with DMA 0 in the pool and the attestation done on that hardware.
