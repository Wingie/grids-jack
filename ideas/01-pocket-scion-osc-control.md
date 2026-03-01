# Idea 1: Pocket Scion Biofeedback Drives Grids Patterns via OSC

## Concept

Add an OSC input listener to grids-jack so the Pocket Scion's biofeedback
data can control pattern generation parameters in real time. Plants/fungi/skin
literally shape the drum patterns.

## How It Works

The Pocket Scion desktop app broadcasts 6 biofeedback values over OSC on
**UDP port 10361** (configurable in the companion app). The values are derived
from a 10-sample sliding window of resistance measurements, all in milliseconds
(the period of the pulse oscillator tracking resistance changes):

| OSC Address | Type | Description |
|-------------|------|-------------|
| `/min` | float | Shortest pulse period in the window (ms) |
| `/max` | float | Longest pulse period in the window (ms) |
| `/mean` | float | Average pulse period across the window (ms) |
| `/delta` | float | Spread: `/max` minus `/min` (ms) |
| `/variance` | float | Average squared deviation from `/mean` |
| `/deviation` | float | Square root of `/variance` (standard deviation) |

These map naturally onto grids-jack's core parameters:

| Pocket Scion OSC Value | grids-jack Parameter | Effect |
|------------------------|---------------------|--------|
| `/mean`                | X position (0-255)  | Shifts drum style along the horizontal pattern map axis |
| `/delta`               | Y position (0-255)  | Shifts drum style along the vertical pattern map axis |
| `/deviation`           | Randomness (0-255)  | More biological variation = more pattern chaos |
| `/max`                 | Density BD (0-255)  | Peak signals = heavier kick patterns |
| `/min`                 | Density HH (0-255)  | Quiet signals = sparser hi-hat activity |
| `/variance`            | Humanize (0.0-1.0)  | Maps to timing jitter amount |

## Implementation

### Library Choice: liblo (Recommended)

| Feature | liblo | oscpack | oscpkt | oscpp |
|---------|-------|---------|--------|-------|
| Language | C (C++ wrapper) | C++ | C++ | C++11 |
| Header-only | No (shared lib) | No | Yes (~650 lines) | Yes |
| Built-in threaded server | Yes | No | No | No |
| pkg-config | Yes | No | N/A | N/A |
| License | LGPL 2.1+ | BSD | Zlib-like | ISC |

**liblo wins** because it provides a built-in threaded server (`lo_server_thread`)
that handles all socket I/O and threading, and integrates via pkg-config which
grids-jack already uses for JACK and sndfile.

### Thread Safety: Atomics (Pattern A)

The JACK process callback runs on a realtime thread that **must never block**.
liblo's server thread dispatches callbacks on a separate non-realtime thread.
Use `std::atomic<float>` for parameter passing -- lock-free on all modern x86/ARM:

```cpp
#include <atomic>

struct OscParams {
    std::atomic<uint8_t> pattern_x{128};
    std::atomic<uint8_t> pattern_y{128};
    std::atomic<uint8_t> randomness{0};
    std::atomic<float> humanize{0.0f};
    // Pocket Scion biofeedback (raw values)
    std::atomic<float> bio_delta{0.0f};
    std::atomic<float> bio_deviation{0.0f};
    std::atomic<float> bio_mean{0.0f};
    std::atomic<float> bio_min{0.0f};
    std::atomic<float> bio_max{0.0f};
    std::atomic<float> bio_variance{0.0f};
};

static OscParams g_osc_params;
```

`memory_order_relaxed` is sufficient because each parameter is independent --
no happens-before relationship needed between reading X and reading Y.

### liblo C++ API with Lambda Handlers

