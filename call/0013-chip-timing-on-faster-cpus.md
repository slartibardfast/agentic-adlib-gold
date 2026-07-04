# Chip timing is enforced in software, since the driver runs on far faster CPUs

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The Ad Lib Gold chips (the OPL3, the MMA, the SP2, and the Control Chip) were specified
in 1992 and carry real timing requirements: minimum delays between register writes,
setup and hold times, and the clocked bit-serial protocol the SP2 needs. Code written
for period CPUs met those delays for free, because the CPU itself was slow. This driver
runs on radically faster CPUs (a modern host, or 98SE on fast silicon), where
back-to-back register writes can outrun what the chips accept, so a naive port misbehaves
in ways that depend on CPU speed.

## Decision

Meet every chip's timing in software, independent of CPU speed. Each register-access
path enforces the datasheet delay explicitly, as a calibrated stall rather than an
instruction count, so the same driver behaves the same on slow and fast CPUs. The SP2
bit-serial clock, the OPL3 inter-write delay, and the MMA and Control Chip access timings
each get their documented minimum.

The timing is specified, not only coded: a `.tla` timing spec (the Specula lane) models
the chip-access orderings and their minimum delays, model-checked in the software's CI,
so a regression that races a chip is caught by the spec rather than by a field report on
one machine.

## Consequences

- Good: the driver is correct across CPU speeds, and the timing contract is a checked
  spec rather than folklore.
- Cost: every hot path carries an explicit delay, so the timing spec and the tests guard
  against both racing the chip and stalling longer than the chip needs.
- The chip minimums come from the SDK manual and the datasheets; the manual is consulted
  for each figure (see the goal, `call/0010`).
