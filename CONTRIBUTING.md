# Contributing

> [!IMPORTANT]
> **This is a port in progress.** tidalamp-win is a fork of
> [tidalamp](https://github.com/wh01s17/tidalamp), a Linux TIDAL client, being moved to
> Windows. At the time of writing the tree is still Linux code: mpv is driven over a
> Unix socket, desktop integration is D-Bus, and the audio stack talks to PipeWire.
> None of it runs on Windows yet. [`windows.md`](windows.md) is the migration plan —
> read it before touching anything.

## First, two lines that are not crossed

1. **No DRM.** `stream.py` rejects encrypted manifests rather than decrypting them, and
   no audio is downloaded to disk. Patches that add decryption or download to a file are
   not accepted. This is not an aesthetic preference: it is what keeps the project clear
   of anti-circumvention law.
2. **No embedded credentials.** No API keys, no tokens, no client secrets of our own in
   the repository.

## The rule that governs this fork

**Change the body of a function, never its contract.** Module names, class names and
public signatures stay as upstream has them, even where the implementation underneath is
replaced wholesale. `player.Mpv` keeps its name and its methods although it speaks to a
named pipe instead of a Unix socket; `mpris.MprisService` keeps `start()`, `stop()`,
`publish()` and `publish_tracks()` although it publishes nothing.

The reason is concrete: upstream keeps moving, and with the signatures intact a fix to
`library.py`, `queue.py` or `screens/` can still be cherry-picked here without
conflicts. Rename the modules and that channel closes permanently.

Where a function has no Windows equivalent, it keeps its signature and returns the
neutral value (`""`, `None`, `False`, an empty list), with a docstring saying why. The
callers already handle that case — the whole codebase is written so that a missing
cava, a missing cover or a missing MPRIS never stops the music.

This is spelled out at length in [`windows.md`](windows.md) §1, along with the list of
the ~13,000 lines that need no changes at all and should not be opened.

## Setting up

You need **Python 3.11 or newer** and **mpv**. Windows Terminal is strongly recommended
over the legacy console host: the TUI works in both, but colours, resizing and repaint
speed differ, and conhost is where things break.

```powershell
winget install mpv            # verify the package id with: winget search mpv
python -m venv .venv
.venv\Scripts\pip install -e ".[dev]"
.venv\Scripts\python -m pytest
```

`cava` is not available on Windows and is not expected to be: the spectrum falls back
to the built-in RMS meter, by design. There is no `dbus-daemon` either, and the MPRIS
integration test that needed one does not apply here.

**Long paths.** Windows caps paths at 260 characters by default, and pytest's temporary
directories plus the generated `.m3u8` names can reach it. If tests fail with
`FileNotFoundError` that make no sense, enable long paths:

```
HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1
```

## What has to pass before a commit

```powershell
.venv\Scripts\ruff format .
.venv\Scripts\ruff check .
.venv\Scripts\mypy
.venv\Scripts\python -m pytest -q --cov
```

All four run in CI. Coverage has a floor of 70%, which is a floor and not a target: it
exists so that a change which empties the tests fails instead of passing quietly.

**mypy needs `platform = "win32"`** in its configuration. Without it, a check run on a
Linux machine or a Linux CI runner validates branches that do not exist here and misses
the ones that do.

## How the tests are written here

They do not touch the network, TIDAL, or any system service. There are doubles for
everything needed: `tests/fake_mpv.py` speaks the real mpv JSON IPC, including the
asynchronous events that arrive interleaved with replies, and has three environment
variables for making it misbehave on purpose (`FAKE_MPV_HANG_ON`, `FAKE_MPV_EOF_ON`,
`FAKE_MPV_SPLIT`). That is what covers the recovery from a hung mpv, which is a
deliberate feature and not an accident.

Two things about tests on Windows that are easy to get wrong:

- **`chmod` is an illusion.** A test that calls `chmod(0o500)` to simulate an unwritable
  directory **passes without testing anything** here. There are two such tests inherited
  from upstream; see `windows.md` §5.13.
- **The transport is injected.** `player.Mpv` takes its transport rather than opening a
  socket itself, so tests can speak TCP over `127.0.0.1` while production uses a named
  pipe. That keeps `fake_mpv.py` simple, and it leaves the pipe transport without
  automatic coverage — which is why there is an integration test that launches a real
  `mpv.exe` and skips when it is not installed. Do not delete it.

When you fix a bug, the test should say **what was breaking**, not only what the
function does. Plenty of the ones in this repository carry the symptom in the name or in
the docstring, and that is deliberate.

## Documentation

The split is by audience, not by preference: what a stranger reads is in English, and
what the maintainer reads is in Spanish.

- `README.md` and `CHANGELOG.md` are the public documentation, in English. So are the
  GitHub release notes.
- `windows.md` is the migration plan, in Spanish: what ties the project to Linux, in
  what order to untie it, and how each step is checked. **While the port is unfinished
  this is the most important file in the repository.** Its §4 is the phase list and its
  §7 the traps that cost an afternoon each.
- `plan.md` is the handover document, in Spanish: what exists, what has been verified
  and **the reasoning behind each decision**. It is inherited from upstream and, outside
  its §0, it describes the *Linux* codebase — which is what is still in this tree. If
  you make an architectural decision, it goes there, with the why.
- `next.md` is the queue of committed work, in Spanish: the open decisions, what has to
  be checked by hand, and what was discarded with the reason. It does not duplicate the
  phases of `windows.md`.
- `publish.md` is the release procedure, in Spanish. `packaging/README.md` is its
  summary plus the packaging-specific decisions, also in Spanish: it is a companion to
  `publish.md`, not a door a stranger comes in through.
- This file is in English. Whoever reads it is deciding whether to contribute, and that
  is a stranger by definition.
- `CHANGELOG.md` is updated in the same commit as the change.

**One more, and it does not travel through git.** `.agents/skills/` is gitignored but
sits in the working tree, and an agent working here will read it and believe it. Two of
those skills — `tidalamp-mpv-ipc` and `tidalamp-mpris-contract` — describe the Unix
socket and the D-Bus contract as the project's architecture. Rewrite them alongside the
code they describe, or delete them. A skill documenting the previous architecture is
worse than no skill.

## Commit messages

Imperative, and explaining **why**, not only what. If the change comes out of a
measurement, the number goes in the message: that is what lets it be argued with later.
