# XY Synth

A browser synth played on an X/Y pad. X picks a note from the chosen scale, Y sets a lowpass filter cutoff. One finger plays a sliding held note; two or more fingers run an arpeggiator through the held notes. A second view has a 16-step drum machine with a synthesised 909-style kit. Built with the Web Audio API, no libraries, no build step.

## Project shape

- Everything lives in one self-contained HTML file (currently `xy-synth.html`; rename to `index.html` for GitHub Pages). CSS and JS are inline.
- Keep it as a single file with no dependencies unless I ask otherwise. The only external resource is the Bricolage Grotesque font from Google Fonts, with a system font fallback.
- A small version label (`<div class="version">`) sits bottom right so I can tell when a push has reached my phone. Bump it (v1.1, v1.2...) with every commit that changes the page.
- To test, just open the file in a browser. Audio only starts after a user gesture (first pointerdown calls `ensureAudio()`), which browsers require.
- GitHub Pages deploys from `main` only. Feature work goes on a branch and gets merged to `main` to test on the phone.
- The arpeggiator needs multitouch, so test it on a phone or tablet. A mouse only gives one pointer.

## How the code is organised

All JS sits inside one IIFE. Main pieces, top to bottom:

**Data**
- `SCALES`: name plus semitone steps from the root. The pad has `ROWS = 2` stacked keyboards, each two octaves (`SPAN = 24`): bottom row C2 to C4 (`LOW = 36`), top row C4 to C6. `buildNotes()` fills `notes` with the bottom row's MIDI numbers in the current scale; each note is one column, and a higher row adds `row * SPAN`.
- `PRESETS`: each sound is a list of oscillators (`type`, `oct` offset, `det` detune in cents, `g` gain) plus filter `q`, attack `atk`, release `rel` and vibrato depth `vib` in cents. To add a sound, add an entry here; the dropdown builds itself.
- Filter cutoff maps exponentially from `CUT_MIN` (60 Hz) at the bottom of a row to `CUT_MAX` (14 kHz) at the top of that row via `cutoffFor(ly)`, so each row has its own full filter sweep.

**Audio graph** (built once in `ensureAudio()`)
```
voice -> master (0.5) -> out -> compressor (-14 dB, 4:1) -> destination
drums -> drumBus (Kit level) -> out          (no delay or reverb on drums)
                 |-> delaySend -> delay <-> tone lowpass 5 kHz -> feedback (loop)
                 |                           \-> out, and -> convolver
                 \-> convolver (reverb) -> reverbWet -> out
```
- Delay echoes also feed the reverb.
- Delay "Decay" is a time in seconds (0.1 to 12, exponential slider). Feedback is derived from it with `feedbackFor(time, decay) = 10^(-3 * time / decay)`, i.e. the echoes fall by 60 dB over the decay time. Capped at 0.95.
- An "ms / notes" switch in the Delay header (`delayUnit`, `setDelayUnit()`) lets Time and Decay be set as note values instead. In notes mode Time steps through `SYNC_TIMES` (1/16 to 1 bar, with T triplet and D dotted) and Decay through `SYNC_DECAYS` (1/2 beat to 8 bars), both stored in beats and converted to seconds from the master tempo, so moving the Tempo slider retimes the delay. Each mode remembers its own slider positions. Delay line max is `MAX_DELAY` (4 s, one bar at 60 BPM).
- Reverb uses a generated impulse response (`makeImpulse(size)`): stereo noise with a power-curve fade, 0.3 to 6 seconds long depending on room size. It's rebuilt on a 120 ms debounce when the size slider moves.
- Effect settings live in `fxState` (read by `readFx()`) so the sliders work before audio has started; `applyFx()` pushes them into the nodes.

**Voices**
- `startVoice(midi, cutoff, t)` builds a fresh voice: oscillators -> per-oscillator gain -> its own lowpass filter -> amp envelope -> `master`. Optional `t` schedules it on the audio clock.
- `stopVoice(v)` releases immediately (held notes). `stopVoiceAt(v, when, onDone)` releases at a scheduled time (arp notes) and cleans up via `onended`.
- `updateVoice()` glides pitch and cutoff with `setTargetAtTime` for smooth sliding.

**Pointers and modes**
- `pointers` is a Map of pointerId to `{x, y, row, ly, idx, midi, gi}`. x and y are normalised 0 to 1 over the whole pad; `mapPointer()` works out `row` (0 = bottom), `ly` (height within that row, 0 to 1), the column `idx`, and `gi` (column counted across all rows, used for colour).
- `refreshMode(prevCount)` handles transitions: 0 fingers stops everything, 1 finger plays a single legato `voice`, 2+ stops that voice and starts the arp. Going from 2 back to 1 returns to the held note.
- The filter follows whichever finger was last pressed or moved (global `cutoff`). In arp mode, `setArpCutoff()` updates every currently sounding arp voice.

