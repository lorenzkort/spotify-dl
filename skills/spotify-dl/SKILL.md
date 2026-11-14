---
name: spotify-dl
description: Save a Spotify track, album, or playlist as tagged MP3 files by matching each track on YouTube. Use when the user gives a Spotify link (open.spotify.com or spotify: URI) and wants the song(s) as local MP3s, asks to download music from a Spotify link, or wants an offline/local copy of a Spotify playlist or album.
argument-hint: "<spotify-url|spotify:uri> [more links...]"
allowed-tools: Bash(spotify-dl *) Bash(*/bin/spotify-dl *) Bash(brew install *) Bash(which *) Bash(ls *) Bash(ffprobe *) Read
---

# spotify-dl

Turn a Spotify link into tagged MP3 files. Spotify streams cannot be
downloaded, so the script reads the track list from Spotify's public embed
metadata, finds the closest match for each track on YouTube, and converts that
audio to MP3 with correct tags and cover art.

Spotify is read only for public metadata; no Spotify audio or DRM is touched.
Only for content the user is entitled to keep a personal copy of. If the user
asks for something else (bypassing Spotify DRM, bulk redistribution), say that
this skill does not do that and stop.

## The script

`${CLAUDE_PLUGIN_ROOT}/bin/spotify-dl` (also on PATH as `spotify-dl` if the
user symlinked it into `~/bin`).

```
spotify-dl [options] <spotify-url|spotify:uri> [more links...]

-o DIR       output directory (default: ~/Music/spotify-dl, or $SPOTIFY_DL_DIR)
-j N         tracks to download in parallel (default: 4, 1 = one at a time)
-n N         YouTube candidates to weigh per track (default: 5)
-S SECS      sleep between requests, with jitter (default: 0)
-L RATE      cap download speed, e.g. 2M
-x PROXY     send traffic through a proxy, e.g. socks5://127.0.0.1:1080
--require-vpn  refuse to download unless a VPN or proxy is in use
-C BROWSER   take cookies from a browser (chrome, safari, firefox...)
-f           re-download even if the MP3 already exists
-d           dry run: list what it would fetch, download nothing
-h           help
```

Accepts track, album, and playlist links, several at once. `?si=...` suffixes
are fine. Files land as `Artist - Title.mp3`; tracks already present are
skipped unless `-f` is given.

## Parallelism

Four tracks are fetched at once by default. Raise it with `-j 8` for a long
playlist on a fast connection, or drop to `-j 1` when you want per-track
progress bars or YouTube is rate-limiting. Much past `-j 8` invites the "confirm
you're not a bot" check.

A parallel run prints one line per finished track and ends with a summary
(downloaded / already had / failed, with each failure named). Completed files
are skipped on a re-run, so an interrupted or cancelled batch resumes where it
stopped — just run the same command again.

## Advise a VPN, don't insist on one

A VPN is recommended, not required. The script prints a notice when traffic is
leaving under the user's own address and then carries on.

What to do with that:

- Before a long batch (an album or playlist rather than a single track),
  mention that a VPN is worth connecting, and say whether one is currently in
  use — `route -n get default` shows the interface, and a tunnel looks like
  `utun4` rather than `en0`. Then let the user decide and proceed either way.
- Do not connect, disconnect, or reconfigure a VPN on your own. Credentials
  live in the client's app, `scutil --nc start "<name>"` silently falls back to
  Disconnected for app-managed profiles like ProtonVPN, and changing someone's
  network state is theirs to choose.
- Pass `--require-vpn` only if the user asks for it. Under that flag the check
  runs per track, so a tunnel that drops mid-batch stops the rest instead of
  leaking the user's IP, and a re-run finishes what's left.
- `-x PROXY` is the other way to satisfy it, if the user has a proxy.

`-d` (dry run) reads only Spotify metadata, so none of this applies to it.

## Rate limiting

