# XY Synth

A browser instrument with three synths and three drum machines. Each synth is played on an X/Y pad (X picks a note from its scale, Y sets a lowpass filter cutoff) or programmed on a step grid. On the pad, one finger plays a sliding note; with Arp on, two or more fingers run an arpeggiator (which can be held), and with Arp off they play chords. The beats view has three 16-step drum machines (A B C, all playing together) with synthesised 909 and 808 kits and a load-your-own break. In Live mode play loops each engine's chosen pattern; in Song mode it plays a song, a row of patterns per bar. Built with the Web Audio API, no libraries, no build step.

## Project shape

- Everything lives in one self-contained HTML file (currently `xy-synth.html`; rename to `index.html` for GitHub Pages). CSS and JS are inline.
- Keep it as a single file with no dependencies unless I ask otherwise. The only external resource is the Barlow Semi Condensed font from Google Fonts, with a system font fallback.
- The version number shows in Help, next to the title, so I can tell when a push has reached my phone. With every commit that changes the page, add a new entry at the top of the Version history list (`#history` in the Help menu): the next number (v2.8, v2.9...) and one short, plain line about what changed. The title's number is read from that top entry, so that's the only place to update.
- To test, just open the file in a browser. Audio only starts after a user gesture (first pointerdown calls `ensureAudio()`), which browsers require.
- GitHub Pages deploys from `main` only. Feature work goes on a branch and gets merged to `main` to test on the phone.
- The arpeggiator, chords and arp hold need multitouch, so test them on a phone or tablet. A mouse only gives one pointer.
- Headless Chrome testing: with `--virtual-time-budget` the audio clock (`ctx.currentTime`) never moves and animation frames barely run, so anything timed against the audio clock (playheads, arp queues) needs a real-time run, e.g. over the DevTools protocol.

## How the code is organised

All JS sits inside one IIFE. Main pieces, top to bottom:

**Data**
- `SCALES`: name plus semitone steps from the root. `scaleNotes(scale, from)` gives two octaves of a scale (plus the C on top) from a C.
- `PRESETS`: each sound is a list of oscillators (`type`, `oct` offset, `det` detune in cents, `g` gain) plus filter resonance `q`, a starting envelope (`atk`, `dec`, `rel` in seconds, `sus` 0 to 1) and vibrato depth `vib` in cents. Picking a sound (`loadPreset()`) copies its envelope and resonance into the synth, and the Sound panel sliders take over from there. To add a sound, add an entry here; the dropdown builds itself.
- The pad has `ROWS` stacked keyboards, each two octaves (`SPAN = 24`) from C2 (`LOW = 36`): 2 rows (to C6) on big screens, 3 rows (to C8) on phones. `ROWS` follows the `phoneMQ` media query and live pointers are remapped (`remapPointers()`) when it changes.