```cpp
#include <lo/lo.h>
#include <lo/lo_cpp.h>

lo::ServerThread st("10361");  // Pocket Scion default port

// Register handlers for all 6 biofeedback values
st.add_method("/delta", "f",
    [](const char *path, const char *types, lo_arg **argv, int argc) {
        g_osc_params.bio_delta.store(argv[0]->f, std::memory_order_relaxed);
        return 0;
    });

st.add_method("/deviation", "f",
    [](const char *path, const char *types, lo_arg **argv, int argc) {
        g_osc_params.bio_deviation.store(argv[0]->f, std::memory_order_relaxed);
        return 0;
    });

// ... same pattern for /min, /max, /mean, /variance

// Also accept direct grids-jack parameter control
st.add_method("/grids/pattern_x", "f",
    [](const char *path, const char *types, lo_arg **argv, int argc) {
        float val = argv[0]->f;
        uint8_t x = (uint8_t)(val < 0 ? 0 : (val > 255 ? 255 : val));
        g_osc_params.pattern_x.store(x, std::memory_order_relaxed);
        return 0;
    });

st.start();
```

### JACK Process Callback Reads (Realtime-Safe)

```cpp
int jack_process_callback(jack_nframes_t nframes, void *arg) {
    // Read latest parameter values (lock-free, wait-free)
    uint8_t new_x = g_osc_params.pattern_x.load(std::memory_order_relaxed);
    if (new_x != g_pattern_generator.GetPatternX()) {
        g_pattern_generator.SetPatternX(new_x);
    }
    // ... same for Y, randomness, humanize
    // ... rest of audio processing
}
```

### Build Integration (CMakeLists.txt)

```cmake
pkg_check_modules(LIBLO REQUIRED liblo)

include_directories(
    ${CMAKE_SOURCE_DIR}
    ${JACK_INCLUDE_DIRS}
    ${SNDFILE_INCLUDE_DIRS}
    ${LIBLO_INCLUDE_DIRS}
)

target_link_libraries(grids-jack
    ${JACK_LIBRARIES}
    ${SNDFILE_LIBRARIES}
    ${LIBLO_LIBRARIES}
)
```

Install: `sudo apt install liblo-dev`

### CLI Flags

- `-O <port>` -- OSC listen port (default 10361, matching Pocket Scion)
- `--osc-map <file>` -- optional JSON mapping file for custom address-to-parameter routing

### Testing Without Hardware

```bash
# Install liblo command-line tools
sudo apt install liblo-tools

# Simulate Pocket Scion biofeedback
oscsend localhost 10361 /delta f 12.5
oscsend localhost 10361 /deviation f 3.2
oscsend localhost 10361 /mean f 450.0

# Direct parameter control
oscsend localhost 10361 /grids/pattern_x f 200.0

# Monitor incoming OSC (useful for debugging real Pocket Scion)
oscdump 10361
```

## Signal Flow

```
Plant/Fungi/Skin
      |
  Pocket Scion (USB)
      |
  Desktop Companion App
      |
  OSC broadcast (UDP port 10361)
      |
  grids-jack (lo::ServerThread)
      |
  std::atomic params (lock-free)
      |
  JACK process callback reads atomics
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
- The Pocket Scion also provides a starter TouchDesigner project on their
  website -- so the same OSC data can drive visuals simultaneously

## Complexity

Medium. The core parameter-update plumbing already exists in
`PatternGeneratorWrapper`. The main work is:
1. Adding liblo dependency (one `pkg_check_modules` line)
2. Creating the OSC server thread with handlers (~50 lines)
3. Adding atomic reads at the top of the JACK process callback (~20 lines)

## References

- [liblo official site](https://liblo.sourceforge.net/)
- [liblo C++ wrapper (lo_cpp.h)](https://github.com/radarsat1/liblo/blob/master/lo/lo_cpp.h)
- [Pocket Scion product page](https://pocketscion.com/) -- includes starter TD project
- [Pocket Scion manual](https://www.manualslib.com/manual/4087816/Instruo-Pocket-Scion.html)
- [Ross Bencina -- Realtime Audio Programming 101](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing)
