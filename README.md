# maxforlive-device-builder

An agent skill that builds complete, ready-to-use **Max for Live devices** as `.maxpat` files.
Drop the output directly into Ableton Live — no patching required.

## What This Skill Builds

| Device Type        | Examples                                                         |
|--------------------|------------------------------------------------------------------|
| Synthesizers       | Subtractive, FM, wavetable, additive, mono/poly                 |
| Drum Machines      | 808-style, sample-based, velocity-sensitive, multi-pad          |
| Granular Instruments | Granular synth, folder scanner, live input granularizer       |
| Sequencers         | Step sequencer, Euclidean, polyrhythmic, generative             |
| Arpeggiators       | Up/down/random, chord, scale quantizer, humanizer               |
| Audio Effects      | Distortion, saturation, reverb, delay, chorus, EQ, compressor   |
| MIDI Processors    | Chord generator, quantizer, probability gate, LFO-to-CC         |
| Samplers           | Folder scanner, multi-sample player, round-robin, timestretcher |

## Install

```bash
npx skills add https://github.com/Simoke98/maxforlive-device-builder
```

## Usage

Ask naturally:

```
"Build me a subtractive mono synth for Ableton"
"Create a granular sampler that loads a whole folder of samples"
"Make a 16-step drum machine with 808-style kick and snare"
"Build an arpeggiator that syncs to Ableton's tempo"
"I need a distortion + reverb effects chain in Max for Live"
"Create a step sequencer with probability per step"
"Build a granularizer that works on live audio input"
```

The agent generates a complete `.maxpat` file. Save it with `.amxd` extension and drag into any Ableton track.

## How to Install Generated Devices in Ableton

1. Save the generated JSON as `DeviceName.amxd`
2. In Ableton: open your User Library → drag the `.amxd` file onto a track
3. The device opens in Max for Live's editor on first use
4. Done — all parameters are automatable from Ableton

## Device Types

| Ableton Slot   | Device Type     | Use for                          |
|----------------|-----------------|----------------------------------|
| MIDI Effect    | MIDI processor  | Arpeggiator, chord, quantizer    |
| Instrument     | Synth/Sampler   | All sound generators             |
| Audio Effect   | FX              | Reverb, distortion, EQ, etc.     |

## Structure

```
maxforlive-device-builder/
├── SKILL.md                            ← Main skill
└── references/
    ├── synth-devices.md                ← Subtractive, FM, wavetable
    ├── drum-and-sampler.md             ← Drum machines, step seq, groove~
    ├── granular-and-sampler.md         ← Granular, folder scanner, pfft~
    ├── midi-and-sequencer.md           ← Arpeggiator, MIDI, sequencers
    ├── effects.md                      ← All audio effects
    └── live-api-and-ui.md              ← M4L UI, Live API, automation
```

## Requirements

- Ableton Live 10+ with Max for Live (included in Live Suite)
- Max 8 (bundled with M4L)

## Author

Built by [SIMOKE](https://github.com/Simoke98) — producer, A&R, DIY electronics. Rome.

## License

MIT
