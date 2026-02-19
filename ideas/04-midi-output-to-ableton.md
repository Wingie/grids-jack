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
| BD (kick) | `60.kick.wav` | Note 60, vel from velocity pattern | MIDI track -> Drum Rack pad |
| SD (snare) | `65.snare.wav` | Note 65, vel from velocity pattern | MIDI track -> Drum Rack pad |
| HH (hat) | `72.hat.wav` | Note 72, vel from velocity pattern | MIDI track -> Drum Rack pad |

## Implementation: JACK MIDI API

### Only One Extra Header

```cpp
#include <jack/midiport.h>  // add alongside existing jack/jack.h
```

### Port Registration

```cpp
static jack_port_t* g_midi_output_port = nullptr;

// In init_jack(), after registering audio ports:
g_midi_output_port = jack_port_register(g_jack_client, "midi_out",
                                        JACK_DEFAULT_MIDI_TYPE,
                                        JackPortIsOutput, 0);
```

No new dependencies -- JACK MIDI is part of the same `libjack` library
already linked. No changes to CMakeLists.txt required.

### Writing MIDI Events in the Process Callback

Three critical rules:
1. **MUST call `jack_midi_clear_buffer()` every cycle** (or it re-sends previous events)
2. **Events MUST be written in ascending sample offset order**
3. **No running status** -- every event must be a complete MIDI message

```cpp
// Helper functions
static inline int send_note_on(void* midi_buf, jack_nframes_t offset,
                                uint8_t channel, uint8_t note, uint8_t vel) {
    jack_midi_data_t data[3] = {
        static_cast<jack_midi_data_t>(0x90 | (channel & 0x0F)),
        static_cast<jack_midi_data_t>(note & 0x7F),
        static_cast<jack_midi_data_t>(vel & 0x7F)
    };
    return jack_midi_event_write(midi_buf, offset, data, 3);
}

static inline int send_note_off(void* midi_buf, jack_nframes_t offset,
                                 uint8_t channel, uint8_t note) {
    jack_midi_data_t data[3] = {
        static_cast<jack_midi_data_t>(0x80 | (channel & 0x0F)),
        static_cast<jack_midi_data_t>(note & 0x7F),
        0
    };
    return jack_midi_event_write(midi_buf, offset, data, 3);
}
```

### Sample-Accurate MIDI Timing

The existing `Process()` loop already iterates frame-by-frame:

```cpp
for (uint32_t i = 0; i < num_frames; ++i) {
    if (frames_since_last_tick_ >= frames_per_pulse_) {
        ProcessTriggers(state);  // trigger fires here
    }
}
```

The loop variable `i` **is already the correct sample offset** for
`jack_midi_event_write()`. Since `i` only increases, the ascending-order
constraint is naturally satisfied. This is the key insight -- no extra work
needed for sample-accurate timing.

At 48kHz with a 256-sample buffer (5.33ms), placing an event at sample 128
means it occurs at 2.67ms into the buffer, not at the start. This precision
matters for tight groove feel.

