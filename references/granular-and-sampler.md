# Granular Synthesis & Advanced Sampler Reference

## Granular Synthesis Fundamentals

Granular synthesis chops audio into tiny "grains" (10–500ms) and
reassembles them with control over position, pitch, density, and size.

### Core granular objects in Max

| Object      | Type        | Description                                      |
|-------------|-------------|--------------------------------------------------|
| `munger~`   | Max package | Classic granular processor (Third Party - CNMAT) |
| `granular~` | M4L/Max8    | Built-in granular engine (Max 8+)               |
| `grainstretch~` | Max     | Granular timestretching                         |
| Manual      | phasor~+buffer~ | DIY granular via grain scheduling          |

---

## DIY Granular Engine (no external packages)

Build granular synthesis from first principles using `poly~`:

### Architecture
```
buffer~ source [audio file]        → stores source material
metro [grain_rate]                 → triggers new grains
  → poly~ grain_voice 32           → 32 simultaneous grain voices
     inside poly~:
       in 1 [position]  → play~ source (start point)
       in 2 [size_ms]   → play~ source (end point = start + size)
       in 3 [pitch]     → speed multiplier
       envelope: hanning window via cycle~ (0.5 Hz) or line~
       out: grained audio
```

### Grain voice subpatcher (grain_voice.maxpat)
```
in 1 [position_samples]
in 2 [grain_size_samples]
in 3 [pitch_ratio]

play~ source [position] [position+size]
  → *~ [hanning_envelope]    (smooth grain window)
  → *~ [amplitude]
  → out 1

hanning envelope:
  line~ 1 [size_ms] 0  → curve~ → *~  (rise + fall)
```

### Control parameters
```
POSITION:   0-1 (position in source file, 0=start, 1=end)
SIZE:       10-500ms (grain duration)
PITCH:      0.25-4.0 (0.5=octave down, 2.0=octave up)
DENSITY:    1-100 grains/sec (metro speed: 1000/density ms)
SPREAD:     ±position randomization per grain
SCATTER:    timing jitter (random delay 0-50ms per grain)
MIX:        wet/dry blend
```

### Granular with position modulation (scrub)
```
live.dial "Position" → * [buffer_length_samples]
  → +~ [random spread] → play~ source position inlet
```

---

## Buffer~ Management

### Load file into buffer
```
buffer~ myfile 0               → create buffer (0 = auto-size)
read /path/to/file.wav         → load from disk
  or
replace /path/to/file.wav      → load and resize buffer to fit
```

### Get buffer info
```
buffer~ myfile → info~ → [length in samples / ms / channels / samplerate]
```

### Record into buffer (live input)
```
plugin~ → record~ myfile       → records Ableton input into buffer
toggle → record~ (start/stop)
```

### Waveform display
```
waveform~ myfile               → visual waveform display
  → selection start/end outlets for loop points
```

---

## Multi-Sample Player with Round Robin

### Architecture for a playable instrument
```
notein [note, vel, ch]
  note → makenote [note vel] → pack → route by note range:
    36-47 (bass) → load bass samples
    48-59 (low mid) → load low mid samples
    60-71 (mid) → load mid samples
    72-84 (high) → load high samples
  
  For each zone:
    note → - [root_note] → * [semitone_ratio] → groove~ speed
    vel  → * 0.00787 → *~
```

**Semitone ratio formula:**
```
ratio = 2^(semitones/12)
In Max: [semitones] → * 0.08333 → exp2 → groove~ speed inlet
```
`exp2` object: raises 2 to the power of input (semitone to ratio converter)

---

## Granular Folder Scanner + Player

### Complete instrument: load folder, granularize any file

```
[button "Load Folder"] → opendialog → folder [scanned path]
folder → filelist [js]   → live.menu [file selector]
live.menu selection → replace~ myfile   → buffer~ myfile

live.dial "Position" → * [buffer_size] → position
live.dial "Size" → grain size (ms)
live.dial "Pitch" → pitch ratio
live.dial "Density" → metro speed = 1000/density
live.dial "Spread" → position randomizer range

metro [density] → poly~ granvoice 32 [position, size, pitch, spread]
poly~ → *~ [mix] → plugout~
```

### File filtering (only show audio files)
```javascript
// In js filefilter.js
function file(name, path) {
    var ext = name.split('.').pop().toLowerCase();
    if (ext === 'wav' || ext === 'aif' || ext === 'aiff' || ext === 'mp3') {
        outlet(0, ["add", name]);
        outlet(1, path);  // path for buffer loading
    }
}
```

---

## Timestretching with pfft~

For high-quality pitch-independent timestretching:

```
plugin~ → pfft~ timestretch 4096 1   (4096 FFT size, overlap 1)
  inside pfft~:
    fftin~ → cartopol~ → [phase accumulator] → poltocar~ → fftout~
    speed inlet → controls phase increment
```

### Simpler approach: granular timestretch
```
phasor~ [1/file_duration_sec * stretch_factor]
  → * [buffer_size_samples]
  → play~ source [position]
```
`stretch_factor`: 1.0 = normal, 0.5 = half speed (stretched), 2.0 = double speed

---

## Live Input Granularizer (audio effect version)

Captures live audio and granularizes in real-time:

```
plugin~ → record~ live_buffer [always recording, circular]
         → circular buffer (use MSP: write head always moving)

phasor~ [0.1] → * [buffer_size] → frozen position
  OR live.dial "Freeze Position" → position

metro [density] → poly~ grain 32 [frozen_pos, size, pitch]
poly~ → *~ [wet]
plugin~ → *~ [dry]
+~ → plugout~
```

**Key technique — freeze:** stop the position from advancing → infinite sustain of any moment.
