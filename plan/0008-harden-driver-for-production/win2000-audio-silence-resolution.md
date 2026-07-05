# Resolution: the Windows 2000 audio silence after the driver loaded

Once the PE-checksum fix (`call/0024`) cleared Code 31 and `v1.0.0-alpha.9` loaded and ran on the
GoldLib hardware, a boot trace confirmed the driver detecting the card, bringing up every
subdevice, and digitally programming both the OPL3 FM synth and the MMA PCM engine correctly.
Yet the card was silent on every path: WAV/PCM playback, MIDI through the FM synth, and both the
DirectSound and DirectMusic tests in dxdiag. This note records how that silence was localized and
cured, so a later session does not re-walk the dead ends.

## The shared analog stage was the first suspect, and a partial cause

Three independent digital sources being silent at once points at the one stage they share on the
way to line-out: the control chip's analog mixer and master output. A code review against the SDK
found the master output volume (reg 04h/05h) mis-programmed. That register is a dB code, not the
linear scheme of the source-volume registers, so its hardcoded default `0xD8` decoded to the off
band rather than a live level. `v1.0.0-alpha.10` corrected it (`call/0025`): master default to
`0xFC` (0 dB), the master node's advertised floor into the audible band, the MMA output volume
written to full, and a self-healing clamp for a stale saved value.

alpha.10 was still silent, and the operator added a decisive observation: the master and wave
mixer sliders were frozen, physically un-draggable, and pinned at the top of their travel. A
volume default alone cannot explain a slider with no travel. Both symptoms shared a deeper cause.

## The true cause: volume values read as register bytes, not decibels

`KSPROPERTY_AUDIO_VOLUMELEVEL` is a signed LONG in units of 1/65536 dB, not a hardware register
byte. Grounding this against Microsoft Learn, the Microsoft SB16 topology sample bundled with the
driver, and the SDK register reference, with an adversarial pass that tried and failed to refute
it, found `PropertyHandler_Level` treating the KS dB LONG as a raw register byte on read, write,
and range advertisement. Two failures followed from the one error:

- When the audio stack opens a stream it sets each source-volume node to unity, a KS value of
  zero (0 dB). The handler narrowed that to a zero byte and clamped it up to the node minimum. For
  the FM (reg 09h/0Ah) and sampling/PCM (reg 0Bh/0Ch) registers that minimum is `0x80`, which the
  SDK documents as silent. So the audio stack muted FM and PCM at their source nodes on every
  stream open, downstream of and after the audible defaults the master fix had restored. That is
  why alpha.10 could not cure it.
- The range advertisement returned the raw byte span (for example `0x80` to `0xFF`) as if it were
  dB. Read as 1/65536 dB that span is about two thousandths of a decibel wide, so the slider had no
  usable travel and pinned; a read returning the raw byte as a tiny positive dB is why it pinned at
  the top rather than the bottom.

## The fix, and how it was verified without hardware

`v1.0.0-alpha.11` (`call/0026`) converts per node between KS dB and the register byte in integer
arithmetic only, since the kernel runs no floating point: the master as a two-dB-per-step dB code
via shifts, and the linear mixing volumes (reg 09h-0Fh) and the record gain (reg 02h/03h) through
build-time dB tables. A unity set now lands on the audible register value (the mixing volumes to
`0xFF`, the master to `0xFC`) instead of the silent minimum, and every slider advertises its real
dB range.

Because the driver cannot be run on hardware here, the conversion was checked by re-deriving the
exact integer logic offline and asserting the load-bearing cases: a unity set is audible on every
path; read-back round-trips within a fraction of a dB; and every read stays inside the advertised
range, so the audio stack sees no range violation. The free build reproduces byte-for-byte and the
checked build additionally logs each control-chip write (`CtrlWr`) and each mixer set
(`VolSet ksval -> register`), compiling out of the free build, so a residual silence can be read
straight off a trace rather than guessed at.

## What remains open

Confirmation is now the operator's hardware test of alpha.11: audible WAV and MIDI, the dxdiag
DirectSound and DirectMusic tests, and moving mixer sliders. If any path is still silent, the
checked build's trace localizes it precisely. The bass and treble tone handler already converts in
dB correctly and was not part of this defect.
