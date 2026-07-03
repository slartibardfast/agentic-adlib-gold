# agentic-adlib-gold

The agentic-project host for the
[`adlib_gold`](https://github.com/slartibardfast/adlib_gold) Windows WDM audio
driver. This repository holds the *thought* about that software: its plans,
decisions, personas, and the rules an agent works under. The *action*, the driver
source itself, lives beneath this host as a bare store with worktrees (the Where
room), pinned in `.host-software`.

It adopts the `host` methodology from
[`connollydavid/host-template`](https://github.com/connollydavid/host-template),
at the revision recorded in the `.host` stamp.

## Read first

- `CLAUDE.md` is the operating manual. Read it before you change anything.
- `STRUCTURE.md` is the one-page map of the rooms.

## The rooms

| Question | Room | Holds |
|----------|------|-------|
| Who  | `cast/` | personas the driver serves |
| What | with the code | specs, in the driver repository |
| When | `plan/` | the milestone index and folders |
| Where | `software/adlib_gold/` | the driver, a bare store with worktrees (gitignored) |
| Why  | `call/` | decisions, in MADR format |
| How  | `CLAUDE.md` + `tools/` | the manual and the verification tools |

## The software it governs

`adlib_gold` is a Windows WDM audio driver for the Ad Lib Gold sound card, with
the v1.01 Developer Toolkit SDK. Its own manual is published as a searchable
mdBook at <https://slartibardfast.github.io/adlib_gold/>. This host keeps that
site as it is and links to it; the host's own mdBook covers the rooms above.

## Working here

1. Materialize the tools and the software:
   `git submodule update --init`, then `host-lifecycle software --materialize .`.
2. Regenerate the skill symlinks with `./link-skills.sh`.
3. Follow `CLAUDE.md`.

## License

Released into the public domain under the [Unlicense](LICENSE).
