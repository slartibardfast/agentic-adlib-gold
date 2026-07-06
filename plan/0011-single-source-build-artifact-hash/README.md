# 0011 One source of truth for the build-artifact hash

The dual-host reproducible-build lane (`call/0007`, `call/0009`) checks each freshly built
`adlibgold.sys` against an expected hash. That hash was a hardcoded `ARTIFACT_SHA` constant copied
into both workflow files, and the copies were hand-maintained. When the driver binary changed
across alpha.9, alpha.10 and alpha.11 the host `.host-software` pin was re-recorded but the two
workflow constants were not, so both reproducible-build jobs failed for three releases against a
stale hash while the build itself reproduced correctly (`call/0027`). This milestone gives the
expected hash one home in the software repository.

## Who

- **Piotr** relies on the reproducible-build gate to catch a real regression. A gate that fails on a
  stale constant instead of a real difference trains him to ignore it; a single, reviewable source
  of truth keeps the gate meaningful.

## The shape of the fix

The software repository records the expected hash once, in a committed `adlibgold.sys.sha256` file
next to the artifact, in `sha256sum` format. Both workflows verify with `sha256sum -c
adlibgold.sys.sha256` and hold no `ARTIFACT_SHA` constant, and the file joins each workflow's
`paths` trigger so a hash change re-runs the check. The host `.host-software` artifact pin stays the
cross-repository production anchor; the remaining host/software copy of one value is a
`host-lifecycle` concern raised upstream (`call/0027`), not patched here.

## Build sequence

### Record the expected artifact hash in one committed file {#checksum-file}

- verify: attested call/0027
- inputs: software/adlib_gold/main/adlibgold.sys.sha256

Add `adlibgold.sys.sha256` beside the artifact, in `sha256sum` format (the hash and the
`adlibgold.sys` name), holding the current reproducible hash. This is the software repository's one
record of the expected artifact.

### Verify from the file in both build workflows {#verify-from-file}

- depends: #checksum-file
- verify: attested operator

Replace the `ARTIFACT_SHA` constant and its inline `sha256sum -c -` check in both
`reproducible-build.yml` and `reproducible-build-windows.yml` with `sha256sum -c
adlibgold.sys.sha256`, and add the checksum file to each `paths` trigger. Both Reproducible Build
jobs pass on the fixed `main`. Then re-pin `.host-software` to the new software commit; the artifact
is unchanged, so both build stanzas keep the same recorded hash.

## Depends on

Milestone `0003` (the reproducible WDK build lane this refines) and `call/0027` (the decision this
milestone carries out).
