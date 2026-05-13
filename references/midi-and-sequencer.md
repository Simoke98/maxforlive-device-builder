# MIDI Processing & Sequencer Reference

## MIDI Objects in Max for Live

### Input
| Object     | Outputs                          | Notes                              |
|------------|----------------------------------|------------------------------------|
| `notein`   | note, velocity, channel          | All incoming MIDI notes            |
| `ctlin`    | value, controller#, channel      | CC messages                        |
| `bendin`   | bend value (-8192 to 8191), ch   | Pitch bend                         |
| `pgmin`    | program number, channel          | Program change                     |
| `midiin`   | raw MIDI bytes                   | Low-level access                   |

### Output
| Object     | Inputs                           | Notes                              |
|------------|----------------------------------|------------------------------------|
| `noteout`  | note, velocity, channel          | Send MIDI notes                    |
| `ctlout`   | value, controller#, channel      | Send CC                            |
| `bendout`  | value (-8192 to 8191), channel   | Send pitch bend                    |
| `pgmout`   | program, channel, port           | Send program change                |

---

## Arpeggiator Architecture

### Basic up arpeggiator
```
notein → [note storage: coll or table]
metro [rate_ms] → counter 0 [n_notes-1]
  → [index into note storage]
  → noteout pitch
  → noteout velocity (fixed or from storage)
```

### Note storage with coll
```
notein [note, vel]
  note-on (vel>0) → store in coll [index, note]
  note-off (vel=0) → remove from coll
  query coll size → update counter max
```

### Arpeggio patterns
```
UP:    counter 0 [n-1] → index
DOWN:  counter [n-1] 0 (count down)
UP/DOWN: [pendulum counter using drunk or fold]
RANDOM: random [n] → index
```

### Complete arpeggiator patch (objects list)
```
notein                         → receive notes
stripnote                      → separate note-on from note-off
route note_on note_off         → manage note storage

[note buffer: table 128]       → store active notes
  note_on → poke table         → add note
  note_off → poke 0            → remove note
  
live.dial "Rate" → [tempo sync options] → metro rate
live.menu "Pattern" [Up/Down/UpDown/Random] → pattern selector
live.dial "Octave Range" 1-4 → octave transpose logic
live.dial "Gate" 1-100% → note duration as % of step

metro → counter → table index → [note+octave offset] → noteout
```

### Tempo sync to Ableton transport
```
transport → [rate in ticks: 480=quarter, 240=8th, 120=16th]
live.dial "Division" [1/4, 1/8, 1/16, 1/32, 1/3, 1/6] → tempo division selector
```

---

## Step Sequencer (MIDI)

### 16-step melodic sequencer
```
transport → metro [step_time] → counter 0 15
  → [16x number boxes: pitch values] → sel 0 1 2...15
     each outlet → noteout [pitch, velocity, channel]

live.step @steps 16 @notes 1 @mode 0   → M4L native UI version
  pitch outlet → noteout
  vel outlet   → noteout velocity
  active outlet → gate (mute individual steps)
```

### Polyrhythmic sequencer (multiple lengths)
```
Track 1: counter 0 15 (16 steps)
Track 2: counter 0 11 (12 steps — creates evolving patterns)
Track 3: counter 0 7  (8 steps)

Each track has own metro → counter → note selection
All noteout to same channel → layers of rhythm
```

### Euclidean sequencer
```javascript
// js euclidean.js
// Inputs: [hits, steps] → outputs: pattern array [1,0,1,0,1,1,0,1]

function msg_int(v) {}  // hits
function list(hits, steps) {
    var pattern = bjorklund(hits, steps);
    outlet(0, pattern);
}

function bjorklund(k, n) {
    if (k >= n) return new Array(n).fill(1);
    var pattern = [];
    var counts = new Array(k).fill(1).concat(new Array(n-k).fill(0));
    var remainders, divisor, level;
    remainders = [n - k];
    divisor = k;
    level = 0;
    while (remainders[level] > 1) {
        counts.push(Math.floor(divisor / remainders[level]));
        remainders.push(divisor % remainders[level]);
        divisor = remainders[level];
        level++;
    }
    // Build pattern
    return buildPattern(counts, remainders, level);
}
```

---

## Chord Generator

```
notein [note] → chord_builder:
  note → [note]           (root)
  note → + [interval1]    (e.g. +4 = major third)
  note → + [interval2]    (e.g. +7 = perfect fifth)
  note → + [interval3]    (e.g. +12 = octave)
  all four → noteout (same velocity, channel 1)
```

### Chord type selector
```
live.menu "Chord" [Major, Minor, Dom7, Maj7, Sus4, Dim, Aug]
  → sel 0 1 2 3 4 5 6
  → message [0 4 7] [0 3 7] [0 4 7 10] [0 4 7 11] [0 5 7] [0 3 6] [0 4 8]
  → unpack → store intervals
```

---

## Scale Quantizer

Quantizes incoming MIDI notes to a musical scale:

```
notein [raw_note] → [note mod 12] → lookup table [scale mask]
                  → nearest in-scale note → noteout
```

**Scale masks (1=in scale, 0=out):**
```
Major:      [1,0,1,0,1,1,0,1,0,1,0,1]
Minor:      [1,0,1,1,0,1,0,1,1,0,1,0]
Pentatonic: [1,0,1,0,1,0,0,1,0,1,0,0]
Dorian:     [1,0,1,1,0,1,0,1,0,1,1,0]
Phrygian:   [1,1,0,1,0,1,0,1,1,0,1,0]
```

**Implementation:**
```javascript
// js quantize.js
var scales = {
    major: [0,2,4,5,7,9,11],
    minor: [0,2,3,5,7,8,10],
    pentatonic: [0,2,4,7,9]
};
var currentScale = 'major';
var rootNote = 0;

function note(n) {
    var octave = Math.floor(n / 12);
    var pc = n % 12;
    var scale = scales[currentScale];
    var quantized = nearestInScale(pc - rootNote, scale);
    outlet(0, octave * 12 + rootNote + quantized);
}

function nearestInScale(pc, scale) {
    pc = ((pc % 12) + 12) % 12;
    var best = scale[0], bestDist = 12;
    for (var i = 0; i < scale.length; i++) {
        var d = Math.abs(scale[i] - pc);
        if (d < bestDist) { bestDist = d; best = scale[i]; }
    }
    return best;
}
```

---

## MIDI Probability / Humanizer

```
notein [note, vel]
  vel → random 20 → - 10 → + [vel] → clip 1 127    (±10 velocity variation)
  trigger → random 100 → < [prob%] → sel 1 → allow note through
                                    → sel 0 → block (swallow note)
  timing: metro → + [random 20] → irregular trigger timing
```

## LFO for CC Automation

```
phasor~ [rate_hz] → cycle~ (shape: sine, saw, square)
  → scale~ -1 1 0 127
  → snapshot~ 30    (sample signal at 30Hz → control rate)
  → ctlout [cc_number] [channel]
```
