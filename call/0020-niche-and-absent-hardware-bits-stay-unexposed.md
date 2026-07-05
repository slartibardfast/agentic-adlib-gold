# A few control-chip and hardware bits stay unexposed

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

The goal is a feature-complete driver, and the audible surface is now complete: PCM 8/16-bit
mono and stereo playback and capture, FM synthesis in two- and four-operator voices, the MIDI
UART, the full control-chip mixer (master, tone, mute, the input volumes, the record gain),
and SP2 surround (auto-detected, present on the GoldLib clone). What remains are a few
register bits that either do not map to a standard mixer control or drive hardware a card does
not carry. The question is whether each earns a control.

## Decision

These bits stay at their sensible power-on defaults and are not exposed as controls:

- **Output source select** (reg 08h, D1-D0: both / left-only / right-only). Windows has no
  standard control for "drive both, or only the left, or only the right, output channel," and
  it is not a `KSNODETYPE_MUX`, which selects among input pins. The default (both) is correct
  stereo; left-only and right-only are diagnostic. The register constant stays defined for a
  future diagnostic control.
- **I/O base relocation** (reg 15h). Plug and Play assigns and owns the card's I/O base; the
  driver honors the assignment and must not move the base under the OS.
- **The MMA timers** (regs 02h-08h of the YMZ263). The manual states the timers are not wired
  on the card, so there is nothing to expose.
- **The telephone, PC-speaker and audio-filter paths** (regs 01h, 10h, 11h). These drive the
  optional telephone module, which the GoldLib clone and a base Gold do not carry; there is no
  connector to control.

## Consequences

- Good: the mixer surface stays standard and uncluttered, and each unexposed bit sits at the
  correct default, so nothing audible is lost.
- Neutral: the diagnostic output modes are reachable only by a future, deliberately-scoped
  control. The constants stay defined, so exposing one later is a small change.
- This is a scope decision about which bits earn a control, not a limitation of the audio
  path, which is complete. It does not touch the SP2 surround (implemented and auto-detected)
  or full-duplex (a real two-DMA capability, implemented separately).
