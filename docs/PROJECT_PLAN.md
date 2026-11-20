# Project Plan — MPC-inspired Sampler Workstation

Status: **planning** (no application code yet) · Plan author: Opus 5.5 · Last updated: 2026-11-20

This is the single source of truth for architecture, scope, and task assignment. Any agent
(Opus / Sonnet / Haiku) picking up a task should read sections 1–4 and its task row in section 9
before writing code. Update this file when a decision changes — do not let it drift.

Confidence tags: **[Certain]** hard evidence · **[Likely]** strong inference · **[Guessing]** needs
verification before relying on it.

---

## 1. Hard constraints (read first)

### 1.1 Spotify cannot be used as a sample source
- **[Certain]** Spotify's Developer Terms prohibit downloading, recording, sampling, ripping or
  modifying streamed content. Building "load a Spotify track onto a pad" would violate them and get
  the app's client ID revoked.
- **[Certain]** The Web Playback SDK streams DRM-protected (EME) audio. Its output cannot be routed
  into Web Audio / an AudioWorklet, so it is technically uncapturable in the browser anyway.
- **[Likely]** The `audio-features` and `audio-analysis` endpoints (BPM, key, beats) were deprecated
  for new apps in Nov 2024. Do not plan around getting BPM/key from Spotify. Verify at build time.
- **[Likely]** Apps in Spotify "development mode" are capped to a small allow-list of users;
  extended quota requires an application to Spotify. Fine for personal use.

**Resulting Spotify scope:** OAuth 2.0 Authorization Code + PKCE, search, browse user library and
playlists, reference playback (Premium only, via Web Playback SDK / Connect), "now playing" display,
save crate/reference lists inside a project. Pads load **only local, user-owned files**.

### 1.2 Trademarks and trade dress
- "AKAI", "MPC", the logos, and the exact hardware look are owned by Akai Professional / inMusic.
- We build **"inspired-by" layouts** with our own product name, skins, fonts and icons. No AKAI/MPC
  logos in the UI, no copied factory sample libraries or firmware assets, no pixel-copied screens.
- Model names may appear only descriptively (e.g. "Vintage 60 profile — inspired by MPC60").
- Pick a product name in task P0-01 (placeholder below: **"PadStation"**).

### 1.3 Third-party plug-in hosting is native-only
- **[Certain]** Browsers cannot load VST3/AU binaries. The web test build ships **built-in effects
  only**. VST3/AU hosting arrives with the native macOS app (Phase 6).

### 1.4 Real-time audio rules (apply to every engine task)
- No allocation, locks, file I/O, logging or UI calls on the audio thread / inside
  `AudioWorkletProcessor.process()` / JUCE `processBlock()`.
- UI ⇄ engine communication through lock-free ring buffers / `postMessage` with
  preallocated `SharedArrayBuffer` where available.
- All DSP must be sample-rate independent (44.1 / 48 / 88.2 / 96 kHz).

---

## 2. Goals and non-goals

**Goals**
1. Visualise 4 MPC families as switchable, interactive front panels.
2. Emulate each family's workflow (pad banks, sequencing limits, swing, note repeat, screens) and its
   sonic character (bit depth, sample rate, filters, swing timing).
3. Load WAV and other common formats onto pads; chop, edit, and sequence them.
4. Record audio and MIDI; resample; bounce/export.
5. Mixer with built-in mixing/mastering plug-ins; native build hosts VST3/AU.
6. Spotify integration within the limits of 1.1.
7. Web app for testing → installable, signed, notarized macOS app.

**Non-goals (v1)**: Windows/Linux builds, iOS, cloud collaboration, loading real MPC firmware,
reading proprietary `.xpj` project files (stretch goal only: `.pgm`/`.snd` import from 2000XL —
see Open Questions), streaming-service sampling.

---

## 3. Architecture

```
                ┌───────────────────────────── ui/ (React + TS, Vite) ─────────────────────────────┐
                │  Front-panel skins · Pad grid · Screens · Sequencer · Mixer · Browser · Spotify  │
                │                    state: Zustand store  ←→  shared/ schemas                     │
                └──────────────────────────────┬──────────────────────────────────────────────────┘
                                               │  AudioEngine (TypeScript interface)
                     ┌─────────────────────────┴─────────────────────────┐
         engine-web/ │ WebAudioEngine                                     │ engine-native/ bridge
   (test phases)     │ AudioWorklet voices + DSP (TS → later WASM)        │ NativeEngine (TS) ──► JUCE 8
                     └────────────────────────────────────────────────────┘   WebBrowserComponent
                                                                              native functions/events
                                                                                    │
                                                         engine-native/ (C++/JUCE 8, CMake)
                                                         voices, sequencer clock, DSP, recorder,
                                                         VST3/AU host, CoreAudio, CoreMIDI
```

