# 0004 16-bit PCM with high-fidelity downsampling

Extend the PCM path from the 8-bit tone (plan/0002) to 16-bit playback at the rates
Windows delivers, downsampled to the 12-bit DAC with high fidelity and within the DPC
budget (`call/0011`). It serves Casey with clean digital audio and Piotr with a tested,
deterministic inner loop.

## Who

- **Primary: Casey** (`cast/casey.md`) hears 16-bit game and music audio without hiss
  or aliasing.
- **Supporting:** Piotr (`cast/piotr.md`) owns the fixed-point resampler and the dither.

## What the skeleton already provides

`algwave.h` carries the 12-bit "format 2" PIO path and a Galois-LFSR TPDF dither
(`DitherSample`). The 16-bit to 12-bit reduction is present. The sample-rate resampler
is the gap this milestone fills.

## Spec

`wave.allium` in the adlib_gold repo, distilled from `algwave.cpp` and then extended,
specifies the format validation, the dither, the resample to a supported rate, and the
stream position. It runs in that repo's CI (allium check plus analyse plus plan) with
the obligations discharged by tests, and a property test or a Kani proof holds the
fixed-point resampler to its fidelity bound. This milestone references the spec by pin.

The spec is authored and live at the software pin: `allium check` and `allium analyse` are
clean, and `spec/wave.obligations` dispositions all 31 `allium plan` obligations (the
behavioural ones waived until this milestone lands the harness).

## Acceptance

- 16-bit PCM at 44100 and 48000 Hz plays on the GoldLib recreation without audible
  aliasing or distortion.
- A test drives the resampler and dither and holds the output to the spec's
  signal-to-noise-and-distortion bound.

## Depends on

- plan/0002 (the 8-bit PCM path is up) and `call/0011` (the downsampling design).
