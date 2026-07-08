# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Bash tooling for surveying and re-encoding a Plex media library (NFS-mounted,
`Ironwolf8_1`/`Barracuda8_1` shares on `files.buddha.lan`). See `README.md` for the
full script list, target-bitrate guidance, and NFS layout — this file only covers
conventions for modifying the scripts.

Split out from a former monorepo that also contained the unrelated VHS digitization
pipeline ([vhs-cli](https://github.com/RyanEiri/vhs-cli),
[vhs-gui](https://github.com/RyanEiri/vhs-gui)). No relationship to those repos —
don't assume shared conventions beyond both being bash.

## Conventions

- `set -euo pipefail` in every script.
- Destructive scripts (`plex_cleanup.sh`, `plex_swap.sh`) default to a dry-run
  preview; require an explicit `--confirm` flag (plus, for extra safety, typing the
  literal action word) before deleting or renaming anything. Preserve this pattern
  in any new destructive script.
- Logs go to `./logs/<script>_<timestamp>.log`.
- `plex_reencode.sh` stages encodes to a local SSD before moving to NFS — never
  encode directly onto the NFS mount.
- Auto-discard-if-larger-than-source is a deliberate safety net in
  `plex_reencode.sh` — don't remove it when touching the encode logic.
- Path-safety guards (`/mnt/media` prefix + minimum directory-depth checks) in
  `plex_cleanup.sh`/`plex_swap.sh` exist specifically to prevent an over-broad list
  file from deleting near the NFS share root — preserve them in any similar script.
