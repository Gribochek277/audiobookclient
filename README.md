> Sanitized mirror of Forgejo `serhii/audiobookclient`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/audiobookclient` | Synced: 2026-10-07T00:57:20Z

---

# audiobook

Offline-first TUI client for a personal audiobook library that lives on a Synology NAS
(`silo`). The app downloads books into a local file cache, plays them with **mpv** (listen
anywhere, offline), and syncs "where am I" progress back to the NAS over the home LAN
(**SMB via rclone**, FTP fallback).

```
▶ CONTINUE  Ray Bradbury - Fahrenheit 451   12:34:56 / 21:00:00  ███████░░░ 62%  [pop-os-1]
┌ Library (14 books, 3 finished) ─────────────── / search   f filter   s sync   q quit
│ ▸ Ray Bradbury - Fahrenheit 451        62%   12:34:56   pop-os-1
│   Stanisław Lem - Solaris             100%   ✓ done
```

## Requirements

| Tool | Used for |
|---|---|
| `rclone` | transport to the NAS (SMB share / FTP) |
| `mpv` | audio playback (JSON IPC) |
| `ffmpeg` | audio visualizer: per-chapter PCM tap (optional; a synthetic animation is the fallback) |
| `ffprobe` | chapter/book durations (optional, lazy) |

## Keys

| Screen | Key | Action |
|---|---|---|
| library | `j`/`k`, `↑`/`↓` | select |
| library | `enter` | open the selected book |
| library | `/` | search (type the query, `enter` keeps the filter, `esc` clears it) |
| library | `f` | filter: all / active / done |
| library | `o` | sort: recent / A–Z / progress |
| library | `s` | sync with the NAS (background; conflict dialog if needed) |
| library | `g` / `G` | first / last |
| library | `q` | quit (pushes unsynced progress if any) |
| library | left-click | select the clicked book; a second click on the same row within 400 ms opens it (Enter) |
| library | wheel up/down | move the selection one row, clamped at the ends |
| book | `space` | play / pause |
| book | `,` / `.` | seek −10 s / +10 s |
| book | `n` / `p` | next / previous chapter |
| book | `d` | restart the whole-book download (re-checks what is cached) |
| book | `q` | save progress and go back |
| book | `ctrl-c` | save and quit the TUI |

When a book opens, its chapters download to the local cache in the background
(starting with the one being played, then the rest in playback order) — the
progress line shows it (`⬇ 3/26 · chapter 4 — 12.4 MB` → `⬇ cached (26/26)`).
Playback waits for the current chapter only; already-cached chapters switch
instantly. The book screen shows position / total / remaining; the total is an
estimate (`~`) until every chapter's length has been measured. The library
marks books that are fully or partly cached with `⬇`.

The cache has a size budget (`[cache] limit_gb`, default **10**, `0` = no limit).
When a whole book has finished downloading — and at startup — the cache is
trimmed back to the budget: **finished** books go first, then the least recently
played ones. The book currently open is never dropped, and a book is only
evicted when the cache is actually over the limit. `audiobook cache status`
shows usage against the limit.

## Setup

```sh
audiobook setup      # writes ~/.config/audiobook/config.toml from a template
# edit it: silo host / user / password / share / library path
audiobook doctor     # checks binaries, config, NAS reachability
```

Config example:

```toml
[silo]
host = "192.168.50.90"
user = "your-user"
password = "your-password"
protocol = "smb"        # or "ftp"
share = "homes"         # SMB share name (ignored for ftp)
path = "Audiobooks"     # library root inside the share / from the FTP root
```

`audiobook` also generates a private `~/.config/audiobook/rclone.conf` (mode 0600)
from the `[silo]` section — you never edit rclone config by hand.

Synology NAS note: for `protocol = "ftp"` the generated rclone section adds
`disable_mlsd = true` automatically — Synology's FTP answers MLSD with a malformed
header that rclone rejects. Also avoid `[` `]` in book folder names: the stock
vsftpd treats `LIST <path>` as a glob, so folders like `… [narrator]` list as empty
(the one-shot migration renames them into the canonical layout). Downloads run
`--multi-thread-streams 1`: the stock vsftpd drops the control connection when rclone
opens several data channels for one file ("broken pipe" mid-transfer).

mpv note (verified on this machine's mpv 0.41.0): there is no `waveform`
property in mpv's JSON-IPC (not in this build, not in the upstream source)
— the audio visualizer decodes cached chapters with ffmpeg into a PCM ring
buffer instead (see `src/viz.rs`, issue #4). `end-file`/`idle` events are
also unreliable here; chapter ends are detected from a frozen `time-pos`
(see `src/playback.rs`).

## Library layout (auto-detected)

- a single audio file → one-file book;
- a directory with audio files → chaptered book (chapters ordered by natural sort of file names).

Canonical target layout (the one-shot migration normalizes existing files into it):

```
Audiobooks/
  Ray Bradbury - Fahrenheit 451/
    01 - Part One.mp3
    02 - Part Two.mp3
  Some Author - One File Book.m4a
