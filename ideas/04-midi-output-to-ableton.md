# Idea 4: MIDI Output to Ableton for Hybrid Instrument Layering

## Concept

Add MIDI output to grids-jack so its generative drum triggers are sent as
MIDI notes to Ableton Live. Instead of (or alongside) playing internal WAV
samples, grids-jack becomes a **pattern brain** that drives Ableton's
instruments -- Drum Racks, Simpler, Operator, or any third-party plugin.
This lets you layer grids-jack patterns with Pocket Scion's melodic MIDI
on separate Ableton tracks.

## How It Works

grids-jack already tags samples with MIDI note numbers (filename format:
`60.kick.wav` = note 60). When a drum part triggers, in addition to playing
the internal sample, it sends the corresponding MIDI note to Ableton:

| Drum Part | Internal Sample | MIDI Note Out | Ableton Track |
|-----------|----------------|---------------|---------------|
| BD (kick) | `60.kick.wav` | Note 60, vel from velocity pattern | MIDI track → Drum Rack pad |
| SD (snare) | `65.snare.wav` | Note 65, vel from velocity pattern | MIDI track → Drum Rack pad |
| HH (hat) | `72.hat.wav` | Note 72, vel from velocity pattern | MIDI track → Drum Rack pad |

## Implementation Options

### Option A: JACK MIDI (Recommended for Linux)

1. Register a JACK MIDI output port alongside the existing audio ports
2. In the JACK process callback, write MIDI note-on/note-off events to the
   MIDI buffer when triggers fire
3. Connect the JACK MIDI port to Ableton's MIDI input (via a2jmidid bridge
   or native JACK MIDI routing)
4. Fully realtime-safe -- MIDI events are sample-accurate within the buffer

### Option B: Virtual MIDI via ALSA

1. Create a virtual ALSA MIDI port using `snd_seq`
2. Send note events from a non-realtime thread (queued from the audio callback)
3. Ableton sees it as a standard MIDI input device
4. Simpler setup but slightly less timing precision

## CLI Flags

- `-M` enable MIDI output
- `-m <channel>` MIDI output channel (default: 10, standard drum channel)
- `--midi-only` suppress internal sample playback (pattern brain mode)
- `--midi-vel-scale <0.0-1.0>` scale velocity output

## Signal Flow

```
grids-jack
    |
    +-- JACK Audio Out (L/R) ----> speakers / mixer (optional)
    |
    +-- JACK MIDI Out -----------> Ableton Live 12
                                        |
                                   MIDI Track
                                        |
                                   Drum Rack / Instrument
                                        |
                                   Mixed with Pocket Scion
                                   MIDI on other tracks
                                        |
                                   Master Out -> speakers
```

## Why This Is Interesting

- **Sound design freedom**: grids-jack generates the *patterns*, but Ableton
  provides the *sounds*. Swap Drum Racks on the fly during performance.
  Use Ableton's effects chains (reverb, delay, sidechain compression) on
  the grids patterns
- **Pocket Scion duet**: Pocket Scion provides bio-driven melodies on MIDI
  channels 1-5, grids-jack provides generative rhythms on channel 10. Both
  feed into Ableton for unified mixing and processing
- **Velocity expression**: grids-jack's 32-step binary velocity patterns
  translate directly to MIDI velocity values, giving Ableton instruments
  dynamic expression data
- **Pattern brain mode** (`--midi-only`): Run grids-jack as a pure sequencer
  with no audio output -- it becomes a lightweight generative MIDI pattern
  source. Useful if you want all sound to come from Ableton
- **Recording**: Ableton can record the MIDI output as clips, letting you
  capture generative patterns and edit them later in the arrangement view

## Complexity

Medium. JACK MIDI is well-documented and the trigger points already exist in
`PatternGeneratorWrapper`. The main work is managing note-on/note-off pairs
correctly (need to send note-off before or at the same time as the next
note-on for the same pitch) and mapping the binary velocity patterns to
useful MIDI velocity ranges (e.g., low=40, high=127 instead of 0.1/1.0).
