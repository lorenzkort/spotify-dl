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

The script defends itself here too: if `-C` is passed anyway, the driver
exports the cookies **once** into a temporary `cookies.txt` and hands the
workers that file, so the keychain is read a single time instead of once per
track. A prompt appearing for every song means an older version is installed —
update it. If the export fails, the run says so and continues without cookies
rather than stalling.

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

## Playlists longer than 100 tracks

The embed page the script reads by default returns only the first 100 tracks,
reports no total, and ignores offset/limit/page. A 444-track playlist therefore
downloads as 100 unless Web API credentials are configured.

The script prints a NOTE whenever a list comes back exactly 100 long. **Do not
ignore it and do not report the run as complete** — tell the user their
playlist is probably longer than what was fetched, and that fetching the rest
needs a free app from developer.spotify.com (Client ID + Secret) saved to
`~/.config/spotify-dl/credentials` or exported as `SPOTIFY_CLIENT_ID` /
`SPOTIFY_CLIENT_SECRET`.

You cannot create that app for them: it requires logging into their Spotify
account. Ask, then re-run the same command — already-downloaded files are
skipped, so only the missing tracks are fetched.

With credentials present the script pages the full list automatically via the
client-credentials flow, which covers public playlists and albums. A private
playlist returns nothing from the API, and the script falls back to the embed
and says so.

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
  address, not the account. Lower `-j` (2 or 1), raise `-S` (3–5), and let the
  user know; if they are on a VPN, reconnecting may land on a clean exit IP.
  Do not reach for `-C` on your own — see "Never pass `-C` on your own" above.
- **"no YouTube match"** — the search came back empty. Three causes, in order of
  how often they come up:
  - YouTube's content filter. A query containing a word the filter trips on
    returns *nothing at all*, even when the word is part of the artist's name,
    and on some addresses that filter is on without being asked for. The script
    retries without those words, which is enough when the word is incidental
    ("Psychedelic Porn Crumpets" finds nothing, "Psychedelic Crumpets" finds the
    band's own channel — which spells itself "Psychedelic Prn Crumpets" to get
    around the same filter). It cannot help when the flagged word *is* the title
    or the whole artist name, because what is left is too vague to search on.
  - The track is genuinely not on YouTube. Album tracks by small artists often
    are not, and have no auto-generated "- Topic" upload either.
  - A query made long by every featured artist appearing twice. The script
    retries with the lead artist and the bare title.

  For the first and third causes, find the video by a route that is not a
  search — list the artist's channel with
  `yt-dlp --flat-playlist --dump-json "<channel_url>/videos"` and grep the
  titles — then pin it (below). Confirm a candidate by its length: Spotify's
  duration for the track is in the embed metadata, and a pin that is more than
  a second or two off is probably the wrong upload.
- **Pinning a track by hand** — `-P FILE`, or `~/.config/spotify-dl/pins` by
  default, maps a Spotify track to a YouTube video, one per line, `#` starting a
  comment:

  ```
  17s9ohMleXceK8Qobgl8Zd  MGt8-poMDD0   # carwash - striptease (161s)
  ```

  Either column accepts a bare id or a full uri/url. A pinned track skips
  searching and scoring altogether, so verify the video first — nothing checks
  that it is the right song. The run logs `. pinned:` when a pin is used.
- **HTTP 429 from Spotify** — too many embed requests from one address. It
  clears on its own; wait about ten minutes and pace the retry (serial, with a
  second or two between requests) rather than re-running the same burst.
- **"could not read Spotify metadata"** — the link is private, region-locked, or
  Spotify changed its embed page. Check that the link opens in a browser; if the
  page loads but the script finds nothing, the `__NEXT_DATA__` extraction in
  `fetch_entity` needs updating.
- **Wrong or missing tags** — the script falls back to saving untagged audio if
  ffmpeg's tagging step fails, and says so. Re-tag with ffmpeg directly.
- **Downloads are slow or stall** — yt-dlp is usually the bottleneck; let it
  run, and keep batches to one album or playlist at a time.
