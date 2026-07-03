# The reproducible WDK build lane runs on Windows CI and on Linux with Wine

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

plan/0002 (the 8-bit PCM test tone) needs a trustworthy build to install. The driver
is `repro-exempt` today (`call/0002`) because its Windows WDK build cannot run on this
Linux/WSL host. Retiring the exemption needs a reproducible build lane and a place to
run it. The driver's `sources` file builds with the DDK `build` environment against
portcls.

## Decision

Build the driver from its one source pin two ways, recorded as per-platform builds in
`.host-software`:

- **Linux with Wine** (`attest-host = linux`): run the DDK build toolchain under Wine
  on a Linux runner. This lets `host-lifecycle software --verify-build` reproduce the
  driver on ordinary Linux CI and on the operator's WSL host, with no Windows machine
  in the loop. It is the primary reproducibility lane.
- **Windows CI runner** (`attest-host = windows`): run the same DDK build natively on
  a Windows runner. This attests the driver on its real target platform and
  cross-checks the Wine build.

Each build records its own `toolchain`, `build`, and `artifact` (the `adlibgold.sys`
path and hash). `--check` and `--verify-build` attest each build only on its host and
skip the foreign one rather than fail on it (the per-platform build rule). When a lane
reproduces the pinned artifact, the `repro-exempt` (`call/0002`) is dropped for that
lane.

## Consequences

- Good: reproducibility no longer depends on owning a Windows machine, since the Wine
  lane runs on Linux, while the Windows lane keeps an authoritative check on the
  target.
- Bad: Wine fidelity for a period DDK is unproven, so the Wine lane may not reach a
  byte-identical artifact. If it cannot, it stays a build check and the Windows lane
  is the reproducibility anchor.
- The exact DDK version and its provenance are recorded when the lane is built
  (plan/0003).
