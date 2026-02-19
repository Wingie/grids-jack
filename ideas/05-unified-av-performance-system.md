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
                    (USB)│        │  /min,/max,/mean,
                         │        │  /delta,/variance,
                         │        │  /deviation)
                         │        │  port 10361
                         │        ▼
                         │   ┌────────────────┐
                         │   │  grids-jack     │
                         │   │                 │
                         │   │  OSC In (10361):│
                         │   │  biofeedback -> │
                         │   │  X,Y,randomness,│◄──── Ableton Link
                         │   │  density,       │      (UDP multicast
                         │   │  humanize       │       port 20808)
                         │   │                 │
                         │   │  Grids engine   │
                         │   │  generates      │
                         │   │  patterns       │
                         │   │                 │
                         │   │  Outputs:       │
                         │   │  - JACK audio   │──── stereo drums
                         │   │  - JACK MIDI    │──── note triggers (ch 10)
                         │   │  - OSC (7770)   │──── pattern data to TD
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
                    │  levels/params  │    │
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
                    │    rampbar (0-1)         │
                    │                          │
                    │   OSC In CHOP (7770):    │
                    │    /grids/trigger/bd,sd, │
                    │    hh (Pulse Mode ON)    │
                    │    /grids/state/x,y      │
                    │                          │
                    │   OSC In CHOP (10361):   │
                    │    Pocket Scion          │
                    │    biofeedback values    │
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

## TDAbleton: Ableton-to-TouchDesigner Bridge