### 3.1 Decisions
| Decision | Choice | Why |
|---|---|---|
| UI | React 18 + TypeScript + Vite, Zustand, Canvas/SVG for panels & waveforms | One UI for web and native; cheapest for Sonnet/Haiku to build |
| Web engine | Web Audio API + AudioWorklet | Only viable low-latency browser path |
| Native shell + engine | **JUCE 8** standalone app, UI in `WebBrowserComponent` (WKWebView on macOS) | JUCE 8 officially supports web UIs; mature VST3/AU hosting (`AudioPluginFormatManager`, `KnownPluginList`); CoreAudio/CoreMIDI |
| Shared DSP | Write DSP once in C++, compile to WASM (Emscripten) for the web worklet **from Phase 4 on**; Phases 1–3 may use TS DSP | Avoid two divergent DSP code bases; the vintage character must sound identical in both builds |
| Data model | Versioned JSON Schema in `shared/schemas`, TS types generated | Same project file opens in both builds |
| Project format | `.psproj` folder bundle: `project.json` + `samples/` (WAV 32-bit float or original) | Human-diffable, robust |
| Tests | Vitest (TS), Playwright (UI e2e), Catch2 (C++), offline-render golden-file audio tests | Audio regressions caught by comparing rendered buffers |
| Licensing | JUCE: personal/indie licence or AGPLv3 — **decide before Phase 6** (see Open Questions) | JUCE licence affects distribution |

### 3.2 `AudioEngine` interface (owned by Opus, task P0-04)
Minimum surface (TS, both implementations must pass the same contract tests):
```ts
interface AudioEngine {
  init(opts: { sampleRate?: number; latencyHint?: 'interactive' | number }): Promise<void>;
  loadSample(id: SampleId, data: ArrayBuffer | FilePath): Promise<SampleInfo>;
  unloadSample(id: SampleId): void;
  setProgram(program: Program): void;            // pad → sample/zone/envelope/filter mapping
  setModelProfile(profile: ModelProfile): void;  // vintage character, swing, limits
  triggerPad(pad: PadRef, velocity: number, pressure?: number): void;
  releasePad(pad: PadRef): void;
  transport: { play(): void; stop(): void; record(arm: RecordArm): void; setBpm(bpm: number): void;
               locate(tick: number): void; onTick(cb: (tick: number) => void): Unsub };
  setSequence(seq: Sequence): void;
  mixer: { setChannel(id: ChannelId, params: ChannelParams): void;
           setInsert(id: ChannelId, slot: number, fx: FxDescriptor | null): void;
           meters(cb: (m: MeterFrame) => void): Unsub };
  recordInput(opts: InputRecordOpts): Promise<RecordingHandle>;
  bounce(opts: BounceOpts): Promise<ArrayBuffer>;   // offline render
  listPlugins?(): Promise<PluginDescriptor[]>;      // native only
}
```
Timing: 960 PPQN internal resolution (MPC60/3000 used 96 PPQN; 2000XL 96; modern 960 —
**[Likely]**). Profiles quantise recorded timing to their native PPQN to reproduce feel.

### 3.3 Repository layout (target)
```
/README.md
/docs/PROJECT_PLAN.md          ← this file
/docs/adr/                     ← architecture decision records (one per decision change)
/docs/model-specs/             ← per-model research notes + references
/shared/schemas/               ← JSON Schemas (project, program, sequence, profile)
/shared/models/*.json          ← model profiles (section 4)
/shared/types/                 ← generated TS types
/ui/                           ← React app
/engine-web/                   ← WebAudioEngine + worklets
/engine-native/                ← JUCE 8 CMake project (+ NativeEngine TS bridge in /engine-native/bridge)
/dsp/                          ← shared C++ DSP (compiled native + WASM)
/tests/golden/                 ← reference renders for audio regression tests
/.github/workflows/            ← CI
```

---

## 4. Model emulation profiles

