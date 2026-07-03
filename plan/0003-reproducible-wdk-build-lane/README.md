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

## Acceptance

- `.host-software` records, per platform, the build recipe (the `toolchain` and the
  `build` command) and the `adlibgold.sys` artifact hash for the `adlib_gold`
  component.
- `host-lifecycle software --verify-build` rebuilds from the pin and reproduces the
  recorded artifact on at least one lane.
- When a lane reproduces, the `repro-exempt = call/0002` line is dropped and
  `software --check` stays clean without it.

## Needs from the operator

- The DDK or WDK version that builds a 98SE-capable WDM driver, and where to obtain it
  (the Windows 2000 DDK is the anchor the `sources` file and the samples assume).
- Whether the Wine lane must reach a byte-identical artifact, or whether a successful
  Wine build with the Windows lane as the reproducibility anchor is enough.
