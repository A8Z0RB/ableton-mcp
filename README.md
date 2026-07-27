# AbletonMCP — Ableton Live Model Context Protocol Server

Control Ableton Live from Claude (or any MCP client): build tracks, write and
**read back** MIDI, load instruments, mix, arrange, and run whole Python
scripts against the Live API in a single call.

This is an actively maintained fork of
[ahujasid/ableton-mcp](https://github.com/ahujasid/ableton-mcp) (which is no
longer maintained). Full credit to [Siddharth Ahuja](https://x.com/sidahuj)
for the original architecture. What this fork adds is driven by daily use in
real music-production sessions — see the [CHANGELOG](CHANGELOG.md).

**Highlights over upstream:**

- **`execute_live_code`** — run a bounded Python script against the Live
  Object Model in one call, instead of paying a round trip per operation.
  Syntax-validated before it's sent, wrapped in a single Live undo step
  (one Ctrl+Z reverts everything), returns `result`/stdout/tracebacks as data.
  Ships with a curated [LOM reference](LOM.md) written for LLM consumption.
- **`get_clip_notes`** — read a clip's MIDI notes back (pitch, timing,
  velocity, mute). Upstream could write notes but never verify them.
- **`set_track_mixer`** — set volume, pan, mute, solo per track.
- **Telemetry is opt-in** (off by default). Upstream phones home on startup;
  this fork never sends anything unless you set `ABLETON_MCP_ENABLE_TELEMETRY=true`.
- Bug fixes: `load_drum_kit` no longer half-mutates your track on failure,
  device loads report the devices actually added, docstrings document URI
  formats that actually resolve.

## How it works

```
MCP client (Claude Desktop / Claude Code / Cursor)
   │  MCP over stdio
   ▼
MCP_Server/server.py          ← this package, run via uvx
   │  JSON over TCP localhost:9877
   ▼
AbletonMCP_Remote_Script/     ← Python remote script running inside Live
   │  Live Object Model
   ▼
Ableton Live
```

Two components must be installed: the **remote script** (inside Live, once)
and the **MCP server** (registered with your MCP client).

## Installation

### Prerequisites

- Ableton Live 10 or newer (Live 12.0.5+ for audio-clip import)
- [uv](https://docs.astral.sh/uv/getting-started/installation/)

### 1. Install the Ableton Remote Script

1. Copy the `AbletonMCP_Remote_Script` folder into Ableton's
   `MIDI Remote Scripts` directory, renamed to `AbletonMCP` (so you end up
   with `.../MIDI Remote Scripts/AbletonMCP/__init__.py`). Common locations:

   **macOS:**
   - Applications → right-click Ableton Live → Show Package Contents →
     `Contents/App-Resources/MIDI Remote Scripts/`
   - or `~/Library/Preferences/Ableton/Live XX/User Remote Scripts/`

   **Windows:**
   - `C:\ProgramData\Ableton\Live XX\Resources\MIDI Remote Scripts\`
   - or `C:\Users\[You]\AppData\Roaming\Ableton\Live x.x.x\Preferences\User Remote Scripts\`
   - or `C:\Program Files\Ableton\Live XX\Resources\MIDI Remote Scripts\`

2. Launch Live → Settings/Preferences → **Link, Tempo & MIDI**
3. Control Surface dropdown → select **AbletonMCP**; set Input and Output to
   **None**. Live's status bar should show
   `AbletonMCP: Listening for commands on port 9877`.

### 2. Register the MCP server

Straight from GitHub (recommended — always current):

```json
{
    "mcpServers": {
        "AbletonMCP": {
            "command": "uvx",
            "args": ["--from", "git+https://github.com/aarx0/ableton-mcp", "ableton-mcp"]
        }
    }
}
```

- **Claude Desktop:** Settings → Developer → Edit Config → `claude_desktop_config.json`
- **Claude Code:** `claude mcp add AbletonMCP -- uvx --from git+https://github.com/aarx0/ableton-mcp ableton-mcp`
- **Cursor:** Settings → MCP → paste the same `uvx` command

From a local clone (for development): replace the `--from` value with the
clone's path.

> Run only one MCP client against Live at a time per session; multiple
> clients work but share the same Live instance.

## Tools

| Tool | What it does |
|---|---|
| `get_session_info` | Tempo, signature, track count, transport state |
| `get_track_info` | Track details: clips, devices, mixer values |
| `get_clip_notes` | **Read back** a clip's MIDI notes |
| `create_midi_track`, `set_track_name` | Track management |
| `create_clip`, `add_notes_to_clip`, `set_clip_name` | MIDI clip authoring |
| `create_audio_clip` | Import an audio file into a clip slot (Live ≥ 12.0.5) |
| `set_track_mixer` | Volume / pan / mute / solo |
| `set_tempo` | Session tempo |
| `fire_clip`, `stop_clip`, `start_playback`, `stop_playback` | Transport |
| `get_browser_tree`, `get_browser_items_at_path` | Browse Live's library |
| `load_instrument_or_effect`, `load_drum_kit` | Load devices by browser URI |
| `switch_to_arrangement_view`, `set_arrangement_time`, `get_arrangement_clips`, `duplicate_to_arrangement` | Arrangement view |
| `execute_live_code` | Run a Python script against the Live API in one call |

### `execute_live_code` in 30 seconds

```python
# One call builds a four-on-the-floor kick pattern and reads it back:
track = song.tracks[0]
clip = track.clip_slots[0].clip
clip.set_notes(tuple((36, float(beat), 0.25, 100, False) for beat in range(4)))
result = [(n.pitch, n.start_time) for n in clip.get_notes_extended(0, 128, 0.0, 4.0)]
```

Scripts run on Live's main thread inside one undo step. Read
[LOM.md](LOM.md) before writing nontrivial scripts — it documents the exact
signatures, the argument-order traps, and what the Live API cannot do.

## Example prompts

- "Create an 80s synthwave track"
- "Build a 4-bar drum loop, then read the notes back and check the timing"
- "Load a 909 kit on track 1 and write a house pattern"
- "Balance the mix: drums at 0 dB, bass slightly down, pan the hats right"
- "Copy the session clips into an arrangement with intro, drop, and outro"

## Troubleshooting

- **"Could not connect to Ableton"** — Live isn't running, or the remote
  script isn't enabled as a Control Surface. Check for the port message in
  Live's status bar.
- **Timeouts on state-changing commands** — Live's main thread is busy (or a
  modal dialog is open). Dismiss dialogs; break big operations into smaller ones.
- **Changed the remote script?** — Live only loads remote scripts at startup;
  restart Live.
- **Still stuck** — restart both Live and your MCP client, in that order.

## Telemetry

**Off by default.** Nothing is ever sent unless you explicitly set
`ABLETON_MCP_ENABLE_TELEMETRY=true` in the server's environment. (The
upstream project's endpoint receives that data, not this fork's maintainers.)

## Contributing

Issues and PRs welcome — especially reports from real production sessions.
This fork's development is itself largely done by LLMs working against real
Ableton sessions, with every change reviewed and dogfooded before merge.

## License & credit

MIT. Original project by [Siddharth Ahuja](https://github.com/ahujasid);
fork maintained at [aarx0/ableton-mcp](https://github.com/aarx0/ableton-mcp).

This is a third-party integration, not made or endorsed by Ableton.
