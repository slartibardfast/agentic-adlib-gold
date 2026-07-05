# Mixer settings persist to the registry; EEPROM save stays off until hardware-tested

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

The card can persist its mixer state two ways: to the Windows registry (per-machine, the
WDM norm) and to an EEPROM on the card (cross-machine, via the Control chip's save/restore
commands, `SaveToEEPROM`/`RestoreFromEEPROM`). The registry path is wired and runs; the
EEPROM path is implemented but never called. The question is whether to auto-persist to the
EEPROM as well.

Three concrete problems block auto-wiring the EEPROM save:

- **The EEPROM has limited write endurance.** Saving on every mixer change (a slider drag
  emits many SETs) would wear it and add a 2.5 ms stall per SET, a visible lag.
- **There is no safe automatic trigger.** The natural low-frequency trigger, a power-down
  transition, runs through `PowerChangeState`, which may be called at `DISPATCH_LEVEL`; but
  `SaveToEEPROM` is pageable and busy-waits 2.5 ms, so calling it there would fault. The
  adapter destructor runs at `PASSIVE_LEVEL` but its port mapping may already be gone.
- **Auto-restoring at boot is risky.** `RestoreFromEEPROM` makes the card load whatever its
  30-year-old EEPROM holds, which could be an extreme or empty state that silences audio on
  first install, where the hardcoded defaults are known-good.

## Decision

Mixer settings persist to the **registry**, and the card EEPROM save is **off by default**.

- `SaveMixerSettingsToRegistry`/`RestoreMixerSettingsFromRegistry` remain the persistence
  path; the boot order stays registry, then hardcoded defaults, unchanged.
- `SaveToEEPROM` is not auto-invoked. It and `RestoreFromEEPROM` stay available for a future
  explicit, hardware-validated "save to card" control, enabled only once the driver's
  functionality is verified on real hardware.

## Consequences

- Good: no EEPROM write wear, no per-SET lag, no risk of a bad EEPROM state silencing a fresh
  install, and no `DISPATCH_LEVEL`-versus-pageable fault. Persistence works today through the
  registry, which is what Windows expects.
- Neutral: the card does not remember settings across machines or reinstalls yet. That is the
  deferred capability, gated on hardware testing.
- The two EEPROM functions stay in the source as the ready implementation for that future
  control, so the capability is one wiring step away once it can be validated.
