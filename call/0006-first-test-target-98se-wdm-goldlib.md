# The first test target is Windows 98SE (WDM) on a GoldLib recreation

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

plan/0001 and plan/0002 left one question open: which Windows 9x release and which
card revision is the first target for bringing up the 8-bit PCM test tone
(`call/0005`). The acceptance needs a concrete test machine to mean anything.

## Decision

The first test target is Windows 98 Second Edition, driven through the WDM path, on
a GoldLib recreation of the Ad Lib Gold.

- **Windows 98SE, WDM.** 98SE is the first Windows 9x release with a usable WDM audio
  stack (portcls), which the driver's `sources` file already links against. It is the
  natural first 9x target for a WDM driver and the release Casey runs.
- **The hardware is a GoldLib recreation.** No original Ad Lib Gold 2000 shipped in
  the wild, so it is out of scope; the GoldLib recreation stands in for the Gold
  hardware under test. The Gold 1000 is the real-card lineage the recreation follows.

## Consequences

- Good: the acceptance in plan/0002 now names a real machine, 98SE with the GoldLib
  recreation, so "the tone is audible on the test hardware" is checkable.
- Neutral: Windows 2000 and XP (Piotr's WDM targets) and any later card follow this
  first target, and 98SE-specific WDM quirks are surfaced during bring-up.
- Testing against a recreation rather than a period card is recorded here, so a later
  discrepancy against original hardware traces back to this choice.
