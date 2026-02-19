# Idea 2: Ableton Link Integration for Tempo/Phase Sync

## Concept

Replace grids-jack's internal BPM clock with Ableton Link so it locks to
the shared session tempo and phase alongside Ableton Live 12 and
TouchDesigner. All three applications stay perfectly synchronized without
any manual BPM matching.

## How It Works

Currently, grids-jack uses an internal clock based on frame counting in
`pattern_generator_wrapper.cpp`:

```cpp
frames_per_pulse_ = sample_rate_ * 60.0 / (bpm_ * 24.0);
// ...
frames_since_last_tick_++;
if (frames_since_last_tick_ >= frames_per_pulse_) {
    frames_since_last_tick_ -= frames_per_pulse_;
    grids::PatternGenerator::TickClock(1);
}
```

With Ableton Link, grids-jack joins the Link session and derives pulse timing
from the shared beat phase. Instead of counting frames to determine when pulses
occur, you ask Link "what beat/phase is it right now?" and derive pulse
positions from that.

## Implementation

### Step 1: Add Link SDK as a Git Submodule

```bash
cd /home/user/grids-jack
git submodule add https://github.com/Ableton/link.git link
cd link && git submodule update --init --recursive  # pulls asio-standalone
```

### Step 2: Core API Usage

```cpp
#include <ableton/Link.hpp>
#include <ableton/link/HostTimeFilter.hpp>

// Initialize with starting tempo
ableton::Link link(120.0);
link.enable(true);
link.enableStartStopSync(true);  // share transport state

// Optional callbacks (called from background thread, NOT audio thread)
link.setNumPeersCallback([](std::size_t numPeers) {
    fprintf(stderr, "Link peers: %zu\n", numPeers);
});
link.setTempoCallback([](double bpm) {
    fprintf(stderr, "Link tempo: %.1f BPM\n", bpm);
});
```

### Step 3: JACK + Link Clock Conversion

The critical challenge: JACK works in sample counts, Link works in microseconds.
The Link SDK provides `HostTimeFilter` which performs **linear regression**
between system clock time and sample-count time for stable, jitter-free
timestamps.

