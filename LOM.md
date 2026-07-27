# Live Object Model (LOM) cheat sheet for `execute_live_code`

A curated reference for writing scripts against the objects in scope
(`song`, `application`, `control_surface`). This is the *remote-script* API
(the same one Ableton's own controller scripts use), not Max for Live's
`live.object`. It is undocumented by Ableton; everything here is verified
in practice or lifted from working remote-script code. When unsure,
introspect first:

```python
result = [a for a in dir(song.tracks[0]) if not a.startswith('_')]
```

## Ground rules

- Scripts run on **Live's main thread**. The UI is frozen while they run and
  a runaway loop cannot be killed. Keep everything bounded; never sleep.
- Scripts run inside **one undo step** — Ctrl+Z in Live reverts the whole call.
- Assign to `result`; `return` is a SyntaxError (module-level exec).
- Most numeric properties are plain floats/ints; setting an out-of-range value
  raises. Collections (`song.tracks`, `track.devices`, …) are Live vectors —
  index and iterate like lists, but they are live views, not copies.

## Song (`song`)

```python
song.tempo                      # float BPM, settable
song.signature_numerator        # int, settable
song.signature_denominator      # int, settable
song.is_playing                 # bool, settable (True starts playback)
song.current_song_time          # float beats, settable (playhead)
song.song_length                # float beats, read-only
song.loop, song.loop_start, song.loop_length   # arrangement loop, settable
song.metronome                  # bool, settable
song.tracks                     # regular tracks (index 0..n-1)
song.return_tracks              # return/send tracks
song.master_track
song.scenes                     # scene list; song.scenes[i].fire()
song.view.selected_track        # settable to a track object
song.view.detail_clip           # clip shown in Detail view, settable
song.create_midi_track(index)   # -1 = end; returns None — re-read song.tracks
song.create_audio_track(index)
song.delete_track(index)
song.duplicate_track(index)
song.begin_undo_step() / song.end_undo_step()   # already wrapped for you
song.undo() / song.redo()
```

## Track (`t = song.tracks[i]`)

```python
t.name                          # str, settable
t.has_midi_input                # True → MIDI track
t.has_audio_input               # True → audio track
t.mute, t.solo, t.arm           # bools, settable (arm only if track can be armed)
t.is_foldable                   # group track?
t.devices                       # device chain, left to right
t.clip_slots                    # session slots, one per scene
t.arrangement_clips             # clips in the Arrangement timeline
t.stop_all_clips()
t.duplicate_clip_to_arrangement(clip, destination_time_beats)  # Live 11+
t.mixer_device.volume.value     # 0.0..1.0 — 0.85 is 0 dB (Live's default)
t.mixer_device.panning.value    # -1.0..1.0
t.mixer_device.sends            # list of DeviceParameter, one per return track
t.mixer_device.sends[0].value   # 0.0..1.0
```

## Clip slot / Clip (`slot = t.clip_slots[j]`)

```python
slot.has_clip                   # ALWAYS check before touching slot.clip
slot.clip                       # Clip or raises if empty
slot.create_clip(length_beats)  # MIDI tracks only, slot must be empty
slot.delete_clip()
slot.fire()                     # launch (quantized like the UI)
slot.stop()
slot.create_audio_clip(path)    # Live >= 12.0.5, audio tracks only

c = slot.clip
c.name                          # str, settable
c.length                        # float beats
c.is_midi_clip / c.is_audio_clip
c.is_playing, c.is_recording
c.looping                       # bool, settable
c.loop_start, c.loop_end        # beats, settable
c.signature_numerator / _denominator
c.fire() / c.stop()
c.color                         # int 0x00RRGGBB, settable
```

### MIDI notes (Live 11+ extended API — prefer this)

```python
notes = c.get_notes_extended(from_pitch, pitch_span, from_time, time_span)
# e.g. everything: c.get_notes_extended(0, 128, 0.0, c.length)
# -> vector of MidiNote: .pitch .start_time .duration .velocity .mute
#    .note_id .probability .velocity_deviation .release_velocity

c.remove_notes_extended(from_pitch, pitch_span, from_time, time_span)

c.add_new_notes(specs)          # takes an iterable of Live.Clip.MidiNoteSpecification
# Building specs needs the Live module:
import Live
spec = Live.Clip.MidiNoteSpecification(
    pitch=60, start_time=0.0, duration=0.5, velocity=100, mute=False)
c.add_new_notes((spec,))

# Legacy API (works everywhere, simpler for writes):
c.set_notes(((pitch, start, dur, vel, mute), ...))   # REPLACES notes in region
c.get_notes(from_time, from_pitch, time_span, pitch_span)  # note the arg order!
```

Gotchas:
- `get_notes` arg order is `(time, pitch, time_span, pitch_span)` but
  `get_notes_extended` is `(pitch, pitch_span, time, time_span)`. Easy to swap.
- To *edit* existing notes: fetch via `get_notes_extended`, mutate the
  MidiNote objects' attributes, then `c.apply_note_modifications(notes)`.
- Drum racks: pitch = pad (C1/36 = kick by convention in Live kits).

## Devices & parameters (`d = t.devices[k]`)

```python
d.name                          # display name ("909 Core Kit")
d.class_name                    # API class ("DrumGroupDevice", "InstrumentGroupDevice",
                                #             "PluginDevice", "OriginalSimpler", ...)
d.parameters                    # list of DeviceParameter
p = d.parameters[0]             # parameters[0] is always "Device On"
p.name, p.value, p.min, p.max   # value settable, clamp to [min, max]
p.is_quantized                  # stepped (enum-like) parameter
str(p)                          # display string with units
d.can_have_drum_pads            # drum rack?
d.drum_pads                     # 128 pads; pad.note, pad.name, pad.chains
```

There is no API to *delete* a device on older Lives; Live 11+:
`t.delete_device(index)`.

## Browser (`application.browser`)

```python
b = application.browser
b.instruments / b.drums / b.sounds / b.audio_effects / b.midi_effects
b.packs / b.user_library / b.plugins   # root folders; each has .children
item.name, item.uri, item.is_loadable, item.is_folder, item.children
b.load_item(item)               # loads onto the SELECTED track — set
                                # song.view.selected_track first!
```

URIs look like `query:Drums#Drum%20Rack` or `query:Drums#FileId_6079`.
Plain paths ("Drums/Drum Rack") are not URIs and won't resolve. Walking
`.children` of large folders is slow — it lazily hits the disk; keep it out of
hot loops.

## Application (`application`)

```python
application.get_major_version()          # e.g. 12
application.view.show_view("Session")    # or "Arranger", "Detail/Clip", "Browser"
application.view.focus_view("Detail/Clip")
```

## Things the LOM cannot do (don't hunt for them)

- Render/export audio, or read audio sample data.
- Load arbitrary files by path (only browser items by URI; `create_audio_clip`
  is the one path-based exception).
- Save the set, open dialogs, or drive most menu commands.
- Access the Groove Pool, tuning systems, or (pre-12.1) most warp-marker edits.
- Create return tracks on older Lives (`song.create_return_track()` exists 11+).
