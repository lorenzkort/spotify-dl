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
--no-vpn     download over the plain connection (do not pass this yourself)
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
## VPN is required by default

The script refuses to download unless the default route leaves through a
tunnel (`utun*`, `ipsec*`, `ppp*`, `tun*`, `gpd*`), or `-x` names a proxy. It
exits 2 with a list of the VPN profiles configured on the machine.

**Do not pass `--no-vpn` to work around this.** If the guard trips, tell the
user their VPN is disconnected and let them decide: connect it and re-run, or
explicitly ask for `--no-vpn`. The script cannot establish a connection itself,
because a VPN client's credentials live in its own app; `scutil --nc start
"<name>"` works only for profiles macOS can bring up unaided, and silently
falls back to Disconnected otherwise.

Each track re-checks, so a tunnel that drops mid-batch stops the remaining
tracks rather than leaking the user's own IP. The summary says so, and a re-run
finishes the rest.

`-d` (dry run) only reads Spotify metadata, so it works with no tunnel.

## Rate limiting

If downloads start failing with "Sign in to confirm you're not a bot", the IP is
being rate-limited. In order: lower `-j`, add `-S 2` (sleeps with jitter), cap
with `-L 2M`, or switch VPN endpoint. It usually clears on its own.

Reach for `-C <browser>` last and say why: it sends the user's logged-in
cookies, which attaches automated downloading to their account rather than just
an IP, and an account can be actioned where an IP is only throttled. Never pass
it without the user asking.

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