[TDAbleton](https://docs.derivative.ca/TDAbleton) is a built-in integration
that connects Ableton Live to TouchDesigner via MIDI Remote Scripts + Max for
Live devices communicating over OSC/UDP.

### What It Provides

| TDAbleton Component | What It Sends to TD | Use Case |
|---------------------|---------------------|----------|
| `abletonLevel` | Track audio levels (spectrum analysis) | Audio-reactive visuals from the full Ableton mix |
| `abletonMapper` | Device parameters (knobs, faders) | Map Ableton effect params to visual properties |
| `abletonSong` | Transport state, tempo, scenes | Global scene management |
| `abletonRack` | Rack macro values | Control visuals via Ableton Rack macros |

### TDAbleton Caveats

- **Undo flooding**: Changing Ableton values FROM TouchDesigner creates undo
  steps in Live, flooding the undo history. Use TDA_Rack devices to avoid this.
- **Duplicate names**: Duplicate track/device names cause confusion in the
  routing system. Give everything unique names.
- **Large Live Sets**: Can overload the OSC connection, especially on macOS.
  Use `TDA_Ignore` devices on tracks you don't need in TD.
- **Bidirectional**: TD can also control Ableton (push parameters back), but
  this should be used sparingly due to the undo issue.

## Port Allocation Plan

Each OSC application needs its own port. Plan carefully to avoid conflicts:

| Application | Protocol | Port | Direction | Notes |
|-------------|----------|------|-----------|-------|
| Pocket Scion Desktop App | OSC/UDP | 10361 | Out | Default, configurable in app |
| grids-jack OSC input | OSC/UDP | 10361 | In | Receives Pocket Scion data |
| grids-jack OSC output | OSC/UDP | 7770 | Out | Sends triggers to TD |
| TouchDesigner OSC In | OSC/UDP | 7770 | In | Receives grids triggers |
| TDAbleton (Ableton→TD) | OSC/UDP | 9000 | In (TD side) | Configurable |
| TDAbleton (TD→Ableton) | OSC/UDP | 9001 | Out (TD side) | Configurable |
| AbletonOSC (optional) | OSC/UDP | 11000/11001 | Bidirectional | Full LOM access |
| Ableton Link | UDP multicast | 20808 | Peer-to-peer | Auto-discovery |

**Key rule**: Multiple apps CANNOT bind to the same UDP port for receiving.
The Pocket Scion desktop app broadcasts on 10361 -- both grids-jack and
TouchDesigner can receive this if they listen on that port (but only one
process per port). Solution: grids-jack listens on 10361, and forwards
relevant data to TD via its own OSC output on 7770.

## Ableton Live on Linux

Ableton Live does not run natively on Linux. Options:

### Option A: Ableton via Wine + wineASIO (Same Machine)

- Audio works well via wineASIO → JACK
- MIDI has known issues (JACK MIDI note-on/off may not work reliably)
- Workaround: Use `a2jmidid` bridge or `snd-virmidi` for MIDI routing
- TDAbleton: Untested under Wine, may work since it uses standard OSC

### Option B: Multi-Machine (Recommended for Performance)

- **Machine 1 (Linux)**: grids-jack + TouchDesigner
- **Machine 2 (Windows/macOS)**: Ableton Live 12 + Pocket Scion
- **Network**: Ethernet cable between machines
- **Sync**: Ableton Link works over network (auto-discovery via UDP multicast)
- **MIDI**: Network MIDI via `qmidinet` or `rtpMIDI`
- **OSC**: All OSC messages work over network (just change target IP)
- **Video**: TouchDesigner NDI Out for remote monitoring (10-60ms latency)

### Option C: Bitwig Studio on Linux (Alternative DAW)

- Native Linux support, JACK integration
- Supports Ableton Link
- Different plugin ecosystem but handles MIDI routing well

## Multi-Machine Latency Budget

| Protocol | Same Machine | Over Network |
|----------|-------------|--------------|
| Ableton Link | < 1ms | < 1ms (UDP multicast) |
| OSC messages | < 1ms | 1-5ms (UDP) |
| JACK MIDI | 0 (sample-accurate) | N/A (use network MIDI) |
| Network MIDI | N/A | 2-10ms |
| Spout/Syphon video | Sub-millisecond | N/A (same machine only) |
| NDI video | N/A | 10-60ms |

For a two-machine setup, total sync error between Ableton (machine 2) and
TouchDesigner (machine 1) is typically < 10ms -- well under perceptual
threshold for audio-visual sync (~40ms).

## Performance Profiling Concerns

When grids-jack handles OSC I/O, Ableton Link, and MIDI output simultaneously,
the JACK audio callback must still meet its deadline:

| Feature | CPU Cost in Audio Callback |
|---------|--------------------------|
| Grids pattern generation | ~1-2 us (simple integer math) |
| Sample playback (3 voices) | ~10-20 us (memory reads) |
| Ableton Link session state | ~1 us (lock-free atomic read) |
| JACK MIDI event writes | < 1 us (buffer writes) |
| OSC ringbuffer writes | < 1 us (memcpy to ringbuffer) |
| **Total** | **~15-25 us** |

At 48kHz with 128-frame buffers, the deadline is 2,667 us. The full pipeline
uses < 1% of the available budget. No issues expected.

The OSC sender thread and liblo server thread run outside the audio callback
and don't affect audio latency.

## Performance Scenarios

### Scenario A: "Garden Set"

A plant sits on stage connected to the Pocket Scion. The performer adjusts
Grids density and Ableton effects. The audience sees TouchDesigner visuals
that react to both the plant's biology and the generated rhythms.

1. Plant biofeedback → Pocket Scion → OSC (10361) → grids-jack (X/Y modulation)
2. Same biofeedback → Pocket Scion → MIDI → Ableton (melodic voices, ch 1-5)
3. grids-jack drum patterns → JACK MIDI → Ableton Drum Rack (ch 10)
4. grids-jack trigger OSC (7770) → TouchDesigner (beat-reactive visuals)
5. Pocket Scion OSC → TouchDesigner (organic visual layer)
6. TDAbleton (9000) → TouchDesigner (audio spectrum from Ableton mix)
7. Ableton Link (20808) keeps everything phase-locked

### Scenario B: "Improvised Duo"

Two performers: one controls the Pocket Scion (touching plants, adjusting
the device), the other live-codes grids-jack parameters via terminal CLI
flags and tweaks Ableton effects. TouchDesigner runs autonomously, reacting
to all incoming data streams.

### Scenario C: "Installation Mode"

No performers. Pocket Scion continuously reads a plant in a gallery.
grids-jack and Ableton generate music autonomously. TouchDesigner produces
evolving projected visuals. The installation runs indefinitely -- always
different because the plant's signals are never the same twice.

## Implementation Phases

### Phase 1: OSC Foundation (Ideas 1 + 3)
- Add liblo dependency to CMakeLists.txt
- Implement OSC input (Pocket Scion biofeedback → pattern params)
- Implement OSC output (triggers → TouchDesigner via ringbuffer)
- Test with `oscsend` / `oscdump` and TouchDesigner OSC In CHOP
- **New dependency**: `liblo` (`sudo apt install liblo-dev`)

### Phase 2: Ableton Link (Idea 2)
- Add Link SDK as git submodule
- Implement `LinkJackBridge` with `HostTimeFilter`
- Replace frame-counting clock with Link beat-phase-driven pulses
- Test: Ableton + grids-jack + TD Link CHOP all synced
- **New dependency**: Link SDK (header-only, vendored as submodule)

### Phase 3: MIDI Output (Idea 4)
- Register JACK MIDI output port
- Add note-on/note-off in ProcessTriggers() with sample-accurate timing
- Implement note-off queue with 50ms fixed duration
- Test: grids-jack MIDI → Ableton Drum Rack alongside Pocket Scion MIDI
- **No new dependency** (JACK MIDI is in libjack)

### Phase 4: Integration and Polish
- Test full pipeline end-to-end
- Add JSON config file for port numbers, MIDI channels, OSC mappings
- Performance profiling with `jack_iodelay` and `jack_lsp -l`
- Create startup script that launches all components
- Document the setup

## Configuration File (Phase 4)

```json
{
  "osc": {
    "listen_port": 10361,
    "send_host": "127.0.0.1",
    "send_port": 7770,
    "pocket_scion_mappings": {
      "/mean": { "param": "pattern_x", "min": 200, "max": 800, "range": [0, 255] },
      "/delta": { "param": "pattern_y", "min": 0, "max": 50, "range": [0, 255] },
      "/deviation": { "param": "randomness", "min": 0, "max": 10, "range": [0, 255] },
      "/variance": { "param": "humanize", "min": 0, "max": 100, "range": [0.0, 1.0] }
    }
  },
  "link": {
    "enabled": true,
    "initial_bpm": 120,
    "quantum": 4.0
  },
  "midi": {
    "enabled": true,
    "channel": 10,
    "velocity_low": 49,
    "velocity_high": 127,
    "note_duration_ms": 50
  }
}
```

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
- **< 1% CPU overhead**: All integration features (OSC, Link, MIDI) cost
  ~15-25 us in the audio callback against a 2,667 us deadline

## References

- [TDAbleton documentation](https://docs.derivative.ca/TDAbleton)
- [TDAbleton user guide](https://derivative.ca/UserGuide/TDAbleton)
- [Ableton + TouchDesigner AV setup (AllTD)](https://alltd.org/ableton-touchdesigner-how-to-build-audio-visual-live-set/)
- [Ableton Link overview](https://ableton.github.io/link/)
- [AbletonOSC (full LOM via OSC)](https://github.com/ideoforms/AbletonOSC)
- [Pocket Scion official site](https://pocketscion.com/)
- [Spout for Windows (GPU texture sharing)](https://spout.zeal.co/)
- [NDI SDK](https://ndi.video/for-developers/ndi-sdk/)