**Synths** (`synths`, three of them; `cur` is the one on screen)
- Each synth object holds everything about it: `preset`, `scale`, `mode` (`"pad"` or `"steps"`), `env` `{a, d, s, r}`, `filt` (a Cutoff and Resonance for each mode), arp settings (`arpOn`, `arpRate`, `arpDir`, `arpHold`), `oct` (the step grid's lowest C), its effects `rack`, its step patterns (`pats`, `editPat`, `playPat`), and its live sound: `bus`, `voices`, `tails`, `arp`, `latch`. Defaults are in `SYNTH_DEFAULTS`: Acid, Supersaw pad, Glass, all minor pentatonic.
- `buildNotes(syn)` fills `syn.notes` (the pad's bottom row; a higher row adds `row * SPAN`) and `syn.seqNotes` (the step grid's rows, two octaves from C(1 + oct)).
- One set of controls serves all three synths. `bindSynthUI()` puts the synth on screen into every control (selects, sliders, arp, mode, step grid); the controls' handlers write into `cur`. Everything a synth is playing (held arps, step patterns) carries on when another synth is on screen.
- The tabs (`#tab0` to `#tab2`) get a dot (`.busy`) while that synth's arp is held (`updateTabs()`).

**Audio graph** (built once in `ensureAudio()`)
```
synth voices -> syn.bus (0.5) per synth          -> that synth's rack   -> mixer fader (chan.node) -> out -> compressor (-14 dB, 4:1) -> destination
drum hits    -> dm.bus (0.8) per drum machine     -> that machine's rack -> mixer fader (chan.node) -> out

each rack (buildRack):
input -> distDry ---------------------------------> mix -> out
      \-> drive -> shaper -> distTone -> distWet -/   |-> delaySend -> delay <-> tone lowpass 5 kHz -> feedback (loop)
                                                       |                           \-> out, and -> reverbIn
                                                       \-> reverbIn (Amount) -> convolver -> level -> out
```
- Buses are made by `newBus(eng)` (a synth or drum machine), after the racks. Stop swaps in fresh ones (see Sequencer).
- Every synth and every drum machine has its own effects rack: distortion, delay and reverb. A rack is data: `rack.values` holds the raw slider positions, `rack.unit`/`rack.saved` the delay's ms/notes mode, `calcFx(rack)` turns values into settings (`rack.st`) and readout text (`rack.out`), `applyFx(rack, rebuildReverb)` pushes settings into the audio nodes (`rack.n`). `loadFxUI(rack)` puts a rack onto its sliders and `showFx(rack)` writes the readouts.
- Sliders: the synth controls have no id prefix (`dAmount`, `xTone`...), the drum racks' start with `k` (`kdAmount`, radio name `kdUnit`). `boundRack(prefix)` says which rack a slider is driving (`dcur.rack`, or `cur.rack`). The synths' Distortion panel (`distHTML("")`) sits in the Sound panel, and Delay and Reverb (`delayRevHTML("")`) in Effects. The drums have all three in Drum effects. Drum racks start with every Amount at 0.
- Distortion: a fixed soft-clip curve (`makeDriveCurve()`, tanh, 2x oversampled). Amount sets the `drive` gain into it (`0.25 * 60^amount`) and turns the output down to roughly match (`1 / (1 + 3 * amount)`). Tone is a lowpass after it (1 to 16 kHz). Clean and driven paths are crossfaded rather than mixed, because the oversampler delays the driven path slightly and mixing the two would comb-filter. At Amount 0 the driven path is unplugged after 150 ms, so it costs no CPU.
- The reverb's Amount is a gain before the convolver (`reverbIn`), so at 0, or with nothing playing, the browser can let the convolver go idle. That matters with four reverbs. The impulse is only built while Amount is above 0 (`reverbStale`), always just after the current event, so it never holds up a note.
- The first `applyFx()` on a fresh rack jumps straight to its settings (`n.fresh`) rather than gliding, so nothing swells in.
- Resting racks (CPU): a delay's feedback loop circulates tiny, never-quite-zero echoes forever, which would keep that delay and the reverb it feeds working even in silence, for every rack that has ever made a sound. A 500 ms check opens a rack's echo loop (`n.feedback.disconnect()`, `n.asleep`) once it has been quiet for longer than its longest tail (release + delay decay + reverb length + 1 s), so the browser can let the delay and reverb idle. `wakeRack()` closes the loop again; it's called from `startVoice()` and `playRow()`. A synth counts as busy while it has finger voices, tails or a running arp; a drum machine while the beats play (its tail allowance is its loaded break's length, or 2 s).
- Delay echoes also feed the reverb.
- Delay "Decay" is a time in seconds (0.1 to 12, exponential slider). Feedback is derived from it with `feedbackFor(time, decay) = 10^(-3 * time / decay)`, i.e. the echoes fall by 60 dB over the decay time. Capped at 0.95.
- An "ms / notes" switch in the Delay header (`setDelayUnit()`) lets Time and Decay be set as note values instead. Every rack starts in notes mode (`makeRack()` switches it, so the ms positions are kept for later). In notes mode Time steps through `SYNC_TIMES` (1/16 to 1 bar, with T triplet and D dotted) and Decay through `SYNC_DECAYS` (1/2 beat to 8 bars), both stored in beats and converted to seconds from the master tempo, so moving the Tempo slider retimes every rack's delay. Each mode remembers its own slider positions. Delay line max is `MAX_DELAY` (4 s, one bar at 60 BPM).
- Reverb uses a generated impulse response (`makeImpulse(size)`): stereo noise with a power-curve fade, 0.3 to 6 seconds long depending on room size. It's rebuilt on a 120 ms debounce when the size slider moves. `swapReverb()` makes a fresh convolver each time and crossfades from the old one over `REVERB_FADE` (0.3 s), because swapping the buffer on a live convolver cuts the tail with a click.

