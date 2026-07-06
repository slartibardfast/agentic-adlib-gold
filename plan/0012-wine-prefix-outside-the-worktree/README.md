# 0012 Move the Wine build prefix out of the worktree

`build.sh` defaulted its Wine prefix to `.winebuild` inside the worktree. Wine fills a prefix with
shell-folder symlinks that escape into `$HOME` and form a cycle, so any recursive worktree walk that
follows symlinks is trapped there and never returns. The host tool `software --check` does that walk;
on this WSL2 `/mnt/c` (9P) host it became a silent multi-hour uninterruptible hang. The walk defect
is the tool's and is reported upstream (`connollydavid/host-lifecycle#15`); this milestone removes the
local trigger by keeping the prefix out of the tree (`call/0028`).

## Who

- **Piotr** runs `software --check` as the gate. A gate that wedges for hours on Wine's build scratch
  is a gate he cannot trust or use; a worktree free of that scratch lets the gate run.

## The shape of the fix

`build.sh` defaults the prefix to `${XDG_CACHE_HOME:-$HOME/.cache}/adlibgold-winebuild` and creates it
with `mkdir -p` before `wineboot`; a caller-supplied `WINEPREFIX` still wins. The existing in-tree
`.winebuild` is removed. The artifact is unaffected, because the toolchain sees only fixed `C:\` Wine
paths, and `build.sh` sits in both reproducible-build workflows' `paths` triggers, so CI re-proves the
`3f70ba6f` hash on the Linux/Wine and native Windows hosts.

## Build sequence

### Move the Wine prefix out of the worktree {#relocate-prefix}

- verify: attested call/0028
- inputs: software/adlib_gold/main/build.sh

Default the Wine prefix to `${XDG_CACHE_HOME:-$HOME/.cache}/adlibgold-winebuild`, add `mkdir -p` before
`wineboot` (the new parent may be absent on a fresh runner), keep the `WINEPREFIX` override, and remove
the existing gitignored `.winebuild` from the worktree.

### Prove the artifact still reproduces {#verify-repro}

- depends: #relocate-prefix
- verify: attested operator

Both reproducible-build workflows re-run on the `build.sh` change and reproduce `adlibgold.sys` to the
recorded `3f70ba6f` hash on the Linux/Wine and native Windows hosts. Then re-pin `.host-software` to
the new software commit; the artifact is unchanged, so both build stanzas keep the same hash.

## Depends on

Milestone `0003` (the reproducible WDK build lane) and `call/0028` (the decision this milestone carries
out). Related: `plan/0011` (the sibling reproducible-build hygiene milestone) and upstream
`host-lifecycle#15` (the tool walk defect this mitigates locally).
