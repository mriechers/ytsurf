---
name: run-safely
description: Try ytsurf while still vetting it — run the script against throwaway, isolated config/cache/download dirs so it can't touch your real ytsurf state, and steer clear of the paths that fetch code or read browser cookies. Use when the user wants to test-drive, experiment with, or sandbox ytsurf before fully trusting it.
disable-model-invocation: true
---

# Run ytsurf safely (while evaluating it)

The user is vetting this third-party script. **Do not run it silently or as a side effect.** This skill runs it deliberately, isolated from real state, and avoids the risky code paths. Explain each command before running it and let the user confirm.

## Before running

1. Offer to run `/security-audit` first if it hasn't been done this session — reading beats trusting.
2. Note there is no true sandbox here (it's a Bash script with full user privileges and it shells out to `yt-dlp`/`mpv`). "Safe" here means *isolated state* and *avoiding the code-fetch / credential paths*, not OS-level confinement. If the user wants real confinement, that's a container/VM, not this skill.

## Isolated run

Point ytsurf's XDG dirs at a throwaway location so it can't read or clobber real `~/.config/ytsurf`, `~/.cache/ytsurf`, or downloads:

```sh
SANDBOX="$(mktemp -d)"                     # e.g. /tmp/tmp.XXXX
XDG_CONFIG_HOME="$SANDBOX/config" \
XDG_CACHE_HOME="$SANDBOX/cache" \
XDG_DOWNLOAD_DIR="$SANDBOX/downloads" \
  ./ytsurf.sh "test query"
```

- First run auto-creates `$SANDBOX/config/ytsurf/config` (a bash-sourced file) — inspect it, don't paste untrusted content into it.
- When done, the whole thing is disposable: `rm -rf "$SANDBOX"`.
- To just observe without a full picker UI, `./ytsurf.sh -V` (version) and `./ytsurf.sh --help` are inert and reveal the option surface.

## Do NOT run these while still evaluating

- **`--update` / `-u`** — fetches `ytsurf.sh` from GitHub `main` and patches the running script, unverified. Never run during evaluation.
- **Subscription sync / "Sync Subscriptions"** — runs `yt-dlp --cookies-from-browser`, reading your real browser cookie store (logged-in YouTube). Skip it until the script is trusted.

## Watch what it does (optional)

- Run with `--debug` to write an xtrace log to `$XDG_CACHE_HOME/ytsurf/ytsurf.log`, then read it back to see every command executed.
- To see network activity, the user can wrap the run in their own tool (e.g. `sudo tcpdump`, Little Snitch, or `strace`/`dtruss`) — suggest it, don't assume it's installed.
