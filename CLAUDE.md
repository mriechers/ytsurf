# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`ytsurf` is a terminal YouTube client (search / stream / download) written as a **single Bash script**: `ytsurf.sh` (~1,700 lines). There is no build step and no compiled artifact — the script *is* the program. This repo is a checkout of the third-party project `Stan-breaks/ytsurf` (GPL-3.0); version lives in the `VERSION` var near the top of `ytsurf.sh`.

## Run / lint / test

- **Run:** `./ytsurf.sh [OPTIONS] [QUERY]` (interactive with no args).
- **Lint:** `shellcheck ytsurf.sh` — the only linter; it runs in CI on Linux and macOS. No shellcheck config, so default settings apply. Keep it clean.
- **Tests:** No real test suite. CI only runs two smoke checks — replicate these before claiming a change is safe:
  - `./ytsurf.sh -V` must exit 0.
  - `./ytsurf.sh --limit "not-a-number"` must exit **non-zero** (CI asserts `! ...`).

## Bash gotchas that will bite you

- **Requires bash ≥ 4** (`mapfile`, `${arr[-1]}`). macOS system bash is 3.2, so the script re-execs itself under Homebrew bash (`ytsurf.sh:3-10`). Testing on macOS needs `/opt/homebrew/bin/bash`.
- **The script uses `set -u` only** — *not* `set -euo pipefail`, despite what `CONTRIBUTING.md` says. Do not assume `-e` (exit-on-error) or `pipefail` semantics when editing; failures do not abort automatically.
- **Some commands are GNU/Linux-only** and break on macOS/BSD: `stat -c "%Y"` (`ytsurf.sh:89,134` — macOS uses `stat -f`) and `sha256sum` (macOS ships `shasum -a 256`). Account for both when touching those paths.
- **Required external tools:** `yt-dlp`, `mpv`, `jq`, `curl`, `perl`, `socat`, plus a picker (`fzf` default; `rofi`/`sentaku`/`tv` alt). `check_dependencies` hard-exits if a required one is missing.
- State/config lives under `~/.config/ytsurf/` and `~/.cache/ytsurf/`; `~/.config/ytsurf/config` is **sourced as bash**.

## Security-sensitive lines — edit with care

This script scrapes YouTube HTML and feeds parsed values into the shell. When modifying these, do not widen the surface (keep URL-encoding, keep values quoted, don't remove validation):

- `eval "$player"` (`ytsurf.sh:1194` mpv, `:1218` iina) — `$player` is built from `$video_url`, whose `$video_id` comes from scraped network JSON. This is the main injection surface.
- Self-updater (`ytsurf.sh:381-397`) — `--update` fetches `main` from GitHub and applies it with `patch`; no signature/checksum check.
- `source "$CONFIG_FILE"` (`ytsurf.sh:441`) — arbitrary code from the user's config file.
- Subscription sync (`ytsurf.sh:648`) — runs `yt-dlp --cookies-from-browser`, reading the browser cookie store.
