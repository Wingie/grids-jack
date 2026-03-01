# Idea 3: OSC Output to TouchDesigner for Beat-Reactive Visuals

## Concept

Add OSC output to grids-jack that broadcasts drum trigger events, pattern
state, and voice activity to TouchDesigner. This gives TD direct access to
the generative pattern data for driving beat-synchronized visuals -- tighter
than audio analysis because the data arrives *before* the sound.

## What Gets Sent

### Per-Trigger Messages (on every drum hit)

| OSC Address | Args | Types | Use in TouchDesigner |
|-------------|------|-------|---------------------|
| `/grids/trigger/bd` | velocity, pan, midi_note, step | `ffii` | Flash/pulse on kick |
| `/grids/trigger/sd` | velocity, pan, midi_note, step | `ffii` | Flash/pulse on snare |
| `/grids/trigger/hh` | velocity, pan, midi_note, step | `ffii` | Flash/pulse on hi-hat |

### Transport (sent every beat/step)

| OSC Address | Args | Types | Use in TouchDesigner |
|-------------|------|-------|---------------------|
| `/grids/beat` | step, bpm | `if` | Step sequencer visualization |
| `/grids/bar` | bar_number | `i` | Scene changes (every 32 steps) |
| `/grids/downbeat` | 1.0 | `f` | Pulse on beat 1 |

### Pattern State (sent on change)

| OSC Address | Args | Types | Use in TouchDesigner |
|-------------|------|-------|---------------------|
| `/grids/pattern/x` | value (0-255) | `i` | Visual parameter axis |
| `/grids/pattern/y` | value (0-255) | `i` | Visual parameter axis |
| `/grids/pattern/randomness` | value (0-255) | `i` | Visual chaos/noise |

Use floats for velocity (0.0-1.0) -- this is TouchDesigner's native CHOP
format and maps directly to visual intensity without conversion.

## Critical Implementation Detail: Realtime Safety

### The Problem

**Never call `lo_send()` from the JACK process callback.** `lo_send()` calls
`sendto()` which is a syscall that can block:

| Scenario | Blocking Time |
|----------|---------------|
| Localhost | ~89 us |
| Same subnet, host online | ~195 us |
| Host nonexistent (ARP timeout) | **~3,941,487 us (3.9 seconds!)** |

At 48kHz with 128 frames, your JACK deadline is 2,667 us. Even the typical
localhost case consumes 3.3% of the budget. The catastrophic ARP failure case
would cause thousands of xruns.

### The Solution: Lock-Free Ring Buffer + Sender Thread

```
JACK process callback (realtime)          Sender thread (non-realtime)
          |                                         |
  detect triggers                           while(running) {
          |                                   if(ringbuffer has data) {
  write OscTriggerEvent to                      read OscTriggerEvent
  jack_ringbuffer (lock-free)                   lo_send(addr, ...)
          |                                   } else {
  continue audio processing                     usleep(200)
          |                                   }
                                              }
```

### Implementation Code

```cpp
#include <jack/ringbuffer.h>
#include <lo/lo.h>

struct OscTriggerEvent {
    uint8_t drum_part;    // 0=BD, 1=SD, 2=HH
    uint8_t midi_note;
    float   velocity;     // 0.0-1.0
    float   pan;          // -1.0 to 1.0
    uint8_t step;         // current pattern step (0-31)
    uint8_t pattern_x;
    uint8_t pattern_y;
};

class OscSender {
    jack_ringbuffer_t* ring_;
    lo_address addr_;
    pthread_t thread_;
    volatile bool running_;

public:
    bool Init(const char* host, const char* port) {
        ring_ = jack_ringbuffer_create(65536);  // 64KB, ~4000 events
        jack_ringbuffer_mlock(ring_);            // lock pages for RT safety
        addr_ = lo_address_new(host, port);
        running_ = true;
        return pthread_create(&thread_, nullptr, SenderLoop, this) == 0;
    }

    // Called from JACK process callback -- REALTIME SAFE
    void QueueTrigger(const OscTriggerEvent& ev) {
        if (jack_ringbuffer_write_space(ring_) >= sizeof(ev)) {
            jack_ringbuffer_write(ring_, (const char*)&ev, sizeof(ev));
        }
        // If full, silently drop -- never block
    }

private:
    static void* SenderLoop(void* arg) {
        auto* self = static_cast<OscSender*>(arg);
        const char* names[] = { "bd", "sd", "hh" };

        while (self->running_) {
            OscTriggerEvent ev;
            while (jack_ringbuffer_read_space(self->ring_) >= sizeof(ev)) {
                jack_ringbuffer_read(self->ring_, (char*)&ev, sizeof(ev));
                char addr[64];
                snprintf(addr, sizeof(addr), "/grids/trigger/%s",
                         names[ev.drum_part]);
                lo_send(self->addr_, addr, "ffii",
                        ev.velocity, ev.pan,
                        (int)ev.midi_note, (int)ev.step);
            }
            usleep(200);  // ~200us poll = sub-millisecond latency
        }
        return nullptr;
    }
};
```

### Integration Point

In `PatternGeneratorWrapper::ProcessTriggers()` (line 292), right alongside
the existing `sample_player_->Trigger()` call:

