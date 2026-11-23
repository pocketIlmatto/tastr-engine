# Changelog

Each release's notes are its section here: the release workflow refuses a version without one
([docs/ci.md](docs/ci.md#release)). Add the section in the pull request that bumps `version` in `Cargo.toml`.

## 0.6.1

- The beat grid stays on one phase through a steady track. The beat tracker could drift onto the off-beat for a
  stretch and back, which put DJ cues half a beat early for that stretch; now a nearly steady track gets one even
  grid on the phase most of its beats share. Core stage 0.8.0.
- `DjOptions::with_grid(GridMode::Constant)` keeps one tempo and phase for the whole track, like a DJ program's
  normal analysis; `GridMode::Auto` stays the default. The `dj` command takes `--constant`.
- `DjResult` carries the track's energy (0 to 1) and its vibe tags, and `tastr-engine dj` prints both.
- Releases also carry the engine as a library (`libtastr_engine` for Linux x86-64 and ARM, macOS on Apple silicon)
  and its WebAssembly build.

## 0.6.0

- First release on crates.io, with the `tastr-engine` program for macOS (Apple silicon) and Linux (x86-64, ARM).
