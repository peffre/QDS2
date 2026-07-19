# QDS-II/B — Per-Partial Resynthesis Engine

A playable additive synthesizer in a single HTML file, modelled on the resynthesis
architecture of the Fairlight CMI Series II. Draw harmonic spectra with a light pen,
sculpt per-partial amplitude contours across time, bend the partial series out of
harmonicity, and play it from a MIDI controller or the computer keyboard.

No build step, no dependencies, no server. Open `fairlight_resynth_v2.html` in a
browser and press a key.

---

## Contents

| File | Description |
| --- | --- |
| `fairlight_resynth_v2.html` | Current engine — per-partial oscillator bank, 16 frames, waterfall display, Web MIDI |
| `fairlight_resynth.html` | Earlier build — PeriodicWave crossfade engine, 8 frames, harmonic-only |
| `README.md` | This document |

Both files are standalone and can be opened independently.

---

## The architecture

The Series II worked by describing a sound as a sequence of harmonic *frames*: a
snapshot of every harmonic's amplitude at one instant in time, with the instrument
scanning through the frames to reconstruct an evolving timbre. This engine follows
that model directly.

**32 partials × 16 frames.** The frame grid is the entire sound. Read across it one
way and you get a spectrum; read the other way and you get one partial's amplitude
envelope. Both readings are editable, and they're the same data.

**One oscillator per partial, per voice.** Each voice instantiates up to 32
independent sine oscillators. Partial *n* gets its own gain node running an amplitude
contour built from the 16 frame breakpoints via `setValueCurveAtTime`. The frame grid
therefore *is* a bank of 32 per-partial envelopes — no interpolation between two
fixed waveforms, no wavetable. Partials that are silent in every frame, or that would
land above Nyquist at the played pitch, are culled at note-on, so sparse patches
stay cheap.

**Inharmonic partial tuning.** Partial frequencies are

```
f(n) = f0 · n^STRETCH · √(1 + B·n²)
```

`STRETCH` warps the series geometrically (1.0 = harmonic; >1 stretches toward
gamelan and stretched-piano tuning). `B` is the stiffness term that produces the
inharmonicity of real struck and stiff-stringed instruments — small values give
piano-string stretch, large values give bell clang. Both are live on held notes, so
a sustained chord can be bent from consonant to metallic in real time.

**Per-note signal path.**

```
32 × [ sine osc → gain (per-partial contour) ]
        ↓
      sum → lowpass filter (ADSR-modulated) → voice VCA (ADSR × velocity)
        ↓
   compressor → master → analyser → output
```

Vibrato, pitch bend, and fine tune are applied to `detune` in cents, which scales all
partials proportionally — inharmonic ratios survive modulation intact, as they should.

8-voice polyphony with oldest-note stealing.

---

## The display

Three views of the same frame grid, switched with the EDIT MODE toggle.

### SPECTRUM
The Page 4 view. 32 amplitude bars for the currently selected frame. Drag the light
pen across them to sweep in a spectrum — the pen interpolates between bars, so a
single gesture writes a whole contour. Clicking a bar also selects that partial for
envelope editing.

Tools: CLEAR FRAME, COPY → NEXT (build evolving sequences one edit at a time),
COPY → ALL, DAMP ½.

### ENVELOPE
The axes rotate. X is time across frames 1–16, Y is the amplitude of the selected
partial, with all other partials drawn behind as dim ghost curves. An amber cursor
sweeps the timeline while a note plays. Step through partials with the ◀ ▶ buttons.

Tools: CLEAR, EXP DECAY and SWELL shape generators, and APPLY SHAPE → ALL PARTIALS,
which normalizes the selected contour and imposes it on every partial scaled to that
partial's own peak — a fast way to give a whole spectrum a shared articulation
without destroying its timbre.

### WATERFALL
The Page D view: oblique-projection ridges receding up and to the right, with
hidden-line removal done the classic way — each ridge painted back-to-front with an
opaque fill beneath its curve, so nearer ridges occlude what's behind them. Ridge
brightness is depth-cued.

Three row orientations:

- **SPECTRA** — one ridge per frame, each showing that frame's 32-partial spectrum.
- **WAVEFORMS** — one ridge per frame, each showing the computed single-cycle
  waveform, recalculated live against the current stretch and inharmonicity. Rows
  share a global normalization so level evolution reads in the ridge heights.
- **CONTOURS** — one ridge per partial, each showing that partial's 16-point
  envelope. The whole envelope bank as terrain.

The ridge being scanned lights amber and sweeps through the stack during playback;
in CONTOURS an amber trace cuts diagonally across all 32 ridges at the current morph
position. The view is select-only: click a ridge to jump to that frame or partial,
then switch to SPECTRUM or ENVELOPE to edit it.

---

## Parameters

**PARTIAL TUNING** — STRETCH (0.5–2.0), INHARM B (0–20 ×10⁻³), with a live readout
of where partial 32 lands so you can see how far the series has been bent.

