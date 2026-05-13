# Audio Effects Reference

## Distortion / Saturation

### Soft clipper (tanh)
```
plugin~ → *~ [drive]     (drive: 1-20)
        → tanh~           (soft saturation curve)
        → *~ [1/drive]    (normalize output)
        → plugout~
```

### Hard clipper
```
plugin~ → *~ [drive]
        → clip~ -0.8 0.8  (hard clip at ±0.8)
        → plugout~
```

### Asymmetric waveshaper (tube-like)
```
plugin~ → *~ [drive]
        → expr~ (in1 > 0) ? tanh(in1) : clip(in1, -0.5, 0)
        → plugout~
```

### Bitcrusher
```
plugin~ → *~ 32767           (scale to int range)
        → round~             (quantize to integers)
        → *~ [1/bit_depth]   (bit reduction: /32767 for 16bit, /255 for 8bit)
downsampling:
        → rate~ [factor]     (hold every N samples → sample rate reduction)
```

### Overdrive with tone control
```
plugin~ → *~ [drive 1-20]
        → tanh~
        → biquad~ [lowpass filtercoeff~ 0.4 0.7]  (tone: cut harsh HF)
        → *~ [level]
        → plugout~
```

---

## Delay / Echo

### Basic stereo delay
```
plugin~ → delay~ 88200 [time_samples]   (max 2 sec at 44100)
        → *~ [feedback]
        → +~ [dry signal]
        → plugout~
```

### Ping-pong delay
```
plugin~ [dry]
  Left:  → delay~ 88200 [time_L] → *~ [fb] → right channel input
  Right: → delay~ 88200 [time_R] → *~ [fb] → left channel input
  Wet/dry mix → plugout~
```

### Tap tempo sync to Ableton
```
transport → [convert ticks to ms: ticks/480 * 60000/bpm]
live.menu "Sync" [1/4, 1/8, 1/16, dotted 1/4, dotted 1/8]
  → delay time in samples
```

### Multi-tap delay
```
plugin~
  → delay~ 88200 [t1] → *~ 0.6
  → delay~ 88200 [t2] → *~ 0.4
  → delay~ 88200 [t3] → *~ 0.2
  all → +~ → plugout~
```

---

## Reverb

### Simple Schroeder reverb (all-pass + comb filters)
```
plugin~ → [4x comb filters in parallel]
           comb1: delay~ 1680 + feedback 0.83
           comb2: delay~ 1800 + feedback 0.83
           comb3: delay~ 2040 + feedback 0.83
           comb4: delay~ 2250 + feedback 0.83
        → +~ (mix all combs)
        → [2x all-pass in series]
           ap1: delay~ 240 + feedback -0.7
           ap2: delay~ 82 + feedback -0.7
        → *~ [wet]
        → +~ [dry * (1-wet)]
        → plugout~
```

### Convolution reverb (using pfft~)
```
buffer~ ir [impulse_response.wav]    (load IR file)
plugin~ → pfft~ convolve 4096 1
  inside pfft~:
    fftin~ 0 [live signal]
    fftin~ 1 [IR signal — preloaded]
    cartopol~ × cartopol~ → poltocar~ → fftout~
```

### Simple plate reverb approximation
```
plugin~
  → [8 all-pass filters, different prime delays]
  → [feedback matrix: cross-couple 4 delay lines]
  → [lowpass on feedback path: lores~ 6000 0.3]
  → output
```

---

## Chorus / Flanger

### Chorus
```
plugin~ [dry]
  → delay~ 4410 [base_delay + lfo]   (base: 20-30ms + LFO ±10ms)
  LFO: cycle~ 0.5 → scale~ -1 1 440 882  (±10ms at 44100 Hz)
  → *~ [wet]
  → +~ dry
  → plugout~
```

### Flanger
```
plugin~ [dry]
  → delay~ 441 [base_delay + lfo]    (base: 1-10ms + LFO ±5ms)
  LFO: cycle~ 0.3 → scale~ -1 1 44 220
  → *~ [wet]
  → +~ [dry] + feedback path
  → plugout~
```

---

## EQ (parametric)

### Parametric band (biquad~)
```
live.dial "Freq" → * 0.0000226757  (Hz to 0-1 normalized: /44100)
live.dial "Q" → filtercoeff~ [type] [freq] [Q]
live.dial "Gain" → filtercoeff~ gain
filtercoeff~ → biquad~
plugin~ → biquad~ → plugout~
```

### Filter types for filtercoeff~
```
lowpass   freq Q         → LP filter
highpass  freq Q         → HP filter
bandpass  freq Q         → BP filter
notch     freq Q         → Notch filter
peaking   freq Q gain    → Peaking EQ (bell)
lowshelf  freq Q gain    → Low shelf
highshelf freq Q gain    → High shelf
```

### 3-band EQ
```
plugin~
  → biquad~ [lowshelf 200Hz]     → *~ [low gain]
  → biquad~ [peaking 1000Hz Q2]  → *~ [mid gain]  
  → biquad~ [highshelf 8000Hz]   → *~ [high gain]
  → plugout~
```

---

## Compressor

### Feedforward RMS compressor
```
plugin~ [input]
  → [rms~] or [abs~ → lores~ 10] → level detection
  → [log envelope: attack/release]
  → [gain computation: above threshold → gain reduction]
  → *~ [input signal]
  → *~ [makeup gain]
  → plugout~
```

### Simplified gain computer
```
level_db = signal level in dB (via expr~ 20*log10(abs($v1)+0.0001))
gain_reduction = max(0, (level_db - threshold) * (1 - 1/ratio))
gain_linear = pow(10, -gain_reduction/20)
```

---

## Reverb + Saturation Chain (complete FX rack example)

```
plugin~
  → *~ [input_gain]
  → tanh~ (subtle saturation, drive 1.2)
  → [schroeder reverb as subpatcher]
  → *~ [wet/dry]
  → biquad~ [highpass 100Hz] (remove low mud from reverb)
  → *~ [output_gain]
  → plugout~
```

## Stereo Widener

```
plugin~ [L, R]
  mid  = (L + R) * 0.5
  side = (L - R) * 0.5
  side → *~ [width]          (width > 1 = wider, < 1 = narrower)
  outL = mid + side
  outR = mid - side
  → plugout~
```
