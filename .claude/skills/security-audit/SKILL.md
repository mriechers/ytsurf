---
name: security-audit
description: Focused security review of ytsurf.sh — aims a security lens at this repo's known risk surface (network-derived eval, self-updater, config sourcing, browser-cookie extraction, packaging). Use before trusting/running the script, after editing it, or whenever the user asks to audit, vet, or security-review ytsurf. Additive to the bundled /security-review.
---

# ytsurf security audit

`ytsurf` is a single Bash script that scrapes YouTube HTML and feeds parsed, network-derived values into the shell. This skill re-checks the specific spots most likely to become a vulnerability. Run it against the **current** `ytsurf.sh` (line numbers below are starting points from an earlier survey — re-locate them, they drift as the file changes).

For a general-purpose pass, the bundled `/security-review` skill still applies; this one is the ytsurf-specific checklist on top of it.

## Method

1. Re-locate each item below in the current `ytsurf.sh` (grep for the construct, don't trust the line number).
2. For each, decide: is untrusted/network input reaching a dangerous sink, and what stops it? Report file:line, the data flow, the actual risk, and — if the user is editing — whether the change widens or narrows the surface.
3. Verify quoting and validation are intact; a "fix" that drops `@uri` encoding or unquotes a variable is a regression here.
4. End with a short verdict: safe-to-run / run-with-caveats / do-not-run, and the specific caveats.

## The known risk surface (check every one)

- **`eval "$player"` (mpv ~1194, iina ~1218).** `$player` is assembled from `$video_url` → `$video_id`, taken from scraped YouTube JSON (`.videoId`) with no whitelist. Real IDs are `[A-Za-z0-9_-]{11}` and safe, but this is network content flowing into `eval` — the single highest-value finding. Check whether `$video_id`/`$video_url` is validated before reaching the eval, and whether any edit changes that.
- **Self-updater (~381-397).** `--update` curls `ytsurf.sh` from GitHub `main` and applies it with `patch` onto the running script. HTTPS but no signature or checksum, and it tracks a branch, not a pinned tag. Confirm it's still opt-in only.
- **`source "$CONFIG_FILE"` (~441).** `~/.config/ytsurf/config` is executed as bash — anything written there runs as the user. Note it as a config-integrity/RCE vector.
- **Subscription sync `yt-dlp --cookies-from-browser` (~648).** Reads the browser cookie store to reach the logged-in YouTube account. Confirm it stays behind explicit user action and that cookies aren't persisted by ytsurf.
- **`.desktop` file writes (~249-253, ~325-329).** Untrusted video `$title` written into `Name=`/`Exec=`. Check for newline/key injection; today `Exec` is only `echo`, so impact is low — confirm that hasn't changed.
- **URL-encoding of untrusted input.** Search queries and channel names must stay `jq @uri`-encoded before hitting curl (~108,138,384); `--limit` must stay numeric-validated (~576). Flag any path that skips this.
- **Temp files.** Must stay `mktemp`/`mktemp -d` (non-predictable names) with the EXIT cleanup trap intact. Flag any predictable `/tmp/...$$` path.
- **Packaging integrity (out-of-band, mention if relevant).** `ytsurf.rb` sha256 is the empty-input hash (placeholder — no real integrity check); `package.nix` installs the binary `install -Dm777` (world-writable mode). Neither is in `ytsurf.sh` but both are supply-chain smells worth noting.

## What the earlier survey already cleared

No obfuscation, no base64 blobs, no telemetry/phone-home beyond `--update`, no real hardcoded secrets (the `AIza...` key is YouTube's public web-client InnerTube key). Re-confirm these still hold if the script changed, but don't re-litigate them from scratch.
