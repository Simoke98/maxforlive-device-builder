# Synthesizer Devices Reference

## Subtractive Synthesizer Architecture

```
MIDI in → pitch/gate → oscillator(s) → mixer → filter → VCA → output
                ↓                          ↑        ↑
           envelope gen ─────────────────────→ ADSR
```

### Mono Subtractive Synth (complete patch blueprint)

**Objects needed:**
```
notein                          → receives MIDI note + velocity
stripnote                       → strips note-off (vel=0) from note-on
mtof                            → MIDI note to Hz
* 0.00787402                    → velocity (0-127) to 0-1 amplitude
saw~ [freq]                     → main oscillator
cycle~ [freq]                   → secondary oscillator (detuned)
+~ 0                            → mix oscillators
lores~ [cutoff] [reso]          → lowpass filter
adsr~ 5 80 0.6 200              → amp envelope (A/D/S/R in ms)
*~ [env]                        → VCA (apply envelope)
*~ 0.7                          → output gain
plugout~                        → M4L audio out
```

**Key connections:**
1. `notein` → `stripnote` → `mtof` → `saw~` (inlet 0, frequency)
2. `notein` → `stripnote` → `* 0.00787402` → `adsr~` (gate: vel>0 = 1, vel=0 = 0)
3. `saw~` → `lores~` → `*~ [adsr output]` → `plugout~`

**MIDI gate logic:**
```
notein [note, vel, ch]
  vel → sel 0 → bang → 0 (note off → send 0 to adsr gate)
  vel → > 0 → 1 (note on → send 1 to adsr gate)
```

---

## FM Synthesizer

### 2-Operator FM

```
Modulator: cycle~ [mod_freq] → *~ [mod_index * carrier_freq] → +~ [carrier_freq] → Carrier: cycle~
```

**Operator ratio formula:**
- `mod_freq = carrier_freq * ratio`
- `mod_depth = carrier_freq * index` (index = modulation index, 0-20 typical)

**Implementation:**
```
mtof → p carrier_freq
     → *~ [ratio]  → cycle~ (modulator)
                    ↓ *~ [index * carrier_freq]
                    ↓ +~ carrier_freq
                    → cycle~ (carrier) → output
```

**M4L controls:**
- RATIO: `live.dial` 0.1–16, integer snapping for harmonic ratios (1,2,3,4,5,6,7,8)
- INDEX: `live.dial` 0–20 (controls brightness/harshness)
- A/D/S/R: four `live.dial` objects

---

## Wavetable Synthesizer

### Single-wavetable lookup via phasor~

```
phasor~ [freq] → *~ 512 → index~ [buffer_name] → output
```

**Buffer setup:**
```
buffer~ wavetable 512     → allocates 512-sample wavetable buffer
(populate via [table] or [jit.matrix] or preloaded .wav)
```

**Wavetable morphing (2 tables):**
```
phasor~ [freq] → index~ table1 → *~ [1-morph]
               → index~ table2 → *~ [morph]
                                  +~ → output
```
morph: 0 = table1, 1 = table2 — automate for evolving timbres

---

## Polyphonic Instrument via poly~

For polyphonic synths, use `poly~` subpatcher:

```
notein → poly~ mysynth 8    (8-voice polyphony)
poly~ outputs audio → plugout~
```

Inside `poly~` subpatcher (`mysynth`):
- Use `thispoly~` to manage voice stealing
- `in 1` = note, `in 2` = velocity
- `out 1` / `out 2` = left/right audio

**Voice-stealing:**
```
thispoly~ → mute signal when voice idle
           → voice number for debugging
```

---

## Essential Synth Objects

### Oscillators with anti-aliasing
- `saw~` — built-in anti-aliased sawtooth
- `rect~` — anti-aliased rectangle/pulse
- Use `cycle~` only for pure sine (it IS aliasing-free)
- For analog-style detuning: two `saw~` with slightly offset frequencies via `+~ 3`

### Filter cookbook
| Sound          | Object + settings                              |
|----------------|------------------------------------------------|
| Dark bassline  | `lores~ 200 0.3` + envelope to cutoff          |
| Acid filter    | `lores~` with reso 0.8–0.95, fast envelope     |
| Moog-style     | 4x `lores~` in series (each at 1/4 Q)          |
| Bright lead    | `svf~` HP output + slight reverb              |
| Vowel formant  | 2x `biquad~` bandpass in parallel             |

### Envelope to filter cutoff
```
adsr~ [A] [D] [S] [R]
  → *~ [env_amount]       env_amount: 0-5000 Hz
  → +~ [base_cutoff]
  → lores~ inlet 1 (frequency)
```

### Portamento / glide
```
mtof → line~ [glide_time]   glide_time: 0 = off, 50-500ms = smooth glide
     → saw~ (frequency inlet)
```

---

## Useful Subpatcher Patterns

### Hard sync
```
phasor~ [master_freq] → [threshold~ 0.5] → bang → [phasor~] reset (slave)
```
Slave oscillator resets phase when master crosses 0.5 → classic sync distortion

### Unison detune (4 voices)
```
mtof [freq]
  → +~ -6  → saw~1
  → +~ -2  → saw~2
  → +~ +2  → saw~3
  → +~ +6  → saw~4
all four → +~ (mix) → *~ 0.25 (normalize)
```

### Ring modulation
```
carrier~ * modulator~ → *~ → output
```
Both operands must be audio signals (~). Result = sum and difference frequencies.

### Waveshaping / distortion on oscillator
```
cycle~ [freq] → *~ [drive]  (drive 1-20)
              → tanh~       (soft clip)
              → *~ [1/drive] (normalize)
```