Profiles are **data** (`shared/models/<id>.json`), not code branches. The engine reads the profile;
the UI picks the skin. Spec numbers below must be re-verified against manuals in task P2-01 —
items marked [Likely]/[Guessing] are not yet confirmed.

| Profile id | Inspired by | Audio character | Pads / banks | Sequencer | Controls to visualise | Screen |
|---|---|---|---|---|---|---|
| `vintage60` | MPC60 (1988) | 12-bit, 40 kHz sampling, 16 voices, characteristic analog output filtering **[Likely]** | 16 pads, 4 banks A–D | 99 sequences, 99 tracks, 96 PPQN, swing 50–75 % | Data wheel, Note Repeat, Full Level, 16 Levels, slider | 2-line-ish LCD (approx. 240×64) |
| `vintage3000` | MPC3000 (1994) | 16-bit, 44.1 kHz, 32 voices, resonant filters **[Likely]** | 16 pads, 4 banks | 99 seq, 99 tracks, 96 PPQN | Data wheel, 2 sliders? **[Guessing]** | Larger LCD |
| `classic2000xl` | MPC2000XL (1999) | 16-bit, 44.1 kHz, 32 voices, filter per pad | 16 pads, 4 banks A–D (64) | 99 seq, 64 tracks, 96 PPQN | Data wheel, Note Variation slider, soft keys F1–F6 | 248×60 LCD **[Likely]** |
| `modernLive` | MPC Live II / One+ | 24-bit/44.1–96 kHz clean engine, built-in FX | 16 pads, 8 banks A–H (128) | Tracks + clips, 960 PPQN, song/arrange | 4 Q-Link knobs, touchscreen, encoder, pad perform modes | 7" touchscreen UI |
| `flagshipX` | MPC X / Key 61 | Same as modernLive | 16 pads, 8 banks (+ 61-key keybed for Key 61 variant) | as modernLive | 16 Q-Links with display strips (X), keybed + pitch/mod (Key 61), virtual CV/Gate outs | 10.1" (X) / 7" (Key 61) |

### 4.1 Profile schema fields
`id, displayName, inspiredBy, skin, pads {count, banks}, voices, sampleRate, bitDepth,
character {resampler, bitCrush, antiAliasFilter, outputFilter, saturation, noiseFloor},
sequencer {maxSequences, maxTracks, ppqn, swingRange, swingAlgorithm, quantizeValues},
controls[] (type, label, binding), screen {width, height, style, pages[]}, features[] (flags)`.

### 4.2 Character emulation approach (Opus designs, Sonnet implements)
- **Rate**: resample to the profile rate with deliberately limited anti-aliasing (aliasing is part of
  the sound), then back to device rate.
- **Bit depth**: quantise with optional dither off; 12-bit for `vintage60`.
- **Filters**: model the pre/post reconstruction filters as fixed low-pass stages (cutoff per profile).
- **Swing**: delay every second 16th (or 8th) by `(swing − 50)/50 × step/…` — implement the MPC-style
  formula and verify by ear + tick-diff test against documented percentages.
- **Pitching**: vintage profiles pitch by sample-rate change (varispeed, couples pitch and time);
  modern uses high-quality resampling + optional time-stretch.
- Character can be applied per-pad on load ("sample through vintage60") or as a bus "Vintage" FX.

---

## 5. Feature map

Legend for build availability: **W** web test build, **N** native macOS build.

### 5.1 Pads & programs (W, N)
- 16 velocity-sensitive pads × profile banks; mouse, keyboard (QWERTY map), MIDI, touch.
- Per pad: sample/zone layers (up to 4), tune, level, pan, ADSR / AD envelope, filter (LP/HP/BP,
  cutoff, resonance, env amount), velocity → level/filter/start, mute groups, choke, poly/mono, one-shot/note-on.
- Performance: Note Repeat (rates 1/4…1/32T, latch), Full Level, 16 Levels (velocity/tune/filter/decay),
  pad mute/solo, pad colours.
- Polyphonic aftertouch where the controller supports it.

