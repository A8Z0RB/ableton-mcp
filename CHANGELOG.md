# Changelog

All notable changes to this fork. Versioning continues from upstream
[ahujasid/ableton-mcp](https://github.com/ahujasid/ableton-mcp) 1.2.0.

## 1.3.0 — 2026-07-27

First release of the maintained fork.

### Added
- **`execute_live_code`**: run a Python script against the Live Object Model
  in one call. Syntax-validated server-side before sending; executed on
  Live's main thread inside a single undo step (Ctrl+Z reverts the whole
  script); returns a `result` value (JSON or repr) plus captured stdout,
  both capped at 50k chars; script errors return the full traceback as data.
- **`LOM.md`**: curated Live Object Model reference (signatures, value
  ranges, argument-order traps, known API limitations) targeted at LLM
  script authors.
- **`get_clip_notes`**: read back a MIDI clip's notes (pitch, start, duration,
  velocity, mute; probability and velocity deviation on Live 11+) via
  `get_notes_extended`, with a legacy `get_notes` fallback.
- **`set_track_mixer`**: set track volume, pan, mute, and/or solo; only the
  parameters passed are changed; returns the resulting mixer state.

### Changed
- **Telemetry is now opt-in** (was opt-out). Nothing is sent unless
  `ABLETON_MCP_ENABLE_TELEMETRY=true` is set; upstream disable variables are
  still honored.
- `load_drum_kit` resolves the kit path **before** loading the drum rack, so
  a bad `kit_path` no longer leaves a bare rack on the track; docstring now
  documents the `query:` URI form the browser actually resolves and points
  .adg kit files at `load_instrument_or_effect`.
- `load_instrument_or_effect` reports the devices actually added
  (`new_devices`, diffed before/after in the remote script) instead of an
  empty list on success.

### Upstream (unchanged in this fork)
- 1.2.0 and earlier: see the upstream repository's history.
