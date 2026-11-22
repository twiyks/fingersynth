# XY Synth

A browser synth played on an X/Y pad. X picks a note from the chosen scale, Y sets a lowpass filter cutoff. One finger plays a sliding held note; two or more fingers run an arpeggiator through the held notes. A second view has a 16-step drum machine with a synthesised 909-style kit. Built with the Web Audio API, no libraries, no build step.

## Project shape

- Everything lives in one self-contained HTML file (currently `xy-synth.html`; rename to `index.html` for GitHub Pages). CSS and JS are inline.
- Keep it as a single file with no dependencies unless I ask otherwise. The only external resource is the Barlow Semi Condensed font from Google Fonts, with a system font fallback.
- A small version label (`<div class="version">`) sits bottom right so I can tell when a push has reached my phone. Bump it (v1.1, v1.2...) with every commit that changes the page.
- To test, just open the file in a browser. Audio only starts after a user gesture (first pointerdown calls `ensureAudio()`), which browsers require.
- GitHub Pages deploys from `main` only. Feature work goes on a branch and gets merged to `main` to test on the phone.
- The arpeggiator needs multitouch, so test it on a phone or tablet. A mouse only gives one pointer.

## How the code is organised

All JS sits inside one IIFE. Main pieces, top to bottom:

**Data**
- `SCALES`: name plus semitone steps from the root. The pad has `ROWS` stacked keyboards, each two octaves (`SPAN = 24`) from C2 (`LOW = 36`): 2 rows (to C6) on big screens, 3 rows (to C8) on phones. `ROWS` follows the `phoneMQ` media query and live pointers are remapped (`remapPointers()`) when it changes. `buildNotes()` fills `notes` with the bottom row's MIDI numbers in the current scale; each note is one column, and a higher row adds `row * SPAN`.
- `PRESETS`: each sound is a list of oscillators (`type`, `oct` offset, `det` detune in cents, `g` gain) plus filter `q`, a starting envelope (`atk`, `dec`, `rel` in seconds, `sus` 0 to 1) and vibrato depth `vib` in cents. To add a sound, add an entry here; the dropdown builds itself.
- Envelope: `env` `{a, d, s, r}` is what new voices use. Picking a sound runs `loadEnv()`, which copies that preset's envelope into `env` and the Envelope sliders; moving a slider changes `env` from there. Time sliders run 0 to 100 and map exponentially over `ENV_TIMES`. Changes apply to the next note, not ones already sounding.
- Filter cutoff maps exponentially from `CUT_MIN` (60 Hz) at the bottom of a row to `CUT_MAX` (14 kHz) at the top of that row via `cutoffFor(ly)`, so each row has its own full filter sweep.

