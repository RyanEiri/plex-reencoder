# plex-reencoder

```
+----+ > PLEX-REENCODER +-----------------------------------------------------+
|                                                                             |
|                     ######.  ##.      #######. ##.  ##.                     |
|                     ##...##. ##:      ##...... .##.##..                     |
|                     ######.. ##:      #####.    .###..                      |
|                     ##.....  ##:      ##....    ##.##.                      |
|                     ##:      #######. #######. ##.. ##.                     |
|                     ...      ........ ........ ...  ...                     |
|                                                                             |
| ######. #######.#######.###.   ##. ######. ######. ######. #######.######.  |
| ##...##.##......##......####.  ##:##......##....##.##...##.##......##...##. |
| ######..#####.  #####.  ##.##. ##:##:     ##:   ##:##:  ##:#####.  ######.. |
| ##...##.##....  ##....  ##:.##.##:##:     ##:   ##:##:  ##:##....  ##...##. |
| ##:  ##:#######.#######.##: .####:.######..######..######..#######.##:  ##: |
| ...  ......................  ..... ....... ....... ....... ...........  ... |
|                                                                             |
|         .:######################################################:.          |
|  > PLAY - EP 6:00:00                                      TRK ===..  * REC  |
+-----------------------------------------------------------------------------+
```

[![License: GPLv3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
![Language: Bash](https://img.shields.io/badge/language-Bash-4EAA25.svg)
![Platform: Linux](https://img.shields.io/badge/platform-Linux-lightgrey.svg)

Bash tooling for surveying and re-encoding a Plex media library (NFS-mounted,
`Ironwolf8_1` and `Barracuda8_1` shares on `files.buddha.lan`) down to
space-efficient H.264/HEVC without a meaningful quality loss. Split out from a
former monorepo that also contained the unrelated VHS digitization pipeline
([vhs-cli](https://github.com/RyanEiri/vhs-cli),
[vhs-gui](https://github.com/RyanEiri/vhs-gui)) — no relationship to those repos.

## Hardware

Encodes run on the same Linux workstation as the VHS pipeline — **AMD Ryzen 9
5900X (12-core)**. Re-encoding here is CPU-only (libx264/libx265 via ffmpeg);
none of these scripts use the machine's GPU (AMD Radeon RX 7800 XT). Encodes
stage to a local drive (`/media/ryan/Patriot/Videos/plex_encode`) before moving
to the NFS server `files.buddha.lan`, which hosts the actual `Ironwolf8_1` /
`Barracuda8_1` media shares this tooling operates on.

## Scripts

| Script | Purpose |
|---|---|
| `plex_space_survey.sh` | Disk usage survey across the NFS-mounted library. |
| `plex_probe_large.sh` | ffprobes the largest files to find bloated/inefficient encodes (codec, bitrate, resolution, duration). |
| `plex_find_dupes.sh` | Detects likely duplicate movies via normalized directory-name matching. |
| `plex_reencode.sh` | The main re-encoder: Blu-ray rip/remux → H.264 CRF 20 (slow preset, capped at 1080p unless overridden), AAC stereo + optional 5.1, subtitle reorder by language preference. Stages to a local SSD before moving to NFS; auto-discards an encode that ends up larger than its source. |
| `plex_cleanup.sh` | Deletes media listed in a file. Dry-run by default; requires `--confirm`. |
| `plex_swap.sh` | Deletes the original `.mkv` and renames the matching `.x264.mkv` → `.mkv` in one step (the manual-cleanup step after `plex_reencode.sh`). Dry-run by default; requires `--confirm`. |
| `reencode_barracuda.sh` | Re-encodes high-bitrate `Barracuda8_1` files to HEVC in place: encode to local staging, verify (duration match, stream presence), copy back over the original, clean up staging. |

All scripts log to `./logs/<script>_<timestamp>.log`. The destructive ones
(`plex_cleanup.sh`, `plex_swap.sh`) default to a dry-run preview and require an
explicit `--confirm` flag to actually act.

## Target bitrates (empirically validated)

For older films/TV (pre-2000s, sitcoms, dramas): **4–6 Mbps at 1080p** via CRF 22 is
a comfortable target — e.g. a 1984–85 sitcom season landed at 4–6 Mbps at CRF 22
from 15–20 Mbps sources (55–70% savings), visually fine.

- **CRF 18** — near-lossless, ~8–20 Mbps output. Only worth it over an
  already-compressed source if that source is above ~12 Mbps.
- **CRF 22** — good quality, ~4–8 Mbps output. Right for older/archive content.
- CRF 18 will *bloat* an already-moderate-bitrate H.264 source (8–12 Mbps) — the
  size check in `plex_reencode.sh` auto-discards these regressions.
- Worth re-encoding: source bitrate **well above 8 Mbps** for older content.
  Skip anything already at or below 6–8 Mbps.

## `plex_reencode.sh` details

- Default CRF 20; override with `CRF=22 ./plex_reencode.sh <list>`.
- Auto-discards an encode if it ends up larger than its source.
- Stages to a local SSD (`/media/ryan/Patriot/Videos/plex_encode`) before moving to
  NFS — this is the scratch space for all encodes.
- Source `.mkv` → output `.x264.mkv` (original kept, needs the manual
  `plex_swap.sh` cleanup step); source `.m2ts`/`.mp4` → output `.mkv` directly.
- `FFMPEG_BIN=/usr/bin/ffmpeg` (libx264), `FFPROBE_BIN=/usr/local/bin/ffprobe`.

**Monitoring:** tail the main script log, not the per-file `_ffmpeg` log —
```bash
tail -f logs/plex_reencode_<timestamp>.log
```
This shows item number, filename, SKIP reasons, stream selection, and scaling
decisions. The `_<N>_ffmpeg.log` file has raw ffmpeg stderr for frame-level
progress if needed.

## NFS library

- Movies: `/mnt/media/movies/Ironwolf8_1/` (canonical) and `Barracuda8_1/`
- TV: `/mnt/media/tvshows/Ironwolf8_1/` and `Barracuda8_1/`
- Server: `files.buddha.lan`

`Ironwolf8_1` content takes precedence over any `Barracuda8_1`/migration
duplicates when reconciling the two shares.
