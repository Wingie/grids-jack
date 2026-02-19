# Idea 2: Ableton Link Integration for Tempo/Phase Sync

## Concept

Replace grids-jack's internal BPM clock with Ableton Link so it locks to
the shared session tempo and phase alongside Ableton Live 12 and
TouchDesigner. All three applications stay perfectly synchronized without
any manual BPM matching.

## How It Works

Currently, grids-jack uses an internal clock based on frame counting:

```
frames_per_pulse = sample_rate * 60.0 / (bpm * 24.0)
```

With Ableton Link, grids-jack joins the Link session and derives its pulse
timing from the shared tempo and beat phase. When you change BPM in Ableton,
grids-jack follows. When TouchDesigner's Ableton Link CHOP reports a beat,
grids-jack is hitting that same beat.

## Implementation

1. Add the [Ableton Link SDK](https://github.com/Ableton/link) as a submodule
   or vendored dependency (it's header-only C++)
2. Create a `LinkClock` class wrapping `ableton::Link`
3. In the JACK process callback, query the Link session state for:
   - Current tempo (replaces `-b` CLI flag when Link is active)
   - Beat phase (used to derive 24 PPQN pulse timing)
   - Whether a pulse should fire this buffer
4. Add CLI flags:
   - `-L` to enable Link (default: off, uses internal clock)
   - `-b` still works as initial tempo suggestion when Link starts
5. The `PatternGeneratorWrapper::Process()` method already ticks per-pulse --
   swap the frame-counting trigger for the Link-derived trigger

## Signal Flow

```
Ableton Live 12                 TouchDesigner
(master or peer)                Ableton Link CHOP
      |                               |
      +----------- Link Session ------+
      |            (UDP multicast     |
      |             port 20808)       |
      |                               |
  grids-jack (LinkClock)
      |
  Pulses derived from shared beat phase
      |
  PatternGeneratorWrapper ticks in sync
      |
  Drum patterns locked to session tempo
```

## Why This Is Interesting

- **Zero-config sync**: No MIDI clock cables, no manual BPM entry. Start
  Ableton, start grids-jack with `-L`, they find each other automatically
- **Phase-coherent**: Not just tempo-matched -- the downbeats align. Grids
  patterns land on the beat with Ableton clips and TouchDesigner's beat ramp
- **Multi-machine**: Link works over local network. Run grids-jack on a
  Raspberry Pi or second laptop and it still syncs
- **Dynamic tempo**: Change BPM in Ableton during performance, grids-jack
  follows smoothly -- great for live transitions
- **Coexists with Pocket Scion**: Pocket Scion feeds MIDI notes to Ableton
  for melodic content, while grids-jack provides the synchronized rhythmic
  foundation

## Dependencies to Add

- Ableton Link SDK (header-only C++11, permissive license)
- ASIO standalone (networking, bundled with Link SDK)

## Complexity

Medium-low. The Link SDK is well-documented with examples. The main challenge
is correctly deriving 24 PPQN pulse timing from Link's beat phase in the
realtime JACK callback. The SDK is designed for exactly this use case and
provides lock-free thread-safe access from audio callbacks.