**Mixer** (`mix`, six channels: synths 1 2 3 then drums A B C, the same order as `engines` and the song's columns)
- Each channel: `name`, `short` (1 2 3 A B C), `rack`, `syn` or `dm`, `level` (slider 0 to 100, 80 is 0 dB, `faderGain()` squares it), `mute`, `solo`, and `node`, a gain after the rack, so level, mute and solo take the echoes and reverb with them. `chanGain()` works out mute and solo (anything soloed silences the rest), `applyMix()` glides every fader there. Fader nodes are made in `ensureAudio()` (`chanOut()`). Each synth and drum machine points back at its channel (`syn.chan`, `dm.chan`).
- The `#mixMenu` popsheet opens from the faders button (`.mixbtn`) on every screen size. All six strips (`.strip`) sit side by side so they fit on one screen, phones included: name and sound or kit, M and S, three short effect sliders (short labels Rev / Dly / Dist on phones), then a vertical Level fader (`.vfader`, `writing-mode: vertical-lr; direction: rtl`) that takes whatever height is left. The sheet has a fixed height and one stretched grid row so the faders can grow. Strips are built in JS; ids start with `m` and the channel number (`m0level`, `m3rMix`, `m1mute`).
- Reverb, Delay and Distortion are the racks' own `rMix`, `dAmount` and `xAmount`, so they move the effects panels' Amount sliders too (when that panel is showing the rack). `showMix()` loads everything when the mixer opens.
- The Drums panel's Level (`kLevel`) is the same value as the mixer fader of the drum machine on screen (`setLevel()` keeps both in step).

**Sound panel** (per synth: Envelope, Filter, Distortion)
- Envelope: time sliders run 0 to 100 and map exponentially over `ENV_TIMES`. Changes apply to the next note.
- Filter: Cutoff (100 Hz to 14 kHz, exponential, `cutFromSlider()`) and Resonance (slider 0 to 100 is Q 0 to 20, `RES_SCALE`). Each mode has its own filter, `syn.filt.pad` and `syn.filt.steps` (`{cut, res}`); the sliders show the one for the synth's current mode and swap when the mode changes. Pad starts fully open with the sound's resonance; Steps starts at both sliders' midpoints (about 1.2 kHz, Q 10). Picking a sound loads its resonance into the current mode's filter.
- In Pad mode Cutoff is the top of the pad's filter range: `cutoffFor(ly, syn)` runs from `CUT_MIN` (60 Hz) at the bottom of a row to the Pad cutoff at the top, so each row has its own sweep. In Steps mode every note's filter sits at the Steps cutoff. `filtFor(ly, syn)` picks the filter for a voice (step notes have no `ly`). Moving either slider sweeps what's already sounding (`retune()`, including notes scheduled but not yet heard).

**Voices**
- `startVoice(syn, midi, ly, t, gate)` builds a fresh voice: oscillators -> per-oscillator gain -> its own lowpass filter (cutoff and Q from `filtFor(ly, syn)`) -> amp envelope -> `syn.bus`. The amp ramps to full over the attack, then curves down to sustain (`setTargetAtTime`, time constant decay / 3) and holds there until release. Optional `t` schedules it on the audio clock. Optional `gate` (arp and step notes) cuts the attack short when the release would start before it finishes. Without that, the pending attack ramp overrides the release and the note rings at full volume until it's hard-stopped (a click).
- Finger notes live in `syn.voices` (pointerId -> voice) and release with `stopVoice(v)`, starting from the real current level via `holdAt()` (`cancelAndHoldAtTime`, with a fallback for Firefox). `glideVoice()` slides pitch and cutoff.
- Arp and step notes go through `playTail(syn, midi, t, gate, ly)`: scheduled with their own release (`stopVoiceAt()`), tracked in `syn.tails`. At most `MAX_TAILS` (12) ring per synth; past that, the oldest (usually a quiet release tail) is stolen (`stealVoice()`) when the new note starts.
- Each voice carries `ly`, the pad height it takes its filter from (null for step notes, which use the Steps cutoff): `voiceCut(v, syn)`. Filter moves glide with `FILTER_GLIDE` (0.08 s time constant) so changes sweep rather than snap.

**Pointers and modes** (fingers always belong to `cur`)
- `pointers` is a Map of pointerId to `{id, x, y, row, ly, idx, midi, gi, latched}`. x and y are normalised 0 to 1 over the whole pad; `mapPointer()` works out `row` (0 = bottom), `ly` (0 at the top of the row to 1 at the bottom), the column `idx`, and `gi` (column counted across all rows, used for colour).
- `refresh(syn)` decides what should be sounding: the arp runs in Pad mode with Arp on and 2+ fingers or a hold; `playsOwnNote()` says which fingers get their own voice. With Arp off that's every finger (chords). With Arp on it's only a single finger (`leadFinger()`), which plays over a held arp if there is one, except a finger that helped make the hold (`p.latched`), so lifting a held chord doesn't play stray notes.
- Switching synth, view or mode calls `dropPointers()`: fingers already down stop counting.
- `lastLy` is the last touched finger's height, for arp notes with no finger of their own.

**Arpeggiator** (one per synth, `syn.arp`)
- Uses a lookahead scheduler: `setInterval(() => arpTick(syn), 25)` schedules any notes falling within the next 100 ms using `ctx.currentTime`, so timing stays tight even if the page stutters. Don't switch this to plain setTimeout per note.
- Follows the master tempo. Rate picks a note value from `ARP_RATES` (1/4 to 1/32, with triplets); `arpStep(syn)` turns it into seconds. Each note releases after `GATE` (0.7) of the step.
- If the beats are playing when an arp starts, `nextGridTime()` delays its first note to the next matching point on the drum grid so the two line up. Starting the beats while an arp runs does the reverse.
- Direction: up, down (sorted notes, stepped by `arp.step`) or random (never repeats the previous note when there's a choice).
- Each note takes its brightness from the finger holding it (`lyOfNote()`), and moving a finger retunes just that finger's arp notes (`setArpCutoffFor()`).
- `arp.queue` records scheduled note times so `draw()` can show which note is actually audible right now (and drains the other synths' queues).
- Arp hold: `checkHold()` runs every frame (from `draw()`) for `cur`. When the same set of notes (`heldNotes()`, so filter moves don't count) has been down for the Hold after time (`ARP_HOLDS`, Off to 5 s, default 2 s), `setLatch(syn)` stores it in `syn.latch` (`notes`, plus each note's `row` and `ly` in `info`), and the arp keeps playing it with no fingers down. `arpNotes(syn)` is what the arp plays: the fingers when `fingersDriveArp(syn)` (2+ down, and not just some of the held notes), otherwise the held set. While an arp is held, any new set of 2+ fingers replaces the hold straight away (no waiting), so letting go keeps the newest notes; lifting fingers one by one only leaves a subset, which doesn't count as new. Fingers resting on held notes still set those notes' brightness. Held notes stay lit on the pad.
- The Hold button (`#arpHold`) is always visible: dimmed (`.off`) until there's something to hold, then it fills while a new set counts up (after 20% of the time, via `--fill`), and stays solid once held. It shows `cur`. Tapping it (`releaseHold()`) stops that synth's arp but not the beats or the other synths. Turning Arp off or switching the synth to Steps also drops its hold.

**Steps mode** (per synth)
- A note grid (`#sgrid`, rebuilt by `renderSynthGrid()` whenever the synth, its scale or octave changes): rows are `syn.seqNotes` with the highest at the top, 16 steps, chords allowed. Patterns are stored by row number up the scale (`[8 patterns][SEQ_ROWS_MAX = 25 rows][16 steps]`), so they follow scale and octave changes. Octave − / + moves the grid's lowest C (C1 to C5). Row labels audition their note.
- Saved in localStorage (`xysynth.synthPatterns.v1`, all three synths). Synth 1's pattern 1 starts with an acid line (`defaultSynthPatterns()`); bump the key if the defaults change.
- Played by the sequencer's clock (below), so step patterns only play while the beats are playing. In Live only synths in Steps mode play theirs; in Song every synth plays what the song gives it, Pad mode included.

**Top bar and views**
- View tabs: synths 1, 2, 3 and Beats. `body[data-view]` is `synth`, `beats` or `song`; CSS hides the other views, and `.synthonly`, `.beatsonly`, `.songonly` hide controls by view. `setView()` switches, drops the fingers, and checks the matching tab (none for the song). `engineView` remembers the last synth or beats screen. Then play/stop (beats and every Steps synth; stop also stops every synth's arp, held or played, via `stopArps()`, which lets go of the fingers too, and cuts all sound with `silence()`), the arp group (`.arpgrp`, a rounded band like the tabs: Arp on/off, Hold, and on phones the Arpeggio settings menu button; in the beats view only Hold shows, and the whole group hides in Steps mode), Pad / Steps (`body[data-mode]` follows `cur.mode`; `.padonly` and `.stepsonly` hide what doesn't apply), and the master Tempo slider (60 to 200 BPM, `tempo()`). `.synthonly` and `.beatsonly` hide controls by view on every screen size.
- The bar is built from two `.barrow` groups. On bigger screens they're `display: contents`, so everything runs along one line. On phones they're two fixed rows (tabs, play, the arp group; then Pad / Steps and the menu buttons), with Tempo on a third. In phone landscape the two rows share one line. The phone rows are full now (six menu buttons in the synth view), so new controls should go elsewhere, e.g. the mode bar.
- The mode bar (`.modebar`) sits under the top bar on every screen size: Live / Song (`playMode` radios), then in the beats view Drums A / B / C (`machine` radios), and in the song view a one-line hint.

**Live and Song** (`playMode`, also on `body[data-play]`)
- Live is how things always worked: each engine plays the pattern picked on its screen. Song plays `song.rows` top to bottom, a bar per row, and loops.
- Picking Song shows the song view. The tabs still open each synth and the beats while Song stays the play mode (so patterns can be edited while the song plays); tapping Song again returns to the song. Picking Live goes back to `engineView`. The playMode radios use `click`, not `change`, so tapping an already-picked Song still works. Changing mode resets `seq.bar`, so a song starts from its first row at the next bar.
- `song` is `{rows: [[p x 6]]}`: a pattern per engine (0 to 7) or -1 for off, in `engines` order. Saved in localStorage (`xysynth.song.v1`; bump if `defaultSong()` changes). The default is drums A alone for 2 bars, then synth 1's acid line joins, 8 bars.
- `renderSong()` rebuilds the list (`#songList`): bar number, six `.slot` buttons (a little gap before the drum columns, `.first`), a × to remove the row, and "+ Add a bar" at the end (copies the last row). The list scrolls; the header (`#songHead`) doesn't. `markSongRow()` lights the row being heard and scrolls it into view while the song view shows.
- Tapping a slot opens `#patPick`, a small fixed popup with Off and 1 to 8 (`openPicker()` / `closePicker()`), placed under the slot (or above it near the bottom), with the scrim behind. Escape or tapping outside closes it.

**Phone layout and menus**
- On phones (`max-width: 520px`, or landscape with `max-height: 540px`; same query as `phoneMQ`) the header is hidden and the pad fills the screen. Menu buttons (`.menubtns`, each with `data-menu` naming its sheet): Scale, Sound, Effects (synth view), Drum effects, Kit (beats view), Mixer, Settings and Help. Closed sheets are hidden (`.sheet:not(.open)`). The Arpeggio settings button lives in the arp group instead.
- Each menu is a `.sheet` overlay (`openMenu()` / `closeMenu()`, one open at a time, `.scrim` behind it, Escape or tapping outside closes). The Sound, Effects, Arpeggio and Drum effects sheets wrap panels that, on bigger screens, sit in a `.fx` grid (`display: contents` on the sheet). The `.fx` grids place panels with named `grid-template-areas` (synth: env, filt, dist, delay, rev, arp; drums: drums, delay, rev, dist); on phones `.fxgroup` resets to `grid-area: auto`.
- The Sound sheet holds the sound list (`.phoneonly`, hidden on bigger screens where the header dropdown does the job), Envelope, Filter and Distortion.
- Scale, Sound and Kit lists are radio lists built from the header selects and kept in step with them (`listSyncs`); picking one sets the select and fires its `change` event. Scale and Kit sheets are `.listsheet` (phones only).
- Settings (`#settingsMenu`, `.setbtn`) has Latency: `LATENCIES` (Lowest is the browser's `"interactive"`, then 20 ms to 1 s; the slider's max follows the list) is passed as `latencyHint` when the AudioContext is made, and saved in localStorage (`xysynth.latency`). Higher gives the device more time per batch of sound, so fewer crackles, but more lag. The browser only takes it at creation, so a change (on the slider's `change`, not `input`) runs `restartAudio()`: it stops the beats and arps, clears every synth's voices and nodes and the drum state, closes the old context, builds a new one with `ensureAudio()`, restarts held arps (`refresh()`) and the beats. Anything new that holds audio nodes must be cleared there too. The readout shows what the browser actually gave (`baseLatency + outputLatency`).
- Mixer, Settings and Help (`.popsheet`) open as menus on every screen size: on bigger screens the menu buttons hide apart from these three, which sit at the end of the bar. It has the title and version (`#verOut`), a Website button top right linking to the XY Synth page on my portfolio site, and two tabs (`helpTab` radios set `.helpgroup[data-tab]`): How to use, and Version history. Keep How to use up to date when features change.
- Open menus share one overlay style (`.sheet.open`) on every screen size, with the `.scrim` behind.

**Drum machines** (`drums`, three of them, A B C; `dcur` is the one on screen)
- Each holds its `kit`, `rack`, `bus`, `openHat`, pattern `banks` with `editPat`/`playPat`, and its break (`brk`: `buf`, `bytes`, `name`, `bars`, `voice`). They all play together; B and C start with empty patterns, so they're silent until something is written.
- One set of Beats controls serves all three, like the synths: `bindDrumUI()` puts `dcur` onto the kit select and list, grid, row labels, break buttons, Drum effects sliders, Level and the Drums panel title (`#dmName`).
- The drum sound helpers play into `hit`, the machine being played right now: `playRow(dm, r, t)` sets it before calling the kit, so the helpers use `hit.bus` and `hit.openHat`.
- Kits are saved per machine (`xysynth.kits`, an array). Drums A picks up the old single `xysynth.kit` setting.

**Drums and kits**
- A Kit menu replaces Sound in beats view (`.kitctl`; `.soundctl` is hidden and `.scalectl` goes invisible but keeps its space). It sets the kit of the drum machine on screen.
- `DRUM_ROWS` is the fixed set of eight rows (bd sd cp rs lt ht ch oh). `KITS` maps each kit to a `play` function per row id: `k909` and `k808` are synthesised from shared helpers (`snare`, `clap`, `rim`, `tom`, `hat`, `closedHat`, `openHatHit`) with different settings. The 808 has a longer round kick, sine snare tones and noise-free hats. A closed hat chokes a ringing open hat via its own `choke` gain.
- `brk` (Break) is special: the user loads an audio file (`#loadBreak` opens the hidden `#breakFile` input; its `accept` lists audio file types by name and must stay audio only: Android adds a camera option for anything it reads as video, such as a `.webm` extension. On Android the file browser may still sit behind the "Photos and videos" picker, since Android picks which app handles the request, not the page). Each machine has its own loaded break. The loop is 1, 2 or 4 bars (`brk.bars`): `guessBars()` picks whichever needs the least speed change at the current tempo when a file loads, and the `#breakBars` button (shown once a loop is loaded) cycles it. It's cut into 8 equal slices across the whole loop, one per row, so in a 2-bar loop a slice lasts 1/4 bar and the preset chops (written for one bar) cut each slice halfway. `playSlice()` speeds the loop up or down so its bars fit the current tempo (this also shifts pitch, like a classic sampler), starts at the slice's offset and plays on until the next slice cuts it off (one slice at a time). The cut-off slice is faded and stopped, not left playing silently.
- The loaded file's bytes, name and length in bars are kept in IndexedDB (`breakStore()`, db `xysynth`, store `files`, record `{bytes, name, bars}`, key from `breakKey(dm)`: `break` for A, so its old loop carries over, `break.B`, `break.C`) and decoded with an `OfflineAudioContext`, so it's ready before the first tap after a reload.
- `playRow(dm, r, t)` is the single entry point for the sequencer, grid audition and row labels.
- Helpers `gainNode`, `filterNode`, `osc`, `noise` build each hit; `cleanup()` disconnects the output node when a source ends. `noiseBuf` is 2 s of white noise made once in `ensureAudio()`.

**Sequencer** (one clock for all six engines)
- Each drum machine has two pattern banks (`dm.banks`): its drum kits share `drums`, the break has `break`. `bankOf(dm)` / `patsOf(dm)` give the one for its current kit. Each is `[8 patterns][8 rows][16 steps]` of 0/1. Keys: A keeps `xysynth.patterns.v2` and `xysynth.breakPatterns`, B and C add `.B` / `.C`.
- Drums A's patterns 1 to 4 start from `PRESET_BEATS` (1, 3 and 4 breakbeats, 2 electro), and its break patterns 1 to 3 from `PRESET_CHOPS` (slice per 1/8; pattern 1 plays the loop straight). Everything else starts empty. Changing presets means bumping that bank's storage key, or saved patterns hide them.
- Same lookahead idea as the arp: `seqTick()` every 25 ms schedules steps due in the next 100 ms (`playRow()` for drums, `playTail()` with gate `GATE` for synths). `seq.count` counts steps since play, used by `nextGridTime()`. `seq.bar` counts bars, for the song.
- `engines` is `[...synths, ...drums]`. Every engine has `editPat` (on screen) and `playPat` (being heard, -1 for nothing). `barStart()` runs at each bar: in Live every `playPat` catches up with its `editPat`, so switching patterns while playing waits for the bar to finish; in Song they come from the song row (`seq.songRow`). The playing pattern's button gets an underline.
- Swing (50 to 75%) pushes every other 1/16 later by up to half a step, for all engines. There's one Swing, shown in the Drums panel.
- Grids: `paintable(el, {get, set, audition, save})` gives both grids press-to-toggle and drag-to-paint (`elementFromPoint`). `seq.queue` entries (`step`, `pats` per engine, song `row`) drive the playheads and the song's lit row in `updateSeqUI()` on its own rAF loop (`heads`, `moveHead()`, `markPat()`, `markSongRow()`). That loop starts on the first frame, after the whole script has run, because it calls into the song code defined further down.
- Stop cuts all sound (`silence()`): the mixer faders dip over 20 ms, every engine gets a fresh bus (`newBus()`) so anything still playing into the old one (scheduled steps, drum hits, a break slice that would ring on to the end of its loop) goes nowhere, finger and arp voices' oscillators are stopped, and 40 ms later the old buses are unplugged, each rack is emptied (`flushRack()`: a new empty delay line, and a new convolver with the same impulse, so no echoes or reverb tail) and the faders come back up.

**Drawing**
- A canvas redrawn every frame in `draw()` for `cur`: each row gets alternating note columns (C columns marked stronger and labelled with their octave, e.g. C3), a soft vertical wash per row (`--row-bright` at the top fading to `--row-dark` at the bottom) that hints the filter is open at the top and closed at the bottom, and a gap between rows. Held columns tinted, sounding notes brighter, a fading trail, and a glow per finger. Glow size grows as the filter opens. Hue runs across the pitch range via `hueFor()`.
- It only redraws when something changed (`padDirty`, set by `markPad()`) or while fingers or a trail are moving; otherwise the last frame stays. Anything that changes what the pad shows must call `markPad()` (already done in `refresh()`, `stopArp()`, `setLatch()`, `resize()`, `remapPointers()`, `bindSynthUI()`, and when the sounding arp note changes). It draws nothing while the canvas is hidden. Colours are read from the CSS tokens once (`readColors()`) and again when light/dark changes, not every frame.
- Canvas is sized to devicePixelRatio (capped at 2) through a ResizeObserver.
- `prefers-reduced-motion` turns the trail off.

## Design and UI conventions

- The palette is a desaturated, almost monochrome teal. The only strong colour is the per-note hue on the pad while playing (`hueFor()`), which stays as it is.
- Colours are CSS custom properties on `:root`, with dark values under `prefers-color-scheme: dark` (guarded by `:root:not([data-theme="light"])`) and again under `:root[data-theme="dark"]`. Canvas colours are read from those tokens with `css()`, so add new colours as tokens, not hard-coded values.
- Layout respects phone safe areas (`viewport-fit=cover` plus `env(safe-area-inset-*)` padding) and uses `height: 100%` rather than `100vh`. On phone widths (max 520px) the side margins drop from 16px to 8px to give the pad and grids more room.
- Short landscape screens (`orientation: landscape` and `max-height: 540px`) use the phone layout above, and the beats view also puts patterns and the Drums panel in a column right of the grid so the steps get wider. Check both orientations after layout changes.
- The pad uses `touch-action: none` and pointer capture per pointer. Keep that, or scrolling and gestures will fight the instrument.
- Controls: selects for scale and sound (radio lists in overlay menus on phones), range inputs with a live `<output>` readout, segmented radio groups (view tabs, Pad / Steps, arp direction, patterns), toggle buttons with `aria-pressed`. Buttons that come and go keep their space so the bar doesn't jump. Keep focus-visible outlines on anything interactive.
- UI copy should be short, plain and conversational. No marketing language, and avoid em dashes.

## Ideas I might pick up next

- Latch mode so a mouse can hold several notes (click to toggle)
- Root key picker
- More arp directions (up-down), gate length and octave spread
- Save and recall settings (patterns already save)
- Per-step accent, pattern copy, step ties and slides
- Filter envelope (for acid squelch in Steps mode)

## Working with me

I come from animation, motion and sound design rather than engineering, and my maths is GCSE level. When explaining audio or DSP concepts, start simple and build up. Prefer small, focused changes and tell me in plain language what changed and why.