### 5.2 Sample import / browser (W, N)
| Format | Web | Native |
|---|---|---|
| WAV (8/16/24/32-bit int & float, mono/stereo) | `decodeAudioData` + own WAV parser for metadata (cue/smpl chunks) | JUCE `WavAudioFormat` |
| AIFF / AIF | decodeAudioData (Safari/Chrome ok, Firefox partial **[Likely]**) → fall back to own parser | JUCE `AiffAudioFormat` |
| FLAC | decodeAudioData | JUCE `FlacAudioFormat` |
| MP3 | decodeAudioData | JUCE/CoreAudio |
| OGG Vorbis | decodeAudioData (not Safari **[Likely]**) | JUCE `OggVorbisAudioFormat` |
| M4A / AAC / ALAC | decodeAudioData (browser-dependent) | `CoreAudioFormat` (macOS) |
- Drag-and-drop to pad, file browser with preview/audition, tags, recent folders.
- Respect `smpl` chunk loop points and root note; `acid`/`bext` BPM metadata if present.
- Editing: trim, normalize, reverse, fade, gain, crossfade loop, time-stretch, pitch-shift.
- Chop: by transient (threshold), by grid (beats/bars at BPM), manual markers; "chop → new program".
- BPM/onset detection is **our own** analysis (not Spotify).

### 5.3 Sequencer (W, N)
- Real-time recording with quantise + swing, overdub, count-in, metronome, undo per pass.
- Step / grid editor (piano-roll + drum grid), event list (vintage-style), Step Edit screen per profile.
- Sequences → Song mode (vintage/2000XL) and Tracks/Clips/Arrange (modern profiles).
- Time signatures, tempo changes, sequence length 1–999 bars.
- Erase while playing (hold ERASE + pad), Timing Correct (MPC naming → "Quantize" in our UI).
- MIDI file import/export (SMF type 0/1).

### 5.4 Recording (W, N)
- Audio input → new sample (threshold-triggered or manual), input monitoring.
- Resample: record master or any bus into a new sample.
- Audio tracks (modern profiles) recorded in sync with the sequencer.
- MIDI recording from external controllers (Web MIDI in W; CoreMIDI in N).
- Bounce: offline render of song/sequence/stems to WAV (16/24/32f, 44.1–96 kHz); MP3 export optional.

### 5.5 Mixer (W, N)
- Channel strip per pad / track / program / return / master: gain, pan, mute, solo, 4 insert slots,
  4 sends, peak + RMS meters.
- Groups/sub-mixes, return buses for reverb/delay.
- Automation of mixer + FX params (record from Q-Links/knobs).

### 5.6 Built-in plug-ins (W, N — same DSP via WASM from Phase 4)
Mixing: parametric EQ (4-band + HP/LP), compressor (VCA-style, sidechain), gate, transient shaper,
saturation/tape, **Vintage Sampler** (bit/rate/filter per profile), filter (LP/HP/BP + LFO),
chorus, flanger/phaser, delay (tempo-synced, ping-pong), reverb (algorithmic), stereo width.
Mastering: linear-phase or minimum-phase EQ, multiband compressor (3-band), glue compressor,
true-peak limiter (oversampled), **LUFS meter** (ITU-R BS.1770-4: momentary/short-term/integrated, LRA,
true peak), mastering presets (Streaming −14 LUFS, Club −9, Lo-Fi vinyl).

### 5.7 Native-only (N)
- VST3 and AU hosting: scan (out-of-process, with blacklist on crash), instrument + effect plug-ins on
  any insert slot, plug-in editor windows, state saved in project, PDC (plug-in delay compensation).
- CoreAudio device selection, buffer size 32–1024, CoreMIDI device mapping, MIDI learn.
- Controller support: generic MIDI pad controllers; real MPC hardware in its MIDI/controller mode
  where it exposes standard MIDI **[Guessing — verify per device]**.
- Optional: build the app also as an AU/VST3 plug-in so it runs inside a DAW.
- Distribution: signed with Developer ID, hardened runtime, notarized, `.dmg`. Universal binary
  (arm64 + x86_64) or arm64-only — decide in P6-01.

### 5.8 Spotify panel (W, N)
- Login (PKCE; redirect `http://127.0.0.1:<port>/callback` for native, origin URL for web — Spotify
  no longer accepts `localhost` redirect URIs, use loopback IP **[Likely]**).
- Search, library, playlists, play/pause/seek a reference track (Premium), now-playing display.
- "Reference crate" saved in project (track URIs + notes + manually entered BPM/key).
- Clear UI copy: "Spotify tracks are for reference listening and can't be loaded onto pads."
- Native: Web Playback SDK may not run inside WKWebView due to EME/Widevine **[Likely]** → fall back to
  Spotify Connect remote control (control the Spotify desktop app via Web API).

