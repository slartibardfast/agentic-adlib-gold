# 0003 Establish the reproducible WDK build lane

Give plan/0002 a trustworthy build to install: make the driver rebuild from its pin
and retire the `repro-exempt` (`call/0002`). This is the prerequisite for any tone, so
it is worked before plan/0002's bring-up.

## Who

- **Piotr** (`cast/piotr.md`) owns the build lane. Casey benefits: the driver Casey
  installs is the pinned, reproducible one.

## Approach

Build the one source pin two ways (`call/0007`), recorded as per-platform builds in
`.host-software`:

- a Linux-with-Wine lane (`attest-host = linux`) that reproduces on ordinary Linux CI
  and on the operator's WSL host;
- a Windows CI lane (`attest-host = windows`) that attests on the real target and
  cross-checks the Wine build.

Both lanes take their toolchain from one hash-pinned dependency bundle: the Windows
2000 DDK, vendored as a `.zip` and recorded as `deps-bundle` (`call/0008`). The bundle
is published as a versioned vendor release, the `pgs-release` pattern, and each lane
downloads and verifies it, then builds offline under `--network none`.

The first step is to produce that vendor release: package the DDK subset the driver
needs, publish it as a tagged release asset where its Microsoft licence permits, record
its sha256 in a committed `deps-bundle.lock`, and set `deps-bundle` in `.host-software`
to the release URL and hash.

## Acceptance

- `.host-software` records, per platform, the build recipe (the `toolchain` and the
  `build` command) and the `adlibgold.sys` artifact hash for the `adlib_gold`
  component.
- `host-lifecycle software --verify-build` rebuilds from the pin and reproduces the
  recorded artifact on at least one lane.
- When a lane reproduces, the `repro-exempt = call/0002` line is dropped and
  `software --check` stays clean without it.

## Needs from the operator

Resolved:

- The toolchain is the Windows 2000 DDK, vendored as a hash-pinned `.zip` deps-bundle
  published as a vendor release (`call/0008`).

Still open:

- The vendor release itself: the packaged DDK `.zip`, its host location (subject to the
  Microsoft licence, see `call/0008`), and the resulting `deps-bundle` URL and sha256.
- Whether the Wine lane must reach a byte-identical artifact, or whether a successful
  Wine build with the Windows lane as the reproducibility anchor is enough.
