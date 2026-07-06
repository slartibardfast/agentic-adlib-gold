# The build-artifact hash is a single committed checksum file, not a duplicated CI constant

- Status: accepted
- Scope: software
- Date: 2026-07-06

## Context and Problem Statement

The dual-host reproducible-build lane (`call/0007`, `call/0009`) proves each build reproduces the
recorded `adlibgold.sys` hash byte-for-byte. Each of the two workflows (Reproducible Build, and
Reproducible Build (Windows)) held that expected hash as its own hardcoded `ARTIFACT_SHA`
environment constant. The driver binary then changed across alpha.9 (`call/0024`), alpha.10
(`call/0025`) and alpha.11 (`call/0026`); the host `.host-software` artifact hash was re-recorded
each time, but the two workflow constants were not. So both reproducible-build jobs failed for
three releases against a stale `92c480bc` while the build itself reproduced correctly. The expected
hash lived in three hand-maintained places (the two workflow constants and the host pin) with no
cross-check, so one copy drifted unnoticed.

## Decision

The software repository records the expected artifact hash in one committed file,
`adlibgold.sys.sha256`, in `sha256sum` format (the hash and the `adlibgold.sys` name). Both
reproducible-build workflows verify the freshly built artifact with `sha256sum -c
adlibgold.sys.sha256` and carry no `ARTIFACT_SHA` constant. The file is a member of each workflow's
`paths` trigger, so a hash change re-runs the verification.

The expected hash now sits in one place within the software repository, beside the artifact it
describes and reviewable as a plain diff, and the linux/windows duplication is gone. The host
`.host-software` artifact pin stays the cross-repository production anchor: on a binary-changing
release both it and `adlibgold.sys.sha256` move to the same value, and
`host-lifecycle software --verify-build` re-derives the recorded hash from the pin, so a forgotten
host-side update fails that gate loudly rather than in silence.

## Consequences

- The two reproducible-build workflows share one source of truth, so a stale copy in one workflow
  can no longer diverge from the other.
- A binary-changing release updates one committed file in the software repository in place of two
  buried environment constants, and the change is visible in review.
- One cross-repository copy remains by construction: the host `.host-software` artifact hash and the
  software `adlibgold.sys.sha256` are two records of one value in two repositories. A single unified
  record (the host verifying its recorded hash against the software file) is a `host-lifecycle`
  concern and is raised upstream, not patched here.
- This amends `call/0007` and `call/0009`. What the reproducible-build lane proves is unchanged;
  only the storage of the expected hash changes, so one committed file replaces the per-workflow
  constant.