### 5.9 Visualisation of the hardware
- Each skin: SVG front panel, interactive pads/knobs/buttons, animated LEDs, screen rendered on canvas
  at the profile's resolution and style (monochrome LCD vs touchscreen).
- Panel ↔ "studio view" toggle: hardware view for feel, expanded editor view for productivity.
- Responsive: panel scales to window; keyboard shortcuts mirror hardware buttons.

---

## 6. Phases & milestones

| Phase | Goal | Exit criteria |
|---|---|---|
| P0 Foundations | Repo, tooling, schemas, engine contract | CI green; `AudioEngine` interface + contract test suite merged; ADR-001 |
| P1 Web MVP | Load samples onto pads, play, sequence 1 track | Drag a WAV onto a pad, play it from mouse/keyboard/MIDI, record a 4-bar loop with swing, < 20 ms perceived latency on Chrome/Mac |
| P2 Model profiles | 4 skins + character DSP + workflows | Switch profile live; golden renders differ per profile as specified; swing tick test passes |
| P3 Recording & editing | Audio in, resample, chop, editor | Record mic → chop by transient → new program → sequence it |
| P4 Mixer & FX | Full mixer + built-in FX (WASM DSP) | Mastering chain hits −14 LUFS ±0.5 on test material; meters verified vs reference |
| P5 Spotify | Reference integration | Login, search, play reference, crate saved/reloaded |
| P6 Native macOS | JUCE app, same UI, VST3/AU | Signed/notarized `.dmg` installs on the user's MacBook; hosts a 3rd-party AU; same project opens in web and native |
| P7 Hardening | Performance, stability, docs | 64 voices at 128-sample buffer without dropouts on M-series; crash-free 2-hour session; user guide |

---

## 7. Model assignment policy (AI agents)

Planning, architecture and review: **Opus 5.5**. Execution goes to the cheapest model that can do the
task reliably.

| Tier | Use for | Do NOT use for |
|---|---|---|
| **Opus 5.5** (`claude-opus-5-5`) | Architecture/ADRs, interfaces & schemas, real-time-safety design, DSP algorithm design (limiter, LUFS, resampler, character), threading/bridge design, VST3/AU host design, review gates on audio-thread code, perf debugging | Boilerplate, CSS, simple components, docs |
| **Sonnet 5.5** (`claude-sonnet-5-5`) | Most feature implementation: engine voices, sequencer, file I/O, chopping, editor, mixer logic, DSP implementation from Opus spec, Spotify OAuth, JUCE port, packaging | Initial architecture decisions |
| **Haiku 5.5** (`claude-haiku-5-5`) | Scaffolding, CI config, lint, UI panels/skins from a precise spec, unit tests from a written spec, schema-to-type generation, fixtures, docs, changelogs | Anything on the audio thread, concurrency, DSP math |

Rules:
1. Every task has an owner tier (section 9). Start there.
2. **Escalate one tier after 2 failed attempts** at meeting acceptance criteria; record it in the task row.
3. Any change touching `process()`/`processBlock()`, lock-free queues, or the WebView bridge requires an
   **Opus review** before merge (marked `R:Opus`).
4. Haiku tasks must be given a spec precise enough to need no design decisions (file paths, props,
   expected output). If the spec is vague, the task is not Haiku's yet — Opus/Sonnet writes the spec first.
5. One task = one branch = one PR; PR description links the task ID.

---

## 8. Definition of done (every task)
- Acceptance criteria in the task row met and demonstrated (test or screenshot/recording for UI).
- Unit tests added; audio tasks add/refresh a golden render.
- No new lint/type errors; CI green.
- This plan updated if scope or a decision changed (add an ADR in `docs/adr/`).

---

## 9. Task backlog

Columns: **ID · Task · Owner · Deps · Acceptance criteria** (`R:Opus` = Opus review required).

