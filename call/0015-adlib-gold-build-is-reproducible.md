# adlib_gold builds reproducibly on this host: repro-exempt retired

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

`call/0002` exempted `adlib_gold` from the reproducible-build rule because its Windows WDK
build could not run on this Linux/WSL host. That premise no longer holds: the Windows 2000
DDK and Visual C++ 6.0 from the operator's media run under Wine here, and the driver
compiles and links into `adlibgold.sys`.

## Decision

Retire the `repro-exempt` and record `adlib_gold` as a reproducible component. The driver
builds byte-reproducibly under Linux/Wine (`call/0007`, `call/0009`):

- `build.sh` stages the pinned DDK+VC6 `deps-bundle` (`call/0008`), compiles and links
  offline, and `pe_normalize.py` zeros the PE `TimeDateStamp` and checksum.
- Two independent from-scratch builds produce the identical `adlibgold.sys`
  (sha256 `3f7d3c3c...`), verified this session.
- `.host-software` now records the `build`, `toolchain`, `deploy`, `attest-host`,
  `artifact`, and `deps-bundle`; `deps-bundle.lock` pins the bundle sha
  (`f60af3fd...`); the `reproducible-build.yml` CI rebuilds and checks the hash.

`call/0002` is superseded by this decision.

## Consequences

- Good: the pin is a true production anchor, and a clean rebuild reproduces the deployed
  binary byte-for-byte, on ordinary Linux CI, offline.
- Neutral: the Microsoft-licensed `deps-bundle` is hosted where its licence permits and
  fetched from a CI secret, so the public repository redistributes no Microsoft code.
- The Windows CI cross-check lane (`call/0007`, `attest-host = windows`) is added later to
  attest the same source pin on the target platform.
