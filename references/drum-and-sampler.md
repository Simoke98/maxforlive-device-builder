# Drum Machine & Sampler Reference

## Sample Playback Architecture

```
trigger → buffer~ [name] → groove~ or play~ → *~ [velocity] → plugout~
```

### buffer~ — loads audio file into memory
```
buffer~ kick 0        → named buffer, auto-size
read /path/to/kick.wav → load file into buffer
```

### groove~ — looping sample player (preferred for drums)
```
groove~ kick 1        → plays buffer "kick", 1 = loop off
  inlet 0: bang = play, 0 = stop
  inlet 1: playback speed (1.0 = normal, 2.0 = double speed)
  outlet 0: left audio
  outlet 1: right audio
  outlet 2: position (0-1)
```

### play~ — one-shot sample player
```
play~ kick            → plays buffer "kick" once
  inlet 0: bang = trigger
  inlet 1: start point (samples)
  inlet 2: end point (samples)
```

---

## Step Sequencer Architecture

### Basic 16-step drum sequencer

```
metro [bpm/4 in ms] → counter 0 15 → sel 0 1 2 ... 15
                                        ↓ each sel outlet → toggle (on/off per step)
                                        ↓ when step active → trigger sample
```

**BPM to step time:**
- 120 BPM, 16th notes = 60000 / 120 / 4 = 125ms per step
- Formula: `step_ms = 60000 / bpm / steps_per_beat`
- `steps_per_beat` = 4 for 16th notes, 2 for 8th notes

### Using live.step (M4L native step sequencer UI)
```
live.step @steps 16 @notes 8 @mode 1
  outlet 0: note pitch
  outlet 1: velocity  
  outlet 2: duration
  outlet 3: step index
  outlet 4: active/inactive
```

### Using live.grid (free grid for custom patterns)
```
live.grid @rows 4 @columns 16
  → outputs [row, column, value] when cell toggled
```

---

## 808-Style Drum Machine (complete blueprint)

### Kick drum
```
bang → line~ 1 0 400          → *~ (amp env: fast decay)
     → line~ 80 40 200        → mtof → cycle~ (pitch env: starts at 80Hz, drops)
                               → *~ [amp env] → plugout~
```

### Snare drum
```
bang → line~ 1 0 150          → *~ (amp env)
noise~ → biquad~ [BP 200Hz]   → *~ 0.3 (tone component)
noise~ → biquad~ [HP 2kHz]    → *~ 0.7 (noise component)
both → +~ → *~ [amp env] → plugout~
```

### Hi-hat (closed)
```
bang → line~ 1 0 50            → *~ (short amp env)
noise~ → biquad~ [HP 8kHz]    → *~ [amp env] → plugout~
```

### Hi-hat (open) — same but longer decay
```
line~ 1 0 300
```

---

## Multi-Sample Drum Pad (velocity-sensitive)

```
notein [note, vel, ch]
  note → sel 36 38 42 46 49 51    (GM drum map: kick, snare, hh, OH, crash, ride)
  vel  → * 0.00787402             → *~ (velocity scaling)

sel 36 → groove~ kick
sel 38 → groove~ snare
sel 42 → groove~ hihat_closed
... etc.
```

**Random velocity humanization:**
```
bang → random 20 → - 10 → * 0.01 → + 1.0    (±10% velocity variation)
     → *~ [base velocity]
```

**Random sample selection (round-robin):**
```
bang → counter 0 3    (4 samples of the same instrument)
     → sel 0 1 2 3
     → route to groove~ kick1 / kick2 / kick3 / kick4
```

---

## Folder Scanner + Sample Player

**Scan a folder and load all .wav files into a menu:**

```javascript
// Use js object with this code:
// filepath object → folder → iterate through files → load into buffers

folder [path] @recursive 0
  → iterate
  → [route "end"] or filename outlet
  → if ".wav" or ".aif" → send to buffer~ and menu
```

### Complete folder scanner blueprint

**Objects:**
```
live.text "Load Folder" → opendialog   (opens folder browser dialog)
opendialog → folder [path]             (scans folder)
folder → route "file" → [js buildlist.js] → live.menu   (populates dropdown)
live.menu → route [index] → buffer~ sample   (loads selected file)
live.dial "Position" → groove~ inlet 1       (playback position)
live.dial "Speed" → groove~ inlet 2          (playback speed)
bang → groove~ inlet 0                       (trigger playback)
```

**JS helper (buildlist.js):**
```javascript
// Max JS object to build a menu from filenames
outlets = 1;
var files = [];

function file(name, path) {
    files.push({name: name, path: path});
    outlet(0, ["append", name]);
}

function clear() {
    files = [];
    outlet(0, "clear");
}

function getpath(idx) {
    if (files[idx]) outlet(0, ["path", files[idx].path]);
}
```

---

## Groove~ Advanced Techniques

### Reverse playback
```
groove~ sample 1
  speed inlet → -1.0     (negative speed = reverse)
  set start/end: inlet 1 = buffer_length, inlet 2 = 0
```

### Timestretching
```
groove~ sample 1
  speed → 0.5   (half speed = pitched down)
  speed → 2.0   (double speed = pitched up)
```
For pitch-independent timestretching use `pfft~` or the `gizmo~` object.

### Random slice player
```
buffer~ sample          (full sample loaded)
random [n_slices]       (random slice index)
  → * [slice_length]    (convert to sample offset)
  → poke~ / play~ (start point)
```

---

## M4L Step Sequencer with Probability

```
live.step @steps 16 @notes 1
  step pitch outlet → noteout pitch
  step vel outlet  → * [probability_factor] → gate decision:
    random 100 → < [threshold] → sel 1 → allow note
                              → sel 0 → block note
```

**Euclidean rhythm generator (via js):**
```javascript
// Generates Euclidean rhythms: bjorklund(hits, steps)
function bjorklund(k, n) {
    // Bresenham-style distribution
    var pattern = [];
    var remainder = k;
    var divisor = n - k;
    var level = 0;
    var counts = [k];
    var remainders = [0];
    // ... (full Bjorklund algorithm)
    return pattern;
}
```
