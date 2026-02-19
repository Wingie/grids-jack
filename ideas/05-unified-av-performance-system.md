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

## Important Platform Note: TouchDesigner Does NOT Run on Linux

TouchDesigner is Windows-only (and macOS). It has no Linux build. This means
the multi-machine setup described in the "Ableton Live on Linux" section below
is likely **required** for a Linux-based grids-jack workflow -- you'll need a
Windows machine for TouchDesigner.

**Linux alternatives for visuals** (if you want everything on one Linux machine):
- **openFrameworks** with ofxOsc -- receives OSC triggers, C++ like grids-jack
- **Processing** with oscP5 -- quick prototyping, good for generative art
- **Pure Data + GEM** -- visual patching, native OSC/MIDI support
- **Godot Engine** -- game engine with OSC plugins, real-time 3D

All of these can receive grids-jack's OSC output on port 7770 identically to
how TouchDesigner would.

## TDAbleton: Ableton-to-TouchDesigner Bridge

[TDAbleton](https://docs.derivative.ca/TDAbleton) is a three-layer system:

1. **MIDI Remote Script** (Python) -- installed in Ableton's MIDI Remote Scripts
   folder. Uses the Live Object Model (LOM) to read/write nearly everything in
   a Live set. Communicates with TD over OSC/UDP.

2. **Max for Live devices** -- placed on tracks for data the Remote Script
   can't access efficiently:
   - **TDA Master** -- on Master track, manages connection, shows IP/ports
   - **TDA Level** -- per-track volume metering via OSC
   - **TDA Rack OSC** -- fast bidirectional control of up to 16 rack macros
   - **TDA MIDI** -- routes MIDI note/CC/program change data
   - **TDA_Ignore** -- excludes tracks from scanning (performance optimization)

3. **TouchDesigner COMP nodes** (drag-and-drop from TDAbleton palette):

| TD Component | What It Provides | Direction |
|---|---|---|
| `abletonSong` | Transport, scenes, cue points, tempo, beat CHOP | Read + Write |
| `abletonTrack` | Clip slots, playing slot, output meters | Read + Write |
| `abletonDeviceParameters` | All params on a device | Read only |
| `abletonRack` | 16 rack macros via fast OSC (bypasses Remote Script) | Read + Write |
| `abletonLevel` | Volume levels + spectrum (with TDA Audio Analyzer Rack) | Read only |
| `abletonMIDI` | MIDI events with callbacks | Read + Write |

### TDAbleton Caveats

- **Undo flooding**: Changing Ableton values FROM TouchDesigner creates undo
  steps in Live, flooding the undo history. Use TDA_Rack devices to avoid this.
- **Duplicate names**: TDAbleton uses names for LOM navigation. Duplicate
  track/device names cause ambiguity and bugs. Give everything unique names.
- **Large Live Sets**: Can overload the OSC connection, especially on macOS.
  Use `TDA_Ignore` devices on tracks you don't need in TD.
- **Output meter bug**: `output_meter_left/right` on `abletonTrack` only
  updates when meters are **visible in the Ableton GUI**. Minimizing a group
  stops updates. Use `abletonLevel` with TDA Level M4L devices instead.
- **M4L loading error**: Starting Ableton via TDAbleton's "Start" button can
  fail to load M4L devices. Start Ableton separately, then connect.
- **Version pinning**: TDAbleton versions are tightly coupled to both TD and
  Ableton versions. Don't mix versions.

### AbletonOSC as Alternative/Complement

[AbletonOSC](https://github.com/ideoforms/AbletonOSC) is a lighter-weight
MIDI Remote Script exposing the full LOM over OSC without Max for Live.
Listens on port 11000, replies on 11001. Useful for programmatic control from
grids-jack directly (e.g., triggering scenes, reading tempo) without needing
TouchDesigner in the loop. Can coexist with TDAbleton.

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
- At 256 samples: ~10.7ms latency, stable. At 64 samples: 4ms possible with
  tuned systems (users report years of professional use)
- MIDI has known issues (JACK MIDI note-on/off may not work reliably)
- Workaround: Use `a2jmidid` bridge or `snd-virmidi` for MIDI routing
- **Pin your Wine version** once working -- Wine-Staging updates frequently
  break wineASIO and VST compatibility
- TDAbleton Remote Scripts work under Wine (they run inside Ableton's Python)

### Option B: Multi-Machine (Recommended for Performance)

Since neither Ableton nor TouchDesigner runs natively on Linux, the most
reliable setup is:

- **Machine 1 (Linux)**: grids-jack (native JACK)
- **Machine 2 (Windows)**: Ableton Live 12 + TouchDesigner + Pocket Scion
- **Network**: Ethernet cable between machines (not WiFi -- causes xruns)
- **Sync**: Ableton Link over network (auto-discovery, < 1ms)
- **MIDI**: Network MIDI via `rtpmidid` (RTP-MIDI with journaling) or `qmidinet`
- **OSC**: grids-jack sends triggers to TD over network (just change target IP)
- **Audio**: Both machines into a hardware mixer (avoids network audio latency)

```
Machine A (Linux)              Machine B (Windows)
┌───────────────────┐         ┌──────────────────────┐
│ grids-jack        │         │ Ableton Live 12      │
│  - Link peer      │◄─Link──►│  - TDAbleton RS      │
│  - OSC out (7770) │──OSC───►│  - Pocket Scion MIDI │
│  - MIDI out       │──MIDI──►│  - Drum Rack (ch 10) │
│                   │         │                      │
│                   │         │ TouchDesigner         │
│                   │──OSC───►│  - OSC In CHOP (7770)│
│                   │         │  - Link CHOP          │
│                   │         │  - TDAbleton COMPs    │
│                   │         │  - Spout → projector  │
└────┬──────────────┘         └──────┬───────────────┘
     │ audio out                     │ audio out
     ▼                               ▼
  ┌──────────────────────────────────────┐
  │        Hardware Mixer                │
  └──────────────────────────────────────┘
```

### Option C: Bitwig Studio on Linux (Alternative DAW)

- Native Linux support (including ARM), JACK integration
- Built-in Ableton Link support
- OSC via [DrivenByMoss](https://github.com/git-moss/DrivenByMoss) controller scripts
- Clip launcher, modular sound design, VST3/CLAP hosting
- Users report stable live performance on Linux

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

## Configuration Strategy

Following patterns from TidalCycles/SuperCollider, use a layered approach:

1. **Compiled defaults** (always work for single-machine use)
2. **Config file** (INI/TOML, text-editable, git-friendly)
3. **Environment variables** (override config -- grids-jack already reads
   `PARTS`, `STEPS`, `LFO`, `VERBOSE` from env)
4. **CLI flags** (override everything -- `-O`, `-T`, `-L`, `-M`, `-b`)

```ini
# ~/.config/grids-jack/config.toml
[osc]
listen_port = 10361
send_target = "192.168.1.50:7770"

[osc.pocket_scion_mappings]
mean = { param = "pattern_x", input_min = 200, input_max = 800, output_range = [0, 255] }
delta = { param = "pattern_y", input_min = 0, input_max = 50, output_range = [0, 255] }
deviation = { param = "randomness", input_min = 0, input_max = 10, output_range = [0, 255] }

[link]
enabled = true
initial_bpm = 120
quantum = 4.0

[midi]
enabled = true
channel = 10
velocity_low = 49
velocity_high = 127
note_duration_ms = 50

[audio]
sample_dir = "/home/user/samples/garden-set"
parts = 6
```

For parsing, [inih](https://github.com/benhoyt/inih) (C, ~600 lines) or
[toml11](https://github.com/ToruNiina/toml11) (header-only C++11) are
lightweight options.

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
- [TDAbleton system components](https://docs.derivative.ca/TDAbleton_System_Components)
- [TDAbleton user guide](https://derivative.ca/UserGuide/TDAbleton)
- [Cyanea Studio TDAbleton guide](https://www.cyaneastudio.com/blog/touchdesigner-and-ableton-a-beginners-guide-to-tdableton)
- [AbletonOSC (full LOM via OSC)](https://github.com/ideoforms/AbletonOSC)
- [AbletonOSC NIME paper](https://nime.org/proceedings/2023/nime2023_60.pdf)
- [Ableton Link overview](https://ableton.github.io/link/)
- [Ableton Link FAQ](https://help.ableton.com/hc/en-us/articles/209776125)
- [Ableton on Linux via Wine](https://github.com/korewaChino/live-on-linux)
- [DrivenByMoss (Bitwig OSC)](https://github.com/git-moss/DrivenByMoss)
- [rtpmidid (network MIDI on Linux)](https://github.com/davidmoreno/rtpmidid)
- [Pocket Scion official site](https://pocketscion.com/)
- [TidalCycles MIDI/OSC config](https://tidalcycles.org/docs/configuration/MIDIOSC/midi/)
- [Ross Bencina -- Realtime Audio Programming 101](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing)
- [Spout for Windows](https://spout.zeal.co/)
- [NDI SDK](https://ndi.video/for-developers/ndi-sdk/)