### P0 — Foundations
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P0-01 | Choose product name, licence (MIT/GPL vs JUCE terms), write ADR-001 (architecture in §3) | Opus | — | ADR-001 merged; name used in README |
| P0-02 | Monorepo scaffold: pnpm workspaces, Vite React TS app in `ui/`, `engine-web/`, `shared/`, ESLint, Prettier, Vitest, Playwright | Haiku | P0-01 | `pnpm i && pnpm test && pnpm dev` work |
| P0-03 | GitHub Actions CI: lint, typecheck, unit, Playwright (Chromium) | Haiku | P0-02 | CI runs on PR, green |
| P0-04 | JSON Schemas (project, program, pad, sequence, event, model profile, mixer) + `AudioEngine` TS interface | Opus | P0-02 | Schemas validate fixtures; types generated |
| P0-05 | Type generation script (schemas → `shared/types`) + fixtures | Haiku | P0-04 | `pnpm gen:types` reproducible, checked in CI |
| P0-06 | `AudioEngine` contract test suite (runs against any implementation, uses offline rendering) | Sonnet | P0-04 | Suite runs against a stub engine |

### P1 — Web MVP
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P1-01 | Pad grid component (16 pads, banks, velocity from click position/keyboard), QWERTY mapping | Haiku | P0-02 | Storybook/Playwright shows pads firing events |
| P1-02 | Sample decode + WAV parser (smpl/cue/acid chunks) + format detection; fallback messages | Sonnet | P0-04 | All formats in §5.2 load in Chrome; WAV metadata tests pass |
| P1-03 | AudioWorklet voice engine: preallocated voice pool, one-shot/note-on, ADSR, choke/mute groups | Sonnet | P0-04, P0-06 | Contract tests pass; no GC in worklet (profiled) · R:Opus |
| P1-04 | Transport + clock (960 PPQN, lookahead scheduler), metronome | Sonnet | P1-03 | Jitter < 1 ms vs offline render · R:Opus |
| P1-05 | Single-track real-time record + quantise + swing (basic formula) + undo | Sonnet | P1-04 | Record 4-bar loop, quantise test passes |
| P1-06 | Drag-and-drop to pad, simple file browser with audition | Haiku | P1-02 | Drop WAV → pad plays it |
| P1-07 | Web MIDI input (notes, velocity, aftertouch) + MIDI learn for pads | Sonnet | P1-03 | External pad controller triggers pads |
| P1-08 | Project save/load (`.psproj` as zip in web via File System Access API) | Sonnet | P0-04 | Round-trip test identical |
| P1-09 | Latency measurement page + docs of results per browser | Haiku | P1-03 | Numbers recorded in `docs/` |

### P2 — Model profiles & visualisation
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P2-01 | Research & verify specs for 5 profiles (manuals, references), write `docs/model-specs/*.md`, fill profile JSON | Opus | P0-04 | All [Likely]/[Guessing] in §4 resolved or flagged |
| P2-02 | Skin spec: per-model SVG layout spec (controls list, coordinates, states) | Opus | P2-01 | Spec precise enough for Haiku (no open design choices) |
| P2-03 | Skin `vintage60` + `vintage3000` | Haiku | P2-02 | Panel renders, all controls bound |
| P2-04 | Skin `classic2000xl` | Haiku | P2-02 | as above |
| P2-05 | Skin `modernLive` (touchscreen-style UI) | Haiku | P2-02 | as above |
| P2-06 | Skin `flagshipX` + Key 61 keybed variant | Haiku | P2-02 | as above |
| P2-07 | LCD/screen renderer (canvas, per-profile pages: Main, Program, Step Edit, Sample Edit) | Sonnet | P2-02 | Screens update from store state |
| P2-08 | Character DSP design: resampler, bit-crush, filters, varispeed pitching, swing formula | Opus | P2-01 | Design doc + reference renders |
| P2-09 | Character DSP implementation (C++ in `/dsp` + WASM build; TS fallback allowed until P4) | Sonnet | P2-08 | Golden renders match within tolerance · R:Opus |
| P2-10 | Profile switching: limits (seq/track counts, banks), Note Repeat, Full Level, 16 Levels | Sonnet | P2-07 | Contract + UI tests per profile |