**Pitfall to avoid**: The [Hydrogen drum machine](https://github.com/hydrogen-music/hydrogen)
had a [bug (#535)](https://github.com/hydrogen-music/hydrogen/issues/535) where
MIDI events were quantized to buffer boundaries (always offset 0) instead of
using the correct intra-buffer position. Don't repeat this -- always pass `i`.

### Note-Off Management

For drum triggers, use a fixed-duration note-off with re-trigger handling:

```cpp
struct PendingNoteOff {
    uint8_t channel;
    uint8_t note;
    int32_t frames_remaining;
    bool active;
};

static const size_t kMaxPendingNoteOffs = 128;
static PendingNoteOff g_pending[kMaxPendingNoteOffs];
static uint32_t g_note_duration = 2400;  // 50ms at 48kHz

void schedule_note(void* midi_buf, jack_nframes_t offset,
                   uint8_t channel, uint8_t note, uint8_t vel) {
    // If same note is already active, send note-off first (re-trigger)
    for (size_t i = 0; i < kMaxPendingNoteOffs; ++i) {
        if (g_pending[i].active &&
            g_pending[i].note == note &&
            g_pending[i].channel == channel) {
            send_note_off(midi_buf, offset, channel, note);
            g_pending[i].active = false;
        }
    }

    // Send note-on
    send_note_on(midi_buf, offset, channel, note, vel);

    // Schedule note-off
    for (size_t i = 0; i < kMaxPendingNoteOffs; ++i) {
        if (!g_pending[i].active) {
            g_pending[i] = { channel, note,
                             static_cast<int32_t>(g_note_duration), true };
            return;
        }
    }
    // Queue full -- immediate note-off as fallback
    send_note_off(midi_buf, offset, channel, note);
}
```

Process pending note-offs in the per-sample loop (they fire at the exact
sample when the countdown reaches 0, maintaining ascending order).

### Velocity Mapping

The current binary velocity pattern outputs `1.0` (loud) or `0.1` (soft).
Map to MIDI velocity range that keeps ghost notes audible:

```cpp
// Simple and effective for binary patterns:
uint8_t midi_vel = high_velocity ? 127 : 49;

// Or generalized for future multi-level velocity:
static inline uint8_t map_velocity(float vel, uint8_t low = 40, uint8_t high = 127) {
    if (vel < 0.0f) vel = 0.0f;
    if (vel > 1.0f) vel = 1.0f;
    return static_cast<uint8_t>(low + vel * (high - low));
}
// 0.1 -> 49 (ghost note), 1.0 -> 127 (full accent)
```

### Modified ProcessTriggers()

```cpp
void PatternGeneratorWrapper::ProcessTriggers(uint8_t state,
                                               void* midi_buf,
                                               jack_nframes_t sample_offset) {
    for (int part = 0; part < grids::kNumParts; ++part) {
        if (state & (1 << part)) {
            for (size_t i = 0; i < sample_mappings_.size(); ++i) {
                if (sample_mappings_[i].drum_part == static_cast<DrumPart>(part)) {
                    bool high_vel = EvaluateVelocityPattern(sample_mappings_[i]);
                    float velocity = high_vel ? 1.0f : 0.1f;

                    // Audio trigger (existing)
                    float pan = sample_mappings_[i].pan;
                    sample_player_->Trigger(sample_mappings_[i].midi_note,
                                            velocity, pan);

                    // MIDI trigger (new)
                    if (midi_buf) {
                        uint8_t midi_vel = high_vel ? 127 : 49;
                        schedule_note(midi_buf, sample_offset,
                                      9,  // GM drum channel (0-indexed)
                                      sample_mappings_[i].midi_note,
                                      midi_vel);
                    }
                }
            }
        }
    }
}
```

### Complete Process Callback

```cpp
int jack_process_callback(jack_nframes_t nframes, void* arg) {
    float* out_left = (float*)jack_port_get_buffer(g_output_port_left, nframes);
    float* out_right = (float*)jack_port_get_buffer(g_output_port_right, nframes);
    void* midi_buf = jack_port_get_buffer(g_midi_output_port, nframes);

    jack_midi_clear_buffer(midi_buf);  // MUST be called every cycle

    g_pattern_generator.Process(nframes, midi_buf);  // generates triggers + MIDI
    g_sample_player.ProcessStereo(out_left, out_right, nframes);

    return 0;
}
```

## Connecting to Ableton

### On Linux (native DAWs: Ardour, Bitwig, Reaper)

Direct JACK MIDI routing -- no bridge needed:
```bash
jack_connect grids-jack:midi_out ardour:MIDI\ in\ 1
```

### On Linux (Ableton via Wine)

Ableton under Wine with wineASIO has known issues with JACK MIDI (note-on/off
may not work reliably). Workaround: use `a2jmidid` to bridge to ALSA MIDI,
or use a virtual ALSA port (`modprobe snd-virmidi`).

Note: `a2jmidid` adds exactly one JACK period of latency (~5.3ms at 48kHz/256).

### Multi-Machine Setup

If Ableton runs on a separate Windows/Mac machine, use network MIDI
(qmidinet, rtpMIDI) to send MIDI from Linux over the network.

### Auto-Connection Code

```cpp
// After jack_activate(), auto-connect to first available MIDI input
const char** midi_ports = jack_get_ports(g_jack_client, nullptr,
                                          JACK_DEFAULT_MIDI_TYPE,
                                          JackPortIsInput);
if (midi_ports && midi_ports[0]) {
    jack_connect(g_jack_client,
                 jack_port_name(g_midi_output_port),
                 midi_ports[0]);
    jack_free(midi_ports);
}
```

## CLI Flags

- `-M` -- enable MIDI output
- `-m <channel>` -- MIDI channel (default: 10 / 0-indexed 9, standard drums)
- `--midi-only` -- suppress internal sample playback (pattern brain mode)
- `--midi-vel-scale <0.0-1.0>` -- scale velocity output

## Signal Flow

```
grids-jack
    |
    +-- JACK Audio Out (L/R) ----> speakers / mixer (optional)
    |
    +-- JACK MIDI Out -----------> Ableton Live 12
                                        |
                                   MIDI Track (ch 10)
                                        |
                                   Drum Rack / Instrument
                                        |
                                   Mixed with Pocket Scion
                                   MIDI on channels 1-5
                                        |
                                   Master Out -> speakers
```

## Why This Is Interesting

- **Sound design freedom**: grids-jack generates *patterns*, Ableton provides
  *sounds*. Swap Drum Racks on the fly during performance
- **Pocket Scion duet**: Pocket Scion on MIDI ch 1-5 (melody), grids-jack on
  ch 10 (rhythm), unified in Ableton's mixer
- **Pattern brain mode** (`--midi-only`): Pure sequencer, zero audio overhead
- **Recording**: Ableton records MIDI clips from grids-jack for later editing
- **Sample-accurate**: Using `i` from the per-sample loop avoids the
  buffer-quantization pitfall that plagued Hydrogen

## Complexity

Medium. No new dependencies (JACK MIDI is in libjack). The trigger points
already exist in ProcessTriggers(). Main work is note-off management (~50
lines) and velocity mapping (~10 lines).

## References

- [JACK MIDI API](https://jackaudio.org/api/group__MIDIAPI.html)
- [JACK midiseq.c example](https://github.com/jackaudio/example-clients/blob/master/midiseq.c)
- [Harry Haaren's JACK-MIDI-Examples](https://github.com/harryhaaren/JACK-MIDI-Examples)
- [x42/jack_midi_clock](https://github.com/x42/jack_midi_clock)
- [Hydrogen MIDI timing bug #535](https://github.com/hydrogen-music/hydrogen/issues/535)
- [a2jmidid bridge](https://github.com/jackaudio/a2jmidid)
