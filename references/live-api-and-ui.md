# Live API & M4L UI Reference

## M4L UI Objects

### live.dial — knob/parameter control
```json
{
  "box": {
    "id": "obj-10",
    "maxclass": "live.dial",
    "patching_rect": [100, 50, 44, 44],
    "varname": "Cutoff",
    "minimum": 20,
    "maximum": 20000,
    "unit_style": "hertz",
    "parameter_enable": 1,
    "saved_attribute_attributes": {
      "valueof": { "parameter_initial": [1000], "parameter_initial_enable": 1 }
    }
  }
}
```

### live.slider — vertical/horizontal fader
```json
{
  "box": {
    "id": "obj-11",
    "maxclass": "live.slider",
    "patching_rect": [150, 50, 20, 80],
    "varname": "Volume",
    "minimum": 0.0,
    "maximum": 1.0,
    "parameter_enable": 1
  }
}
```

### live.button — toggle/momentary button
```json
{
  "box": {
    "id": "obj-12",
    "maxclass": "live.button",
    "patching_rect": [200, 50, 24, 24],
    "varname": "Active",
    "parameter_enable": 1
  }
}
```

### live.menu — dropdown selector
```json
{
  "box": {
    "id": "obj-13",
    "maxclass": "live.menu",
    "patching_rect": [100, 100, 120, 20],
    "varname": "Waveform",
    "items": "Sine:Saw:Square:Triangle",
    "parameter_enable": 1
  }
}
```

### live.step — step sequencer UI
```json
{
  "box": {
    "id": "obj-14",
    "maxclass": "live.step",
    "patching_rect": [50, 200, 400, 80],
    "varname": "Sequence",
    "steps": 16,
    "notes": 1,
    "parameter_enable": 1
  }
}
```

---

## Parameter Automation

### Making a parameter automatable in Ableton
Every `live.dial`, `live.slider`, `live.button`, `live.menu` with `parameter_enable 1`
appears in Ableton's automation lane.

**Naming:** set `varname` to a descriptive name — this shows in Ableton's automation.

**Unit styles for live.dial:**
```
int         → integer values
float       → decimal values
decibels    → dB display
semitones   → musical pitch offset
percent     → 0-100%
hz          → frequency in Hz
milliseconds → time in ms
pan         → -50 to +50 pan
```

---

## Live API — Reading Ableton State

Access Ableton Live's internal state from Max:

### live.object + live.path
```
live.path "live_set"             → root Live Set object
live.path "live_set tracks 0"    → first track
live.path "live_set view"        → current view
```

### Get BPM
```
live.path "live_set" → live.object → get tempo
  → output: current BPM as float
```

### Get current playing position
```
live.path "live_set" → live.object → get current_song_time
  → bars.beats.sixteenths as list
```

### Trigger a clip
```
live.path "live_set tracks 0 clip_slots 0 clip"
  → live.object → call fire
```

### Observe tempo changes
```
live.path "live_set" → live.observer tempo → output: BPM on change
```

---

## Transport Sync

### Sync to Ableton tempo
```
transport → [tempo, position, state outputs]
  tempo outlet → / 60 → metro rate
  state outlet → sel 1 → start metro
               → sel 0 → stop metro
```

### Musical time divisions
```
transport @clock 1        → outputs ticks (480 ticks = 1 quarter note)
  → [/ 480] → bars
  → [% 480] → position within bar
  → [/ 120] → 8th note position
  → [/ 60]  → 16th note position
  → [/ 30]  → 32nd note position
```

### Sync metro to Ableton beat grid
```
transport → clock outlet (ticks)
  → [== 0] or [% 120 == 0]     (every 16th note)
  → bang → trigger your sequencer step
```

---

## M4L Presentation Mode

### Setting up device presentation (visible face in Ableton)

In your patch, objects in **Presentation Mode** are what users see in Ableton.
Set `presentation: 1` and `presentation_rect` for each UI object:

```json
{
  "box": {
    "id": "obj-10",
    "maxclass": "live.dial",
    "patching_rect": [100, 50, 44, 44],
    "presentation": 1,
    "presentation_rect": [10, 10, 44, 44]
  }
}
```

**Typical M4L device panel size:** 500×120px (standard), up to 500×200px (tall)

### Background panel
```json
{
  "box": {
    "id": "obj-bg",
    "maxclass": "panel",
    "patching_rect": [0, 0, 500, 120],
    "presentation": 1,
    "presentation_rect": [0, 0, 500, 120],
    "bgcolor": [0.15, 0.15, 0.15, 1.0],
    "bordercolor": [0.3, 0.3, 0.3, 1.0],
    "rounded": 5
  }
}
```

---

## Complete .maxpat Template

```json
{
  "patcher": {
    "fileversion": 1,
    "appversion": {
      "major": 8,
      "minor": 5,
      "revision": 0,
      "architecture": "x64",
      "modernui": 1
    },
    "classnamespace": "dsp.topology",
    "rect": [100, 100, 800, 500],
    "openrect": [0, 0, 500, 120],
    "openinpresentation": 1,
    "boxes": [],
    "lines": [],
    "parameters": {
      "obj-10": ["Cutoff", ["Cutoff", 0]]
    }
  }
}
```

`"openinpresentation": 1` → device opens in presentation view in Ableton (user-facing).
`"openrect"` → size of the device panel in Ableton.

---

## Common Patch Patterns

### Smooth parameter (no zipper noise)
```
live.dial → line~ [ramp_time_ms]    (ramp_time: 5-20ms)
          → *~ or lores~ (use ramped value)
```
Direct connection from live.dial to audio parameters causes "zipper noise" (audible steps).
Always go through `line~` for smooth interpolation.

### Preset system
```
live.dial values → pattrstorage → [save/recall presets]
live.menu "Preset" → pattrstorage recall [slot]
button "Save" → pattrstorage store [slot]
```

### CPU-efficient LFO
```
phasor~ [rate]         → runs at audio rate (expensive)
vs.
metro [1000/rate]      → snapshot~ → line~ (cheaper for slow LFOs < 10Hz)
```
Use `phasor~` only for audio-rate modulation. For parameter modulation (< 30Hz), use `metro` + `snapshot~`.
