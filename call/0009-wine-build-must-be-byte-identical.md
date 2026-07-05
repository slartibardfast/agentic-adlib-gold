# The Wine build must be byte-identical to the Windows build

- Status: accepted
- Scope: software
- Date: 2026-07-04

> Amended by `call/0024`: the normalization that achieves this byte-identity must recompute the
> PE checksum, not zero it (zeroing it discarded the `/release` checksum a driver should carry).
> The byte-identical requirement here stands.

## Context and Problem Statement

`call/0007` set up two build lanes for the driver, a Linux/Wine lane and a Windows CI
lane, and left open whether the Wine lane must reach a byte-identical artifact or
whether a successful Wine build, with the Windows lane as the reproducibility anchor,
is enough. The weaker reading makes the Wine lane only a build check; the stronger one
makes it a true reproduction.

## Decision

The Wine lane must produce a byte-identical `adlibgold.sys`: the same bytes the Windows
CI lane produces from the same pin and the same DDK deps-bundle (`call/0008`). A Wine
build that merely succeeds without matching is a failure of this lane, not a pass. The
Windows-as-anchor fallback that `call/0007` allowed is withdrawn.

This makes reproducibility checkable on Linux, whether ordinary CI or the operator's
WSL host, with the same authority as the Windows build rather than trusting Windows
alone.

## Consequences

- Good: `host-lifecycle software --verify-build` on the Linux/Wine lane proves the
  deployed driver bit-for-bit, so a rebuild anywhere reproduces what ships.
- Cost: the build must be deterministic. The PE `TimeDateStamp` and any embedded build
  paths must be pinned or normalised, since the DDK linker stamps a timestamp by
  default. plan/0003 carries this as build work.
- If Wine cannot be made byte-identical, that is a blocker to resolve in plan/0003, not
  a reason to weaken the requirement.
