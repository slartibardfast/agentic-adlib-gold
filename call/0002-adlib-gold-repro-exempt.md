# adlib_gold is not yet reproducible on this host: repro-exempt

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The methodology asks each Where-room component to carry a byte-reproducible build
recorded in `.host-software`, proven by `host-lifecycle software --verify-build`.
`adlib_gold` is pre-existing software brought under the methodology, not initiated
greenfield. It builds with the Windows WDK toolchain, and this host operates on
Linux under WSL, which cannot reproduce that build.

## Decision

Record `repro-exempt = call/0002` on the `adlib_gold` component in `.host-software`.
`--verify-build` then reports the component and skips the rebuild comparison, while
`--check` still requires this citation to resolve.

Interim provenance: the deployed driver is built from the pinned source with the
Windows WDK on a Windows host, which is outside this host's reach.

## Consequences

- Good: the host stays honest about what it can verify here rather than asserting a
  reproducibility it cannot run.
- Bad: build reproducibility is unproven on this host until a Windows build lane is
  recorded.
- The exemption is meant to be retired. Once a reproducible WDK recipe is recorded
  (a pinned toolchain with an `attest-host = windows` build), drop the exemption.