If downloads start failing with "Sign in to confirm you're not a bot", the IP
is being rate-limited. In order: lower `-j`, add `-S 2` (sleeps with jitter),
cap with `-L 2M`, or switch VPN endpoint. It usually clears on its own.

### Never pass `-C` on your own

`-C <browser>` makes yt-dlp read the browser's cookie database. On macOS those
cookies are encrypted with the login keychain, so the run **stops dead behind a
system password prompt** ("security wants to use your confidential information
stored in Chrome Safe Storage"). The user has to type their Mac login password
before anything continues — in an unattended or background run it just hangs.

It also attaches automated downloading to their logged-in account rather than
to an IP, and an account can be actioned where an IP is only throttled.

So: do not pass `-C` to get past a bot check, do not suggest it as the first
fix, and never add it to a batch the user is not watching. The normal path —
no cookies at all — needs no keychain, no browser, and no password.

If the user explicitly asks for cookies, warn about the prompt first, and
prefer `-k FILE` with an exported cookies.txt (one prompt when exporting,
none afterwards) over `-C`.

## Invoke it by full path

`spotify-dl` is often not on `PATH` in a fresh shell, which shows up as
`command not found: spotify-dl`. Use `${CLAUDE_PLUGIN_ROOT}/bin/spotify-dl`,
or `~/bin/spotify-dl` if the user symlinked it there.
## How to run it

1. **Check dependencies once.** The script needs `yt-dlp`, `ffmpeg`, `jq`, and
   `curl`, and exits with an install hint if one is missing. Install with
   `brew install yt-dlp ffmpeg jq`. On an Apple Silicon Mac where the shell is
   running under Rosetta, Homebrew refuses to install into `/opt/homebrew`;
   use `arch -arm64 brew install ...` instead.
2. **Dry run first for an album or playlist** (`-d`): it prints every track it
   found with its duration and target filename. Show the user that list before
   downloading dozens of files.
3. **Run it** and report where the files went and anything that failed. The
   script keeps going past a failed track rather than aborting the batch.
4. **Spot-check a result** when the user cares about quality:
   `ffprobe -hide_banner <file>.mp3` shows the embedded cover art stream and
   the ID3 tags.

## How the matching works

For each track the script searches YouTube and scores candidates: closest
duration to the Spotify track wins, official `- Topic` channels and "official
audio" titles get a bonus, and live/cover/karaoke/instrumental/sped-up/
nightcore results are penalised — unless the Spotify title itself says "Live"
or "Remix", in which case that penalty is lifted.

It is a heuristic. For obscure tracks, or songs with many remixes and live
versions, the top hit can be wrong. Remedies, in order:

- `-n 10` to weigh more candidates.
- Check the file: `ffprobe` for duration, or play it.
- Download that one track by hand with `yt-dlp` from a URL the user picked, then
  keep the naming convention `Artist - Title.mp3`.

## Tags

Each MP3 gets title, artist, album, album artist, the `spotify:track:...` URI in
the comment field (handy for tracing a file back to its source), and cover art:
Spotify's own album art for track and album links, the YouTube thumbnail for
playlist tracks (a playlist's artwork is not any one track's artwork).

## Troubleshooting

- **"Sign in to confirm you're not a bot"** — YouTube is rate-limiting the
  machine. Re-run with `-C chrome` (or `safari`, `firefox`) so yt-dlp uses the
  user's browser cookies.
- **"no YouTube match"** — usually a very obscure or region-blocked track. Try
  `-n 10`, or fall back to a hand-picked URL.
- **"could not read Spotify metadata"** — the link is private, region-locked, or
  Spotify changed its embed page. Check that the link opens in a browser; if the
  page loads but the script finds nothing, the `__NEXT_DATA__` extraction in
  `fetch_entity` needs updating.
- **Wrong or missing tags** — the script falls back to saving untagged audio if
  ffmpeg's tagging step fails, and says so. Re-tag with ffmpeg directly.
- **Downloads are slow or stall** — yt-dlp is usually the bottleneck; let it
  run, and keep batches to one album or playlist at a time.
