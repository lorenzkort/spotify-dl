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
