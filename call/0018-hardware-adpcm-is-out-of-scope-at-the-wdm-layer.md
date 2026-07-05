# The chip's hardware ADPCM is not exposed at the WDM driver layer

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

The YMZ263 (MMA) can compress digital audio to 4-bit ADPCM: clearing the PCM bit in the
play register (reg 09h) selects ADPCM, and the FIFO data port (reg 0Bh) then carries two
4-bit samples per access, high nibble first (manual ch07). The card's DOS Developer Toolkit
exposes this as its own `WAVE_FORMAT_ADPCM4` format alongside `WAVE_FORMAT_PCM8/12/16`
(`WAVE.C`), and ADPCM runs at half the PCM sample rates (44.1 kHz "does not work in 4-bit
ADPCM"; manual ch04). The development plan lists ADPCM as an optional "if desired" feature.
The question for feature-completeness is whether the WDM driver should expose it.

## Decision

The driver does not expose hardware ADPCM. It advertises and plays only PCM (8-bit and
16-bit, mono and stereo), which is the WDM-correct design, for two independent reasons:

- **The WDM audio stack never hands a kernel WaveCyclic miniport a compressed stream.**
  Windows decompresses ADPCM to PCM in user mode (the ACM codecs), and KMixer mixes and
  feeds the driver in PCM. A non-PCM format reaches a driver only over a dedicated raw or
  passthrough path (as with AC-3 over S/PDIF), which does not exist for an ISA card's
  proprietary sample compression in the Windows 98 SE audio stack. So the miniport is fed
  PCM regardless, and the chip's ADPCM decoder is never on the path the OS uses.
- **The chip's ADPCM is a Yamaha format no Windows source produces.** Windows' ADPCM codecs
  emit Microsoft-ADPCM or IMA/DVI-ADPCM, which use different prediction and step tables than
  the YMZ263's Yamaha ADPCM. Advertising the chip's ADPCM would match no Windows source, and
  feeding it a Windows-codec ADPCM stream would decode as noise. Even Ad Lib's own DOS
  Sample Maker left its PCM-to-ADPCM converter "not completely implemented" (manual ch04).

## Consequences

- Good: the driver is feature-complete for WDM digital audio, since the OS handles every
  compressed format by decompressing to PCM before the driver, and the driver plays PCM. This
  matches how the DDK's own SB16 WDM sample treats that chip's ADPCM modes: it does not
  expose them.
- Neutral: the card's hardware ADPCM compression, a DOS-era memory-and-bus optimization, is
  unused under WDM. Nothing is lost audibly, since the same audio plays as PCM.
- This is an architectural decision, not a deferral: there is no WDM path that would make
  exposing the chip's Yamaha ADPCM useful, so it is not revisited unless a raw-format
  passthrough requirement appears.