**Audio graph** (built once in `ensureAudio()`)
```
voices -> master (0.5)         -> synth rack -> out -> compressor (-14 dB, 4:1) -> destination
drums  -> drumBus (Drums level) -> drum rack  -> out

each rack (buildRack):
input -> distDry ---------------------------------> mix -> out
      \-> drive -> shaper -> distTone -> distWet -/   |-> delaySend -> delay <-> tone lowpass 5 kHz -> feedback (loop)
                                                       |                           \-> out, and -> reverbIn
                                                       \-> reverbIn (Amount) -> convolver -> level -> out
```
- The synth and the drums each have their own effects rack: distortion, delay and reverb, with the same controls. `makeRack(prefix, sheetId, amounts)` builds a rack's panels from `rackHTML()` into its sheet and returns `{p, el, unit, saved, st, n}`. The synth's ids have no prefix (`dAmount`, `xTone`...), the drum rack's start with `k` (`kdAmount`, radio name `kdUnit`). `readFx(rack)` reads sliders into `rack.st`, and `applyFx(rack, rebuildReverb)` pushes them into `rack.n`. Every effects function takes the rack. The drum rack starts with every Amount at 0.
- Distortion: a fixed soft-clip curve (`makeDriveCurve()`, tanh, 2x oversampled). Amount sets the `drive` gain into it (`0.25 * 60^amount`) and turns the output down to roughly match (`1 / (1 + 3 * amount)`). Tone is a lowpass after it (1 to 16 kHz). Clean and driven paths are crossfaded rather than mixed, because the oversampler delays the driven path slightly and mixing the two would comb-filter. At Amount 0 the driven path is unplugged after 150 ms, so it costs no CPU.
- The reverb's Amount is a gain before the convolver (`reverbIn`), so at 0 the browser can let the convolver go idle. The impulse is only built while Amount is above 0 (`reverbStale`), always just after the current event, so it never holds up a note.
- The first `applyFx()` on a fresh rack jumps straight to its settings (`n.fresh`) rather than gliding, so nothing swells in.
- Delay echoes also feed the reverb.
- Delay "Decay" is a time in seconds (0.1 to 12, exponential slider). Feedback is derived from it with `feedbackFor(time, decay) = 10^(-3 * time / decay)`, i.e. the echoes fall by 60 dB over the decay time. Capped at 0.95.
- An "ms / notes" switch in the Delay header (`delayUnit`, `setDelayUnit()`) lets Time and Decay be set as note values instead. In notes mode Time steps through `SYNC_TIMES` (1/16 to 1 bar, with T triplet and D dotted) and Decay through `SYNC_DECAYS` (1/2 beat to 8 bars), both stored in beats and converted to seconds from the master tempo, so moving the Tempo slider retimes the delay. Each mode remembers its own slider positions. Delay line max is `MAX_DELAY` (4 s, one bar at 60 BPM).
- Reverb uses a generated impulse response (`makeImpulse(size)`): stereo noise with a power-curve fade, 0.3 to 6 seconds long depending on room size. It's rebuilt on a 120 ms debounce when the size slider moves. `swapReverb()` makes a fresh convolver each time and crossfades from the old one over `REVERB_FADE` (0.3 s), because swapping the buffer on a live convolver cuts the tail with a click. The first impulse is built just after the first note (`setTimeout`) so it doesn't delay it.
- Effect settings live in each rack's `st` (read by `readFx(rack)`) so the sliders work before audio has started; `applyFx(rack)` pushes them into the nodes.

**Voices**
- `startVoice(midi, cutoff, t, gate)` builds a fresh voice: oscillators -> per-oscillator gain -> its own lowpass filter -> amp envelope -> `master`. The amp ramps to full over the attack, then curves down to sustain (`setTargetAtTime`, time constant decay / 3) and holds there until release. Optional `t` schedules it on the audio clock. Optional `gate` (arp notes) cuts the attack short when the release would start before it finishes. Without that, the pending attack ramp overrides the release and the note rings at full volume until it's hard-stopped (a click).
- `stopVoice(v)` releases immediately (held notes), starting from the real current level via `holdAt()` (`cancelAndHoldAtTime`, with a fallback for Firefox). `stopVoiceAt(v, when, onDone)` releases at a scheduled time (arp notes) and cleans up via `onended`. `stealVoice(v, t)` fades a voice out fast.
- `updateVoice()` glides pitch and cutoff with `setTargetAtTime` for smooth sliding.

**Pointers and modes**
- `pointers` is a Map of pointerId to `{x, y, row, ly, idx, midi, gi}`. x and y are normalised 0 to 1 over the whole pad; `mapPointer()` works out `row` (0 = bottom), `ly` (height within that row, 0 to 1), the column `idx`, and `gi` (column counted across all rows, used for colour).
- `refreshMode(prevCount)` handles transitions: 0 fingers stops everything, 1 finger plays a single legato `voice`, 2+ stops that voice and starts the arp. Going from 2 back to 1 returns to the held note.
- Each finger has its own filter setting (`p.cutoff`, from its height within its row). The held note follows the single finger; global `cutoff` tracks the last touched finger for that and the readout.
- In arp mode each note takes its brightness from the finger holding it (`cutoffOfNote()`), and moving a finger only retunes that finger's arp voices, including ones scheduled but not yet heard (`setArpCutoffFor()`). Arp voices carry `midi` and `start` for this.
- Filter moves glide with `FILTER_GLIDE` (0.08 s time constant, settling in about a quarter of a second) so changes sweep rather than snap.

