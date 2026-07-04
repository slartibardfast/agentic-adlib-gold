# The driver uses its own OPL3 FM synth, not the stock DDK synth

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

An OPL3 FM voice is already available to Casey (`call/0005`), but through a separate,
stock FM synth driver. The skeleton carries its own OPL3 miniport (`fmsynth.cpp`),
adapted from the DDK sample. The project wants the Gold's FM to run through the driver's
own synth, so the FM path is the project's own to specify and ship rather than a
dependency on a driver it does not control.

## Decision

The driver's own `fmsynth` miniport is the FM synth. The stock or separate FM driver is
not relied on: the Gold's OPL3 is driven by this driver end-to-end, and plan/0006 brings
that miniport up and under spec.

## Consequences

- Good: the FM voice, its mixer coupling (the FM volume node), and its timing
  (`call/0013`) are all the project's own, specified and tested here.
- Cost: "FM already works" (`call/0005`) no longer means the job is done. plan/0006 owns
  the real bring-up of the project's synth, not a verification of someone else's.
- The OPL3 register specifics come from the SDK manual and the datasheet, consulted as
  needed (`call/0010`).