### P3 — Recording & editing
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P3-01 | Audio input recording (getUserMedia, threshold trigger, monitoring) | Sonnet | P1-03 | Recorded sample appears in pool |
| P3-02 | Resample bus (master/bus → sample) | Sonnet | P3-01 | Resampled audio matches bus render |
| P3-03 | Waveform editor: zoom, markers, trim, normalize, reverse, fade, loop | Sonnet | P1-02 | Edits non-destructive + undoable |
| P3-04 | Chop: transient detection, grid chop, manual; "chop → program" | Sonnet | P3-03 | Drum loop → 16 pads in one action |
| P3-05 | Time-stretch & pitch-shift (library choice: Rubber Band (GPL/commercial) vs SoundTouch (LGPL) — Opus decides) | Opus→Sonnet | P3-03 | Quality listening test + licence ADR |
| P3-06 | Multi-track sequencer, grid editor, event list, song mode, tracks/clips for modern profiles | Sonnet | P1-05 | Feature parity with §5.3 |
| P3-07 | MIDI file import/export | Haiku | P3-06 | Round-trip test with reference SMFs |
| P3-08 | Tests for P3 features from specs | Haiku | P3-01..07 | Coverage ≥ 80 % on new modules |

### P4 — Mixer & built-in FX
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P4-01 | Mixer graph design (buses, sends, inserts, PDC model) | Opus | P1-03 | ADR + graph contract |
| P4-02 | Mixer UI (strips, meters, sends, insert slots) | Haiku | P4-01 | Renders from store, all controls bound |
| P4-03 | Mixer engine (web) | Sonnet | P4-01 | Contract tests pass · R:Opus |
| P4-04 | WASM DSP pipeline (Emscripten build of `/dsp`, worklet loader) | Sonnet | P2-09 | Same golden renders in C++ native test and WASM |
| P4-05 | Mixing FX: EQ, comp, gate, transient, saturation, filter, chorus, phaser, delay, reverb, width | Sonnet | P4-04 | Each FX has unit + golden tests |
| P4-06 | Mastering: multiband comp, glue comp, oversampled true-peak limiter, LUFS meter (BS.1770-4) | Opus (design) → Sonnet | P4-04 | LUFS within ±0.1 of EBU test vectors; limiter no overs at −1 dBTP |
| P4-07 | FX preset system + mastering presets | Haiku | P4-05 | Presets load/save |
| P4-08 | Automation recording/playback | Sonnet | P4-03 | Knob moves recorded and replayed |
| P4-09 | Bounce/export WAV + stems (offline render) | Sonnet | P4-03 | Bit-identical across two renders |

### P5 — Spotify
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P5-01 | Verify current Spotify API/ToS (endpoints, redirect rules, quotas, SDK in WKWebView) → ADR | Opus | — | ADR-Spotify; §5.8 updated |
| P5-02 | PKCE auth, token refresh, secure storage (web: memory/sessionStorage; native: macOS Keychain) | Sonnet | P5-01 | Login/logout works; no client secret shipped |
| P5-03 | Search, library, playlists, player (SDK on web; Connect fallback on native) | Sonnet | P5-02 | Play a reference track |
| P5-04 | Spotify panel UI + reference crate in project | Haiku | P5-03 | Crate persists in project file |

### P6 — Native macOS
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P6-01 | Native architecture: JUCE 8 app, WebView bridge protocol, threading, universal vs arm64 | Opus | P0-04 | ADR + bridge spec |
| P6-02 | JUCE CMake project, WebBrowserComponent hosting built `ui/`, dev mode loading Vite server | Sonnet | P6-01 | App opens UI on macOS |
| P6-03 | `NativeEngine` TS bridge implementing `AudioEngine` | Sonnet | P6-02 | Contract tests pass against native · R:Opus |
| P6-04 | C++ voice engine, sequencer clock, mixer (reusing `/dsp`) | Sonnet | P6-03, P4-04 | Golden renders match web · R:Opus |
| P6-05 | VST3/AU hosting: out-of-process scan, insert slots, editor windows, state, PDC | Opus | P6-04 | Loads a free 3rd-party AU + VST3, state restored |
| P6-06 | CoreAudio/CoreMIDI device settings UI, MIDI learn | Sonnet | P6-04 | Device switching without restart |
| P6-07 | Native file I/O (all formats via JUCE/CoreAudio), project bundles on disk | Sonnet | P6-04 | Same project opens in web and native |
| P6-08 | Codesign, hardened runtime, entitlements (mic, network), notarization, `.dmg` in CI | Sonnet | P6-02 | Notarized dmg installs on a clean Mac |
| P6-09 | Optional: AU/VST3 build of the app itself | Sonnet | P6-04 | Loads in a DAW |

