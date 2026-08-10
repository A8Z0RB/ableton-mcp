# Changelog

All notable changes to this fork. Versioning continues from upstream
[ahujasid/ableton-mcp](https://github.com/ahujasid/ableton-mcp) 1.2.0.

## 1.4.0 — merge of closestfriend plugin tools + farmhut mixing tools

Merged from two additional MIT-licensed forks (see
[ACKNOWLEDGEMENTS.md](ACKNOWLEDGEMENTS.md) for full attribution).

### Added (from closestfriend/ableton-mcp — VST/AU plugin toolset)
- **`get_available_plugins`**: enumerate plugins from `application.browser.plugins`,
  filterable by `vst` / `vst3` / `au` / `all`.
- **`load_vst_plugin`**: select a track, find a plugin by name (type auto-detect
  or forced), load it via `browser.load_item`.
- **`get_plugin_parameters`**: read every parameter of a plugin device
  (index, name, value, min, max, is_quantized, is_enabled).
- **`set_plugin_parameter`**: set a plugin parameter by index with min/max clamping.

### Added (from farmhutsoftwareteam/ableton-mcp-extended — device & mixing toolset)
- **`get_device_parameters`**: full parameter list (incl. `value_items` for
  quantized params) on `regular` / `return` / `master` tracks.
- **`set_device_parameter`**: set a device parameter by index, clipped to
  min..max; returns requested vs applied value.
- **`delete_track`** / **`delete_device`**: remove tracks and devices.
- **`set_track_send`**: send levels to return tracks.
- **`get_device_routings`** / **`set_device_routing`**: inspect and set input
  routings by name (e.g. sidechain source selection), incl. `audio_inputs`
  fallback.
- `_resolve_track` / `_resolve_device` helpers with `track_kind` resolution.

### Changed
- **`set_track_mixer`** now accepts `track_kind` (`regular` / `return` /
  `master`), letting the AI mix on return and master tracks.
- New modifying commands registered in the server's socket timeout logic.

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
