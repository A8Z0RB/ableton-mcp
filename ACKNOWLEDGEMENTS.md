# Acknowledgements

This project is a merge of **three MIT-licensed open-source projects**. It is
released under the MIT License; the copyright of each contributed part remains
with its original author(s). Nothing here is used without attribution.

## Fork lineage

```
Siddharth Ahuja (MCPBlender/ableton-mcp, the original architecture, MIT)
└── aarx0/ableton-mcp            (maintained fork, MIT)     ← our base
    └── THIS PROJECT (music-arrangement-learner)
        + closestfriend/ableton-mcp          (VST/AU plugin toolset, MIT)
        + farmhutsoftwareteam/ableton-mcp-extended (device/mixing toolset, MIT)
```

## Base: [aarx0/ableton-mcp](https://github.com/aarx0/ableton-mcp)

Forked from this repository. We keep its entire codebase (remote script,
MCP server, telemetry modules, LOM.md, Dockerfile, pyproject.toml, uv.lock,
smithery.yaml, .python-version). Notable contributions kept as-is or
lightly modified:

| Component | Author |
|---|---|
| `execute_live_code` tool + `_execute_code` handler | aarx0 (upstream of it: MCPBlender/ableton-mcp) |
| `get_clip_notes` tool + handler | aarx0 |
| `set_track_mixer` tool + handler | aarx0 (we extended it with `track_kind`) |
| Telemetry opt-in design (`telemetry.py`, `telemetry_decorator.py`) | aarx0 / MCPBlender |
| `LOM.md` LLM reference | aarx0 |
| Original architecture: `AbletonMCP_Remote_Script/__init__.py` socket server, `MCP_Server/server.py` | Siddharth Ahuja (MCPBlender/ableton-mcp) |

## Contributed: [closestfriend/ableton-mcp](https://github.com/closestfriend/ableton-mcp)

The **VST/AU plugin toolset** was ported from this fork (its
`AbletonMCP_Remote_Script/__init__.py` "VST/AU Plugin Methods" section and its
`MCP_Server/server.py` "VST/AU Plugin Functions" section). Ported with minor
adaptations (attribution comments in code):

- `get_available_plugins` (remote `_get_available_plugins`) — enumerate
  `application.browser.plugins`, filtered by `vst` / `vst3` / `au` / `all`
- `load_vst_plugin` (remote `_load_vst_plugin`) — select track, find plugin by
  name in the browser plugin folders, `browser.load_item`
- `get_plugin_parameters` (remote `_get_plugin_parameters`) — read all
  parameters (index, name, value, min, max, is_quantized, is_enabled)
- `set_plugin_parameter` (remote `_set_plugin_parameter`) — set a parameter by
  index with min/max clamping

## Contributed: [farmhutsoftwareteam/ableton-mcp-extended](https://github.com/farmhutsoftwareteam/ableton-mcp-extended)

The **device & mixing toolset** was ported from this fork (its
`AbletonMCP_Remote_Script/__init__.py` "Extended tools" section and matching
`MCP_Server/server.py` tools). Ported with minor adaptations:

- `get_device_parameters` (remote `_get_device_parameters`) — full parameter
  list incl. `value_items` for quantized parameters, on regular/return/master
  tracks (`track_kind`)
- `set_device_parameter` (remote `_set_device_parameter`) — set by index with
  min/max clipping, returns requested vs applied value
- `delete_track` (remote `_delete_track`)
- `delete_device` (remote `_delete_device`)
- `set_track_send` (remote `_set_track_send`) — send levels to return tracks
- `get_device_routings` (remote `_get_device_routings`) — available input
  routing types/channels incl. `audio_inputs` sidechain introspection
- `set_device_routing` (remote `_set_device_routing`) — sidechain source
  selection by name matching, with `audio_inputs[].routing` fallback
- `_resolve_track` / `_resolve_device` helpers (track_kind resolution)

## Not included (on purpose)

- closestfriend's Arrangement View + Max for Live (`ArrangementHelper.maxpat`)
  work was **not** merged — it adds a Max for Live dependency. Arrangement
  basics already exist in the base fork.
- farmhut's audio import / arrangement / locator extras were **not** merged —
  the base fork already covers audio-clip import and arrangement operations.

## License

[MIT](LICENSE) — original copyright (c) 2025 Siddharth Ahuja; merged portions
copyright their respective authors (see above). All three source projects are
MIT licensed, so merging is permitted with attribution.
