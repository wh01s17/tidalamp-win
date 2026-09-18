# tidalamp-win

A terminal TIDAL client for Windows with a retro player interface. No official API app
registration and no browser in the middle: device flow + mpv.

![The same tidalamp layout cycling through six palettes](https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/tidalamp-banner.svg?v=0.13.0)

> [!WARNING]
> **This does not run on Windows yet.** tidalamp-win is a port in progress of
> [tidalamp](https://github.com/wh01s17/tidalamp), which is a Linux application. The
> code in this repository is still the Linux code: mpv is driven over a Unix socket,
> desktop integration is D-Bus, and the audio stack talks to PipeWire. Nothing has been
> migrated, there is nothing published on PyPI or winget, and the installation
> instructions below describe the **intended** result, not something you can run today.
>
> The migration plan is [`windows.md`](windows.md). Everything else in this README
> describes the player as it behaves on Linux, which is what the port is aiming to
> reproduce.
>
> **If you are on Linux, you want [the original](https://github.com/wh01s17/tidalamp)**,
> which is finished and published.

## Quick start

*Once the port lands. See the warning above.*

```powershell
winget install mpv            # verify the id with: winget search mpv
pipx install "tidalamp-win[art]"
tidalamp
```

Authorize once through the link it prints, and the session is kept at
`%APPDATA%\tidalamp\session.json`. Then: `/` searches, `l` opens your library, and
`z` `x` `c` `v` are previous, play/pause, stop and next.

## Contents

- [Installation](#installation)
- [Hi-res, all the way to the DAC](#hi-res-all-the-way-to-the-dac)
- [Keys](#keys)
- [Queue and library](#queue-and-library)
- [The track menu](#the-track-menu)
- [Quality](#quality)
  - [Important limitation: DRM](#important-limitation-drm)
- [Themes and colours](#themes-and-colours)
- [Settings](#settings)
  - [The audio stack](#the-audio-stack)
  - [Language](#language)
- [Lyrics](#lyrics)
- [Equalizer and balance](#equalizer-and-balance)
- [About the analyzer](#about-the-analyzer)
- [Cover art](#cover-art)
- [Desktop integration](#desktop-integration)
  - [In the Start menu](#in-the-start-menu)
- [Help and about](#help-and-about)
- [Troubleshooting](#troubleshooting)
- [Platform support](#platform-support)
- [License](#license)
- [Disclaimer](#disclaimer)

## Installation

> [!IMPORTANT]
> **Nothing is published yet.** There is no `tidalamp-win` on PyPI and no winget
> manifest, because the port is not finished. Everything in this section describes the
> intended channels. The only thing you can do with this repository today is read
> [`windows.md`](windows.md) and work on it.

### Requirements

**Windows 10 (21H2) or newer.** Windows 11 is what development targets.

**Windows Terminal, not the legacy console host.** The TUI runs in both, but `conhost`
handles colours, resizing and repaint speed differently, and a full-screen retro
interface is exactly the kind of program that shows the difference. Windows Terminal
ships with Windows 11 and installs from the Store on Windows 10.

**Python 3.11 or newer**, if you install from PyPI. The standalone installer carries
its own interpreter and does not need one.

**pip does not install `mpv`.** It is a separate program and must be present, or
tidalamp exits on startup saying so.

| | mpv |
| --- | --- |
| winget (built into Windows 11) | `winget install mpv` |
| Scoop | `scoop install mpv` |
| Chocolatey | `choco install mpv` |
| By hand | a build from [mpv.io](https://mpv.io/installation/), unpacked somewhere on `PATH` |

If mpv is missing, tidalamp looks for winget, Scoop and Chocolatey in that order and
names the command for whichever you have, so the error is actionable rather than merely
true. It also looks for `mpv.exe` in the usual install locations before giving up, since
mpv is often installed without ending up on `PATH`; failing that, `mpv_path` in the
config file points at it directly.

**`cava` has no Windows build**, so the FFT spectrum is not available and the visualizer
uses the built-in RMS meter. Nothing else changes.

### From PyPI

For anyone who already has Python.

```powershell
pipx install "tidalamp-win[art]"  # the art extra adds Pillow for cover rendering
tidalamp
```

Without the `art` extra everything works except the cover, which is simply not drawn;
tidalamp says so once in the status line at startup.

The package is `tidalamp-win`; the command it installs is `tidalamp`.

The first launch has to be from a terminal (`tidalamp login`, then `tidalamp`). The
player then asks whether to add itself to the Start menu: see
[In the Start menu](#in-the-start-menu).

### From winget

For anyone who would rather not know Python exists. This installs a standalone build
with its own interpreter, and creates the Start menu shortcut itself.

```powershell
winget install wh01s17.tidalamp-win
```

You still need mpv separately, for the reason in Requirements.

### From the repository

```powershell
python -m venv .venv
.venv\Scripts\pip install -e ".[art]"   # drop [art] only if you do not want cover art
.venv\Scripts\tidalamp login
.venv\Scripts\tidalamp
```

Running `tidalamp` with no subcommand opens the player. The explicit `tidalamp tui`
command remains available and does exactly the same thing.

The release process is documented in [`publish.md`](publish.md), and packaging notes
in [`packaging/README.md`](packaging/README.md).

### Updating

```powershell
pipx upgrade tidalamp-win      # installed from PyPI
winget upgrade wh01s17.tidalamp-win   # installed from winget
```

pipx reinstalls from the spec it was given, so the `art` extra is kept. `pipx
upgrade-all` covers tidalamp along with everything else pipx manages.

Right after a release, pipx may still answer *already at latest version* with the
previous number: pip caches the package index for a few minutes. Either wait, or skip
the cache for one run:

```powershell
$env:PIP_NO_CACHE_DIR=1; pipx upgrade tidalamp-win
```

From a repository checkout instead:

```powershell
git pull
.venv\Scripts\pip install -e ".[art]"
```

`mpv` is a separate program — whatever you installed it with updates it, not pipx.

To check which version is running:

```powershell
tidalamp --version    # or -v
```

The same number is on the help screen (`?`, then `→` for _About_), together with the
notes for each release; `pipx list` shows what is installed.

### Minimum size

The interface needs **60×18** cells, and below 80×26 it switches to a compact layout
that drops the cover and the balance row. Under the minimum the fixed layout would
overlap, so tidalamp covers it with a message showing the current and required
dimensions; the normal interface returns on its own when the terminal is enlarged.

## Hi-res, all the way to the DAC

tidalamp asks TIDAL for `HI_RES_LOSSLESS` by default and plays FLAC up to
**24-bit / 192 kHz**, with the bit depth and sample rate of the running stream on
screen. When TIDAL delivers less than was asked for, the status bar says so instead of
leaving the badge to imply otherwise.

Three things had to be right for that, and two of them are not in the player:

- **The stream.** Hi-res arrives as a segmented DASH manifest, which needs rewriting
  before ffmpeg will open it.
- **The output mode.** In shared mode the Windows audio engine resamples everything to
  the device's configured Default Format, so a 24/96 stream commonly reaches the DAC at
  48 kHz while every badge tells the truth about the stream. Exclusive mode is what
  gets around it — see [The audio stack](#the-audio-stack).
- **The chain.** No software volume attenuation and no filters: with the balance
  centred and the equalizer flat, mpv carries `astats` alone, which measures and does
  not touch the signal.

Bluetooth cannot carry any of this, whatever the rates say, and the settings window
warns when the output is a Bluetooth device.

## Keys

The transport defaults are Winamp's `z x c v`, except that play and pause share one
key. All of them may be changed from [`config.toml`](#settings), and the buttons then
show the key that actually works.

| Key             | Action                                             |
| --------------- | -------------------------------------------------- |
| `z` `x` `c` `v` | previous / play-pause / stop / next                |
| `space`         | play / pause, the same as `x`                      |
| `/`             | search TIDAL                                       |
| `ctrl+f`        | search the queue                                   |
| `g`             | go to the track that is playing                    |
| `m`             | open the track menu on the queue row                |
| `p`             | save the queue as a TIDAL playlist                 |
| `↑` `↓` `Enter` | navigate; on a track, open the track menu          |
| `l`             | open the library browser                           |
| `f` `F`         | add to / remove from favourites                    |
| `/`             | inside the browser, filter the level you are on    |
| `R`             | reload the level, bypassing the cache              |
| `s`             | inside the browser, sort the level you are on      |
| `v`             | inside the browser, show the level as a list or as a grid of covers |
| `d`             | inside the browser, remove from favourites or from the open playlist |
| `m`             | inside the browser, open the menu of a track, album, artist or playlist |
| `y`             | show lyrics for the current track                  |
| `s` `r`         | shuffle (`⇄`) / repeat (`↻`), on the transport row |
| `d`             | remove from the queue                              |
| `alt+↑` `alt+↓` | move the track in the queue                        |
| `e`             | open the equalizer                                 |
| `,` `.` `\`     | balance left / right / centre                      |
| `C`             | clear the queue                                    |
| `u`             | undo the last clear                                |
| `←` `→`         | seek ±5 seconds                                    |
| `+` `-`         | change volume                                      |
| `t`             | toggle elapsed / remaining time                    |
| `w`             | full screen: the cover large; `esc` comes back     |
| `b`             | playback speed, from 0.25× to 2×                   |
| `o`             | open the settings window                           |
| `?` `h`         | open the help window                               |
| `q`             | quit, after asking (`q` again confirms)            |
| `ctrl+c`        | quit at once, without asking                       |

Navigation keys are fixed — arrows, Page Up/Down, Enter and Esc — because a typo there
could make the browser unusable.

### Full screen

`w` opens a full-screen view: the cover as large as the terminal allows, centred, and a
bar at the foot with the track on the left, the controls, the seek bar and the times in
the middle, and the quality and a button for the queue on the right. `tab` (or a click
on that button) opens the queue beside the cover, which shrinks to make room; `↑` `↓`
walk it and `Enter` plays, and the queue's own keys work in it as they do in the
player's: `g` goes to the playing track (and opens the panel if it is closed), `d`
removes, `alt+↑` `alt+↓` move, `m` opens the track menu, `f` `F` favourite. With the
panel closed those keys ask for it rather than act on a row you cannot see. `?` lists
this view's keys alone. `w` again, or `esc`, comes back to the player. The view takes
its colours from the palette and its frame from the theme in use, and the transport keys
work in it as they do everywhere.

<table>
  <tr>
    <td width="50%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/fullscreen.webp?v=0.13.0" alt="The full-screen view: an album cover centred on a black ground, and a bar at the foot with the track, artist and album on the left, the shuffle, previous, pause, next and repeat controls over the seek bar and the times in the middle, and the quality, the queue button and the keys on the right"></td>
    <td width="50%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/fullscreen-queue.webp?v=0.13.0" alt="The same view with the queue open on the right: thirty numbered tracks with their lengths, the playing one highlighted, and the cover shifted left to make room"></td>
  </tr>
  <tr>
    <td align="center"><sub>The cover as large as the terminal allows</sub></td>
    <td align="center"><sub><code>tab</code> puts the queue beside it</sub></td>
  </tr>
</table>

## Queue and library

`/` searches for tracks and displays them directly, with albums, artists and playlists
in three category rows above; a category is fetched only when opened.

Press `l` to open the library browser: playlists, favourite tracks, albums, artists,
and your mixes (the daily ones, discovery, new arrivals and the rest TIDAL makes for
your account). Enter a level with `↵` and go back with `⌫`. A mix opens like a
playlist, but it cannot be sorted or edited, so `s` and `d` do nothing there.

An artist opens to its sections: popular tracks, albums, EPs and singles, and other
(compilations and appearances), each disc opening to its tracks. They are the
sections TIDAL has, in its own order, and one with nothing in it is left out. `m` and
`a` on an artist still play its popular tracks.

**Discover**, the last row of the library, is what TIDAL proposes rather than what you
keep: its home page (recently played, albums you will enjoy, new tracks, your mixes and
radio stations), For you, and Explore, whose genres, moods and decades each open to a
page of their own. Every page opens to its categories and every category to what it
holds, which plays and opens like anything else in the library. Videos and TIDAL's
banners are left out, and a category shows what TIDAL puts on the page, usually the
first ten.

**The grid.** `v` in the browser (or Library view in the settings window) shows a level
of albums, playlists, artists or mixes as tiles, each with its cover, its name and a
line under it; `v` again goes back to the list, and the choice is kept in
`config.toml` as `library_view`. The arrow keys walk the tiles, `⌫` goes back, and
every other key of the browser works as it does in the list. A level of tracks is
always a list. The covers are drawn with text, whatever the terminal can do, because an
image sent with kitty or sixel is painted over the text and would cover the menu, the
help and every question that opens on top of the grid. In kitty, ghostty, WezTerm and
foot they use sextants, six pixels to a cell, which those terminals draw themselves;
anywhere else, the quadrants every font has, four to a cell. `TIDALAMP_SEXTANTS=1`
turns sextants on for another terminal whose font has them, and `0` turns them off.
They arrive a moment after the tiles, the ones on screen first.

Tracks play one into the next with no gap: the next one is fetched from TIDAL shortly
before the current one ends and handed to mpv ahead of time, along with its cover (and
its lyrics, when the split view or the lyrics window is showing them).

Quitting keeps your place: the queue comes back on the next start with the cursor on
the track you were hearing, the status line says the second it will resume at, and
playing that track picks up there.

`g` brings the queue cursor back to the track that is playing. If the queue search is
hiding it, the search is cleared first. `p` asks for a name and saves a snapshot of the
queue as a TIDAL playlist, in queue order rather than shuffle order. Large queues are
sent in batches; if TIDAL stops accepting them part-way through, the partial playlist
is kept and the status line says exactly how many tracks made it.

- `↵` on a track opens the [track menu](#the-track-menu); its **Play now** queues the
  entire level, so the rest of the album or playlist follows it.
- `m` opens the same menu, and on an album, an artist or a playlist it offers the same
  verbs over everything inside it.
- `a` appends an item without interrupting the current track. On a playlist or album,
  it appends all of its contents, every page of it.
- `A` appends every track in the current level.

Modal windows — the library, search, settings, lyrics, help — take a share of the
terminal rather than a fixed 84x26, so a large screen gets a large library.

Turn `transparency` on (in the settings window, or in the config file) and they open
over a translucent scrim instead, leaving the player visible and dimmed behind them: a
terminal cannot blur, and the scrim is what stands in for it. While a modal is open the
player stops redrawing itself, so the scrim costs less than the opaque window did.

Turning it on also limits the cover to `blocks` or `off` for as long as it lasts, and
switches to `blocks` when it has to, saying so. Blocks are ordinary
characters, so the window draws over them; an image sent with the kitty or sixel
protocol is painted by the terminal *over* the text, which would put the album art on
top of the window you are reading. The change applies immediately — the cover is redrawn
with the new protocol without restarting tidalamp.

<p align="center">
  <img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/transparency.webp?v=0.13.0" alt="The settings window over a translucent scrim, with the player dimmed behind it: transparency is on and the cover has been moved to blocks" width="880">
</p>

The settings window over the scrim. `Transparencia` is on, and `Carátula` sitting on
`blocks` underneath it is the move this made on your behalf.

- `/` **filters the level you are on**, from a bar at the foot of the window that
  narrows the list underneath instead of covering it. What it filters is whatever the
  level holds: tracks in your favourites, playlists in `My playlists`, albums, artists,
  or a category of search results. Case and accents are ignored — `sinfonia` finds
  *Sinfonía* — every word you type has to match somewhere, and a track is also found by
  its album, which is not on the line unless you turned that column on. `↵` applies the
  filter and gives the arrows back to the list; `Esc` clears it and leaves the browser
  open, on the row you had reached. In the library, typing a filter brings in the rest
  of the level, page by page, so it searches the whole collection and not the pages
  already loaded. A search is not fetched whole, having no end worth reaching: there,
  the `more…` row is never filtered out, and `↵` on it fetches the next page.

`f` adds the selected track, album, artist, or playlist to TIDAL favourites; `F`
removes it.

`b` opens the speed window: 0.25× to 2× in quarters, 1× as recorded. Walk it with
`↑` `↓` and apply with `↵` (or click a speed); `esc` leaves the speed alone. The
transport's speed button says the current one and lights up off 1×. mpv keeps the
pitch, which it does with a filter, so a speed other than 1× is not bit-perfect.
The speed lasts until you quit.

`s` toggles shuffle and `r` cycles repeat (off → queue → track). Both sit on the
transport row as buttons, lit in the palette's accent while they are on. Every state
is also readable without colour: `⇄○`/`⇄●` for shuffle, and `↻–`/`↻A`/`↻1` for the
three repeat modes — the `retro`, `nova` and `ascii` layouts spell the same states out
as `SHUFFLE ○` and `REPEAT 1`.

`p` **saves the queue as a new TIDAL playlist**: it asks for a name and creates it with
the tracks in queue order, not in shuffle order, because what is saved is the list and
not the listening session. To append to a playlist that already exists, use `≡` in the
track menu instead.

`ctrl+f` **searches the queue**, from a bar under the playlist, the same gesture the
browser's `/` is: type and the queue narrows to what matches, without covering it. It
matches the same way — case and accents ignored, every word has to appear somewhere,
and a track is also found by its album. The rows keep the number they really have in
the queue, so a match numbered `47` tells you where it is in the playing order rather
than pretending to be the first track. `↵` gives the arrows back to the list with the
filter still applied, and `Esc` clears it and leaves the cursor on the track you had
reached. Everything that acts on the selected row — `↵`, `d`, `alt+↑`, `alt+↓`, `f` —
acts on that track and not on its place on screen.

`alt+↑` and `alt+↓` move the selected track. With shuffle enabled, moving a row does
not reshuffle what comes next. While the queue is filtered the rows may not visibly
reorder — the track it swapped with can be one the filter is hiding — but the number
at the head of the line changes, because that is the queue position.

Long levels come from TIDAL a hundred at a time, and the next hundred is fetched on
its own as the cursor nears the end, in the background, into the same level and
without losing the cursor's place: there is nothing to press. A level shorter than the
window fills it at once. The final row, `more…`, is there while a page is still to
come, and `↵` on it fetches it too.
Opened levels are cached for the lifetime of the application, so returning to one is
instant. `R` fetches the current level again, which is useful after creating a
playlist on another device.

The queue is stored at `%LOCALAPPDATA%\tidalamp\state\queue.json` and restored on startup,
including the previous cursor. Anything slow runs in the background, and a spinner
names what is pending — `⠋ opening My playlist…` — instead of freezing the interface.

### Queue columns

A wide enough terminal splits the queue into columns instead of running the artist
into the title. Which ones is up to you: the settings window (`o`) has a **Queue
columns** row that opens a picker, and the choice applies to the queue already on
screen rather than to the next one loaded.

| name | shows |
|---|---|
| `track` | the number the song carries on its own album — not its place in the queue, which is always drawn |
| `version` | `Remastered 2011` and the like, when TIDAL has one |
| `artist` | on by default |
| `album` | on by default |
| `year` | on by default |
| `quality` | `HI-RES`, `LOSSLESS`, `HIGH`, `LOW` |
| `explicit` | `E` |
| `popularity` | TIDAL's 0–100 |
| `disc` | disc number on a multi-disc release |
| `isrc` | the recording's ISRC |
| `duration` | on by default |

The queue position on the left and the title are always drawn, and a field TIDAL has
no answer for leaves its cell blank rather than inventing a value.

Columns are dropped as the window narrows, in the order they can be spared — the year
before the album, the album before the artist — ending at `artist - title` on one line
with the duration on the right. The artist only leaves the title when it has a column
of its own to go to.

## The track menu

`↵` on a song — in search results or in the library — opens a small menu instead of
assuming what you meant. `m` opens the same menu, in the browser and on the queue row
under the cursor, where `↵` already plays it:

|     | Action            | Key | What it does                                                                     |
| --- | ----------------- | --- | -------------------------------------------------------------------------------- |
| `▶` | Play now          | `a` | Queues the whole level and starts on this track.                                 |
| `↳` | Play next         | `c` | Inserts just this track after the one playing. Under shuffle it really is next.  |
| `≈` | Track radio       | `d` | Plays TIDAL's station for this track: the seed first, then the similar songs.    |
| `♥` | Add to favourites | `v` | Adds it to your TIDAL favourites, leaving the queue alone.                       |
| `≡` | Add to a playlist | `l` | Picks one of the playlists you created and appends the track to it.             |
| `◉` | Go to the artist  | `t` | Opens the browser on the track's artist; with several, asks which one.          |
| `◎` | Go to the album   | `b` | Opens the browser on the track's album.                                          |

`↑` `↓` and `↵` pick, `Esc` backs out. `↵` on an album, artist or playlist still opens
it: a level has one obvious thing to do.

Going to the artist or the album from the browser opens it on top of the level you
were on, and `⌫` comes back to it. From the queue, the browser opens right there, and
`⌫` goes back to the root of the library instead of closing it.

On a playlist of yours, one in `My playlists`, the menu has three more lines: **rename**
(`n`), **change the description** (`e`), typed over the one it has, and an empty one
clears it, and **delete the playlist** (`x`), which asks first with the cursor on Cancel.
Inside one of your playlists, `alt+↑` and `alt+↓` move the track under the cursor, as in
the queue, and TIDAL is asked afterwards whether it landed where it was sent. That only
works with the playlist in its own order and unfiltered: sorted with `s` or narrowed
with `/`, a row's place on screen is not its place in the playlist, and the browser says
so instead of moving it.

`m` on an album, an artist or a playlist opens the same menu for everything inside it:
**play it all now** (`a`), **play it all next** (`c`), **add it to your favourites**
(`v`) or **add it all to a playlist** (`l`). The whole of it comes along, every page
and not only the first hundred tracks, in the order you picked for it with `s`. There
is no radio in this one: a station grows from a single track. For an artist, "all" is
their popular tracks, not every disc in their sections.

Not every track has a radio station — TIDAL simply has none for some obscure releases
— and when it does not, the status line says so and nothing is queued.

## Quality

The default requested quality is `HI_RES_LOSSLESS`. TIDAL answers with one of two
manifest kinds: `BTS` is a progressive URL mpv opens directly, and `MPD` is the
segmented DASH used for hi-res. Measured against a real account on 2026-09-08:

| Requested         | `HIRES_LOSSLESS` track        | `LOSSLESS`-only track |
| ----------------- | ----------------------------- | --------------------- |
| `LOW`             | BTS, LOW, 96 kbps             | same                  |
| `HIGH`            | BTS, HIGH, 320 kbps           | same                  |
| `LOSSLESS`        | BTS, **HIGH**                 | BTS, **HIGH**         |
| `HI_RES_LOSSLESS` | **MPD**, FLAC 24-bit / 96 kHz | BTS, HIGH             |

In other words, **requesting `LOSSLESS` never produced lossless audio** through the
device-flow client: TIDAL returned `HIGH` even for tracks it labels `LOSSLESS`.
Requesting `HI_RES_LOSSLESS` yields FLAC where available and `HIGH` otherwise, so it is
strictly better than the old default.

When TIDAL delivers less than requested, the status bar says so (`TIDAL delivered
HIGH, not HI_RES_LOSSLESS`) instead of leaving the badge to imply it.

```powershell
$env:TIDALAMP_QUALITY="HIGH"; tidalamp
```

Valid values are `LOW`, `HIGH`, `LOSSLESS`, and `HI_RES_LOSSLESS`.

### Important limitation: DRM

Tracks with encrypted Widevine manifests **cannot be played by mpv** because there is
no CDM to decrypt them. tidalamp detects this and reports it in the status bar instead
of failing with a codec error. If it happens frequently, lower the quality with
`TIDALAMP_QUALITY=HIGH`.

## Themes and colours

Two independent settings. `theme` picks the **layout** — how the interface is drawn.
`palette` picks the **colours** it is drawn in. Any layout works with any palette, and
both can be changed from the settings window (`o`) without restarting playback.

| `theme`   | Look                                                                                                                                                                                                   |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `quattro` | The default. Flat, modern, short dividers, left-aligned headings.                                                                                                                                      |
| `retro`   | The 1997 skin as far as a terminal goes: title bars drawn as a rule with the heading centred on it, square transport keys packed shoulder to shoulder, and the toggles spelled `SHUFFLE` and `REPEAT`. |
| `nova`    | Frameless. One flat ground, no boxes anywhere, and colour reserved for the two controls that carry state — an accent rule under whichever toggle is on.                                                |
| `ascii`   | A terminal before it had box drawing: `[ z << ]` bracket keys, rules made of `=` and `-`, and no glyph in the chrome you could not type. The meters keep their block characters.                       |

<table>
  <tr>
    <td width="50%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/theme-quattro.webp?v=0.13.0" alt="The quattro layout"></td>
    <td width="50%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/theme-retro.webp?v=0.13.0" alt="The retro layout"></td>
  </tr>
  <tr>
    <td align="center"><code>theme = "quattro"</code></td>
    <td align="center"><code>theme = "retro"</code></td>
  </tr>
  <tr>
    <td width="50%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/theme-nova.webp?v=0.13.0" alt="The nova layout"></td>
    <td width="50%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/theme-ascii.webp?v=0.13.0" alt="The ascii layout"></td>
  </tr>
  <tr>
    <td align="center"><code>theme = "nova"</code></td>
    <td align="center"><code>theme = "ascii"</code></td>
  </tr>
</table>

### Themed looks

Ten more layouts come with a palette of their own name. Choosing one in the settings
window also sets `palette` to its colours, once: after that the palette is yours again,
so a themed layout in `nord` is one keypress away and nothing puts the pair back. Their
palettes can be picked on their own too, under any layout.

| `theme`         | Look                                                                         |
| --------------- | ---------------------------------------------------------------------------- |
| `unidad-morada` | Purple armour, lime for what is lit, orange warning stripes, a heavy frame.  |
| `pirata`        | Straw yellow on open sea, a rounded frame and a log for a queue.             |
| `cuaderno`      | A black notebook: no frame, a red margin rule, headings on a ruled line.     |
| `neon-noir`     | Night city neon, yellow and cyan, a thick frame and hard flat bars.          |
| `runas`         | Old gold on a dark forest, a double frame and square keys.                   |
| `reggae`        | Red, gold and green on black, a wide frame.                                  |
| `comodin`       | The wild card: a purple suit, green hair, a dashed card edge and the suits.  |
| `gotico`        | Crimson and violet under a pointed arch, square keys and centred headings.   |
| `death-metal`   | Bone on black, blood red, a tall frame and noise at the edges.               |
| `bosque`        | Leaf green on moss, a rounded frame with a vine along the top, a leaf at its foot. |

### Two columns

`arrangement = "split"` puts the queue in a column to the right of the player instead of
under it. The player's column shows the lyrics of the playing track above the cover,
following the sung line when TIDAL has timed lyrics, and the transport keys and the
menu run across both columns at the bottom. It needs a terminal at least 160×26; on a
smaller one it stays stacked on its own, and the settings window says so. Switching
(from the settings window, `o`) moves nothing but the layout: the track keeps playing
and the queue's cursor stays where it was. It works with every layout and palette.

Each of the ten also has a picture and a line of its own. The picture is drawn behind
the queue the way the cover is drawn, darkened so the rows on top still read; the line
stands in for the title while nothing is playing. The picture needs Pillow (the `art`
extra), like the cover.

The picture is a setting of its own, `backdrop` (Queue backdrop in the settings window):
`auto` is the theme's, `none` removes it, and any themed look's name borrows its
picture. Choosing a themed look sets its palette and its picture once; after that both
are yours to change, so any theme, palette and picture can be mixed.

<p align="center">
  <img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/split-backdrop.webp?v=0.13.0" alt="The split arrangement: timed lyrics above the cover and the clock on the left, the queue on the right with a purple armoured figure drawn dimly behind its rows, and the transport keys running across both columns" width="880">
</p>

Split, with the timed lyrics following the song above the cover. The queue carries the
`unidad-morada` picture under a layout and palette that are not its own.

### Palettes

`palette` accepts `classic` (green-on-black, the player's own), the built-ins
`tokyo-night`, `catppuccin`, `nord`, `gruvbox` and `black` (pure black with grey and
white accents, where lightness carries what hue carries elsewhere), the ten that come
with the themed looks, or the name of a TOML file you drop in
`%APPDATA%\tidalamp\palettes\`.

Custom palettes keep the format the Linux version uses, which is
[Omarchy](https://omarchy.org)'s `colors.toml`. That is not an accident and not a
leftover: a palette file written on a Linux machine works here unchanged, and the
format is a plain table of colour names, so nothing about it needs a desktop
environment. `auto` follows the system theme where there is one to follow, and falls
back to `classic` here.

The same layout, repainted.

<table>
  <tr>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-emerald.webp?v=0.13.0" alt="Bright green on black"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-green.webp?v=0.13.0" alt="Muted green on black"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-mint.webp?v=0.13.0" alt="Green on navy"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-amber.webp?v=0.13.0" alt="Amber on black"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-sand.webp?v=0.13.0" alt="Sand on warm grey"></td>
  </tr>
  <tr>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-orange.webp?v=0.13.0" alt="Orange on navy"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-cyan.webp?v=0.13.0" alt="Cyan on a dark ground"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-blue.webp?v=0.13.0" alt="Blue on navy"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-daylight.webp?v=0.13.0" alt="Blue on cream"></td>
    <td width="20%"><img src="https://raw.githubusercontent.com/wh01s17/tidalamp-win/main/img/palette-paper.webp?v=0.13.0" alt="Grey on white"></td>
  </tr>
</table>

A palette file that is missing or invalid falls back to the player's own green-on-black
`classic` rather than failing — the same treatment a typo in the config file gets.

## Settings

`o` opens a settings window — the transport bar lists it, next to `? help`. Every row
writes `%APPDATA%\tidalamp\config.toml`, so a change made once stays made.

| Setting   | Values                                    | Takes effect   |
| --------- | ----------------------------------------- | -------------- |
| Quality   | `LOW` `HIGH` `LOSSLESS` `HI_RES_LOSSLESS` | the next track |
| Cover art | `auto` `kitty` `sixel` `blocks` `off`     | on restart     |
| Cover shape | `square` `rounded` `round`              | immediately    |
| Language  | `auto` `es` `en`                          | on restart     |
| Visualizer | `bars` `mirror` `curve` `fine`            | immediately   |
| Autoplay  | on / off                                  | immediately    |
| Normalised volume | `off` `track` `album`             | immediately    |
| Library view | `list` `grid`                          | the next level opened |
| Exclusive mode | on / off                             | the next track |
| Output device | the WASAPI devices mpv reports         | the next track |
| Debug log | on / off                                  | immediately    |
| Menu shortcut | created / not created                 | immediately    |
| Log out   | action                                    | closes tidalamp |

Normalised volume uses the ReplayGain TIDAL sends with every stream: `track` evens
out every track, `album` keeps the loud and quiet songs of one record as the record
has them. A track is never raised past its own peak, so nothing clips. While it is on,
the badge line under the clock says the gain applied to the track playing, such as
`RG -7.5 dB`, or `RG —` when TIDAL sent no gain for it.

The lines at the foot of the window (the note on the current row, the output and any
warning) glide there and back when they do not fit, as the track title does.

Log out deletes the saved session and closes tidalamp; to use it again, run
`tidalamp login`. Its question has a box to delete tidalamp's data too:
`%APPDATA%\tidalamp`, `%LOCALAPPDATA%\tidalamp\state` and
`%LOCALAPPDATA%\tidalamp\cache` (settings, queue, equaliser, covers) and any shortcut
for tidalamp in your Start menu, with its icon. The next start is then a first start,
and asks about the menu again. Shortcuts installed machine-wide by an installer are
left alone.

Quality, Exclusive mode, Output device, Menu shortcut and Log out do not change with
the arrows, since a stray press on any of them costs more than a colour: Enter opens a
list to choose from, and Log out's list opens on Cancel.

`tidalamp config` shows the effective settings and creates the file if it does not
exist:

```toml
quality = "HI_RES_LOSSLESS"   # LOW, HIGH, LOSSLESS, or HI_RES_LOSSLESS
artwork = "auto"              # auto, kitty, sixel, blocks, or off
cover_shape = "square"        # square, rounded (rounded corners), or round (a disc)
language = "auto"             # auto follows the locale; es or en pin it
columns = "artist,album,year,duration"   # queue columns, comma separated
theme = "quattro"             # layout: quattro, retro, nova, ascii, or a themed look
palette = "auto"              # colours: auto, classic, a built-in, or your own
arrangement = "stacked"       # stacked, or split: the queue in a column on the right
backdrop = "auto"             # the picture behind the queue: auto, none, or a themed look
visualizer = "bars"           # analyzer shape: bars, mirror, curve, or fine
autoplay = false              # when the queue ends, carry on with the last track's radio
replaygain = "off"            # normalised volume: off, track, or album
library_view = "list"         # the library as a list, or as a grid of covers
exclusive = false             # WASAPI exclusive mode: bit-perfect, and nothing else sounds
mpv_path = ""                 # where mpv.exe is, when it is not on PATH
debug = false                 # log to %LOCALAPPDATA%\tidalamp\state\tidalamp.log

[keys]
play = "p"
quit = "ctrl+q"
```

Precedence is **environment → file → default**. `TIDALAMP_QUALITY`, `TIDALAMP_ART`,
`TIDALAMP_LANG`, `TIDALAMP_COLUMNS`, `TIDALAMP_THEME`, `TIDALAMP_PALETTE`,
`TIDALAMP_VISUALIZER`, `TIDALAMP_LIBRARY_VIEW`, and `TIDALAMP_DEBUG` therefore override the file for one-off runs; the settings window
labels a row whose value is being shadowed that way, rather than showing a value the
app is not using. A syntax error in the file does not prevent startup; it is logged
and the defaults take over.

Under `[keys]`, the action is on the left and the key on the right; separate multiple
keys with commas. Valid actions are the ones in the [key table](#keys), and
`tidalamp config` warns about unknown ones.

### The audio stack

The same window shows what is underneath mpv, because nothing else can:

```
  Exclusive mode (WASAPI)    off
                               the audio engine is at 48000 Hz and resamples…
  Output device              Your USB DAC (WASAPI)

  Output: Your USB DAC · 48000 Hz s32le
```

Windows mixes every application into one stream at one sample rate, the device's
**Default Format** in the Sound control panel, and resamples anything that does not
match. So a 24/96 stream reaches the DAC at 48 kHz: the badge in the player is telling
the truth about the stream, and the DAC still never sees hi-res. Nothing inside a
shared-mode application can change that, which is why the window says it rather than
pretending otherwise.

**Exclusive mode** is the way around it. mpv takes the device for itself and opens it at
the source's own rate, which is bit-perfect and follows every track without the manual
dance the control panel would need. The cost is in the name: while tidalamp is playing,
**nothing else on the machine can make a sound** — no notifications, no browser, no
other player. That is why it is off by default and has to be turned on deliberately.

Without exclusive mode, the other option is manual: set the Default Format in the Sound
control panel to the rate you listen at most, and accept resampling for the rest. If the
track's rate and the device's differ, the `OUT` badge says `resampled from … kHz`.

The window also names the output and warns when it is Bluetooth, which cannot carry
lossless whatever the rates say. Exclusive mode is reversible from the same row, and
nothing it does survives turning it off.

### Language

English and Spanish are built in; Spanish is the fallback for unsupported locales.
With `auto`, the standard `LANGUAGE`, `LC_ALL`, `LC_MESSAGES`, and `LANG` variables are
consulted in that order — they are not usually set on Windows, and when none of them is,
the Windows display language decides.

```toml
language = "en"   # auto, es, or en
```

Theme and palette names follow the language on screen: in English `bosque` reads as
forest, `pirata` as pirate, `unidad-morada` as purple-unit. `config.toml` keeps the
names in the table above whichever language is set, so a file written under one
still works under the other.

To override for one run:

```powershell
$env:TIDALAMP_LANG="en"; tidalamp
```

## Lyrics

`y` opens lyrics for the current track without stopping playback. When TIDAL provides
LRC subtitles, the active line is highlighted and the window follows mpv's position.
Plain text can be scrolled with `↑`, `↓`, `PageUp`, and `PageDown`.

Not every track has lyrics, and regional licences do not always expose them. In that
case, the window displays an error and playback continues normally.

## Equalizer and balance

`e` opens a ten-band equalizer (60 Hz … 16 kHz, matching Winamp) with a ±12 dB range.
Use `←→` to select a band, `↑↓` to adjust it, and `0` to flatten it. Changes are
applied while you move them; an equalizer you cannot hear until pressing “OK” is not
useful.

`p` and `P` walk through eight presets — flat, rock, pop, jazz, classical, vocal, bass
and treble — and the one you are on is named under the bands. Move a band afterwards and
it stops being that preset and says `manual`, because the name is worked out from the
gains rather than remembered.

`,` and `.` move the balance, and `\` centres it, including from the main window.
The position, volume and balance bars also accept a click; clicking position while no
track is loaded does nothing.

Settings are stored in `%LOCALAPPDATA%\tidalamp\state\settings.json` and reapplied on
startup.

## About the analyzer

The quality display identifies which of two modes is active:

- **`RMS`:** mpv exposes levels through its `astats` filter, and the analyzer becomes a
  band-shaped meter with fast attack and slow decay. It reacts to music but is not a
  frequency breakdown, and the badge says so. **This is what you get on Windows.**
- **`FFT`:** a real spectrum, measured by [cava](https://github.com/karlstav/cava). cava
  is a Linux program with no Windows build, so this mode belongs to
  [the Linux version](https://github.com/wh01s17/tidalamp) and the badge never shows it
  here.

A real FFT is possible on Windows — loopback capture through WASAPI, with the transform
done in-process — and it would fix the caveat cava has always had on Linux, where it
listens to the whole output device rather than to tidalamp specifically and so shows
whatever the machine is playing. It is not implemented; see `next.md`.

### Shapes

The `visualizer` setting picks how that meter is drawn. All four read the same frame, so
switching between them costs a redraw and nothing else — the music is never interrupted.

| Shape    | What it draws                                                |
| -------- | ------------------------------------------------------------ |
| `bars`   | The default: upright bars with falling peaks                 |
| `mirror` | Bars growing up and down from a centre line                  |
| `curve`  | The contour of the spectrum as a line, one glyph per column  |
| `fine`   | The same line on the Braille dot grid: twice the horizontal resolution, and joined up into a stroke |

`fine` needs a font with Braille. Most have it — every Nerd Font, DejaVu, the Noto
family — but a font without it draws boxes, and a terminal cannot be asked beforehand,
so it is a shape you choose rather than one anything falls back to.

All four are drawn where the analyzer has always been — beside the cover, under the
track details — and all three run to the right edge of the window. Nothing moves and
no row is taken from the queue.

The bars are capped at 64 bands and made wider to cover the width, rather than growing
thinner as the terminal grows: on a 4K display the uncapped version cost the app 55% of
a core at ten frames a second, because every band is an escape sequence the terminal
has to chew through. Capped and with runs of one colour merged, the same picture costs
about 8%.

Change it from the settings window (`o`, under _Appearance_), from `config.toml`, or
with `TIDALAMP_VISUALIZER=curve tidalamp`. It applies immediately.

## Cover art

Album art is drawn to the left of the display, in a box that grows with the terminal
from 18×9 cells up to 40×20. The renderer is selected automatically from the
terminal's capabilities:

| Protocol       | Terminals on Windows        | Result                        |
| -------------- | --------------------------- | ----------------------------- |
| kitty graphics | WezTerm                     | real pixels                   |
| sixel          | Windows Terminal 1.22+      | real pixels                   |
| blocks         | any other terminal, including conhost | four samples per cell, two colours |

**Blocks are the default here**, and on most Windows machines they are what you get.
The pixel protocols are a Unix terminal tradition that Windows has only recently begun
to pick up: Windows Terminal gained sixel in 1.22, WezTerm speaks kitty graphics
because it is a port of a Unix terminal, and the console host speaks neither.

Detection reads `$WT_SESSION`, `$TERM_PROGRAM`, `$WEZTERM_EXECUTABLE` and `$TERM`, and
falls back to blocks, which work everywhere. Blocks are also what you want if anything
looks wrong: a sixel sent to a terminal that does not understand it prints garbage
across the screen, while blocks are ordinary characters and cannot fail.

`cover_shape = "round"` (Cover shape in the settings window) draws the cover as a disc,
under any theme. The corners are painted in the band's own colour rather than left
transparent, since sixel and blocks have no transparency to leave, so the disc looks
the same with every protocol. `rounded` keeps the square and rounds its corners, with a
radius in proportion to the side so it still shows in `blocks`.

`blocks` draws with the quadrant glyphs (`▘▝▖▗▚▞…`), so each cell carries **four**
samples: two across and two down. A cell still holds only two colours, so where its four
pixels disagree they are split into a light group and a dark one and each is averaged —
on a photograph neighbouring pixels rarely disagree by much. Before this the renderer
used `▀` alone, which spent a whole cell's width on a single pixel and made covers look
stretched. The glyphs come from the same Block Elements range as `▀` and `█`, so nothing
is asked of a font that was not asked before.

To force a renderer:

```powershell
$env:TIDALAMP_ART="blocks"; tidalamp   # kitty | sixel | blocks | off
```

**Pillow** is required to decode images, and the `art` extra installs it
(`pipx install "tidalamp-win[art]"`). Without it, cover art is omitted and everything
else keeps working — the same treatment a terminal with no pixel protocol gets — and the
status bar says so at startup. Covers are cached under
`%LOCALAPPDATA%\tidalamp\cache\art\`, keyed by URL.

## Desktop integration

> [!NOTE]
> **This is the one feature the port loses, at least for now.** On Linux, tidalamp
> publishes itself over MPRIS, so media keys, the desktop's own media widget and status
> bars all see the track and can control it. Windows has an equivalent — the System
> Media Transport Controls, the panel that appears when you press a media key — but it
> is bound to a window handle, and a terminal application does not have one. Getting
> there means creating a hidden window with its own message loop, which is real work
> and is not done.
>
> Until it is, the media keys will not reach tidalamp and nothing outside the terminal
> knows what is playing. Everything inside the player works exactly as documented.

Playback is controlled from the player's own keys: `z` `x` `c` `v` for previous,
play/pause, stop and next, with the full list under `?`.

The speed lives in its own window (`b`), between 0.25× and 2× in quarters, with the
pitch preserved.

### In the Start menu

pipx installs a command and nothing else, so the Start menu does not know about
tidalamp. The first time the player opens (logged in, with mpv present) it asks in a
dialog whether to add it:

- **yes** writes `TidalAmp.lnk` into your Start menu, with its icon. It opens in
  Windows Terminal where that is installed, and in the console host otherwise.
- **no** is remembered, and the question is not asked again.
- **Esc** leaves it for the next start.

If a shortcut for tidalamp already exists — the winget installer creates one — nothing
is asked. To take it out of the menu, delete it: it will not be offered again.

**Menu shortcut**, under General in the settings window (`o`), does the same at any
time, even after a no: it says `created` or `not created`, Enter offers to create it,
and when one already exists it shows where instead. `TIDALAMP_NO_DESKTOP_ENTRY=1`
turns off the first-start question.

## Help and about

`?` (or `h`) opens a window listing every key with what it does, grouped by what you
are doing: playback, volume, the queue, the windows, favourites, and the keys that
only apply inside search and the library. It reads the bindings from the running app,
so a key rebound in `config.toml` shows up there as the key you actually have to
press.

The window has two tabs. `→` moves to _About_ — version, author, repository, licence
and a summary of what each released version brought — and `←` comes back to the keys,
where you left them.

`↑` `↓` scroll, `PgUp` `PgDn` a page, `Home` `End` jump to either end, and `?`, `h` or
`Esc` close it.

Inside the browser, whether it is showing your library or search results, `?` opens
the same window with the browser's keys alone; the browser's footer names only `?`
and `esc`.

`/` searches the tab you are on: a box opens under the text, and what you type narrows
it to the lines that match, each under its section's heading, ignoring accents and
case. `Enter` hands the arrows back to the page with the search still on, and `Esc`
clears it before it closes the window. The footer names the key.

## Troubleshooting

If mpv dies, tidalamp starts a fresh process and reloads the current track. Expired
tokens are refreshed automatically; `tidalamp login` is only needed when there is no
usable refresh token. Failed TIDAL calls are retried with backoff.

**If the interface looks wrong — smeared colours, a cover that is a grey slab, rows that
do not line up** — try Windows Terminal before anything else. The console host renders
differently and it is where this kind of thing shows up first.

**If `OUT` sits below `SRC`**, the audio engine is resampling. That is the normal state
of a shared-mode Windows output and it is not a fault: see
[The audio stack](#the-audio-stack) for the two ways around it.

**If the rate is right and the DAC still says otherwise**, the limit is the USB link
rather than anything in software. A cable that negotiates full speed instead of high
speed caps many DACs at 16 bit / 96 kHz. Device Manager will show the device connected
to a full-speed hub; try another cable and a port directly on the machine before
suspecting the player, because the badge is reporting the truth about hardware that has
quietly downgraded itself.

**If accented characters or `«»` come out as garbage** in the messages printed before
the interface opens, the console is using a legacy code page. `PYTHONUTF8=1` fixes it
for every Python program, not just this one.

For anything else, `TIDALAMP_DEBUG=1` writes to
`%LOCALAPPDATA%\tidalamp\state\tidalamp.log`. The TUI owns the terminal, so logging
goes to a file.

## Platform support

tidalamp-win targets Windows. **Nothing in this table is tested yet**, because the port
is not finished; it says what is intended and what the known obstacles are.

| Platform                                | Status                                                                                                  |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Windows 11 + Windows Terminal           | The target. What development and CI will run against                                                    |
| Windows 10 21H2+ + Windows Terminal     | Intended to work; Windows Terminal installs from the Store                                              |
| Windows 10/11 + console host (conhost)   | Expected to work with blocks for cover art, and to look worse. Checked last, which is how it usually breaks |
| Windows on ARM                          | Untested. Python and mpv both have ARM64 builds, so it is plausible rather than planned                 |
| WSL2                                    | Use [the Linux version](https://github.com/wh01s17/tidalamp) inside it — that is what it is for. Audio needs separate setup either way |
| Linux, macOS, BSD                       | Not this project. Linux is [upstream](https://github.com/wh01s17/tidalamp); macOS and BSD are unsupported in both |

Cover art falls back to terminal blocks when no pixel protocol is available, which on
Windows is most of the time. Desktop media controls are not implemented — see
[Desktop integration](#desktop-integration).

## License

GPL-3.0-or-later. See [LICENSE](LICENSE) for the complete text.

The themed looks' pictures in `tidalamp/emblems/` are the exception: they are fan
tributes to the works each look nods to, those characters and designs belong to their
owners, and the GPL does not cover them. See
[`tidalamp/emblems/NOTICE`](tidalamp/emblems/NOTICE). If you hold rights over one and
want it gone, open an issue and it leaves in the next release.

## Disclaimer

tidalamp is an independent project. **It is not affiliated with, sponsored by, or
endorsed by TIDAL, Aspiro, Square, or the owners of the Winamp trademark.** Names are
used only descriptively to identify the service it communicates with and the
player its key defaults and equalizer bands come from.

- You need **your own TIDAL subscription**. tidalamp provides no access beyond what
  your account already has.
- tidalamp **does not circumvent technical protection measures**. Tracks with
  encrypted Widevine manifests are rejected with a message; no decryption is
  attempted. This boundary is deliberate, and patches that add decryption or download
  audio to files will not be accepted.
- tidalamp **does not download or redistribute music**. Audio is streamed. The only
  on-disk playback artifact is a temporary HLS playlist containing URLs, not audio.
- Authentication uses TIDAL's device authorization flow through `tidalapi`, not the
  developer API. Using an unofficial client may conflict with TIDAL's terms of
  service; users accept that decision and any risk to their account.

The themed looks evoke other works without naming them. Their pictures are fan
tributes, not affiliated with or endorsed by the owners of what they nod to.

The licence applies to this code. It is not, and cannot be, permission from TIDAL.
