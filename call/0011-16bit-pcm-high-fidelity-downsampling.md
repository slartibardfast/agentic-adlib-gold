# 16-bit PCM: high-fidelity, high-performance downsampling to the 12-bit DAC

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The MMA digital-audio DAC is 12-bit and clocks a fixed set of sample rates (44100,
22050, 11025, and 7350 Hz; see `algwave.h`). Windows delivers 16-bit PCM at rates it
chooses, often 44100 or 48000 Hz. So the 16-bit path has two gaps to close in kernel
mode, with no FPU: it must reduce 16-bit samples to 12 bits without audible
quantization artefacts, and it must convert an unsupported source rate to a supported
one without audible aliasing.

## Decision

Close both with fixed-point, integer-only arithmetic sized for the WaveCyclic DPC
budget.

- **Bit depth, 16 down to 12.** Apply triangular-PDF dither before truncation, so the
  quantization error is decorrelated from the signal and heard as steady low-level
  noise rather than distortion. The skeleton already carries a Galois-LFSR TPDF dither
  (`DitherSample` in `algwave.h`); that is the fidelity floor. First-order
  error-feedback noise shaping is the reach goal, since it pushes that noise toward the
  band edge where the ear is least sensitive.
- **Sample rate.** Resample to the nearest supported hardware rate with a fixed-point
  polyphase FIR whose coefficients are precomputed once. The FIR band-limits ahead of
  decimation, so it keeps aliasing out of the audible band, and its polyphase form
  spends only a small, constant count of multiply-accumulates per output sample. Linear
  interpolation is the low-cost fallback where the FIR cannot meet the DPC budget.

The 12-bit "format 2" register mode (`MMA_DATA_FMT_12B_2`) is the transport, since it
is little-endian-compatible with 16-bit PCM.

## Consequences

- Good: 16-bit PCM plays at the best fidelity the hardware allows, with the quality and
  the cost both stated up front and checkable.
- Cost: the resampler and the dither are the driver's most arithmetic-heavy inner loop.
  The spec fixes a fidelity target (a signal-to-noise-and-distortion bound) and the
  tests hold the build to it.
- This is a design proposal grounded in `algwave.h`. The `.allium` spec and its tests (a
  property test, or a Kani proof of the fixed-point resampler) decide whether the reach
  goal is kept or the fallback stands.
