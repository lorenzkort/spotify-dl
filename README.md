# spotify-dl

Take a Spotify track, album, or playlist link and save tagged MP3 files, by
matching each track on YouTube.

Spotify streams themselves cannot be downloaded. This reads the track list from
Spotify's public embed metadata (no API key, no login), finds the closest match
for each track on YouTube with `yt-dlp`, converts it to MP3, and writes proper
ID3 tags and cover art with `ffmpeg`.

## Intended use

This is a convenience wrapper around [yt-dlp](https://github.com/yt-dlp/yt-dlp)
and [ffmpeg](https://ffmpeg.org) for making personal copies of music you are
entitled to copy — your own recordings, freely licensed or public-domain
material, or tracks where personal-use copying is permitted where you live.

It does **not** touch Spotify audio or Spotify DRM. Spotify is read only for
public track metadata — name, artist, duration, cover art — exactly what the
public embed widget shows. The audio always comes from YouTube, via yt-dlp.

Do not use it to redistribute recordings, to assemble a catalogue you have no
rights to, or anywhere personal-use copying is not permitted. Respect the terms
of the services you point it at. How you use it is your responsibility.

## Install

```bash
brew install yt-dlp ffmpeg jq          # on Rosetta: arch -arm64 brew install ...
git clone https://github.com/lorenzkort/spotify-dl.git
ln -s "$PWD/spotify-dl/bin/spotify-dl" ~/bin/spotify-dl
```

## Use

```bash
spotify-dl https://open.spotify.com/track/4cOdK2wGLETKBW3PvgPWqT
spotify-dl -o ~/Music/mixes https://open.spotify.com/playlist/37i9dQZF1DXcBWIGoYBM5M
spotify-dl -d <link>              # dry run: list the tracks, download nothing
spotify-dl -j 8 <link>            # more tracks at once (8 parallel downloads)
spotify-dl -C chrome <link>       # use browser cookies if YouTube does a bot check
spotify-dl -f <link>              # re-download over existing files
```

| Option | Meaning |
| --- | --- |
| `-o DIR` | output directory (default `~/Music/spotify-dl`, or `$SPOTIFY_DL_DIR`) |
| `-j N` | tracks to download in parallel (default 4, `1` = one at a time) |
| `-n N` | YouTube candidates to weigh per track (default 5) |
| `-S SECS` | sleep between requests, with jitter (default 0) |
| `-L RATE` | cap download speed, e.g. `2M` |
| `-x PROXY` | send traffic through a proxy, e.g. `socks5://127.0.0.1:1080` |
| `--require-vpn` | refuse to download unless a VPN or proxy is in use |
| `-C BROWSER` | pass cookies from `chrome`, `safari`, `firefox`, ... to yt-dlp |
| `-f` | re-download even if the MP3 exists |
| `-d` | dry run |
| `-h` | help |

Track, album, and playlist links all work, several at a time, as
`https://open.spotify.com/...` URLs or `spotify:...` URIs. Files are named
`Artist - Title.mp3`; existing files are skipped unless `-f` is passed.

## Parallel downloads

By default four tracks are fetched at once; `-j N` changes that and `-j 1` goes
back to one at a time. Each track is an independent YouTube search, download,
MP3 conversion and tagging pass, so this scales close to linearly until the
network or the CPU saturates.

Two things to know:

- Progress bars are only shown with `-j 1`. Several at once are unreadable, so
  parallel runs print one line per finished track instead.
- Going much past `-j 8` makes YouTube rate-limiting ("confirm you're not a
  bot") more likely. If that happens, lower `-j` or pass `-C <browser>`.

A run ends with a summary of how many tracks were downloaded, already present,
and failed, naming each failure. Finished files are skipped on a re-run, so an
interrupted batch resumes where it stopped.

## VPN

A VPN is recommended, not required. When the default route does not leave
through a tunnel (`utun*`, `ipsec*`, `ppp*`, `tun*`, `gpd*`) and `-x` names no
proxy, the run says so and carries on under your own address:

```
Note: no VPN — traffic leaves through en0, under your own address.
      Worth connecting one for a long batch. --require-vpn refuses without it.
```

`--require-vpn` turns the notice into a refusal, exits 2, and lists the VPN
profiles configured on the machine. Under that flag the check runs per track,
so a tunnel dropping mid-batch stops the rest rather than falling back to your
own IP; reconnect and re-run to finish, since completed files are skipped.

The script cannot *establish* a connection — a VPN client's credentials live in
its own app, and `scutil --nc start "<name>"` only works for profiles macOS can
bring up unaided. Connect in the app, then download.

`-d` (dry run) reads only Spotify metadata, so the notice never applies.

## Being polite to YouTube

Nothing here hides that you are a script, so the way to avoid trouble is to
look unremarkable:

- `-S SECS` sleeps between requests, randomised between `SECS` and `2×SECS+1`
  so the gaps aren't a fixed heartbeat. `-S 2` is a good default for a long
  playlist.
- `-L RATE` caps download speed (`-L 2M`), which keeps a big batch from
  saturating the link.
- A lower `-j` matters more than either. Volume and burstiness are what get
  noticed, not concurrency as such.

Hitting "Sign in to confirm you're not a bot" means the IP is being
rate-limited; it normally clears by itself. Prefer waiting, a lower `-j`, or a
different VPN endpoint over `-C`.

### A warning about `-C`

`-C <browser>` clears a bot check by sending that browser's cookies, which ties
automated downloading to your **logged-in account** instead of just an IP.
That is a different and larger risk: an IP gets throttled for a while, an
account can be actioned. Treat it as a last resort, not a default.

## Playlists longer than 100 tracks

The embed page returns **at most the first 100 tracks**, carries no total, and
ignores `offset`, `limit` and `page` entirely — so a 444-track playlist looks
like a 100-track one. When a list comes back exactly full, the run says so
instead of quietly truncating.

To fetch the whole thing, give it Web API credentials:

1. Go to [developer.spotify.com/dashboard](https://developer.spotify.com/dashboard),
   create an app (any name; redirect URI is unused), and copy the Client ID and
   Client Secret.
2. Put them somewhere the script can read:

```bash
mkdir -p ~/.config/spotify-dl
cat > ~/.config/spotify-dl/credentials <<'CREDS'
SPOTIFY_CLIENT_ID=your_client_id
SPOTIFY_CLIENT_SECRET=your_client_secret
CREDS
chmod 600 ~/.config/spotify-dl/credentials
```

Or export `SPOTIFY_CLIENT_ID` and `SPOTIFY_CLIENT_SECRET` in your shell.

With credentials present the script pages through the full track list (100 at a
time) using the client-credentials flow, which reads public catalogue data and
needs no user login — so it works for public playlists and albums, not private
ones. Without them, nothing changes: the embed is used and the cap is reported.

## How a match is picked

For each track, the top N YouTube search results are scored:

- closest duration to the Spotify track wins;
- official `- Topic` channels and "official audio" titles get a bonus;
- live / cover / karaoke / instrumental / sped-up / nightcore results are
  penalised — unless the Spotify title itself says "Live" or "Remix".

It is a heuristic. For obscure tracks or songs with many versions, check the
result and try `-n 10`.

## Tags

Title, artist, album, album artist, the `spotify:track:...` URI in the comment
field, and cover art — Spotify's own album art for track and album links, the
YouTube thumbnail for playlist tracks.

## Claude Code skill

The repo is also a Claude Code plugin, so an agent can drive the script:

```
/plugin marketplace add lorenzkort/spotify-dl
/plugin install spotify-dl@spotify-dl
```

Or install the skill by hand:

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD/spotify-dl/skills/spotify-dl" ~/.claude/skills/spotify-dl
```

Then ask the agent for the songs behind a Spotify link in plain words. The skill
covers dependency checks, dry runs before large playlists, the cookie flag for
YouTube bot checks, verifying tags with `ffprobe`, and what to do when a match
is wrong.

## Caveats

- Matching is a heuristic, not a lookup. It can pick the wrong upload for
  obscure tracks, or for songs with many live and remix versions.
- Spotify's embed page is not a documented API. If Spotify changes it, the
  metadata step breaks and `fetch_entity` in `bin/spotify-dl` needs updating.
- YouTube rate-limits automated requests. Use `-C <browser>` to pass cookies
  when that happens; the cookies stay on your machine and go only to yt-dlp.
- No credentials or API keys are used, stored, or sent anywhere.

## Requirements

`yt-dlp`, `ffmpeg`, `jq`, `curl`, and bash. Written for macOS; the script is
plain bash 3.2 and should work on Linux unchanged.

yt-dlp and ffmpeg are separate projects, installed separately, and used under
their own licenses. This repo only shells out to them.

## License

MIT — see [LICENSE](LICENSE). Provided as is, with no warranty.
