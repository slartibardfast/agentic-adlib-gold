# KS volume level is a 1/65536-dB LONG, not a raw register byte

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

After the master-volume dB-code fix (call/0025, shipped as alpha.10), the driver on Windows 2000
still produced no audible output on any path, and the operator additionally reported that the
mixer volume sliders for master and wave were physically stuck: they could not be dragged, and
sat pinned at the top of their travel. A dB-code default alone did not explain a total silence
that survived an audible master default, nor a slider with no travel.

A grounded review (Microsoft Learn, the DDK SB16 topology sample bundled with the driver in
`doc/wdm.txt`, and the Ad Lib Gold SDK register reference), with an adversarial pass that tried
and failed to refute it, found a single units error that produces every symptom at once.

`KSPROPERTY_AUDIO_VOLUMELEVEL` is a signed LONG in units of 1/65536 dB (0x00010000 == 1 dB), not
a hardware register byte. The DDK SB16 topology miniport is the reference: its BASICSUPPORT range
bounds and stepping are all in 1/65536 dB (SignedMaximum `0` == 0 dB, SignedMinimum `0xFFC20000`
== -62 dB, SteppingDelta `0x20000` == 2 dB), its GET returns a cached dB LONG, and its SET
converts the incoming dB LONG to the register by integer shift arithmetic.

`PropertyHandler_Level` did none of that. It read the KS LONG as a raw register byte on GET, SET,
and BASICSUPPORT. Two failures followed:

- **Silence.** When the audio stack opens a stream it SETs each source-volume node to unity, KS
  LONG `0` (0 dB). The handler narrowed that to `(BYTE)0 == 0x00` and clamped up to the node
  minimum. For the FM (reg 09h/0Ah) and sampling/PCM (reg 0Bh/0Ch) linear registers that minimum
  is `0x80`, which the SDK documents as **silent** (code 128 = silent, 255 = maximum). So the
  audio stack itself muted FM and PCM at their source nodes, immediately after `ControlRegReset`
  had written the audible `0xC0` defaults. This is downstream of the master, which is why the
  master-default fix could not cure it. The master likewise clamped a unity SET to `0xDC` (-64 dB).
- **Stuck sliders.** BASICSUPPORT advertised the raw byte range (for example 128..255) as if it
  were 1/65536 dB. Read as dB that is a range about 0.002 dB wide, so the slider has no usable
  travel and pins; GET returning the raw byte read as a tiny positive dB is why it pinned at the
  top rather than the bottom.

The Control Chip has two register families that need two conversions. The master (reg 04h/05h) is
already a 2-dB-per-step dB code, so it converts with SB16-style integer shifts. The mixing volumes
(reg 09h-0Fh) are linear-in-amplitude, and the record gain (reg 02h/03h) is a linear gain
`(val*10)/256`, so both need a precomputed dB lookup table (kernel code runs no floating point).
The bass/treble tone handler already converts in dB correctly and is not part of this defect.

## Decision

`PropertyHandler_Level` converts between KS 1/65536-dB LONGs and the Control Chip register byte
per node, in integer arithmetic only.

- Each level node declares its encoding family (dB code, linear volume, or linear gain) and its
  advertised dB range and stepping. Two build-time constant tables map the linear families to dB:
  a 127-entry table for the mixing volumes (code 129..255, code 255 = 0 dB) and a 255-entry table
  for the record gain.
- GET converts the register byte to dB; SET converts the incoming dB LONG to the nearest register
  byte, clamped to the node range; BASICSUPPORT advertises the dB range and stepping. A raw-register
  backstop clamp is retained after the conversion.
- A unity (0 dB) SET now lands on the audible register value (mixing volumes to code 255, master to
  `0xFC`) instead of clamping a `(BYTE)0` up to the silent minimum. The MUTE node keeps ownership of
  the -infinity / off state; the volume nodes advertise a finite floor.

This amends call/0025: the master-default and MMA-output fixes there remain correct, but they were
not sufficient, because the silence was inflicted downstream by the stack's own unity SET striking
the raw-byte mis-mapping fixed here.

## Consequences

- FM and PCM are no longer muted at their source nodes when a stream opens; a unity stream plays at
  full scale. All three output paths (PCM, FM/MIDI, DirectSound/DirectMusic) share this fix.
- The master, wave, FM, aux, mic, and record-gain sliders now advertise real dB travel and move.
- The conversion is exercised in the checked build's `VolSet` trace, which logs each mixer SET as
  `ksval -> register byte`, so a residual silence can be localized against ground truth.
- The linear-volume 0-dB anchor at code 255 is inferred, not stated by the SDK; any monotonic
  linear-amplitude table with a finite floor and true-silence deferred to MUTE satisfies the fix, so
  the exact anchor is not load-bearing.
