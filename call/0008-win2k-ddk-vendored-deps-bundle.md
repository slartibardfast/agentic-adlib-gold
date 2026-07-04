# The build toolchain is the Windows 2000 DDK, vendored as a hash-pinned bundle

- Status: accepted
- Scope: software
- Date: 2026-07-04

## Context and Problem Statement

The reproducible build lane (`call/0007`, plan/0003) needs a toolchain that builds a
98SE-capable WDM driver. The driver's `sources` file is a DDK build script that links
against portcls, and the skeleton derives from the Windows 2000 DDK audio samples. A
hermetic, reproducible build cannot fetch that toolchain from a live network at build
time; it needs a pinned, offline copy.

## Decision

The build toolchain is the Windows 2000 DDK, vendored once as a hash-pinned `.zip` and
recorded as the component's dependency bundle in `.host-software`:

```
toolchain    = win2k-ddk
deps-bundle  = <url> <sha256>
```

Both build lanes (`call/0007`) download and verify that one bundle, then build offline
under `--network none`. This is the methodology's hermetic-build mechanism
(`deps-bundle`), so the DDK is a pinned input rather than a live dependency, and the
producer commits a `deps-bundle.lock` as its source of truth.

The bundle is published as a versioned, tagged vendor release, the pattern
`pgs-release` uses for its prebuilt sysroot: a dedicated release whose asset is the
pinned DDK `.zip`, named by URL and sha256 in the recipe. Unlike that sysroot, the DDK
is a prebuilt toolchain, so producing the release is packaging the DDK subset the
driver needs rather than compiling one.

## Consequences

- Good: the build is hermetic and reproducible from pinned inputs, with no need for a
  live Microsoft download at build time.
- Caution: the Windows 2000 DDK is Microsoft-licensed. Host the vendored bundle where
  its license permits, a private or access-controlled location, and record that
  provenance with the bundle.
- The `deps-bundle` URL and sha256 are filled in when the bundle is published. Until
  then, plan/0003 carries the pending pin and `.host-software` keeps the current
  `repro-exempt` (`call/0002`).
