# Idea 3: OSC Output to TouchDesigner for Beat-Reactive Visuals

## Concept

Add OSC output to grids-jack that broadcasts drum trigger events, pattern
state, and voice activity to TouchDesigner. This gives TD direct access to
the generative pattern data for driving beat-synchronized visuals -- tighter
than audio analysis because the data arrives *before* the sound.

## What Gets Sent

### Per-Trigger Messages (on every drum hit)

| OSC Address | Type | Value | Use in TouchDesigner |
|-------------|------|-------|---------------------|
| `/grids/trigger/bd` | float | velocity (0.0-1.0) | Flash/pulse on kick |
| `/grids/trigger/sd` | float | velocity (0.0-1.0) | Flash/pulse on snare |
| `/grids/trigger/hh` | float | velocity (0.0-1.0) | Flash/pulse on hi-hat |
| `/grids/trigger/step` | int | current step (0-31) | Step sequencer visualization |

### Pattern State (sent on change or periodically)

| OSC Address | Type | Value | Use in TouchDesigner |
|-------------|------|-------|---------------------|
| `/grids/state/x` | int | 0-255 | Map to visual parameter axis |
| `/grids/state/y` | int | 0-255 | Map to visual parameter axis |
| `/grids/state/randomness` | int | 0-255 | Drive visual chaos/noise |
| `/grids/state/density/bd` | int | 0-255 | Visual density of kick layer |
| `/grids/state/density/sd` | int | 0-255 | Visual density of snare layer |
| `/grids/state/density/hh` | int | 0-255 | Visual density of hat layer |
| `/grids/state/bpm` | float | current BPM | Display / animation speed |
| `/grids/state/voices` | int | active voice count | Particle count, complexity |

## Implementation

1. Add `liblo` (or `oscpack`) for OSC output
2. Create an `OscSender` class with a target IP:port (default `127.0.0.1:8000`)
3. Hook into `PatternGeneratorWrapper` at the trigger evaluation point --
   when a drum part fires, send the corresponding OSC trigger message
4. After each pattern cycle (32 steps), send a state update bundle
5. Add CLI flags:
   - `-T <ip:port>` for TouchDesigner OSC target
   - Messages are fire-and-forget UDP, zero impact on realtime audio

## TouchDesigner Patch Architecture

```
OSC In CHOP (port 8000)
    |
    +-- /grids/trigger/bd --> Trigger CHOP --> Envelope (attack/decay)
    |                              |
    |                         Visual layer: geometry scale, color pulse,
    |                         particle burst, camera shake
    |
    +-- /grids/trigger/sd --> Trigger CHOP --> Envelope
    |                              |
    |                         Visual layer: flash, ripple, displacement
    |
    +-- /grids/trigger/hh --> Trigger CHOP --> Envelope
    |                              |
    |                         Visual layer: sparkle, high-freq noise
    |
    +-- /grids/state/* --> Math CHOPs --> Parameter mapping
                               |
                          Global visual parameters:
                          color palette, complexity, camera movement
```

## Why This Is Interesting

- **Pre-audio timing**: OSC trigger messages arrive before the sound reaches
  the speakers (audio has buffer latency, UDP is near-instant). Visuals can
  actually *anticipate* beats for tighter perceived sync
- **Structured data, not audio analysis**: Instead of FFT-guessing "was that
  a kick?", TouchDesigner gets explicit `/trigger/bd` with exact velocity.
  No false positives, no threshold tuning
- **Pattern visualization**: The X/Y state data lets you build a visual
  representation of the Grids 2D map -- show where you are in pattern-space
  as a moving dot on a 2D field
- **Combines with Pocket Scion OSC**: If Idea 1 is also implemented, the
  biofeedback data flows in via OSC *and* the resulting pattern triggers
  flow out via OSC -- TouchDesigner can visualize both the cause (biology)
  and effect (rhythm)

## Complexity

Low-medium. OSC output is simpler than input (no threading needed -- just
fire UDP packets from the JACK callback or a helper thread). The main design
decision is whether to send from the realtime audio thread (fast but
technically not realtime-safe due to UDP syscalls) or queue messages for a
sender thread.
