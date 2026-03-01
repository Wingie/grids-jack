# Idea 2: Ableton Link Integration for Tempo/Phase Sync

## Concept

Replace grids-jack's internal BPM clock with Ableton Link so it locks to
the shared session tempo and phase alongside Ableton Live 12 and
TouchDesigner. All three applications stay perfectly synchronized without
any manual BPM matching.

## How It Works

Currently, grids-jack uses an internal clock based on frame counting
(in `pattern_generator_wrapper.cpp`):

```cpp
frames_per_pulse_ = sample_rate_ * 60.0 / (bpm_ * 24.0);
// ...
frames_since_last_tick_++;
if (frames_since_last_tick_ >= frames_per_pulse_) {
    frames_since_last_tick_ -= frames_per_pulse_;
    grids::PatternGenerator::TickClock(1);
}
```

With Ableton Link, grids-jack joins the Link session and derives its pulse
timing from the shared tempo and beat phase. When you change BPM in Ableton,
grids-jack follows. When TouchDesigner's Ableton Link CHOP reports a beat,
grids-jack is hitting that same beat.

## Implementation Details

### Ableton Link SDK

**Repository**: [github.com/Ableton/link](https://github.com/Ableton/link)

Header-only C++ library. Primary header: `include/ableton/Link.hpp`. The SDK
provides two pairs of capture/commit functions: `captureAudioSessionState()` /
`commitAudioSessionState()` for the realtime audio thread (lock-free), and
`captureAppSessionState()` / `commitAppSessionState()` for the UI thread (may block).

### Key API

```cpp
#include <ableton/Link.hpp>
#include <ableton/link/HostTimeFilter.hpp>

// Initialize with initial BPM
ableton::Link link(120.0);
link.enable(true);
link.enableStartStopSync(true);

// Register callbacks (called from background thread, NOT audio thread)
link.setNumPeersCallback([](std::size_t numPeers) {
    fprintf(stderr, "Link peers: %zu\n", numPeers);
});
link.setTempoCallback([](double bpm) {
    fprintf(stderr, "Link tempo: %.1f BPM\n", bpm);
});
```

### The Clock Conversion Problem

JACK works in samples. Link works in microseconds. The Link SDK provides
`HostTimeFilter` which performs linear regression between system clock time
and sample-count time to produce stable, jitter-free timestamps.

From the official JACK example (`examples/linkaudio/AudioPlatform_Jack.ipp`):

```cpp
ableton::link::HostTimeFilter<ableton::Link::Clock> mHostTimeFilter;
double mSampleTime = 0.0;  // continuously incrementing, NEVER reset

int audioCallback(jack_nframes_t nframes) {
    // Convert sample time to Link host time
    const auto hostTime = mHostTimeFilter.sampleTimeToHostTime(mSampleTime);
    mSampleTime += nframes;  // must increment every callback

    // Account for output latency
    const auto bufferBeginAtOutput = hostTime + mOutputLatency.load();

    // Capture session state (lock-free, realtime-safe)
    auto sessionState = link.captureAudioSessionState();

    double quantum = 4.0;  // 4/4 time
    double tempo = sessionState.tempo();

    // Per-sample processing for precise timing
    for (jack_nframes_t i = 0; i < nframes; ++i) {
        auto sampleHostTime = bufferBeginAtOutput
            + std::chrono::microseconds(
                llround(static_cast<double>(i) / sampleRate * 1.0e6));

        double beat = sessionState.beatAtTime(sampleHostTime, quantum);
        double phase = sessionState.phaseAtTime(sampleHostTime, quantum);
        // ... use beat/phase to drive pattern generator
    }
    return 0;
}
```

### Deriving 24 PPQN Pulses from Link Beat Phase

Grids uses 24 PPQN (pulses per quarter note) with `kPulsesPerStep = 3` and
`kStepsPerPattern = 32`, giving 96 pulses per 4-beat pattern. To derive
these from Link's continuous beat value:

```cpp
static int lastPulseIndex = -1;

double currentBeat = sessionState.beatAtTime(hostTime, 4.0);

// 24 PPQN: a pulse fires every 1/24th of a beat
int currentPulse = static_cast<int>(std::floor(currentBeat * 24.0));

if (currentPulse != lastPulseIndex && lastPulseIndex >= 0) {
    int pulsesElapsed = currentPulse - lastPulseIndex;
    if (pulsesElapsed < 0) pulsesElapsed += 96;  // wrapped around pattern

    for (int p = 0; p < pulsesElapsed; p++) {
        grids::PatternGenerator::TickClock(1);
        uint8_t state = grids::PatternGenerator::state();
        // ... ProcessTriggers(state)
        grids::PatternGenerator::IncrementPulseCounter();
    }
}
lastPulseIndex = currentPulse;
```

**Key architectural change**: Instead of counting frames to determine when
pulses occur (free-running clock), you ask Link "what beat/phase is it right
now?" and derive pulse positions from that. The `frames_since_last_tick_` /
`frames_per_pulse_` mechanism becomes unnecessary.

### Output Latency Callback

```cpp
static void latencyCallback(jack_latency_callback_mode_t mode, void* userData) {
    if (mode == JackPlaybackLatency) {
        jack_latency_range_t range;
        jack_port_get_latency_range(outputPort, JackPlaybackLatency, &range);
        outputLatency.store(
            std::chrono::microseconds(llround(1.0e6 * range.max / sampleRate)));
    }
}
```

### Beat Boundary Detection (for downbeat events)

Detect when a beat boundary falls within a buffer (used by the official Link
metronome example):

```cpp
// If phase wrapped around, a new beat just started
if (sessionState.phaseAtTime(currentTime, 1) <
    sessionState.phaseAtTime(previousTime, 1)) {
    // New beat! Fire downbeat event
}
```

### Build Integration

```bash
# Add as git submodule
cd /home/user/grids-jack
git submodule add https://github.com/Ableton/link.git link
cd link && git submodule update --init --recursive  # pulls asio-standalone
```

**CMakeLists.txt additions**:

```cmake
# Ableton Link (header-only)
include_directories(
    ${CMAKE_SOURCE_DIR}/link/include
    ${CMAKE_SOURCE_DIR}/link/modules/asio-standalone/asio/include
)
add_definitions(-DLINK_PLATFORM_LINUX=1 -DASIO_STANDALONE=1)

# Link needs pthread (already linked for most JACK apps)
target_link_libraries(grids-jack ${JACK_LIBRARIES} ${SNDFILE_LIBRARIES} pthread)
```

No additional shared libraries needed -- Link is purely header-only.

## What TouchDesigner Sees

When grids-jack joins a Link session, TouchDesigner's
[Ableton Link CHOP](https://docs.derivative.ca/Ableton_Link_CHOP) exposes
the same shared data:

| TD CHOP Channel | grids-jack Equivalent |
|---|---|
| `tempo` | `sessionState.tempo()` |
| `beats` | `sessionState.beatAtTime(hostTime, quantum)` |
| `phase` | `sessionState.phaseAtTime(hostTime, quantum)` |
| `rampbeat` | `fmod(beat, 1.0)` |
| `rampbar` | `phase / quantum` (0-1 per bar) |
| `bar` | `floor(beat / quantum)` |
| `beat` | `floor(fmod(beat, quantum)) + 1` (1-based) |
| `numpeers` | `link.numPeers()` |

All three tools (Ableton, grids-jack, TouchDesigner) share the same beat grid.

## Signal Flow

```
Ableton Live 12                 TouchDesigner
(master or peer)                Ableton Link CHOP
      |                               |
      +----------- Link Session ------+
      |            (UDP multicast     |
      |             port 20808)       |
      |                               |
  grids-jack (Link peer)
      |
  HostTimeFilter converts JACK samples -> Link microseconds
      |
  captureAudioSessionState() reads shared tempo + phase
      |
  24 PPQN pulses derived from beat position
      |
  PatternGeneratorWrapper ticks in sync
      |
  Drum patterns locked to session tempo + phase
```

## Existing Reference Projects

- **[Ableton/link official JACK example](https://github.com/Ableton/link/tree/master/examples/linkaudio)** --
  `AudioPlatform_Jack.hpp/.ipp` shows the complete HostTimeFilter + JACK integration
- **[rncbc/jack_link](https://github.com/rncbc/jack_link)** -- Bridges JACK transport
  BBT (Bar-Beat-Tick) with Link. Shows how to convert Link beats to bar/beat/tick

## Why This Is Interesting

- **Zero-config sync**: No MIDI clock cables, no manual BPM entry. Start
  Ableton, start grids-jack with `-L`, they find each other automatically
- **Phase-coherent**: Not just tempo-matched -- the downbeats align. Grids
  patterns land on the beat with Ableton clips and TouchDesigner's beat ramp
- **Multi-machine**: Link works over local network. Run grids-jack on a
  Raspberry Pi or second laptop and it still syncs
- **Dynamic tempo**: Change BPM in Ableton during performance, grids-jack
  follows smoothly via `sessionState.tempo()`
- **Coexists with Pocket Scion**: Pocket Scion feeds MIDI notes to Ableton
  for melodic content, while grids-jack provides the synchronized rhythmic
  foundation

## Complexity

Medium. The Link SDK is well-documented with a complete JACK example. The main
challenge is correctly deriving 24 PPQN pulse timing from Link's continuous beat
phase. The HostTimeFilter needs a warm-up period of several callbacks before
estimates stabilize. The SDK is designed for exactly this use case and provides
lock-free thread-safe access from audio callbacks.

## References

- [Ableton Link GitHub](https://github.com/Ableton/link)
- [Link.hpp API header](https://github.com/Ableton/link/blob/master/include/ableton/Link.hpp)
- [Official JACK example](https://github.com/Ableton/link/tree/master/examples/linkaudio)
- [HostTimeFilter.hpp](https://github.com/Ableton/link/blob/master/include/ableton/link/HostTimeFilter.hpp)
- [rncbc/jack_link](https://github.com/rncbc/jack_link) -- JACK transport bridge
- [Ableton Link CHOP (TouchDesigner)](https://docs.derivative.ca/Ableton_Link_CHOP)
- [Ableton Link documentation](https://ableton.github.io/link/)
