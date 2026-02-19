# Idea 5: Unified AV Performance System -- Full Pipeline

## Concept

Combine Ideas 1-4 into a complete audiovisual performance system where
grids-jack sits at the center of a Pocket Scion + Ableton + TouchDesigner
pipeline. Biology drives rhythm, rhythm drives sound and visuals, and
everything stays in sync via Ableton Link.

## The Full Signal Flow

```
                    ┌──────────────────────┐
                    │   POCKET SCION       │
                    │   (plant/fungi/skin)  │
                    └────┬────────┬────────┘
                         │        │
                    MIDI │        │ OSC (biofeedback:
                    (USB)│        │  min,max,avg,delta,
                         │        │  std_var,std_dev)
                         │        │
                         │        ▼
                         │   ┌────────────────┐
                         │   │  grids-jack     │
                         │   │                 │
                         │   │  OSC In:        │
                         │   │  biofeedback -> │
                         │   │  X,Y,randomness,│◄──── Ableton Link
                         │   │  density,       │      (tempo/phase sync)
                         │   │  humanize       │
                         │   │                 │
                         │   │  Grids engine   │
                         │   │  generates      │
                         │   │  patterns       │
                         │   │                 │
                         │   │  Outputs:       │
                         │   │  - JACK audio   │──── stereo drums
                         │   │  - JACK MIDI    │──── note triggers
                         │   │  - OSC triggers │──── pattern data
                         │   └──┬──────┬───┬───┘
                         │     │      │   │
                         │     │audio │   │ OSC triggers
                         │     │MIDI  │   │ (/grids/trigger/bd,sd,hh)
                         │     │      │   │ (/grids/state/x,y,...)
                         ▼     ▼      │   │
                    ┌──────────────┐   │   │
                    │ ABLETON 12   │   │   │
                    │              │   │   │
                    │ Track 1-5:   │   │   │
                    │  Pocket Scion│   │   │
                    │  MIDI →      │   │   │
                    │  instruments │   │   │
                    │              │   │   │
                    │ Track 6:     │   │   │
                    │  grids-jack  │   │   │
                    │  MIDI →      │   │   │
                    │  Drum Rack   │   │   │
                    │              │   │   │
                    │ Track 7:     │   │   │
                    │  grids-jack  │   │   │
                    │  JACK audio  │   │   │
                    │  (raw drums) │   │   │
                    │              │───┘   │
                    │ Link: tempo  │       │
                    │ TDAbleton:   │──┐    │
                    │  audio levels│  │    │
                    │  device params  │    │
                    └──────────────┘  │    │
                                     │    │
                         ┌───────────┘    │
                         │  OSC           │
                         │  (TDAbleton)   │
                         ▼                ▼
                    ┌─────────────────────────┐
                    │   TOUCHDESIGNER          │
                    │                          │
                    │   Ableton Link CHOP:     │
                    │    tempo, beat, phase,   │
                    │    bar ramp (0-1)        │
                    │                          │
                    │   OSC In CHOP (grids):   │
                    │    /trigger/bd,sd,hh     │
                    │    /state/x,y,randomness │
                    │                          │
                    │   OSC In CHOP (scion):   │
                    │    biofeedback values    │
                    │    (via desktop app)     │
                    │                          │
                    │   TDAbleton:             │
                    │    audio spectrum,       │
                    │    track levels,         │
                    │    device parameters     │
                    │                          │
                    │   Visual layers:         │
                    │    - Beat-reactive geo   │
                    │    - Biofeedback particles│
                    │    - Pattern map viz     │
                    │    - Audio-reactive FX   │
                    │                          │
                    │   Output:                │
                    │    Spout/NDI → projector  │
                    └──────────────────────────┘
```

## Performance Scenarios

### Scenario A: "Garden Set"

A plant sits on stage connected to the Pocket Scion. The performer adjusts
Grids density and Ableton effects. The audience sees TouchDesigner visuals
that react to both the plant's biology and the generated rhythms.

1. Plant biofeedback → Pocket Scion → OSC → grids-jack (X/Y modulation)
2. Same biofeedback → Pocket Scion → MIDI → Ableton (melodic voices)
3. grids-jack drum patterns → JACK audio + MIDI → Ableton (mixing/FX)
4. grids-jack trigger OSC → TouchDesigner (beat-reactive visuals)
5. Pocket Scion biofeedback OSC → TouchDesigner (organic visual layer)
6. TDAbleton → TouchDesigner (audio analysis for spectrum visuals)
7. Ableton Link keeps everything phase-locked

### Scenario B: "Improvised Duo"

Two performers: one controls the Pocket Scion (touching plants, adjusting
the device), the other live-codes grids-jack parameters via a terminal
and tweaks Ableton effects. TouchDesigner runs autonomously, reacting to
all incoming data.

### Scenario C: "Installation Mode"

No performers. Pocket Scion continuously reads a plant in a gallery.
grids-jack and Ableton generate music autonomously. TouchDesigner
produces evolving projected visuals. The installation runs indefinitely,
always different because the plant's signals are never the same twice.

## Implementation Phases

### Phase 1: OSC Foundation
- Add OSC input (Idea 1) and output (Idea 3) to grids-jack
- Test with TouchDesigner receiving trigger data
- Dependency: `liblo`

### Phase 2: Ableton Link
- Add Link sync (Idea 2) so grids-jack locks to Ableton's tempo
- Test multi-app sync: Ableton + grids-jack + TouchDesigner Link CHOP
- Dependency: Ableton Link SDK

### Phase 3: MIDI Output
- Add JACK MIDI output (Idea 4) for Ableton instrument control
- Test pattern brain mode alongside Pocket Scion MIDI in Ableton
- Dependency: existing JACK MIDI API

### Phase 4: Integration & Polish
- Test full pipeline end-to-end
- Add a config file or preset system for saving parameter mappings
- Performance profiling (ensure OSC/Link/MIDI don't impact audio latency)
- Document the setup for reproducibility

## Port Allocation Plan

| Application | Protocol | Port | Direction |
|-------------|----------|------|-----------|
| Pocket Scion Desktop App | OSC | 7000 (configurable) | Out |
| grids-jack OSC input | OSC | 7000 | In (from Scion) |
| grids-jack OSC output | OSC | 8000 | Out (to TouchDesigner) |
| TDAbleton | OSC | 9000/9001 | Bidirectional |
| AbletonOSC (optional) | OSC | 11000/11001 | Bidirectional |
| Ableton Link | UDP multicast | 20808 | Peer-to-peer |

## Why This Is The Endgame

- **Biology → Rhythm → Sound → Visuals**: A complete creative pipeline where
  living organisms generate art through code
- **Each tool does what it's best at**: Pocket Scion captures biology,
  grids-jack generates patterns algorithmically, Ableton handles mixing and
  sound design, TouchDesigner handles visuals
- **grids-jack is the bridge**: It sits between the biological input (Pocket
  Scion) and the creative output tools (Ableton, TouchDesigner), translating
  nature into rhythm
- **Everything is live**: All parameters can change in real time. No
  pre-rendered anything. The performance is different every time
- **Modular**: Each piece works independently. Start with just grids-jack +
  Ableton Link (Phase 2), add pieces as you build out the system
