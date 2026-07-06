# The reproducible build keeps its Wine prefix outside the worktree

- Status: accepted
- Scope: software
- Date: 2026-07-06

## Context and Problem Statement

`build.sh` builds `adlibgold.sys` under Wine and defaulted its Wine prefix to `$HERE/.winebuild`,
inside the worktree (gitignored). Wine fills a prefix with shell-folder symlinks: `My Documents`
points at `$HOME`, and `My Videos`, `My Music`, `My Pictures`, `Templates` and `Downloads` point at
`My Documents`. Two properties of that link set are hostile to any process that walks the worktree.
The links escape the repository into `$HOME`, and `My Videos -> My Documents -> $HOME` plus
`$HOME/.wine` re-enters the same links, so the structure is an unbounded cycle.

A recursive worktree walk that follows symlinks with no cycle guard therefore never terminates once
it enters the prefix. The host tool `software --check` does exactly that walk; on this WSL2 host,
where the worktree sits on a 9P `/mnt/c` mount, each directory read blocks in a 9P RPC, so the walk
became a silent multi-hour uninterruptible hang (nineteen such processes accumulated over a day).
The walk defect is filesystem-independent and belongs to the tool, reported upstream
(`connollydavid/host-lifecycle#15`). This decision records the software-side mitigation.

## Decision

`build.sh` defaults the Wine prefix to `${XDG_CACHE_HOME:-$HOME/.cache}/adlibgold-winebuild`, a
location outside the worktree. It creates that prefix with `mkdir -p` before `wineboot`, since the new
parent may be absent on a fresh runner. A caller-supplied `WINEPREFIX` still wins. The worktree then holds no Wine prefix, so a worktree walk cannot be trapped by the prefix's
symlinks.

The relocation does not change the artifact. The compiler and linker see only fixed Wine paths
(`C:\drv`, `C:\tc`, `C:\temp`); the host location of the prefix never reaches the toolchain and is not
embedded in `adlibgold.sys`. `build.sh` is a member of both reproducible-build workflows' path
triggers, so the change re-runs both, and CI re-proves the recorded `3f70ba6f` hash byte-for-byte on
the Linux/Wine host and the native Windows host. The native Windows build (`build.bat`) uses no Wine
prefix and stays untouched.

## Consequences

- A worktree walk (`software --check`, a linter, an editor index) can no longer be trapped by the
  Wine prefix's symlink cycle, and build scratch no longer sits in the source tree.
- The prefix under `~/.cache` persists across local builds. `build.sh` recompiles the fixed source
  set, re-extracts the toolchain, and links with `/incremental:no` over a fixed input list, so a
  reused prefix still produces the recorded hash; CI never reuses, since each runner is fresh.
- The tool defect itself stays open upstream (`host-lifecycle#15`). This mitigation removes the local
  trigger, not the underlying walk bug, which affects any adopter whose worktree holds a gitignored
  directory with a symlink cycle.