From the [official JACK example](https://github.com/Ableton/link/blob/master/examples/linkaudio/AudioPlatform_Jack.ipp):

```cpp
class LinkJackBridge {
    ableton::Link mLink;
    ableton::link::HostTimeFilter<ableton::Link::Clock> mHostTimeFilter;
    double mSampleTime;     // continuously incrementing sample counter
    double mSampleRate;

    int audioCallback(jack_nframes_t nframes) {
        // Convert sample time to host time using the filter
        const auto hostTime = mHostTimeFilter.sampleTimeToHostTime(mSampleTime);
        mSampleTime += nframes;  // MUST never be reset

        // Account for output latency
        const auto bufferBeginAtOutput = hostTime + mOutputLatency.load();

        // Capture session state (lock-free, realtime-safe)
        auto sessionState = mLink.captureAudioSessionState();
        double tempo = sessionState.tempo();

        // Process each sample to find exact pulse boundaries
        for (jack_nframes_t i = 0; i < nframes; ++i) {
            auto sampleHostTime = bufferBeginAtOutput
                + std::chrono::microseconds(
                    llround(static_cast<double>(i) / mSampleRate * 1.0e6));

            double beat = sessionState.beatAtTime(sampleHostTime, quantum);
            double phase = sessionState.phaseAtTime(sampleHostTime, quantum);
            // ... derive 24 PPQN pulses from beat position
        }
        return 0;
    }
};
```

### Step 4: Deriving 24 PPQN from Link Beat Phase

The Grids pattern generator uses 24 PPQN: `kPulsesPerStep = 3`,
`kStepsPerPattern = 32`, so 96 pulses per pattern = 24 pulses per quarter note.

```cpp
static int g_last_pulse_index = -1;

// Inside the per-sample loop:
double beat = sessionState.beatAtTime(sampleHostTime, 4.0);  // quantum=4
int pulseIndex = static_cast<int>(std::floor(beat * 24.0));

if (pulseIndex != g_last_pulse_index && g_last_pulse_index >= 0) {
    int pulsesElapsed = pulseIndex - g_last_pulse_index;
    if (pulsesElapsed < 0) pulsesElapsed += 96;  // wrapped around

    for (int p = 0; p < pulsesElapsed; p++) {
        grids::PatternGenerator::TickClock(1);
        uint8_t state = grids::PatternGenerator::state();
        // ProcessTriggers(state) at exact sample position i
        grids::PatternGenerator::IncrementPulseCounter();
    }
}
g_last_pulse_index = pulseIndex;
```

### Step 5: Output Latency Callback

```cpp
static void latencyCallback(jack_latency_callback_mode_t mode, void* userData) {
    if (mode == JackPlaybackLatency) {
        jack_latency_range_t range;
        jack_port_get_latency_range(outputPort, JackPlaybackLatency, &range);
        bridge->mOutputLatency.store(
            std::chrono::microseconds(
                llround(1.0e6 * range.max / sampleRate)));
    }
}
```

### Step 6: Build Integration (CMakeLists.txt)

```cmake
set(CMAKE_CXX_STANDARD 11)  # minimum for Link

# Add Link include paths
include_directories(
    ${CMAKE_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/link/include
    ${CMAKE_SOURCE_DIR}/link/modules/asio-standalone/asio/include
    ${JACK_INCLUDE_DIRS}
    ${SNDFILE_INCLUDE_DIRS}
)

# Platform defines
add_definitions(-DLINK_PLATFORM_LINUX=1 -DASIO_STANDALONE=1)

# Link needs pthread
target_link_libraries(grids-jack
    ${JACK_LIBRARIES}
    ${SNDFILE_LIBRARIES}
    pthread
)
```

No external packages to install -- Link is entirely header-only with ASIO
bundled as a submodule.

### CLI Flags

- `-L` -- enable Ableton Link (default: off, uses internal clock)
- `-b` -- still sets initial tempo suggestion when Link starts
- `--link-quantum <beats>` -- set quantum for phase sync (default: 4.0)

## What TouchDesigner Sees

TouchDesigner's [Ableton Link CHOP](https://docs.derivative.ca/Ableton_Link_CHOP)
exposes the same session data that grids-jack reads:

| TD CHOP Channel | grids-jack Equivalent |
|---|---|
| `tempo` | `sessionState.tempo()` |
| `beats` | `sessionState.beatAtTime(hostTime, quantum)` |
| `phase` | `sessionState.phaseAtTime(hostTime, quantum)` |
| `rampbeat` | `fmod(beat, 1.0)` |
| `rampbar` | `phase / quantum` |
| `bar` | `floor(beat / quantum)` |
| `numpeers` | `link.numPeers()` |

All three apps (Ableton, grids-jack, TD) see the same beats and phase because
they are all Link peers on the same network session.

## Signal Flow

```
Ableton Live 12                 TouchDesigner
(master or peer)                Ableton Link CHOP
      |                               |
      +----------- Link Session ------+
      |          (UDP multicast       |
      |           port 20808)         |
      |                               |
  grids-jack (LinkJackBridge)
      |
  HostTimeFilter: samples -> microseconds
      |
  captureAudioSessionState() (lock-free)
      |
  beatAtTime() -> 24 PPQN pulses
      |
  PatternGeneratorWrapper ticks in sync
      |
  Drum patterns locked to session tempo and phase
```

## Existing Open-Source References

- **[Ableton/link](https://github.com/Ableton/link)** -- SDK with official
  JACK example in `examples/linkaudio/AudioPlatform_Jack.*`
- **[rncbc/jack_link](https://github.com/rncbc/jack_link)** -- Bridges JACK
  Transport BBT with Link. Shows clock conversion and `forceBeatAtTime()` usage.

## Why This Is Interesting

- **Zero-config sync**: No MIDI clock cables, no manual BPM entry. Start
  Ableton, start grids-jack with `-L`, they find each other automatically
- **Phase-coherent**: Not just tempo-matched -- the downbeats align. Grids
  patterns land on the beat with Ableton clips and TouchDesigner's beat ramp
- **Multi-machine**: Link works over local network. Run grids-jack on a
  Raspberry Pi or second laptop and it still syncs
- **Dynamic tempo**: Change BPM in Ableton during performance, grids-jack
  follows smoothly via `captureAudioSessionState()` each callback
- **Coexists with Pocket Scion**: Pocket Scion feeds MIDI notes to Ableton
  for melodic content, while grids-jack provides the synchronized rhythmic
  foundation
- **HostTimeFilter warm-up**: The linear regression needs a few callbacks to
  stabilize. After that, timing is rock solid.

## Complexity

Medium-low. The Link SDK is well-documented with an official JACK example.
The main architectural change is replacing `frames_since_last_tick_` /
`frames_per_pulse_` with Link's beat-phase-driven pulse detection. The
`TickClock()` / `IncrementPulseCounter()` interface stays the same.

## References

- [Ableton Link SDK](https://github.com/Ableton/link)
- [Link.hpp API header](https://github.com/Ableton/link/blob/master/include/ableton/Link.hpp)
- [Official JACK example](https://github.com/Ableton/link/blob/master/examples/linkaudio/AudioPlatform_Jack.hpp)
- [HostTimeFilter source](https://github.com/Ableton/link/blob/master/include/ableton/link/HostTimeFilter.hpp)
- [jack_link bridge](https://github.com/rncbc/jack_link)
- [Ableton Link CHOP docs (TD)](https://docs.derivative.ca/Ableton_Link_CHOP)
- [Ableton Link protocol overview](https://ableton.github.io/link/)