**Arpeggiator**
- Uses a lookahead scheduler: `setInterval(arpTick, 25)` schedules any notes falling within the next 100 ms using `ctx.currentTime`, so timing stays tight even if the page stutters. Don't switch this to plain setTimeout per note.
- Follows the master tempo. The Rate slider picks a note value from `ARP_RATES` (1/4 to 1/32, with triplets); `arpStep()` turns it into seconds.
- If the beats are playing when the arp starts, `nextGridTime()` delays its first note to the next matching point on the drum grid so the two line up. Starting the beats while the arp runs does the reverse: the sequencer begins on the arp's next note.
- Each step creates a new voice and releases it after `GATE` (0.7) of the step length. Active arp voices are tracked in `arpVoices`.
- Direction: up, down (sorted held notes, stepped by `arp.step`) or random (never repeats the previous note when there's a choice).
- `arp.queue` records scheduled note times so `draw()` can show which note is actually audible right now.

**Top bar and views**
- Under the header: a Synth / Beats icon switch (sets `body[data-view]`, CSS hides the other view), play/stop for the beats, and the master Tempo slider (60 to 200 BPM, `tempo()`). Beats keep playing while you're on the synth view.

**Drums**
- `DRUMS` lists the eight kit pieces (id, short label, name, `play(t)`). Everything is synthesised: kick is a sine sweeping 190 to 48 Hz plus a noise click; snare is two triangle tones plus highpassed noise; clap is bandpassed noise in three quick bursts then a tail; hats are six detuned square waves (`HAT_FREQS`) plus noise, high and bandpassed. A closed hat chokes a ringing open hat via its own `choke` gain.
- Helpers `gainNode`, `filterNode`, `osc`, `noise` build each hit; `cleanup()` disconnects the output node when the longest source ends. `noiseBuf` is 2 s of white noise made once in `ensureAudio()`.

**Sequencer**
- `patterns[8][8 drums][16 steps]` of 0/1, saved to localStorage (`xysynth.patterns.v2`, wrapped in try/catch). Patterns 1 to 4 start with example beats from `PRESET_BEATS` (1, 3 and 4 are breakbeats, 2 is electro); 5 to 8 start empty. Changing the presets means bumping the storage key, or saved patterns hide them.
- Same lookahead idea as the arp: `seqTick()` every 25 ms schedules steps due in the next 100 ms. `seq.count` counts steps since play, used by `nextGridTime()`.
- `editPat` is the pattern on screen; `playPat` is the one being heard and catches up at the start of each bar, so switching patterns while playing waits for the bar to finish. The playing pattern's button gets an underline.
- Swing (50 to 75%) pushes every other 1/16 later by up to half a step.
- Grid: press a step to toggle it and drag to paint that state across others (`paint`, `elementFromPoint`). Row labels audition the sound. `seq.queue` drives the playhead in `updateSeqUI()` on its own rAF loop.

**Drawing**
- A canvas redrawn every frame in `draw()`: each row gets alternating note columns (C columns marked stronger and labelled with their octave, e.g. C3), Bright/Dark hints drawn on the canvas, and a gap between rows. Held columns tinted, the sounding note brighter, a fading trail, and a glow per finger. Glow size grows as the filter opens. Hue runs across the pitch range via `hueFor()`.
- Canvas is sized to devicePixelRatio (capped at 2) through a ResizeObserver.
- `prefers-reduced-motion` turns the trail off.

## Design and UI conventions

- Colours are CSS custom properties on `:root`, with dark values under `prefers-color-scheme: dark` (guarded by `:root:not([data-theme="light"])`) and again under `:root[data-theme="dark"]`. Canvas colours are read from those tokens with `css()`, so add new colours as tokens, not hard-coded values.
- Layout respects phone safe areas (`viewport-fit=cover` plus `env(safe-area-inset-*)` padding) and uses `height: 100%` rather than `100vh`. On phone widths (max 520px) the side margins drop from 16px to 8px to give the pad and beat grid more room.
- Short landscape screens (`orientation: landscape` and `max-height: 540px`) get their own layout: header and bar share one row, the synth view puts the fx panels in a scrolling column right of the pad, and the beats view puts patterns and Kit in a column right of the grid so the steps get wider. Check both orientations after layout changes.
- The pad uses `touch-action: none` and pointer capture per pointer. Keep that, or scrolling and gestures will fight the instrument.
- Controls: selects for scale and sound, range inputs with a live `<output>` readout, a segmented radio group for arp direction. Keep focus-visible outlines on anything interactive.
- UI copy should be short, plain and conversational. No marketing language, and avoid em dashes.

## Ideas I might pick up next

- Latch mode so a mouse can hold several notes (click to toggle)
- Root key picker, and octave range control
- More arp directions (up-down), gate length and octave spread
- Save and recall settings (patterns already save)
- Per-step accent, pattern copy, drum sends to delay and reverb

## Working with me

I come from animation, motion and sound design rather than engineering, and my maths is GCSE level. When explaining audio or DSP concepts, start simple and build up. Prefer small, focused changes and tell me in plain language what changed and why.
