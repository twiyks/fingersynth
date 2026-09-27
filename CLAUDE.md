# XY Synth

A browser synth played on an X/Y pad. X picks a note from the chosen scale, Y sets a lowpass filter cutoff. One finger plays a sliding held note; two or more fingers run an arpeggiator through the held notes. Built with the Web Audio API, no libraries, no build step.

## Project shape

- Everything lives in one self-contained HTML file (currently `xy-synth.html`; rename to `index.html` for GitHub Pages). CSS and JS are inline.
- Keep it as a single file with no dependencies unless I ask otherwise. The only external resource is the Bricolage Grotesque font from Google Fonts, with a system font fallback.
- A small version label (`<div class="version">`) sits bottom right so I can tell when a push has reached my phone. Bump it (v1.1, v1.2...) with every commit that changes the page.
- To test, just open the file in a browser. Audio only starts after a user gesture (first pointerdown calls `ensureAudio()`), which browsers require.
- The arpeggiator needs multitouch, so test it on a phone or tablet. A mouse only gives one pointer.

## How the code is organised

All JS sits inside one IIFE. Main pieces, top to bottom:

**Data**
- `SCALES`: name plus semitone steps from the root. The pad covers two octaves from C3 to C5 (`LOW = 48`, `SPAN = 24`), and `buildNotes()` fills `notes` with the MIDI numbers in the current scale. Each note is one column on the pad.
- `PRESETS`: each sound is a list of oscillators (`type`, `oct` offset, `det` detune in cents, `g` gain) plus filter `q`, attack `atk`, release `rel` and vibrato depth `vib` in cents. To add a sound, add an entry here; the dropdown builds itself.
- Filter cutoff maps exponentially from `CUT_MIN` (60 Hz) at the bottom to `CUT_MAX` (14 kHz) at the top via `cutoffFor(y)`.

**Audio graph** (built once in `ensureAudio()`)
```
voice -> master (0.5) -> out -> compressor (-14 dB, 4:1) -> destination
                 |-> delaySend -> delay <-> tone lowpass 5 kHz -> feedback (loop)
                 |                           \-> out, and -> convolver
                 \-> convolver (reverb) -> reverbWet -> out
```
- Delay echoes also feed the reverb.
- Delay "Decay" is a time in seconds (0.1 to 12, exponential slider). Feedback is derived from it with `feedbackFor(time, decay) = 10^(-3 * time / decay)`, i.e. the echoes fall by 60 dB over the decay time. Capped at 0.95.
- Reverb uses a generated impulse response (`makeImpulse(size)`): stereo noise with a power-curve fade, 0.3 to 6 seconds long depending on room size. It's rebuilt on a 120 ms debounce when the size slider moves.
- Effect settings live in `fxState` (read by `readFx()`) so the sliders work before audio has started; `applyFx()` pushes them into the nodes.

**Voices**
- `startVoice(midi, cutoff, t)` builds a fresh voice: oscillators -> per-oscillator gain -> its own lowpass filter -> amp envelope -> `master`. Optional `t` schedules it on the audio clock.
- `stopVoice(v)` releases immediately (held notes). `stopVoiceAt(v, when, onDone)` releases at a scheduled time (arp notes) and cleans up via `onended`.
- `updateVoice()` glides pitch and cutoff with `setTargetAtTime` for smooth sliding.

**Pointers and modes**
- `pointers` is a Map of pointerId to `{x, y, idx, midi}` with x and y normalised 0 to 1.
- `refreshMode(prevCount)` handles transitions: 0 fingers stops everything, 1 finger plays a single legato `voice`, 2+ stops that voice and starts the arp. Going from 2 back to 1 returns to the held note.
- The filter follows whichever finger was last pressed or moved (global `cutoff`). In arp mode, `setArpCutoff()` updates every currently sounding arp voice.

**Arpeggiator**
- Uses a lookahead scheduler: `setInterval(arpTick, 25)` schedules any notes falling within the next 100 ms using `ctx.currentTime`, so timing stays tight even if the page stutters. Don't switch this to plain setTimeout per note.
- Each step creates a new voice and releases it after `GATE` (0.7) of the step length. Active arp voices are tracked in `arpVoices`.
- Direction: up, down (sorted held notes, stepped by `arp.step`) or random (never repeats the previous note when there's a choice).
- `arp.queue` records scheduled note times so `draw()` can show which note is actually audible right now.

**Drawing**
- A canvas redrawn every frame in `draw()`: alternating note columns (C columns marked stronger), held columns tinted, the sounding note brighter, a fading trail, and a glow per finger. Glow size grows as the filter opens. Hue runs across the pitch range via `hueFor()`.
- Canvas is sized to devicePixelRatio (capped at 2) through a ResizeObserver.
- `prefers-reduced-motion` turns the trail off.

## Design and UI conventions

- Colours are CSS custom properties on `:root`, with dark values under `prefers-color-scheme: dark` (guarded by `:root:not([data-theme="light"])`) and again under `:root[data-theme="dark"]`. Canvas colours are read from those tokens with `css()`, so add new colours as tokens, not hard-coded values.
- Layout respects phone safe areas (`viewport-fit=cover` plus `env(safe-area-inset-*)` padding) and uses `height: 100%` rather than `100vh`.
- The pad uses `touch-action: none` and pointer capture per pointer. Keep that, or scrolling and gestures will fight the instrument.
- Controls: selects for scale and sound, range inputs with a live `<output>` readout, a segmented radio group for arp direction. Keep focus-visible outlines on anything interactive.
- UI copy should be short, plain and conversational. No marketing language, and avoid em dashes.

## Ideas I might pick up next

- Latch mode so a mouse can hold several notes (click to toggle)
- Root key picker, and octave range control
- Tempo-synced delay that follows the arp speed
- More arp directions (up-down), gate length and octave spread
- Save and recall settings

## Working with me

I come from animation, motion and sound design rather than engineering, and my maths is GCSE level. When explaining audio or DSP concepts, start simple and build up. Prefer small, focused changes and tell me in plain language what changed and why.