```cpp
if (osc_sender_) {
    OscTriggerEvent ev;
    ev.drum_part = static_cast<uint8_t>(part);
    ev.midi_note = sample_mappings_[i].midi_note;
    ev.velocity = velocity;
    ev.pan = pan;
    ev.step = grids::PatternGenerator::step();
    ev.pattern_x = GetPatternX();
    ev.pattern_y = GetPatternY();
    osc_sender_->QueueTrigger(ev);
}
```

### CLI Flags

- `-T <ip:port>` -- TouchDesigner OSC target (default `127.0.0.1:7770`)

## Latency Analysis: Direct OSC vs. FFT Audio Analysis

### Direct OSC trigger path (this approach)

| Stage | Latency |
|-------|---------|
| Trigger detection in ProcessTriggers() | 0 (sample-accurate) |
| Ringbuffer write | < 1 us |
| Sender thread poll + wake | ~100-500 us |
| UDP sendto() to localhost | ~89 us |
| Network stack loopback | ~10-50 us |
| TouchDesigner next frame (60 FPS) | 0 - 16.7 ms |
| **Total** | **~0.3 - 17 ms (avg ~8.5 ms)** |

### FFT audio analysis path (traditional approach)

| Stage | Latency |
|-------|---------|
| JACK audio buffer (128 frames @ 48kHz) | 2.67 ms |
| Audio routing to TD | 2.67 ms |
| TD Audio In CHOP buffering | 0-16.7 ms |
| FFT window (min 1024 samples @ 48kHz) | 21.3 ms |
| Onset detection (2 FFT windows) | 42.7 ms |
| Threshold + trigger logic | 16.7-33.3 ms |
| **Total** | **~86 - 119 ms** |

### The Fundamental Advantage

**grids-jack knows the trigger *before* the sound plays.** In `ProcessTriggers()`,
the event is generated at the exact moment the pattern generator decides to fire
-- before the audio sample even begins playing. With humanize enabled, the visual
trigger arrives before the humanized audio offset is applied. This means visuals
can actually *anticipate* beats.

**Net advantage: ~80-100 ms faster than FFT**, which is nearly a full 16th note
at 120 BPM (125 ms). FFT-based detection is useless for tight sync to fast
hi-hat patterns. Direct OSC is within one video frame of perfect sync.

## TouchDesigner Setup

### Recommended Dual-Receiver Architecture

1. **OSC In CHOP** (port 7770) with **Pulse Mode ON** for `/grids/trigger/*`
   - Drives visual envelopes, particle triggers, geometry instancing
   - Pulse Mode creates a single-frame spike, then resets to 0
   - Without Pulse Mode, the value holds forever until next hit (wrong for triggers)

2. **OSC In DAT** (same port 7770) with Python callbacks for `/grids/beat`
   - Processes messages at full rate regardless of TD frame rate
   - Better for discrete events (scene changes, color shifts)

### TD Patch Architecture

```
OSC In CHOP (port 7770, Pulse Mode ON)
    |
    +-- trigger/bd --> Filter CHOP --> Envelope (attack/decay)
    |                       |
    |                  geometry scale, color pulse, particle burst
    |
    +-- trigger/sd --> Filter CHOP --> Envelope
    |                       |
    |                  flash, ripple, displacement map
    |
    +-- trigger/hh --> Filter CHOP --> Envelope
    |                       |
    |                  sparkle, high-frequency noise
    |
    +-- pattern/* --> Math CHOPs --> global visual params
                          |
                     color palette, complexity, camera

OSC In DAT (port 7770, Python callbacks)
    |
    +-- /grids/beat --> scene management, phrase structure
    +-- /grids/bar  --> color palette cycling
```

### Channel Auto-Creation

When `/grids/trigger/bd` arrives with args `f:0.8 f:0.2 i:36 i:4`, the OSC In
CHOP auto-creates channels: `trigger/bd:chan1` (0.8), `trigger/bd:chan2` (0.2), etc.

At 120 BPM, expect 8-24 trigger messages/second -- well under TD's 60 Hz frame
rate, so no missed messages.

## Why This Is Interesting

- **Pre-audio timing**: ~80-100 ms faster than FFT analysis
- **Structured data**: Explicit `/trigger/bd` with velocity, not FFT-guessing
  "was that a kick?" -- no false positives, no threshold tuning
- **Pattern visualization**: X/Y state data lets you build a visual 2D map
  showing where you are in pattern-space as a moving dot
- **Combines with Pocket Scion OSC**: Biofeedback flows in via OSC (Idea 1)
  and pattern triggers flow out via OSC -- TD can visualize both cause
  (biology) and effect (rhythm)

## Complexity

Low-medium. The ring buffer + sender thread is ~100 lines. Main design
decision is already resolved: never send from the realtime thread, always
queue via `jack_ringbuffer_t`.

## References

- [JACK Ringbuffer API](https://jackaudio.org/api/ringbuffer_8h.html)
- [Adam Sawicki -- UDP send() Blocking Measurements](https://asawicki.info/news_1455_udp_sockets_send_function_blocks.html)
- [Ross Bencina -- Realtime Audio Programming 101](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing)
- [rtosc -- Realtime Safe OSC (used by ZynAddSubFX)](https://github.com/fundamental/rtosc)
- [TouchDesigner OSC In CHOP docs](https://docs.derivative.ca/OSC_In_CHOP)
- [TouchDesigner OSC In DAT docs](https://docs.derivative.ca/OSC_In_DAT)
- [TD forum -- OSC sampling rate and missed messages](https://forum.derivative.ca/t/osc-in-and-sampling-rate-and-how-to-not-miss-messages/167184)