**FRAME SCAN** — FRAME TIME 10–600 ms per frame; scan mode 1-SHOT (holds the final
frame), LOOP, or PING-PONG. At short frame times the morph itself becomes an
audible modulation.

**AMPLITUDE ENVELOPE** — ADSR over the whole voice, scaled by MIDI velocity.

**FILTER** — resonant lowpass, log-scaled cutoff (40 Hz–20 kHz), Q to 18, with a
bipolar envelope amount of ±4 octaves. Cutoff and Q are live on held notes.

**VIBRATO** — rate, depth in cents, and onset delay.

**OUTPUT** — octave (±2), fine tune (±100 cents), level.

---

## Voice library

### Classic
- **SAWTOOTH** — 1/n.
- **TRIANGLE** — odd harmonics at 1/n². Note: the engine stores magnitudes only
  (all partials in sine phase), so the waveform display won't show the pointed
  shape a true sign-alternating triangle produces. The magnitude spectrum, and the
  sound, are correct.
- **HOLLOW SQUARE** — odd harmonics at 1/n.
- **SUPER SAW** — not a detune stack, but what a detune stack actually *sounds*
  like: a full saw spectrum where every partial beats at its own incommensurate
  rate, with beat depth growing from ~13% on the fundamental to ~85% at the top of
  the series. Loads at 50 ms ping-pong so the beating runs continuously. Worth
  viewing in the CONTOURS waterfall.
- **DRAWBAR ORGAN** — classic drawbar registration with slight per-frame drift.
- **STRING ENSEMBLE** — 1/n^1.15 with shimmer.
- **BRASS SWELL** — dark-to-bright spectral bloom; the canonical resynthesis demo.

### Formant
Built on a formant bank — F1/F2/F3 centers, bandwidths, and relative gains for five
vowels, mapped onto the partial series at a 110 Hz reference fundamental over a
−0.55-slope glottal source tilt.

- **VOX HUMANA** — the simple two-formant morph.
- **VOWEL AH / EE / OO** — static vowels with a little breath drift.
- **VOWEL CYCLE A-E-I-O-U** — continuous five-vowel morph, ping-pong at 150 ms. The
  formant peaks visibly migrate through the SPECTRA waterfall.
- **DIPHTHONG AH → OO** — an eased one-shot "ow" gesture, retriggered per note.

Because the formant centers are fixed in the *partial* domain rather than in absolute
frequency, vowel identity is pitch-relative: most convincingly vocal around the
110 Hz reference octave, more abstract as you play higher. This is itself a
characteristic artifact of frame-based additive vocal synthesis.

### Inharmonic
- **TRUE BELL** — B = 8, sparse partial set, upper partials decaying fastest.
- **STIFF STRING** — B = 0.6 with a hammer-noise burst in the opening frames.
- **GAMELAN** — stretch 1.24, with shimmer beating written into the envelopes.

Presets carry their own tuning coefficients, and the animated ones also set frame
time and scan mode, so they arrive playing correctly.

---

## Playing it

**Computer keyboard** — `A S D F G H J K L ;` for white keys, `W E T Y U O P` for
sharps, `Z` / `X` to shift octave. Or click the on-screen keyboard.

**MIDI** — omni-channel Web MIDI, wired automatically with hot-plug support. The
status readout in the keyboard header shows connected device names.

| Message | Response |
| --- | --- |
| Note on/off | Velocity via a 0.10 + 0.90·v^1.6 curve, scaling peak and sustain |
| CC1 (mod wheel) | Adds up to 45 cents of vibrato depth over the panel setting |
| CC64 (sustain) | Releases deferred while down, flushed on pedal-up |
| Pitch bend | ±200 cents, applied across all partials of all held voices |
| CC120 / CC123 | All sound off / all notes off |

Running-status note-off (note-on with velocity 0) is handled.

---

## Requirements and limitations

Requires a browser with Web Audio. Chrome, Edge, and Firefox are all fine; Safari
runs the audio engine but does not support Web MIDI.

`requestMIDIAccess` needs a secure context and a permission grant. Inside a sandboxed
preview frame it may be blocked and the status line will read NOT SUPPORTED — open
the file directly in a browser to get the permission prompt.

Audio contexts can't start without user interaction, so press AUDIO: ON or simply
play a note.

At full 8-voice polyphony with dense spectra the engine is running up to 256
oscillators. Modern machines handle this comfortably, but on constrained hardware
sparse patches (bells, organ, formants) are noticeably lighter than dense ones
(super saw, sawtooth).

Partials are stored as magnitudes with no phase information, and the frame grid is a
fixed 16-step resolution.

---

## Possible extensions

- Per-partial oscillator bank moved into an `AudioWorklet` for higher polyphony
- Drag-editing directly on the waterfall ridges; rotatable view angle
- Velocity mapped to spectrum (brighter partials or frame offset with harder playing)
- Pitch-invariant formants — evaluating the formant curves against absolute
  frequency at note-on rather than baking them into the partial domain
- Patch save/load and export of frame data
