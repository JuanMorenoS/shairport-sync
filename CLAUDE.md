# Shairport Sync — Claude working notes

AirPlay / AirPlay 2 audio receiver in C (C11 + a little C++ for Apple ALAC / FFTConvolver).
Upstream: https://github.com/mikebrady/shairport-sync (maintainer: Mike Brady).
Local dev machine: macOS (Apple Silicon, Homebrew).

## Contribution workflow (upstream rules)
- Active work happens on **`development`**, not `master`. Branch from `upstream/development`, open PRs against `development`.
- Remotes: `origin` = my fork (JuanMorenoS/shairport-sync), `upstream` = mikebrady, `mman` = Martin Man (author of #2312 fix branch `fix-ptp-anchor-64bit`).
- Local Claude setup lives on fork branch `claude-setup` (never PR it upstream). Working-tree copies are git-excluded via `.git/info/exclude`.
- One bug per branch/PR (`fix/<issue#>-short-name`). Small, surgical diffs — maintainer prefers minimal changes.
- **Never** include `CLAUDE.md`, `.claude/`, or reformatting of untouched code in a PR (these files are git-excluded locally).
- Reference the GitHub issue in the commit/PR (`Fixes #1234`). Explain root cause, not just symptom.
- `RELEASENOTES.md` is maintained by the maintainer — don't edit it.

## Build (macOS)
One-time deps:
```
brew install autoconf automake libtool pkg-config popt libconfig openssl ffmpeg libsoxr
```
Classic AirPlay build (same as CI `check_classic_mac_basic.yml`):
```
autoreconf -fi
./configure --with-os=darwin --with-coreaudio --with-ao --with-stdout --with-dns_sd --with-ssl=openssl
make -j"$(sysctl -n hw.ncpu)"
./shairport-sync -V   # prints version + compiled-in features
```
- Re-run `autoreconf -fi && ./configure ...` only when `configure.ac` / `Makefile.am` change; otherwise just `make`.
- AirPlay 2 (`--with-airplay-2`) needs NQPTP + Linux-ish deps (libplist, libsodium, libgcrypt, uuid, pipewire/alsa); build it in Docker (`docker/`, `tests/Dockerfile.testrig`) rather than natively on macOS.
- Linux-only backends (ALSA, PipeWire, PulseAudio, systemd, avahi, D-Bus/MPRIS) can't be built on macOS → verify those via Docker or CI.
- Treat new compiler warnings in touched files as errors to fix.

## Testing
- No unit-test suite. Verification = (1) clean build, (2) `./shairport-sync -V`, (3) run it and play from an Apple device:
  `./shairport-sync -c /dev/null -o stdout -vvv > /dev/null` or `-o coreaudio -vvv`.
- `tests/configure_test.sh` checks configure/build feature combos (Linux only, run in Docker).
- Memory bugs: build with `CFLAGS="-g -O1 -fsanitize=address,undefined"` and reproduce. On Linux, `valgrind --suppressions=valgrind-suppressions.txt`.
- When fixing a bug, first write down a concrete repro (config + steps + log at `-vvv`), then confirm it is gone after the fix.

## Architecture map
- `shairport.c` — `main`, CLI/config parsing (libconfig), global `config` struct (defined in `common.h`).
- `rtsp.c` — RTSP server: connection threads, session setup, AirPlay 2 pairing/`SETUP`/`SETRATEANCHORTIME`, metadata. Biggest & most bug-prone.
- `player.c` / `player.h` — playback engine: packet buffer, sync/timing, stuffing/resampling, volume. `rtsp_conn_info` (player.h) is the per-connection state struct.
- `rtp.c` — classic-AirPlay RTP audio/control/timing sockets; AP2 realtime + buffered streams.
- `ap2_buffered_audio_processor.c`, `ap2_event_receiver.c` — AirPlay 2 buffered audio / event channel.
- `audio.c` + `audio_*.c` — output backend abstraction (`audio_output` struct of function pointers), one file per backend.
- `mdns*.c`, `tinysvcmdns.c` — Bonjour advertising backends.
- `dacp.c` — remote control back to the source (DACP).
- `metadata`/`mqtt.c`/`dbus/` — metadata pipe, MQTT, D-Bus & MPRIS interfaces.
- `pair_ap/` — vendored HomeKit pairing (avoid editing; upstream-owned).
- `utilities/` — helpers; logging lives in `utilities/debug.h`.
- `FFTConvolver/`, `tinyhttp/`, `alac.c` — vendored code; avoid touching.

## Code conventions
- 2-space indent, ~100-column lines, braces on same line (clang-format LLVM-ish). Match surrounding code; don't reformat whole files.
- Logging: `debug(level, fmt, ...)` (1–3, shown with `-v`…`-vvv`), `inform()`, `warn()`, `die()` from `utilities/debug.h`. New debug output at level 2/3 unless genuinely important.
- Heavy pthreads use: every thread has cleanup handlers (`pthread_cleanup_push/pop`); thread cancellation is how connections are torn down. When adding allocations or locks inside a thread, add matching cleanup handlers and respect lock ordering. Prefer existing helpers (`pthread_mutex_lock_and_cleanup_push`, `get_play_lock`/`release_play_lock`, `debug_mutex_lock`) over bare locks.
- Feature code is guarded by `#ifdef CONFIG_AIRPLAY_2`, `CONFIG_METADATA`, `CONFIG_DBUS_INTERFACE`, `CONFIG_MQTT`, etc. A fix must compile with the feature **on and off**.
- Time is mostly `uint64_t` nanoseconds (`get_absolute_time_in_ns()`); RTP timestamps are 32-bit and wrap — use `modulo_32_offset()` (player.h) / `a_minus_b_mod23()` (utilities/mod23.h), never naive subtraction/comparison.
- Network input (RTSP headers, plists, RTP packets) is untrusted: bounds-check lengths, check every `malloc`/`strdup`.

## Known repo quirk
- `.github/workflows/CodeQL.yml` and `codeql.yml` differ only by case. On macOS's case-insensitive FS one overwrites the other, so `git status` always shows one as modified. Ignore it; never commit it.
