# The non-PnP install carries one forced boot config, not arbitratable LogConfigs

- Status: accepted
- Scope: software
- Date: 2026-07-05

## Context and Problem Statement

Installing the driver on the GoldLib (Windows 98 SE) blue-screens with a fatal page
fault at `0028:C0031B0A`. The address is byte-identical across five completely
different driver binaries (`v1.0.0-alpha.1` through `.5`), so it is in a fixed
Windows routine, not in `adlibgold.sys`. Operator observation pinned the moment: the
crash fires the instant the driver is selected in the Have-Disk wizard, and
`adlibgold.sys` is never copied to disk. So the fault is in the earliest install
phase, before `CopyFiles`, in the ring-0 config manager (CONFIGMG), not in driver
code or in resource arbitration.

The Ad Lib Gold (1992) predates ISA Plug-and-Play and reports no PnP hardware id, so
it installs as a manually-created **root devnode** with a synthetic hardware id
(`ALG1000`). The DDK SB16 sample the driver is modelled on never exercises this path:
its devnode is auto-created by the ISA-PnP enumerator matching `*PNPB003`, which hands
CONFIGMG a real boot configuration. Our INF instead offered the devnode two
arbitratable `LogConfig` sections at `ConfigPriority=NORMAL` (the Gold 1000 and Gold
2000 IRQ/DMA ranges) plus a `FactDef` also at `NORMAL`. Microsoft's INF reference is
explicit that a manually-installed non-PnP device must carry a **forced** boot
configuration in its `DDInstall.FactDef` section; a devnode with only arbitratable
configs and no forced config leaves CONFIGMG with no boot config to instantiate for
the root devnode, and it faults building the logical-configuration records.

Prior install-fix attempts mis-targeted this. `d499dce` corrected the hardware-id
binding (real, necessary). `fcef658` (alpha.4) replaced the 9x `AlsoInstall=...(...)`
form with `Include`/`Needs` on the false premise that the parenthesised form "is not a
valid setupx directive" -- the DDK SB16 INF uses that exact form and installs, and the
crash predated and outlived the change, so it was never the cause.

## Decision

The manually-installed non-PnP devnode carries exactly **one forced boot configuration**:

- `[AdLibGold_Device.FactDef]` uses `ConfigPriority=FORCECONFIG`, giving CONFIGMG the
  forced boot config it requires for a root devnode: I/O `388h` (8 ports, `388h-38Fh`),
  IRQ `7`, DMA `1` -- the GoldLib's jumpered, fixed resources.
- The arbitratable `LogConfig` sections (`ALG.LC1`, `ALG.LC2`) are **removed**. The card
  is jumpered to fixed resources, so a forced config is the correct expression; there is
  nothing to arbitrate.
- The 9x install section reverts to the proven `AlsoInstall=KS.Registration(ks.inf),
  WDMAUDIO.Registration(wdmaudio.inf)` form (the form the DDK SB16 and the WDMHDA 98SE
  audio driver both use). `Include`/`Needs` + `AddService` stay in the `.NT` section,
  where they belong. This is correctness hygiene; it is not reached at the crash point
  (CONFIGMG faults registering the devnode config before the install section executes).

The DriverVer bumps to `1.00.0000.6` so the corrected INF outranks the prior attempts.

## Consequences

- Good: the devnode gets a valid forced boot config, which is the documented requirement
  for a manually-installed non-PnP card, so CONFIGMG has no malformed arbitratable record
  to fault on. This is the leading-hypothesis fix, grounded in the Microsoft INF reference
  and an independent research pass, and it targets the confirmed crash phase.
- Trade-off: forcing `388h/7/1` removes the per-model IRQ/DMA choice the arbitratable
  LogConfigs offered. This is deliberate and temporary: the GoldLib is the only card under
  test and uses exactly those resources, and the user's directive was to get the driver
  installing before adding breadth. The Gold 1000/2000 alternatives return as arbitrated
  `LogConfig`s, layered onto the forced config, once the base install is confirmed on
  hardware.
- Unverifiable here: the driver is untestable on this Linux/WSL host, so this fix is
  confirmed only by compile and byte-reproduction; the install itself is validated by the
  operator on the GoldLib. If the crash survives this change, the fallback is the pure
  bisect -- strip `FactDef` as well and install with no forced resources -- to prove
  whether CONFIGMG faults on any config record for this root devnode or specifically on
  the arbitratable ones.
- The full re-analysis and the ground-truth observations are recorded in
  `plan/0008-harden-driver-for-production/install-crash-invariant-address.md`.
