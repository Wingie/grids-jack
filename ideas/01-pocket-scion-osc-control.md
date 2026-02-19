# Idea 1: Pocket Scion Biofeedback Drives Grids Patterns via OSC

## Concept

Add an OSC input listener to grids-jack so the Pocket Scion's biofeedback
data can control pattern generation parameters in real time. Plants/fungi/skin
literally shape the drum patterns.

## How It Works

The Pocket Scion desktop app broadcasts 6 biofeedback values over OSC:
**min, max, average, delta, standard variance, and standard deviation** of
resistance variation. These map naturally onto grids-jack's core parameters:

| Pocket Scion OSC Value | grids-jack Parameter | Effect |
|------------------------|---------------------|--------|
| average                | X position (0-255)  | Shifts drum style along the horizontal pattern map axis |
| delta                  | Y position (0-255)  | Shifts drum style along the vertical pattern map axis |
| standard deviation     | Randomness (0-255)  | More biological variation = more pattern chaos |
| max                    | Density BD (0-255)  | Peak signals = heavier kick patterns |
| min                    | Density HH (0-255)  | Quiet signals = sparser hi-hat activity |
| standard variance      | Humanize (0.0-1.0)  | Maps to timing jitter amount |

## Implementation

1. Add a lightweight OSC library (e.g., `liblo` or `oscpack`) as a dependency
2. Create an `OscReceiver` class that listens on a configurable UDP port
3. In the main loop, poll incoming OSC messages and update
   `PatternGeneratorWrapper` parameters (X, Y, randomness, density, humanize)
4. Add CLI flags: `-O <port>` for OSC listen port (default 9000)
5. The parameter updates feed into the existing per-pulse pattern evaluation,
   so changes take effect on the next clock tick -- tight and responsive

## Signal Flow

```
Plant/Fungi/Skin
      |
  Pocket Scion (USB)
      |
  Desktop Companion App
      |
  OSC broadcast (UDP port 9000)
      |
  grids-jack OscReceiver
      |
  PatternGeneratorWrapper
  (X, Y, randomness, density, humanize updated)
      |
  Generative drum patterns morph in response to biology
      |
  JACK stereo output -> speakers / Ableton / etc.
```

## Why This Is Interesting

- The Grids 2D pattern map has 25x25 nodes with interpolation between them --
  smoothly varying biofeedback data will produce continuously evolving but
  musically coherent drum patterns
- The existing LFO drift feature (`-l` flag) already modulates X/Y over
  15-45 second periods -- biofeedback would replace this with organic,
  unpredictable modulation from a living organism
- No MIDI note knowledge required -- the Pocket Scion's OSC stream is pure
  continuous control data, which maps perfectly to Grids' continuous parameters

## Dependencies to Add

- `liblo` (Lightweight OSC library) -- available via apt: `sudo apt install liblo-dev`
- OR `oscpack` (header-only, no external deps)

## Complexity

Medium. The core parameter-update plumbing already exists in
`PatternGeneratorWrapper`. The main work is adding the OSC listener thread
and ensuring thread-safe parameter updates (a simple atomic or lock-free
approach since updates are infrequent relative to audio callbacks).
