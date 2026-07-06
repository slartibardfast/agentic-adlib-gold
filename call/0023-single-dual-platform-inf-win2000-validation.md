# Ship one dual-platform INF for 9x and Windows 2000, with Windows 2000 as the primary validation path

- Status: accepted
- Scope: software
- Date: 2026-07-05
- Amends: call/0022 (drops the A/B second INF; keeps its resource-form decision)

## Context and Problem Statement

`call/0022` shipped the grounded non-PnP install as alpha.7 with two INF files: the primary
`adlibgold.inf` (a `HARDWIRED` `LogConfig` plus a `NORMAL` `FactDef`) and an A/B alternate
`adlibgold-forceconfig.inf` (the `FORCECONFIG` `FactDef` form), so one hardware session on the
GoldLib could distinguish the two resource forms on Windows 98 SE. That distinction was worth
making while 98 SE was the only test platform.

A follow-up grounding pass on Windows 2000 (recorded in
`plan/0008-harden-driver-for-production/win2000-install-compatibility.md`, five research
searches plus three adversarial reviews plus a local inspection of the driver binary) found
that the same `adlibgold.sys` and the grounded INF install and run on Windows 2000 with no INF
change. `Signature="$CHICAGO$"` with `.NT` sections is the documented cross-platform pattern;
the binary imports only Windows 2000 era PortCls and kernel functions and its PE header targets
the Windows 2000 subsystem; and the 98 SE fault (`0028:C0031B0A`, a Windows 9x ring-0 VxD fault)
is architecturally impossible on Windows 2000, which installs through `setupapi` and the PnP
manager rather than the 9x `setupx` and `CONFIGMG`. A real shipping Windows 2000 audio driver
(C-Media CMI8338) uses this INF's exact `Signature`, `Class`, `ClassGUID`, `.NT`, and
`Needs` pattern.

Two facts follow. The grounded INF is a single dual-platform installer: Windows 9x reads its
undecorated install block and Windows 2000 reads its `.NT` block. And Windows 2000 is the safest
of the four targets, giving a readable Device Manager error instead of a blue screen, so it is a
validation path that does not depend on solving the 98 SE manual-install crash first.

## Decision

Ship a single INF, `adlibgold.inf` (the grounded form `call/0022` chose), as the release for
both Windows 9x and Windows 2000, and drop the A/B alternate `adlibgold-forceconfig.inf`.

- The resource form matches `call/0022`. It uses a `HARDWIRED` `LogConfig` (`ALG.LC0`), bound to
  the undecorated install section and to the `.NT` section; a `NORMAL` `FactDef` beside it; and the
  fixed resources I/O `388-397`, IRQ 7, and DMA 1. The grounding confirmed this form is honored on Windows 2000
  (the `LogConfig` reference ships a byte-for-byte identical `HARDWIRED` pattern for a non-PnP ISA
  controller).
- Remove `adlibgold-forceconfig.inf` from the driver repository. The A/B was a 98 SE tactic; with
  Windows 2000 as a clean validation path and the crash-screen module name as a stronger 98 SE
  diagnostic, the two-file A/B is no longer worth the operator friction.
- Release the single package as alpha.8. `DriverVer` bumps to `1.00.0000.8` so it outranks the
  A/B release. `adlibgold.sys` is unchanged and byte-reproduces (`92c480bc`).
- Recommend Windows 2000 as the first validation platform.

## Consequences

- Good: one installer serves both platforms, which is simpler for the operator and matches how a
  dual-platform INF is meant to work. Windows 2000 validation exercises the whole driver (binary
  load, subdevice registration, audio) independently of the 98 SE install crash, which converts a
  blocked path into a testable one.
- Trade-off, stated so it is a conscious choice: dropping the alternate INF gives up the ability
  to distinguish `HARDWIRED`-`LogConfig` from `FORCECONFIG`-`FactDef` on 98 SE in a single
  session. This is acceptable because the grounded form is the better-supported one, the captured
  crash-screen VxD name is a stronger 98 SE diagnostic than the A/B, and Windows 2000 no longer
  makes the 98 SE result a blocker for validating the driver.
- Windows 2000 caveats, documented, not defects: the unsigned driver raises a dismissable
  signature warning under the default Warn policy, and the single fixed configuration has no
  fallback, so an occupied IRQ 7 or DMA 1 yields a Device Manager Code 12. The remedy is to free
  IRQ 7, DMA 1, and port `388h` (for example by disabling the LPT1 or ECP parallel port), which is
  correct because the card physically decodes those resources.
- The resource-form decision of `call/0022` remains in force; only its A/B second-INF tactic is
  amended here.