**Arpeggiator**
- Uses a lookahead scheduler: `setInterval(arpTick, 25)` schedules any notes falling within the next 100 ms using `ctx.currentTime`, so timing stays tight even if the page stutters. Don't switch this to plain setTimeout per note.
- Follows the master tempo. The Rate slider picks a note value from `ARP_RATES` (1/4 to 1/32, with triplets); `arpStep()` turns it into seconds.
- If the beats are playing when the arp starts, `nextGridTime()` delays its first note to the next matching point on the drum grid so the two line up. Starting the beats while the arp runs does the reverse: the sequencer begins on the arp's next note.
- Each step creates a new voice and releases it after `GATE` (0.7) of the step length. Active arp voices are tracked in `arpVoices`. At most `MAX_ARP_VOICES` (10) ring at once; past that, the oldest (usually a quiet release tail) is stolen when the new note starts. This keeps CPU in check with long releases at fast rates.
- Direction: up, down (sorted held notes, stepped by `arp.step`) or random (never repeats the previous note when there's a choice).
- `arp.queue` records scheduled note times so `draw()` can show which note is actually audible right now.

**Top bar and views**
- Under the header: a Synth / Beats icon switch (sets `body[data-view]`, CSS hides the other view), play/stop for the beats, and the master Tempo slider (60 to 200 BPM, `tempo()`). Beats keep playing while you're on the synth view.

**Phone layout and menus**
- On phones (`max-width: 520px`, or landscape with `max-height: 540px`; same query as `phoneMQ`) the header is hidden and the pad fills the screen. The top row holds the view switch, play, and icon buttons (`.menubtns`, each with `data-menu` naming its sheet): Scale, Sound, Effects, Arpeggio (synth view, `.synthonly`), Drum effects, Kit (beats view, `.beatsonly`), and Help. Tempo sits on its own full-width row below.
- Each menu is a `.sheet` overlay (`openMenu()` / `closeMenu()`, one open at a time, `.scrim` behind it, Escape or tapping outside closes). The Sound, Effects, Arpeggio and Drum effects sheets wrap panels that, on bigger screens, sit in a `.fx` grid (`display: contents` on the sheet). The `.fx` grids place panels with named `grid-template-areas` (env, arp, delay, rev, dist, drums); on phones `.fxgroup` resets to `grid-area: auto`.
- The Sound sheet holds the sound list (`.phoneonly`, hidden on bigger screens where the header dropdown does the job) and the Envelope panel.
- Scale, Sound and Kit lists are radio lists built from the header selects and kept in step with them; picking one sets the select and fires its `change` event. Scale and Kit sheets are `.listsheet` (phones only).
- Help sheet links to the XY Synth page on my portfolio site.

**Drums and kits**
- A Kit menu replaces Sound in beats view (`.kitctl`; `.soundctl` is hidden and `.scalectl` goes invisible but keeps its space). The choice is remembered in localStorage (`xysynth.kit`).
- `DRUM_ROWS` is the fixed set of eight rows (bd sd cp rs lt ht ch oh). `KITS` maps each kit to a `play` function per row id: `k909` and `k808` are synthesised from shared helpers (`snare`, `clap`, `rim`, `tom`, `hat`, `closedHat`, `openHatHit`) with different settings. The 808 has a longer round kick, sine snare tones and noise-free hats. A closed hat chokes a ringing open hat via its own `choke` gain.
- `brk` (Break) is special: the user loads an audio file (`#loadBreak` opens the hidden `#breakFile` input), assumed to be a one-bar loop. It's cut into 8 equal slices, one per row. `playSlice()` speeds the loop up or down so one bar fits the current tempo (this also shifts pitch, like a classic sampler), starts at the slice's offset and plays on until the next slice cuts it off (one slice at a time). The cut-off slice is faded and stopped, not left playing silently.
- The loaded file's bytes are kept in IndexedDB (`breakStore()`, db `xysynth`, store `files`, key `break`) and decoded with an `OfflineAudioContext`, so it's ready before the first tap after a reload.
- `playRow(r, t)` is the single entry point for the sequencer, grid audition and row labels.
- Helpers `gainNode`, `filterNode`, `osc`, `noise` build each hit; `cleanup()` disconnects the output node when a source ends. `noiseBuf` is 2 s of white noise made once in `ensureAudio()`.

**Sequencer**
- Two pattern banks (`banks`): the drum kits share `drums` (localStorage `xysynth.patterns.v2`), the break has its own `break` bank (`xysynth.breakPatterns`). `patterns` points at the current kit's bank and is swapped on kit change. Each is `[8 patterns][8 rows][16 steps]` of 0/1.
- Drum patterns 1 to 4 start from `PRESET_BEATS` (1, 3 and 4 breakbeats, 2 electro). Break patterns 1 to 3 start from `PRESET_CHOPS` (slice per 1/8; pattern 1 plays the loop straight). The rest start empty. Changing presets means bumping that bank's storage key, or saved patterns hide them.
- Same lookahead idea as the arp: `seqTick()` every 25 ms schedules steps due in the next 100 ms. `seq.count` counts steps since play, used by `nextGridTime()`.
- `editPat` is the pattern on screen; `playPat` is the one being heard and catches up at the start of each bar, so switching patterns while playing waits for the bar to finish. The playing pattern's button gets an underline.
- Swing (50 to 75%) pushes every other 1/16 later by up to half a step.
- Grid: press a step to toggle it and drag to paint that state across others (`paint`, `elementFromPoint`). Row labels audition the sound. `seq.queue` drives the playhead in `updateSeqUI()` on its own rAF loop.

**Drawing**
- A canvas redrawn every frame in `draw()`: each row gets alternating note columns (C columns marked stronger and labelled with their octave, e.g. C3), a soft vertical wash per row (`--row-bright` at the top fading to `--row-dark` at the bottom) that hints the filter is open at the top and closed at the bottom, and a gap between rows. Held columns tinted, the sounding note brighter, a fading trail, and a glow per finger. Glow size grows as the filter opens. Hue runs across the pitch range via `hueFor()`.
- Canvas is sized to devicePixelRatio (capped at 2) through a ResizeObserver.
- `prefers-reduced-motion` turns the trail off.

## Design and UI conventions

- The palette is a desaturated, almost monochrome teal. The only strong colour is the per-note hue on the pad while playing (`hueFor()`), which stays as it is.
- Colours are CSS custom properties on `:root`, with dark values under `prefers-color-scheme: dark` (guarded by `:root:not([data-theme="light"])`) and again under `:root[data-theme="dark"]`. Canvas colours are read from those tokens with `css()`, so add new colours as tokens, not hard-coded values.
- Layout respects phone safe areas (`viewport-fit=cover` plus `env(safe-area-inset-*)` padding) and uses `height: 100%` rather than `100vh`. On phone widths (max 520px) the side margins drop from 16px to 8px to give the pad and beat grid more room.
- Short landscape screens (`orientation: landscape` and `max-height: 540px`) use the phone layout above, and the beats view also puts patterns and the Drums panel in a column right of the grid so the steps get wider. Check both orientations after layout changes.
- The pad uses `touch-action: none` and pointer capture per pointer. Keep that, or scrolling and gestures will fight the instrument.
- Controls: selects for scale and sound (radio lists in overlay menus on phones), range inputs with a live `<output>` readout, a segmented radio group for arp direction. Keep focus-visible outlines on anything interactive.
- UI copy should be short, plain and conversational. No marketing language, and avoid em dashes.

## Ideas I might pick up next

- Latch mode so a mouse can hold several notes (click to toggle)
- Root key picker, and octave range control
- More arp directions (up-down), gate length and octave spread
- Save and recall settings (patterns already save)
- Per-step accent, pattern copy

## Working with me

I come from animation, motion and sound design rather than engineering, and my maths is GCSE level. When explaining audio or DSP concepts, start simple and build up. Prefer small, focused changes and tell me in plain language what changed and why.
