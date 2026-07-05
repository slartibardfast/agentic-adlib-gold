# Stereo digital audio is implemented over the DMA interleave path

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

`call/0016` deferred stereo digital audio to mono, for two reasons: the driver had no
channel-1 access, and the manual's channel-1 addressing looked ambiguous. Both are now
resolved. A trace of the Miles AIL Ad Lib Gold driver's 16-bit-stereo path, validated
register by register against the manual (`plan/0008/stereo-mma-reference.md`), shows the
exact hardware contract a known-good driver uses, and it resolves the channel-1 addressing:
`MMA_write <channel>, <reg>, <value>` selects channel 1 through its own port pair
(base+6/base+7), which the driver already names as `ALG_REG_MMA1_ADDR`/`ALG_REG_MMA1_DATA`.

## Decision

Implement stereo, superseding `call/0016`, along the reference's DMA interleave path:

- A stereo stream runs over DMA with the chip's interleave (ILV), not the mono 16-bit PIO
  fill. Both channels' play register (reg 9) and format register (reg 12) are programmed:
  ILV is set on channel 0 with ENB on both channels (interleave needs DMA), channel 1's
  FIFO interrupt is masked so channel 0 drives the shared interrupt, and channel 0 routes to
  the right output and channel 1 to the left, exactly as the validated reference sets them.
- The DMA carries one interleaved left-right stream that the chip de-interleaves into the two
  FIFOs, so stereo needs no second DMA channel and no software interleave. A stereo 16-bit
  source transfers raw (format 2), and the 12-bit DAC reproduces the top bits; the software
  dither stays on the mono 16-bit PIO path, which is unchanged.
- The pin advertises `MaximumChannels = 2` again, and `ValidateFormat` accepts a stereo
  format. The per-channel register values are derived in the pure, tested `wavereg.h` helper.

## Consequences

- Good: the driver plays stereo, the feature the card was built for, from a proven register
  sequence rather than a guess.
- Attested on hardware: the channel-to-output mapping (channel 0 right, channel 1 left) runs
  opposite to the manual's default channel naming, so whether left and right come out
  un-swapped depends on the interleaved DMA byte order and can only be confirmed on the card.
  Continuous stereo playback and the absence of a FIFO overrun are likewise attested on the
  GoldLib, as the mono paths are.
- `call/0016` is superseded by this decision.