```

Supported audio extensions: mp3, m4a, aac, flac, ogg, opus, wav, wma.

## Progress & sync

State lives in **one file** — `audiobook-library.json` in the library root on the NAS
(single source of truth) — plus a local SQLite copy per device:

```json
{ "version": 1, "updated_by": "pop-os-1", "updated_at": "2026-…Z",
  "books": { "Ray Bradbury - Fahrenheit 451/": {
    "position_sec": 4567, "chapter_idx": 3, "total_sec": 75000,
    "finished": false, "last_played_at": "2026-…Z", "client": "pop-os-1" } } }
```

Sync rules (per book): local newer → push; remote newer → pull; both changed since the
last sync with different values → **conflict** (device vs. server, both positions and
timestamps shown). The NAS file is written atomically (temp file + server-side rename).

In the TUI, sync runs on a background worker (the UI never freezes):

- **startup** — automatic; offline → the status line says so, the library stays usable;
- **`s`** — manual sync from the library screen (always re-lists the NAS);
- **conflict** — an in-TUI dialog (`[d]` keep device / `[s]` keep server);
- **exit** — if the local store has unsynced records, one last push runs after the
  TUI closes (conflicts there keep the device's progress, the outcome is printed).

Listing the whole library costs a full recursive `lsjson -R` (~2 s on the real
NAS), so the listing is cached locally (`~/.local/share/audiobook/nas-manifest.json`)
and reused for up to 6 h: automatic syncs (startup, exit) and opening a book that
is missing from the local store then skip the listing entirely. An explicit sync
(`s`) and the CLI always re-list; anything older than the TTL refreshes itself.
A cached listing is never treated as proof that a book was deleted on the NAS —
the status line says `· library list cached (s to refresh)` when one was used.

The CLI (`audiobook sync`) does the same round with an interactive terminal prompt.

## Commands

| Command | Milestone |
|---|---|
| `audiobook` (TUI) | M6 |
| `audiobook doctor` | ✅ M0 |
| `audiobook setup` | ✅ M0 |
| `audiobook list [--json]` | M2 |
| `audiobook scan` (dry-run layout report) | M2 |
| `audiobook cache status\|clean` | M3 |
| `audiobook sync` | M4 |

## Development

```sh
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

The transport sits behind a trait; `LocalDirTransport` (a temp dir acting as "silo")
lets the entire core run in tests without a NAS.

Milestones: M0 bootstrap → M1 core (layout/state/transport-mock + tests) →
M2 rclone SMB transport + list/scan → M3 cache/download → M4 sync engine + conflicts →
M5 mpv player module → M6 TUI library → M7 TUI book/playback → M8 TUI sync flows →
M9 one-shot silo migration + E2E polish. (v2: phone client, statistics, imports.)

## Security notes

- The NAS is reachable **only from the home LAN** by design (no tailscale, no exposure).
- Credentials live in `~/.config/audiobook/{config.toml,rclone.conf}`, mode 0600.