### P7 — Hardening
| ID | Task | Owner | Deps | Acceptance |
|---|---|---|---|---|
| P7-01 | Performance profiling (voices, DSP, UI render) and fixes | Opus | P6 | Targets in §6 met |
| P7-02 | Regression test expansion, fuzzing file parsers | Haiku | P6 | Parsers survive fuzz corpus |
| P7-03 | Crash reporting (opt-in), logging outside audio thread | Sonnet | P6 | Crash reports captured locally |
| P7-04 | User guide + per-profile quick-start | Haiku | P6 | `docs/user-guide/` complete |

Rough split: ~10 Opus tasks (design/review), ~30 Sonnet, ~17 Haiku. Opus review gates add
roughly one review per audio-critical PR.

---

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Users expect Spotify-to-pad sampling | Expectation mismatch | Stated in §1.1 and in UI copy |
| Trademark/trade-dress complaint | Takedown | §1.2 rules; own name/skins; disclaimer |
| Web Audio latency too high for finger drumming | Test build feels bad | Acceptable for testing; native is the real target; measure in P1-09 |
| Two DSP code bases diverge | Web vs native sound different | Single C++ `/dsp` compiled to WASM from P4 |
| Haiku produces subtly wrong DSP/RT code | Glitches, dropouts | Haiku never owns audio-thread work; `R:Opus` gates |
| JUCE / time-stretch library licences | Can't distribute as planned | Decide in P0-01 / P3-05 before code depends on them |
| Spotify SDK won't run in WKWebView | No reference playback in native | Connect remote-control fallback (P5-03) |
| VST/AU plug-ins crash host | Data loss | Out-of-process scan, autosave, blacklist |
| Spec inaccuracy of vintage models | "Doesn't sound/feel right" | P2-01 research with manuals; A/B against reference recordings |

## 11. Open questions (owner: user unless stated)
1. Product name? (P0-01)
2. Personal use only, or distributed/sold? Drives JUCE licence (AGPLv3 vs paid) and Rubber Band licence.
3. Spotify Premium available? (needed for reference playback)
4. MacBook chip (Apple Silicon vs Intel) and macOS version → universal binary or arm64-only.
5. Apple Developer account ($99/yr) for notarization? Without it the app runs only via right-click → Open / Gatekeeper override.
6. Import of legacy MPC files (`.pgm`, `.snd`, `.seq` from 2000XL) wanted? Stretch goal; formats are partly reverse-engineered **[Guessing]**.
7. Which hardware controllers will be used for testing?

## 12. Glossary (MPC terms → our terms)
| MPC term | Meaning | Our UI term |
|---|---|---|
| Program | Mapping of pads to samples + params | Program / Kit |
| Pad bank | Group of 16 pads (A–D vintage, A–H modern) | Bank |
| Note Repeat | Retrigger held pad at a rhythmic rate | Note Repeat |
| Full Level | Ignore velocity, max level | Full Level |
| 16 Levels | Spread one sound over 16 pads by velocity/tune/filter/decay | 16 Levels |
| Timing Correct | Quantise | Quantize |
| Swing | Delay of off-beat steps, 50–75 % | Swing |
| Q-Link | Assignable knob/slider | Q-Link / Macro |
| Chop | Slice a sample into regions | Chop |
| Resample | Record internal output into a new sample | Resample |
| Song mode | Chain sequences | Song |
| Note Variation slider (2000XL) | Real-time tune/decay/attack/filter variation | Variation slider |

## 13. References (verify links at use)
- JUCE 8 WebView UI: JUCE docs `WebBrowserComponent`, `juce::WebBrowserComponent::Options::withNativeFunction`
- JUCE plug-in hosting: `AudioPluginFormatManager`, `KnownPluginList`, `PluginListComponent`, AudioPluginHost example
- Web Audio API & AudioWorklet: MDN; W3C Web Audio spec
- Spotify Web API, Web Playback SDK, Developer Terms (developer.spotify.com)
- ITU-R BS.1770-4 & EBU R128 / Tech 3341 test signals (LUFS)
- Apple: notarizing macOS software, hardened runtime entitlements
- MPC60 / MPC3000 / MPC2000XL / MPC Live II / One+ / MPC X / Key 61 owner's manuals (Akai Pro support site)

## 14. Change log
- 2026-11-20 — Initial plan (Opus 5.5). Web for test phases, native macOS (JUCE 8) as release target; all
  four model families in scope.
