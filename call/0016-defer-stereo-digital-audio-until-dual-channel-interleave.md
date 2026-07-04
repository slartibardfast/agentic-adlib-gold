# Digital-audio output is mono until a hardware-verified dual-channel path exists

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The WaveCyclic miniport advertised and accepted stereo PCM, but never programmed the
channel count into the YMZ263. The play register (reg 09h) was always written with both
the L and R output bits, and the format register (reg 0Ch) interleave bit was never set,
so a stereo stream played as a garbled mono stream. A review flagged the missing channel
programming and the plan proposed deriving the play-register channel bits and the
format-register interleave bit from the stream channel count.

Consulting the hardware manual (ch07) while implementing this surfaced two problems with
that remedy, both stated directly in the manual.

First, stereo on this chip is interleaving, and interleaving is a DMA feature. Reg 0Ch ILV
"will cause the chip to do interleaving ... ENB must be 1 for both channels, otherwise the
data transfer is not performed", and ENB is the DMA-mode-enable bit. The high-fidelity
16-bit path runs in PIO because it dithers each 16-bit sample to the 12-bit DAC in
software, so ENB is 0 on that path and the chip cannot interleave it. Setting ILV there
would have no effect.

Second, correct stereo drives channel 0 to the left output and channel 1 to the right, so
it must program channel 1's registers and, on the PIO path, write the right samples into
channel 1's FIFO. The driver has no channel-1 access at all: `WriteMMA`/`ReadMMA` address
only channel 0 (base+4/base+5). The manual's channel-1 addressing is described two ways
that do not obviously agree (a single shared register-select port at 38CH with a separate
channel-1 data port at 38FH, against the driver's symmetric channel-1 select/data at
base+6/base+7), and there is no worked stereo example to settle it. That path cannot be
verified on this host, which has no card.

## Decision

Digital-audio output is mono until a dual-channel interleaved path is implemented and
verified on hardware. Concretely:

- The wave pin advertises `MaximumChannels = 1`, so Windows negotiates a mono format and
  downmixes a stereo source to mono, which the driver plays correctly. `ValidateFormat`
  also rejects a two-channel format as defense in depth.
- The play-register channel bits and the format-register interleave bit are derived from
  the channel count in a pure, unit-tested helper (`wavereg.h`), grounded in the manual:
  mono routes its one FIFO to both outputs with interleave off; the stereo mapping (channel
  0 to the left output, interleave on) is defined and tested as the target for the future
  path, but no admitted stream reaches it.
- A `wave.allium` rule records that an accepted format is mono and routes to both outputs,
  discharged by the helper's test.

## Consequences

- Good: the driver stops corrupting stereo playback, and it ships no untestable hardware
  code. The register mapping is pinned by a test so a later change cannot silently break
  the mono routing.
- Neutral: stereo separation is lost until the dual-channel path lands. A downmixed mono
  output is the interim behavior, which is what a stereo application already hears on a
  mono device.
- The deferred work is the channel-1 register and FIFO programming, the 8-bit DMA
  interleave path (ILV with both channels enabled), and a software dual-FIFO interleave for
  the 16-bit PIO path, each to be verified on the card. This decision is retired when that
  path exists and is attested on hardware.
