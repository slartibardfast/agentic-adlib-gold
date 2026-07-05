# The non-PnP install declares fixed resources as a HARDWIRED LogConfig, not a FORCECONFIG FactDef

- Status: accepted
- Scope: software
- Date: 2026-07-05
- Supersedes: call/0021

## Context and Problem Statement

`call/0021` fixed the Windows 98 SE install crash (`0028:C0031B0A`, at Have-Disk select,
before any file copy) by putting `ConfigPriority=FORCECONFIG` in the INF's
`DDInstall.FactDef` section, pinning I/O `388-38F` with IRQ 7 and DMA 1, and deleting the
arbitratable `NORMAL`-priority `LogConfig` sections. That fix shipped as alpha.6 but was not
validated on hardware.

A web and DDK grounding pass (recorded in
`plan/0008-harden-driver-for-production/non-pnp-inf-install-grounding.md`, seven research
searches plus three adversarial reviews) confirmed part of `call/0021` and refuted part of it.

- Confirmed: the byte-identical crash address across five different `adlibgold.sys` binaries
  proves the fault is in fixed ring-0 operating-system code, not in the driver, so an
  INF-only remedy is on the correct axis.
- Refuted premise: no Microsoft source states that a manually-installed non-PnP device
  requires a forced boot configuration to install. Real Windows 98 SE non-PnP INF files
  (`SSMCIRDA.INF`, `MSPORTS.INF`) install with arbitratable `HARDRECONFIG` or `NORMAL`
  `LogConfig` sections and no `FactDef` at all.
- Wrong idiom: the one real INF that installs an Adlib/OPL FM chip at base `388` is the
  Windows 95 `MIDI.INF`. It declares the fixed FM resources with `ConfigPriority=HARDWIRED`
  in a `LogConfig`, keeps a `NORMAL` `FactDef`, and binds the standard compatible ids. No
  real shipping Windows 9x sound INF uses `FORCECONFIG` anywhere.
- Over-stated cause: nothing in the primary sources ties an arbitratable `LogConfig`, or the
  absence of a forced config, to a page fault at select time. The attribution to the
  configuration manager is a leading hypothesis, not an established fact.

A second, independent defect surfaced during the pass: the shipped INF claimed
`IOConfig=388-38F` (eight ports), but the driver's own architecture document records that the
MMA chip occupies `base+00h` through `base+0Eh` and that the card decodes sixteen ports from
`388h`. The range under-claimed the card's I/O by half.

## Decision

Declare the card's fixed resources the documented, empirically-attested way, and correct the
range.

- Move the resource declaration into a `LogConfig` section at `ConfigPriority=HARDWIRED`,
  bound to the 9x install section by a `LogConfig=ALG.LC0` directive. This is the only
  forced-style priority that appears in a real Microsoft-authored INF, and it is exactly the
  Windows 95 `MIDI.INF` idiom for an immovable Adlib FM at base `388`.
- Keep a `FactDef` section, at `ConfigPriority=NORMAL` (not `FORCECONFIG`), with the same
  resources. The Ad Lib Gold selects its I/O, IRQ, and DMA in software, and the reference
  advises that such a card should also use `LogConfig`, so deleting all arbitratable configs
  was more restrictive than documented practice.
- Widen the I/O range to `388-397` (sixteen ports, `(ffff::)` 16-bit decode), the card's real
  footprint. IRQ 7 and DMA 1 (the GoldLib's jumpering) are unchanged and were never wrong.
- Retain the `AlsoInstall=KS.Registration(ks.inf),WDMAUDIO.Registration(wdmaudio.inf)` form
  in the 9x section. KS registration declares no resources and is orthogonal to the fix.

Because the current Microsoft reference lists `FORCECONFIG` as valid in `FactDef` while the
empirical corpus never uses it there, and no fetched primary source resolves that
disagreement for Windows 98 SE, ship the alpha.6 `FORCECONFIG` variant as a second INF in the
same release. The operator tries the grounded variant first and the `FORCECONFIG` variant
only if the first still faults. Both variants use the identical `adlibgold.sys`, so the test
isolates the INF. If either still faults, the operator captures the `in VXD NAME(nn) + offset`
line from the crash screen, which names the faulting driver directly.

The DriverVer bumps to `1.00.0000.7` so the grounded INF outranks the prior attempts.

## Consequences

- Good: the fixed resources are declared the documented and empirically-used way, the range
  covers the whole card, and arbitration is preserved rather than deleted on an unproven
  premise. The A/B release settles the one genuine documentation-versus-practice disagreement
  in a single hardware session instead of over serial round-trips.
- Honest limits: the crash mechanism is still an inference and the faulting module is still a
  hypothesis. This decision does not claim `HARDWIRED` is proven necessary; it claims it is
  better grounded than `FORCECONFIG`-in-`FactDef` and avoids the ChkInf hazard reported for
  that placement. The captured crash-screen module name and the A/B result are what will
  convert inference into fact.
- Unverifiable here: the driver is untestable on this Linux and WSL host, so this change is
  confirmed only by compile and byte-reproduction (the `.sys` is unchanged from alpha.5 and
  alpha.6, since the INF is not a build input). The install is validated by the operator on
  the GoldLib.
- Documented alternative, not taken: binding the standard compatible ids (`*PNPB005`,
  `*PNPB020`, `*PNPB006`) as `MIDI.INF` does. It is held back because Windows ships inbox
  drivers for those ids, which could be ranked over this aggregate WDM adapter. It returns as
  a candidate if both A/B variants still fault.
