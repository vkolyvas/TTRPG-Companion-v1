# TTRPG Companion — CLAUDE.md v2
# Build Once. Last Forever.

> **Read this entire file before writing any code.**
> This is the authoritative source for every architectural decision, constraint,
> schema, API contract, and convention in this codebase. Decisions marked
> **[LOCKED]** are not open for reconsideration without a documented blocking
> technical reason. Decisions marked **[OPEN]** are tracked but unresolved —
> raise an issue before acting on them.
>
> When product decisions change: update this file first.
> When implementation diverges from this file: update this file or revert the
> implementation. Never silently let them drift apart.

---

## Table of Contents

1. [Project Context & Design Philosophy](#1-project-context--design-philosophy)
2. [Repository Structure](#2-repository-structure)
3. [Tech Stack — Locked Decisions](#3-tech-stack--locked-decisions)
4. [Architecture Overview](#4-architecture-overview)
5. [Audio Processing Pipeline — Detailed](#5-audio-processing-pipeline--detailed)
6. [ML Model Strategy — ONNX-Native](#6-ml-model-strategy--onnx-native)
7. [Non-Negotiable Constraints](#7-non-negotiable-constraints)
8. [Platform-Specific Requirements](#8-platform-specific-requirements)
9. [Startup Sequence & Phase Model](#9-startup-sequence--phase-model)
10. [Build, Run & Test Commands](#10-build-run--test-commands)
11. [CI/CD Pipeline](#11-cicd-pipeline)
12. [Module Breakdown & Acceptance Criteria](#12-module-breakdown--acceptance-criteria)
13. [Data Schemas — Canonical Definitions](#13-data-schemas--canonical-definitions)
14. [Detection FSM — Complete Specification](#14-detection-fsm--complete-specification)
15. [IPC & Event Contracts](#15-ipc--event-contracts)
16. [Coding Conventions](#16-coding-conventions)
17. [Testing Strategy](#17-testing-strategy)
18. [Performance Budgets](#18-performance-budgets)
19. [Phase Gates](#19-phase-gates)
20. [Open Questions Register](#20-open-questions-register)

---

## 1. Project Context & Design Philosophy

### What This Is

A desktop background application (Windows + macOS) for tabletop RPG Game Masters.
It listens to the GM narrating scenes via the laptop microphone, detects scene
context through keyword matching and vocal analysis, and automatically crossfades
ambient music and triggers sound effects in real time.

**The product is an audio intelligence layer.** Not a VTT. Not a prep tool.
Not a music library. It takes what is happening at the table and makes it sound
right, in real time, without requiring the GM to operate software.

### Design Mantra — Applies to Every Decision

> **"The GM is narrating, not operating software."**

If a feature requires the GM to stop talking to their players and think about the
app, it is wrong. If it can be done with a keystroke while mid-sentence without
switching windows, it may be right. Evaluate everything here.

### MVP Success Criterion

Does this produce the moment where a GM tells another GM
*"you need to try this thing"*?

That moment is: the music shifts on its own and it feels right.
Not most of the time. At least once in the first session, unmistakably.
Everything else is enhancement around that moment.

### Target Hardware Baseline

Any laptop from approximately 2020 onward. 8 GB RAM minimum. No dedicated GPU
required. Mediocre built-in microphone. Noisy room with 3–5 people.
If it doesn't work on this hardware, it doesn't work.

---

## 2. Repository Structure

```
ttrpg-companion/
├── CLAUDE.md                          ← This file. Single source of truth.
├── Cargo.toml                         ← Workspace root
├── Cargo.lock
├── .github/
│   └── workflows/
│       ├── ci.yml                     ← Defined in §11
│       └── release.yml                ← Tauri build + itch.io upload
│
├── src-tauri/                         ← Rust backend (Tauri application)
│   ├── Cargo.toml
│   ├── tauri.conf.json                ← Platform capabilities, bundle config
│   ├── build.rs                       ← ONNX model path embedding
│   ├── capabilities/                  ← Tauri 2.x capability files
│   │   ├── default.json               ← Base capabilities
│   │   └── microphone.json            ← Mic permission capability
│   └── src/
│       ├── main.rs                    ← Entry point, Tauri builder, plugin registration
│       ├── lib.rs                     ← pub use re-exports for integration tests
│       ├── error.rs                   ← AppError type (thiserror), all error variants
│       ├── state.rs                   ← AppState, SessionState, all shared types
│       ├── startup.rs                 ← Two-phase startup orchestration
│       │
│       ├── commands/                  ← Tauri #[tauri::command] handlers
│       │   ├── mod.rs                 ← register_all_commands()
│       │   ├── session.rs             ← start_session, end_session, set_mood, next_track, toggle_lock
│       │   ├── library.rs             ← import_tracks, get_tracks, update_mood_tag, remove_byom_track
│       │   ├── profile.rs             ← start_enrollment, record_passage, complete_enrollment, delete_profile
│       │   └── config.rs              ← get_config, update_config, get_audio_devices, reset_config
│       │
│       ├── audio/
│       │   ├── mod.rs
│       │   ├── engine.rs              ← AudioEngine trait + FmodEngine impl
│       │   ├── silent_engine.rs       ← SilentEngine (test double)
│       │   ├── crossfade.rs           ← CrossfadeType, transition timing constants
│       │   ├── capture.rs             ← CPAL stream management
│       │   ├── resampler.rs           ← Native-rate → 16kHz conversion (rubato)
│       │   └── normalizer.rs          ← RMS normalization to -16 dBFS target
│       │
│       ├── detection/
│       │   ├── mod.rs
│       │   ├── pipeline.rs            ← Orchestrates all detection components
│       │   ├── vad.rs                 ← Silero VAD via ort, 30ms frames
│       │   ├── speaker.rs             ← Resemblyzer ONNX via ort
│       │   ├── whisper.rs             ← whisper-rs wrapper, segment buffering
│       │   ├── keyword.rs             ← Fuzzy matching engine, vocabulary hot-reload
│       │   ├── fsm.rs                 ← Detection state machine (see §14)
│       │   └── session_logger.rs      ← DetectionEvent logging (both auto + corrections)
│       │
│       ├── ml/
│       │   ├── mod.rs
│       │   ├── ort_env.rs             ← Shared OrtEnvironment singleton
│       │   ├── vad_model.rs           ← Silero VAD ONNX session
│       │   ├── speaker_model.rs       ← Resemblyzer ONNX session
│       │   └── emotion_model.rs       ← emotion2vec ONNX session (V1.x, scaffolded)
│       │
│       ├── db/
│       │   ├── mod.rs                 ← Connection pool (r2d2 + rusqlite), migration runner
│       │   ├── migrations/
│       │   │   ├── 001_initial.sql
│       │   │   ├── 002_corrections.sql
│       │   │   └── 003_detection_events.sql
│       │   ├── tracks.rs
│       │   ├── genres.rs              ← track_genres junction table operations
│       │   ├── sfx.rs
│       │   ├── keywords.rs
│       │   └── sessions.rs
│       │
│       ├── profile/
│       │   ├── mod.rs
│       │   ├── storage.rs             ← Encrypted profile DB (AES-256-GCM + OS keychain)
│       │   └── consent.rs             ← Consent flag management, enforcement
│       │
│       ├── tray.rs                    ← System tray / menu bar (platform-adaptive)
│       ├── hotkeys.rs                 ← Global hotkey registration + conflict detection
│       └── permissions.rs             ← Mic permission check + request (Mac + Windows)
│
├── src/                               ← Svelte + Vite frontend (NOT SvelteKit)
│   ├── index.html                     ← Vite entry point
│   ├── main.ts                        ← Mount App.svelte
│   ├── App.svelte                     ← Root: Setup window OR Session popover
│   ├── vite.config.ts
│   ├── tsconfig.json                  ← strict: true, no implicit any
│   ├── tailwind.config.js
│   └── lib/
│       ├── invoke.ts                  ← ALL Tauri command calls (typed wrappers only)
│       ├── events.ts                  ← ALL Tauri event listeners (typed wrappers only)
│       ├── types.ts                   ← Shared TypeScript types (mirrors Rust types)
│       ├── stores/
│       │   ├── session.ts             ← sessionStore: SessionState
│       │   ├── library.ts             ← libraryStore: Track[]
│       │   ├── profile.ts             ← profileStore: GmProfile | null
│       │   └── startup.ts             ← startupStore: StartupPhase
│       └── components/
│           ├── setup/                 ← Pre-session UI
│           │   ├── Navigation.svelte
│           │   ├── LibraryScreen.svelte
│           │   ├── VoiceScreen.svelte
│           │   ├── SettingsScreen.svelte
│           │   └── AboutScreen.svelte
│           └── session/               ← Session-active UI
│               ├── MoodPopover.svelte
│               └── StartupOverlay.svelte
│
├── models/                            ← ONNX model files (gitignored — downloaded by setup script)
│   ├── .gitkeep
│   ├── silero_vad.onnx                ← ~2 MB
│   ├── resemblyzer.onnx               ← ~7 MB
│   └── emotion2vec_base.onnx          ← ~350 MB (V1.x, downloaded on demand)
│
├── tools/
│   └── audio_tagger/                  ← Standalone Python CLI (not shipped with app)
│       ├── main.py
│       ├── classifier.py
│       ├── requirements.txt           ← librosa, soundfile, mutagen, numpy
│       └── README.md
│
├── scripts/
│   ├── download_models.sh             ← Downloads + validates ONNX models to models/
│   ├── export_resemblyzer.py          ← Exports Resemblyzer weights to ONNX
│   └── export_emotion2vec.py          ← Exports emotion2vec base to ONNX
│
├── content/
│   ├── music/
│   │   ├── manifest.json              ← Track metadata (committed, no binary audio)
│   │   └── .gitignore                 ← Ignores *.mp3 *.flac *.wav *.ogg *.m4a
│   ├── sfx/
│   │   ├── manifest.json
│   │   └── .gitignore
│   ├── keywords/
│   │   ├── en_fantasy_v1.json
│   │   └── el_fantasy_v1.json
│   └── passages/
│       ├── en/
│       │   ├── calm.md
│       │   ├── tension.md
│       │   ├── combat.md
│       │   ├── mystery.md
│       │   ├── travel.md
│       │   ├── celebration.md
│       │   └── sorrow.md
│       └── el/                        ← Greek passages — native, not translated
│           └── (same structure)
│
├── profiles/                          ← GM profiles (gitignored entirely)
│   └── .gitignore
│
└── tests/
    ├── integration/
    │   ├── pipeline_test.rs
    │   ├── audio_engine_test.rs
    │   ├── keyword_engine_test.rs
    │   └── fsm_test.rs
    └── fixtures/
        ├── audio/                     ← Test WAV samples (gitignored for size)
        │   └── .gitkeep
        ├── audio_manifest.json        ← Metadata for test fixtures
        ├── keyword_test_cases.json    ← 50 positive + 50 negative keyword cases
        └── real_world_log.md          ← Manual real-session test results (committed)
```

---

## 3. Tech Stack — Locked Decisions

**Do not propose alternatives without a blocking technical reason.**
All decisions below were reached after spike evaluation and are treated as
constraints, not options.

### Application Framework

| Component | Choice | Status | Rationale |
|-----------|--------|--------|-----------|
| Desktop framework | **Tauri 2.x** | LOCKED | Rust backend + web frontend. Native binaries ~6 MB. System tray, global hotkeys, IPC, capability-based permissions all built-in. Actively maintained. |
| Frontend | **Vite + Svelte 5** | LOCKED | **Not SvelteKit.** This is a single-window desktop app — SSR, routing, load functions are irrelevant and add noise. Vite + Svelte compiles to vanilla JS with no runtime. Fastest load for a modest UI surface. |
| Frontend styling | **Tailwind CSS 3** | LOCKED | Utility-first, consistent with static asset bundling. |
| Rust edition | **2021** | LOCKED | |
| Async runtime | **tokio** | LOCKED | Multi-threaded runtime. All I/O async. CPU-bound work via `spawn_blocking`. |

### ML Inference — ONNX-Native, No Python Runtime

**[LOCKED]** All ML inference runs natively in Rust via the `ort` crate (ONNX Runtime
bindings). There are no Python sidecars, no Python runtime dependency, no user-facing
installation step for ML dependencies. Models are ONNX files shipped with or
downloaded by the app.

**Why this matters:** A Python sidecar requires either bundling a full Python
runtime (adds ~50–200 MB, introduces version conflicts) or requiring the user to
have Python installed (breaks the low-friction install story on Windows for most
users). ONNX Runtime eliminates both problems. The export scripts in `scripts/`
convert source model weights to ONNX format once, at release time.

| ML Component | Source Model | ONNX File | Crate | Status |
|---|---|---|---|---|
| Voice Activity Detection | Silero VAD v4 | `silero_vad.onnx` (~2 MB) | `ort` | LOCKED, V1 |
| Speaker Verification | Resemblyzer | `resemblyzer.onnx` (~7 MB) | `ort` | LOCKED, V1 |
| Vocal Mood Classifier | emotion2vec+ base | `emotion2vec_base.onnx` (~350 MB) | `ort` | LOCKED, V1.x — downloaded on demand |

All three models share a single `OrtEnvironment` singleton (one per process).
See §6 for ONNX model architecture details.

### Audio Stack

| Component | Choice | Status | Rationale |
|-----------|--------|--------|-----------|
| Audio engine | **FMOD Studio API** | LOCKED (pending S3) | Industry standard for dynamic game audio. Free under $200K ARR. Gapless loops, crossfades, SFX layering, ducking — all native. Fallback: platform-native if S3 fails. |
| Audio capture | **CPAL** | LOCKED | Cross-platform Rust audio I/O. |
| Sample rate conversion | **rubato** | LOCKED | High-quality resampling from device native rate to 16kHz. Mandatory pre-processing step — never assume the device opens at 16kHz. |
| Audio normalization | **In-house** | LOCKED | RMS normalization to -16 dBFS per speech segment before Whisper. See §5. |

### Speech-to-Text

| Component | Choice | Status | Rationale |
|-----------|--------|--------|-----------|
| STT engine | **whisper-rs** (whisper.cpp Rust FFI) | LOCKED | Runs natively, no network call. Medium model selected for Greek accuracy — smaller models degrade measurably for Greek. |
| Model | **medium** (769M params, ~2.6 GB RAM) | LOCKED | Configurable to `small` on constrained hardware, `large-v3` on capable hardware. Medium is the default. |
| Language mode | **"auto"** | LOCKED | Handles Greek/English code-switching within a session. |
| Segment buffering | **8–12 second windows** | LOCKED | Whisper accuracy degrades on short clips. Buffer speech to minimum 8 seconds before firing inference. See §5. |

### Data

| Component | Choice | Status |
|-----------|--------|--------|
| Database | **SQLite via rusqlite** | LOCKED |
| Connection pool | **r2d2 + r2d2_sqlite** | LOCKED |
| Migrations | **Manual SQL files + schema_migrations table** | LOCKED |
| Profile encryption | **AES-256-GCM, key from OS keychain** | LOCKED |
| OS keychain | **keyring crate** (macOS Keychain / Windows DPAPI) | LOCKED |
| Vocabulary hot-reload | **notify crate** (fs watcher) | LOCKED |

### Distribution

| Component | Choice |
|-----------|--------|
| Primary | **itch.io** (TTRPG community, no review, creator-friendly rev split) |
| Package (Windows) | `.msi` via Tauri bundler, code-signed (EV cert target) |
| Package (Mac) | `.dmg` + notarized `.app`, universal binary (Intel + Apple Silicon) |
| Updates | **Tauri updater** → GitHub Releases manifest |
| Model delivery | ONNX models bundled in installer for V1 models; V1.x model downloaded post-install |

---

## 4. Architecture Overview

### Thread Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Main Thread — Tauri event loop, IPC, tray, hotkeys             │
└───────────────────────┬─────────────────────────────────────────┘
                        │ tokio channels (bounded)
        ┌───────────────┼───────────────────┐
        ▼               ▼                   ▼
┌──────────────┐ ┌─────────────┐  ┌────────────────┐
│ Audio Capture│ │  Detection  │  │  FMOD Thread   │
│ Thread (CPAL)│ │  Thread     │  │  (FMOD-managed)│
│              │ │             │  │                │
│ Captures mic │ │ VAD →       │  │ Music playback │
│ Resamples to │ │ Speaker →   │  │ SFX layering   │
│ 16kHz        │ │ Buffer →    │  │ Crossfades     │
│ Normalizes   │ │ Whisper →   │  │                │
│              │ │ Keywords →  │  │                │
│ → audio_tx   │ │ FSM         │  │ ← audio_cmd_rx │
└──────────────┘ └─────────────┘  └────────────────┘
        │               │
        ▼               ▼
 bounded channel  bounded channel
  audio_tx/rx     event_tx/rx → IPC → Svelte frontend
```

### Channel Specifications — All Bounded

**No unbounded channels on any hot path.** VAD produces frames at ~33Hz.
An unbounded channel is a memory leak waiting to happen during a 4-hour session.

```rust
// All channel capacities are named constants in state.rs — never magic numbers

pub const AUDIO_FRAME_CHANNEL_CAPACITY: usize = 100;  // ~3 seconds of VAD frames
pub const SPEECH_SEGMENT_CHANNEL_CAPACITY: usize = 10; // VAD → Whisper buffer
pub const DETECTION_EVENT_CHANNEL_CAPACITY: usize = 50;
pub const AUDIO_CMD_CHANNEL_CAPACITY: usize = 20;
pub const IPC_EVENT_CHANNEL_CAPACITY: usize = 50;
```

**Overflow behavior:** Drop oldest (not newest). The detection pipeline cares about
recent audio, not queued old audio. Implement with a custom `overwrite_send` helper
or a ringbuffer.

### IPC Pattern — Rust ↔ Frontend

- **Rust → Frontend:** `app_handle.emit(event_name, payload)` — **Tauri 2.x API**.
  `emit_all()` is deprecated and will emit compiler warnings. Use `emit()`.
- **Frontend → Rust:** `invoke(command, args)` via typed wrappers in `invoke.ts`.
  Never call `invoke()` directly from a Svelte component.
- **Rule:** Frontend is display-only. All business logic and mutable state live in
  Rust. Frontend renders what Rust tells it to render.

---

## 5. Audio Processing Pipeline — Detailed

Every audio sample passes through this exact pipeline. No shortcuts.
Steps marked **[mandatory]** cannot be omitted under any condition.

```
[Mic Input — CPAL]
       │
       │  Device native sample rate (44100 or 48000 Hz typical)
       │  Device native channel count (1 or 2)
       ▼
[Channel Downmix — mandatory]
       │  Stereo → mono if needed (average channels)
       │  Output: mono stream
       ▼
[Resample to 16kHz — mandatory]
       │  rubato SincFixedIn resampler
       │  Required: Whisper, Silero VAD, Resemblyzer all expect 16kHz
       │  Never assume device opens at 16kHz — it almost never does
       │  Output: 16kHz mono f32 samples
       ▼
[Frame Assembly — 30ms frames]
       │  480 samples per frame at 16kHz
       │  Ring buffer: hold last 10 frames for pre-padding
       ▼
[Voice Activity Detection — Silero VAD via ort]
       │  Input: 480-sample frame
       │  Output: speech probability 0.0–1.0 per frame
       │  Threshold: 0.6 (configurable 0.4–0.8)
       │
       ├── probability < threshold → SILENCE → discard frame
       │
       └── probability ≥ threshold → SPEECH
              │
              │  Assemble speech segment:
              │  - Pre-padding: prepend last 5 frames (150ms) from ring buffer
              │    (captures start of utterance that preceded VAD detection)
              │  - Continue accumulating until VAD drops below threshold
              │  - Post-padding: append 10 frames (300ms) after VAD drops
              │    (captures trailing phonemes)
              │  - Minimum segment length: 0.5 seconds (discard shorter)
              │  - Maximum segment length: 30 seconds (force-flush, feed to Whisper)
              ▼
[Normalization — mandatory]
       │  RMS normalization to -16 dBFS target per segment
       │  Formula: scale = 10^(-16/20) / rms(segment)
       │  Clamp scale: [0.1, 10.0] — prevent extreme amplification of silence
       │  Reason: Whisper accuracy varies significantly with input level.
       │  A quiet narrator in a large room produces low-amplitude audio
       │  that degrades recognition. This makes the pipeline level-independent.
       ▼
[Segment Buffering — mandatory]
       │  Accumulate normalized segments until buffer reaches 8–12 seconds
       │  of speech (not wall-clock time — only voiced frames count)
       │
       │  Flush conditions:
       │  - Buffer reaches 12 seconds → flush immediately
       │  - Inactivity timeout: no new speech for 3 seconds → flush whatever is buffered
       │  - High-energy VAD spike (probability > 0.95) → flush immediately
       │    (likely a keyword-dense moment — prioritize low latency)
       │
       │  Reason: Whisper accuracy degrades significantly on short clips.
       │  A 3-word phrase like "roll for initiative" on 1.5s of audio
       │  produces more errors than the same phrase in a longer segment.
       │  The 8–12s buffer trades ~8s of max latency for materially better accuracy.
       ▼
[Speaker Verification — Resemblyzer ONNX via ort]
       │  Input: buffered speech segment (16kHz mono f32)
       │  Extract d-vector embedding (256-dim)
       │  Cosine similarity vs enrolled GM embedding
       │  Threshold: 0.75 (configurable 0.60–0.90)
       │
       ├── similarity < threshold → REJECT → discard segment, log SpeakerRejected event
       │
       └── similarity ≥ threshold → ACCEPT → forward to Whisper
              ▼
[Whisper STT — whisper-rs]
       │  Model: medium (default), configurable to small/large-v3
       │  Language: "auto" (handles en/el code-switching)
       │  Beam size: 5 (quality/speed balance; reduce to 3 on constrained hardware)
       │  Word-level timestamps: enabled (needed for keyword timing)
       │  Processing: spawn_blocking (CPU-bound, not async-friendly)
       │  Output: Vec<WhisperToken> with timestamps + transcription text
       ▼
[Keyword Engine — Rust]
       │  Load vocabulary from DB for active (language, genre) pair
       │  Fuzzy match: normalized Levenshtein distance ≤ 0.25 on word tokens
       │  Whisper transcription errors are systematic — edit distance handles them
       │  Return: best match + confidence score + trigger type
       │
       ├── no match OR confidence < threshold → log LowConfidence event → IDLE
       │
       └── match found, confidence ≥ threshold
              ▼
[Detection FSM — see §14]
       │
       ├── Cooldown active → log CooldownActive event → IDLE
       ├── FSM LOCKED → log Suppressed event → IDLE
       │
       └── TRIGGER state entered
              │
              ├── trigger_type = MoodShift → FMOD crossfade + log DetectionEvent (auto)
              └── trigger_type = SfxTrigger → FMOD one-shot + log DetectionEvent (auto)
```

### V1.x Addition — Vocal Classifier Path

When emotion2vec is active, this runs in parallel with Whisper on accepted segments:

```
[Accepted speech segment from Speaker Verification]
       │
       ▼
[emotion2vec ONNX — ort]
       │  Frozen backbone: extract 768-dim embedding
       │  Per-GM mapping layer: embedding → 7-class probability distribution
       │  Output: top_mood + probability vector
       ▼
[Dual-Signal Fusion]
       │  Keyword signal: mood + confidence (from Keyword Engine)
       │  Vocal signal: mood + confidence (from emotion2vec mapping)
       │
       │  Fusion rules:
       │  - Keyword confidence ≥ threshold: keyword signal wins
       │  - Keyword confidence < threshold AND vocal confidence ≥ 0.70: vocal wins
       │  - Both below threshold: no action
       │  - Signals agree: boost combined confidence, lower effective threshold
       │  - Signals disagree at similar confidence: keyword wins (conservative)
       │
       │  Weight toward vocal signal shifts as per-GM correction data accumulates:
       │  vocal_weight = min(0.4, corrections_count / 500.0)
       ▼
[Detection FSM]
```

---

## 6. ML Model Strategy — ONNX-Native

### OrtEnvironment Singleton

```rust
// ml/ort_env.rs
use ort::{Environment, ExecutionProvider};
use std::sync::Arc;
use once_cell::sync::Lazy;

pub static ORT_ENV: Lazy<Arc<Environment>> = Lazy::new(|| {
    Arc::new(
        Environment::builder()
            .with_name("ttrpg-companion")
            .with_execution_providers([
                ExecutionProvider::CoreML(Default::default()),  // Mac (Apple Silicon)
                ExecutionProvider::DirectML(Default::default()), // Windows GPU if available
                ExecutionProvider::CPU(Default::default()),      // Fallback — always available
            ])
            .build()
            .expect("ORT environment initialization failed — this is unrecoverable"),
    )
});
```

All three model sessions are created from this environment. One environment per process.
CoreML and DirectML are opportunistic — if unavailable, CPU fallback is used silently.

### Model File Paths

Model files are embedded at compile time via `build.rs` using `include_bytes!` for
small models (VAD, Resemblyzer) to guarantee availability. emotion2vec is large
(~350 MB) and is downloaded post-install on first Pro activation.

```rust
// build.rs — embed small models into binary or verify at build time
// Large models: path resolved at runtime from models/ directory
pub const VAD_MODEL: &[u8] = include_bytes!("../../models/silero_vad.onnx");
pub const SPEAKER_MODEL: &[u8] = include_bytes!("../../models/resemblyzer.onnx");
// emotion2vec: loaded from file path, not embedded
```

### Silero VAD (silero_vad.onnx)

- **Input:** Float32 tensor [1, 480] (30ms at 16kHz)
- **Output:** Float32 scalar [1, 1] — speech probability
- **State:** Silero VAD v4 is stateful (carries hidden state between frames).
  Maintain state tensors across frames. Reset state on session start and on
  speech segment completion.
- **Source:** https://github.com/snakers4/silero-vad — ONNX export available directly.

### Resemblyzer Speaker Verification (resemblyzer.onnx)

- **Input:** Float32 tensor [N, 40] — mel-spectrogram frames (variable N)
- **Output:** Float32 tensor [256] — d-vector embedding
- **Similarity:** cosine similarity between enrolled embedding and test embedding
- **Export:** `scripts/export_resemblyzer.py` converts the PyTorch checkpoint to ONNX.
  Use `torch.onnx.export` with `dynamic_axes` for variable-length input.

### emotion2vec+ Base (emotion2vec_base.onnx)

- **Input:** Float32 tensor [N] — raw 16kHz audio samples
- **Output:** Float32 tensor [768] — universal speech emotion embedding
- **Architecture:** Frozen backbone only in production. Per-GM mapping layer
  (`Linear(768, 7)`) is a separate small model, trained and stored per-GM in
  their profile. The ONNX file is the backbone; the mapping layer runs as a
  pure Rust matrix multiply (no ONNX needed for a 768×7 linear layer).
- **V1.x activation:** Model downloaded when user first enables Pro tier.
  Stored in `models/emotion2vec_base.onnx`. SHA-256 verified on download.

### Hardware Self-Regulation

Detect available hardware on startup and set `ComputeProfile`:

```rust
pub enum ComputeProfile {
    Full,        // Runs VAD + Speaker + Whisper medium + emotion2vec (V1.x)
    Standard,    // Runs VAD + Speaker + Whisper medium (no emotion2vec)
    Minimal,     // Runs VAD + Speaker + Whisper small
}

impl ComputeProfile {
    pub fn detect() -> Self {
        let available_ram_gb = system_ram_gb();
        let has_fast_cpu = cpu_benchmark_score() > BENCHMARK_THRESHOLD;

        match (available_ram_gb, has_fast_cpu) {
            (ram, _) if ram >= 12.0 => ComputeProfile::Full,
            (ram, true) if ram >= 8.0 => ComputeProfile::Standard,
            _ => ComputeProfile::Minimal,
        }
    }
}
```

Degradation is silent and automatic. The GM is never asked to choose a performance
profile. If they want to override, it's in Advanced Settings, hidden by default.

---

## 7. Non-Negotiable Constraints

**These are absolute. No exceptions. No workarounds.**

### Privacy Constraints

```
[P1] NO RAW AUDIO EVER WRITTEN TO DISK.
     Audio buffer → processed in memory → discarded.
     Whisper transcript text: may be logged.
     Resemblyzer/emotion2vec embeddings: may be stored (they are numerical vectors,
     not recoverable audio — document this explicitly in the privacy policy).
     Actual audio samples: never persisted, ever.
     Enforcement: CI lint check. Any call to File::create / std::fs::write
     in the audio capture module fails the build.

[P2] NO VOICE DATA LEAVES THE DEVICE.
     whisper-rs: local. ort models: local.
     Speaker embeddings: local, encrypted.
     emotion2vec embeddings: local, encrypted.
     No HTTP calls from any code path that handles audio or voice data.
     Enforcement: reqwest / hyper cannot be imported in audio/ or ml/ modules.
     The Tauri network capability does not grant access to these paths.

[P3] VOCAL ANALYSIS REQUIRES EXPLICIT OPT-IN.
     If gm_consent.vocal_analysis_consent = false:
       - emotion2vec ONNX session is never created
       - emotion2vec model is never loaded into memory
     This check is enforced in startup.rs and session.rs (Rust core).
     A frontend bug cannot bypass it.
     Enforcement: The consent flag is checked before any ORT session creation.
     There is no code path from session start to emotion2vec init that bypasses
     the consent flag check.

[P4] CONSENT MUST PRECEDE ALL PROCESSING.
     gm_consent record must exist and both fields must have been explicitly set
     before the first session can start.
     start_session() returns AppError::ConsentRequired if no consent record exists.
     Enrollment cannot proceed without the consent UI completing successfully.
```

### Reliability Constraints

```
[R1] SESSION-TIME CODE PATH MUST NOT PANIC.
     The detection pipeline handles all errors internally via Result<>.
     Errors are logged (structured, with context) and degraded around.
     Panics in the detection thread restart the thread (supervisor, see below).
     Panics in the audio thread restart the thread.
     Panics in the FMOD thread: log + switch to SilentEngine.
     The GM's session must not crash because Whisper returned an unexpected token.

[R2] AUDIO CONTINUITY IS SACRED.
     The FMOD audio engine runs independently of detection state.
     If the detection thread dies: music continues at last mood.
     If speaker verification fails repeatedly: keyword detection continues alone.
     If Whisper times out: log, reset FSM to IDLE, continue.
     There is no failure mode in detection that silences the audio.

[R3] DETECTION THREAD SUPERVISOR.
     The detection thread runs inside a supervisor loop:
       - On panic or unrecoverable error: log at ERROR level
       - Wait 3 seconds, restart thread (up to 5 times)
       - After 5 restarts: log FATAL, emit detection-error IPC event,
         continue app in manual-only mode (no auto-detection, all hotkeys active)
     The GM sees a single notification in the popover. The session continues.

[R4] SESSION MODE PRESENTS NO UNSOLICITED UI.
     When a session is active, the app's only GM-visible surface is:
       - System tray icon (color + tooltip)
       - Tray popover (GM-initiated, click to open)
     Prohibited during session:
       - Modal dialogs of any kind
       - Toast/notification popups (OS-level or in-app)
       - Any window appearing or gaining focus without GM action
     Exception: Persistent audio hardware failure (> 5 seconds dropout) →
       single status line in popover only, no popup.
```

### Data Integrity Constraints

```
[D1] GENRE IS ALWAYS AN ARRAY, NEVER A STRING.
     In SQLite: track_genres junction table (track_id, genre). Not a TEXT column.
     In Rust: Vec<String> on the Track struct.
     In TypeScript: string[] on the Track type.
     A Track with genre ["fantasy", "horror"] stores two rows in track_genres.
     Query "all fantasy tracks" uses a JOIN on track_genres, not JSON parsing.

[D2] SUBCATEGORY TAGS PRE-ASSIGNED ON ALL BUNDLED TRACKS.
     Every bundled track has mood_sub set at content-database creation time.
     Free-tier users do not see this field in the UI.
     Pro upgrade activates mood_sub display and matching — no migration needed.
     This is the foundation of "upgrade feels instant."

[D3] ALL DETECTION EVENTS ARE LOGGED — AUTO AND MANUAL.
     Two event types, both mandatory:
     - AUTO: system detected a keyword, triggered a mood shift. Logged with
       transcript, keyword_matched, confidence, vocal_embedding (null in V1).
     - CORRECTION: GM used Shift hotkey to change mood. Logged with
       detected_mood (what system had), corrected_mood (what GM set),
       transcript, vocal_embedding (null in V1).
     Both are in detection_events table. Both are training data for V1.x.
     Neither is optional. Neither is skipped for performance reasons.
     An AUTO event held uncorrected for > 60 seconds is an implicit positive label.

[D4] SCHEMA_MIGRATIONS TABLE IS THE MIGRATION SOURCE OF TRUTH.
     migrations table: (version TEXT PRIMARY KEY, applied_at DATETIME).
     On startup: check which versions have been applied, run outstanding migrations
     in version order, record each. Idempotent — safe to run multiple times.
     Migration files are named NNN_description.sql (zero-padded, 3 digits).
     Never modify an applied migration. Create a new one.
```

---

## 8. Platform-Specific Requirements

### macOS

**Microphone entitlement — mandatory, app will silently fail without this.**

```json
// src-tauri/capabilities/microphone.json
{
  "$schema": "https://schema.tauri.app/config/2/capability.json",
  "identifier": "microphone",
  "description": "Microphone access for session listening",
  "platforms": ["macOS"],
  "permissions": ["core:default", "microphone:allow-open"]
}
```

```xml
<!-- src-tauri/Info.plist addition -->
<key>NSMicrophoneUsageDescription</key>
<string>This app listens to your narration to automatically match music to your scene. Audio is processed locally and never transmitted.</string>
```

```xml
<!-- src-tauri/entitlements/macos.entitlements -->
<key>com.apple.security.device.audio-input</key>
<true/>
```

**Permission request flow (permissions.rs):**

```rust
pub async fn check_and_request_mic_permission() -> Result<MicPermissionStatus, AppError> {
    // Use tauri-plugin-microphone or direct macOS AVFoundation bridge
    // Return: Granted | Denied | NotDetermined
    // If NotDetermined: request permission, await user response
    // If Denied: return Denied — user must go to System Preferences
    // Never silently continue with a denied mic — CPAL returns empty stream
}
```

**Universal binary:** Build target must be `universal-apple-darwin` for release.
`cargo tauri build --target universal-apple-darwin`. CI enforces this.

**Notarization:** Required for Gatekeeper. Configure in `tauri.conf.json`:
`"macOS": { "notarizationCredentials": { ... } }`. Without notarization, users
see a "malicious software" warning. Budget $99/yr Apple Developer Program.

### Windows

**Microphone permission — Windows 10/11 Privacy Settings.**

Windows does not show a permission dialog the way macOS does. Instead, the user must
have "Microphone access" enabled for desktop apps in Settings → Privacy → Microphone.
If this is disabled, CPAL opens the device but returns silence. No error is thrown.

**Detection at startup (permissions.rs):**

```rust
// Windows: write a 1-second test recording, measure RMS
// If RMS < -60 dBFS → mic is likely blocked by privacy settings
// Show a setup screen explaining how to enable microphone access
// Do not assume silence = quiet room. Distinguish.
```

**Code signing:** Required to avoid Windows Defender SmartScreen warnings.
Budget ~$200/yr for a standard OV code signing certificate. EV certificate
($300–700/yr) removes SmartScreen entirely — worth it post-revenue.

**Windows path considerations:**

```rust
// App data directory: use tauri::api::path::app_data_dir()
// Never hardcode %APPDATA% or $HOME — use the Tauri path API
// Profile directory: {app_data_dir}/profiles/{gm_id}/
// Content DB: {app_data_dir}/content.db
// Models: {app_data_dir}/models/ (emotion2vec downloaded here)
// Logs: {app_data_dir}/logs/
```

### Both Platforms

**FMOD library placement:**

```
src-tauri/
└── vendor/
    └── fmod/
        ├── api/
        │   └── core/
        │       ├── inc/           ← fmod.h headers
        │       └── lib/
        │           ├── x86_64/    ← Linux (not currently targeted)
        │           ├── mac/       ← libfmod.dylib
        │           └── x64/       ← fmod.dll (Windows)
        └── .gitignore             ← FMOD SDK is not redistributed in repo
```

`FMOD_SDK_PATH` environment variable must be set for builds. CI sets it from
a repository secret. Document in README.md setup instructions.

---

## 9. Startup Sequence & Phase Model

Cold start to "detection ready" spans two phases. The app is interactive (tray
visible, UI usable) well before detection is ready. This is the correct UX and
also the only way to hit a reasonable cold-start budget.

### Phase 1 — UI Ready (target: ≤ 3 seconds)

```
App process starts
  → Tauri window created (invisible)
  → DB connection pool initialized
  → Migrations run (idempotent, fast)
  → Config loaded from DB
  → GM profile loaded (existence check only, no decryption)
  → Consent flags loaded
  → Tray icon shown (grey / "loading" state if no profile, mood color if profile exists)
  → Setup window shown (or hidden if profile exists — await user to open)
  → IPC event emitted: startup-phase-changed { phase: "ui_ready" }
  → Frontend shows setup window in "ready" state
```

### Phase 2 — Detection Ready (target: ≤ 15 seconds from process start)

Runs async in the background after Phase 1 completes. GM can use the setup
window, import music, and configure settings during this phase.

```
[Background, after Phase 1]
  → Load silero_vad.onnx into ORT session
  → Load resemblyzer.onnx into ORT session
  → Initialize CPAL audio capture (request mic permission if needed — Mac)
  → Check mic availability (Windows silence test)
  → Load Whisper medium model (this is the slow step: 3–8 seconds)
  → Initialize FMOD engine
  → Load keyword vocabularies from DB
  → Initialize vocabulary file watcher (notify)
  → [If vocal_analysis_consent AND emotion2vec model exists AND ComputeProfile::Full]
      → Load emotion2vec_base.onnx into ORT session
  → IPC event emitted: startup-phase-changed { phase: "detection_ready" }
  → Tray icon transitions to full-color mood state
  → "Start Session" button becomes enabled in setup window
```

### Startup Phase Store (Frontend)

```typescript
// lib/stores/startup.ts
export type StartupPhase = 'initializing' | 'ui_ready' | 'detection_ready' | 'error';

export const startupStore = writable<{
  phase: StartupPhase;
  error: string | null;
  micPermission: 'granted' | 'denied' | 'unknown';
}>({ phase: 'initializing', error: null, micPermission: 'unknown' });
```

The "Start Session" button is disabled until `phase === 'detection_ready'`.
A non-blocking status line shows "Preparing audio engine..." during Phase 2.

---

## 10. Build, Run & Test Commands

### Prerequisites

```bash
# 1. Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add aarch64-apple-darwin x86_64-apple-darwin  # Mac universal
rustup target add x86_64-pc-windows-msvc                    # Windows

# 2. Node.js — use mise or nvm, Node 20 LTS minimum
mise install node@20  # or: nvm install 20

# 3. Tauri CLI v2
cargo install tauri-cli --version "^2.0"

# 4. FMOD SDK
# Download from https://www.fmod.com/download → FMOD Engine → API
# Extract to src-tauri/vendor/fmod/
export FMOD_SDK_PATH="$(pwd)/src-tauri/vendor/fmod"

# 5. ONNX models — run setup script
chmod +x scripts/download_models.sh
./scripts/download_models.sh
# This downloads silero_vad.onnx and resemblyzer.onnx, verifies SHA-256
# emotion2vec is NOT downloaded here — it's downloaded by the app on demand

# 6. Python environment (tools only — not shipped)
python3 -m venv .venv && source .venv/bin/activate
pip install -r tools/audio_tagger/requirements.txt
pip install -r scripts/requirements.txt  # for export scripts

# 7. Frontend dependencies
cd src && npm install && cd ..
```

### Development

```bash
# Full dev build (hot reload frontend + Rust recompile on change)
cargo tauri dev

# Debug with verbose detection logging
TTRPG_LOG=detection=debug,audio=info cargo tauri dev

# Rust backend tests only (fast — no Tauri runtime needed)
cargo test -p ttrpg-companion-tauri

# Specific test
cargo test -p ttrpg-companion-tauri -- fsm_test::test_lock_exits_to_idle

# Frontend type check
cd src && npm run check

# Frontend dev without Tauri (UI iteration only)
cd src && npm run dev

# Watch mode — Rust (useful for detection pipeline development)
cargo watch -x "test -p ttrpg-companion-tauri -- detection"
```

### Code Quality

```bash
# Format (required before commit)
cargo fmt --all
cd src && npm run format

# Lint — Rust (zero warnings policy in CI)
cargo clippy --all-targets --all-features -- \
  -D warnings \
  -D clippy::unwrap_used \
  -D clippy::expect_used \
  -D clippy::panic \
  -D clippy::unimplemented \
  -D clippy::todo

# Lint — TypeScript
cd src && npm run lint

# Type check — Python tools
mypy tools/ scripts/ --strict --ignore-missing-imports
```

### Content & Models

```bash
# Run audio tagger on a folder — produces manifest.json
python tools/audio_tagger/main.py \
  --input /path/to/music/ \
  --output content/music/manifest.json \
  --accuracy-report \
  --ground-truth tests/fixtures/audio_manifest.json

# Validate keyword vocabulary
cargo run --bin validate-keywords -- content/keywords/en_fantasy_v1.json

# Export Resemblyzer to ONNX (run once when updating model)
python scripts/export_resemblyzer.py --output models/resemblyzer.onnx

# Export emotion2vec to ONNX (run once when updating model)
python scripts/export_emotion2vec.py --output models/emotion2vec_base.onnx
```

### Release

```bash
# Release build (current platform)
cargo tauri build

# Universal Mac binary
cargo tauri build --target universal-apple-darwin

# Verify binary size (FMOD + ONNX models add bulk — track this)
ls -lh src-tauri/target/release/bundle/
```

### Database Inspection

```bash
# Content DB (development)
sqlite3 "$(echo ~/.local/share/ttrpg-companion/content.db)"

# Profile DB (decrypt first — cannot open .enc file directly)
# Use the app's CLI export command:
cargo run --bin profile-tool -- export --gm-id <uuid> --output /tmp/profile_export/
```

---

## 11. CI/CD Pipeline

### ci.yml — Runs on Every Push and Pull Request

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  FMOD_SDK_PATH: ${{ secrets.FMOD_SDK_PATH_CI }}
  RUST_BACKTRACE: 1

jobs:
  rust-checks:
    name: Rust — Format, Lint, Test
    runs-on: ubuntu-latest  # Unit tests run on Linux (no Tauri window needed)
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      - name: Cache cargo
        uses: Swatinem/rust-cache@v2

      - name: Setup ONNX models (stubs for CI)
        run: ./scripts/download_models.sh --ci-stub
        # CI stub: creates empty files at expected paths so include_bytes! compiles

      - name: Format check
        run: cargo fmt --all -- --check

      - name: Clippy (zero warnings, zero panics)
        run: |
          cargo clippy --all-targets --all-features -- \
            -D warnings \
            -D clippy::unwrap_used \
            -D clippy::expect_used \
            -D clippy::panic \
            -D clippy::unimplemented \
            -D clippy::todo

      - name: Unit tests
        run: cargo test -p ttrpg-companion-tauri --lib

      - name: Integration tests (no audio hardware)
        run: cargo test -p ttrpg-companion-tauri --test integration -- --skip real_audio

  frontend-checks:
    name: Frontend — Type Check, Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: src/package-lock.json

      - name: Install dependencies
        run: cd src && npm ci

      - name: Type check
        run: cd src && npm run check

      - name: Lint
        run: cd src && npm run lint

  python-checks:
    name: Python — Type Check, Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r tools/audio_tagger/requirements.txt
          pip install -r scripts/requirements.txt
          pip install mypy

      - name: Type check
        run: mypy tools/ scripts/ --strict --ignore-missing-imports

      - name: Test audio tagger
        run: python -m pytest tools/ -v

  keyword-validation:
    name: Keyword Vocabulary Validation
    runs-on: ubuntu-latest
    needs: rust-checks
    steps:
      - uses: actions/checkout@v4
      - name: Validate all vocabulary files
        run: |
          for f in content/keywords/*.json; do
            cargo run --bin validate-keywords -- "$f"
          done
```

### release.yml — Runs on Version Tags

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-mac:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build universal Mac binary
        run: cargo tauri build --target universal-apple-darwin
        env:
          APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
          APPLE_CERTIFICATE: ${{ secrets.APPLE_CERTIFICATE }}
          APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
          APPLE_NOTARIZATION_USERNAME: ${{ secrets.APPLE_NOTARIZATION_USERNAME }}
          APPLE_NOTARIZATION_PASSWORD: ${{ secrets.APPLE_NOTARIZATION_PASSWORD }}
          FMOD_SDK_PATH: ${{ secrets.FMOD_SDK_PATH_CI }}
      - name: Upload DMG artifact
        uses: actions/upload-artifact@v4
        with:
          name: mac-dmg
          path: src-tauri/target/universal-apple-darwin/release/bundle/dmg/*.dmg

  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Windows installer
        run: cargo tauri build
        env:
          WINDOWS_CERTIFICATE: ${{ secrets.WINDOWS_CERTIFICATE }}
          WINDOWS_CERTIFICATE_PASSWORD: ${{ secrets.WINDOWS_CERTIFICATE_PASSWORD }}
          FMOD_SDK_PATH: ${{ secrets.FMOD_SDK_PATH_CI }}
      - name: Upload MSI artifact
        uses: actions/upload-artifact@v4
        with:
          name: windows-msi
          path: src-tauri/target/release/bundle/msi/*.msi

  publish-itch:
    needs: [build-mac, build-windows]
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
      - name: Push to itch.io
        uses: josephbmanley/butler-publish-itchio-action@v1
        env:
          BUTLER_CREDENTIALS: ${{ secrets.ITCHIO_BUTLER_KEY }}
          CHANNEL_MAC: mac
          CHANNEL_WINDOWS: windows
          ITCH_GAME: ttrpg-companion  # replace with actual itch.io game slug
          ITCH_USER: petros           # replace with itch.io username
          VERSION: ${{ github.ref_name }}
```

---

## 12. Module Breakdown & Acceptance Criteria

### Phase 0 — Existential Spikes

Run all spikes before writing production code. Each is a throwaway prototype
(separate branch, not merged to main). Results documented in
`tests/fixtures/real_world_log.md`.

---

#### Spike S1 — Tauri Shell Viability

**Question:** Can Tauri 2.x cleanly handle system tray + global hotkeys + background
operation without platform-specific structural problems?

**Deliverable:** Throwaway Tauri 2.x app. System tray icon with 7 color states.
Three global hotkeys that change tray color. No audio. No window required.

**Pass criteria:**
- [ ] Hotkeys fire when any other application is focused (PDF reader, browser)
- [ ] Tray icon renders correctly: Windows 10/11, macOS 12+ Intel, macOS 14+ ARM
- [ ] No window appears or gains focus on hotkey press
- [ ] Cold start to interactive tray: ≤ 3 seconds on 2020-era hardware
- [ ] Idle RAM: ≤ 50 MB

**Fail path:** If hotkeys conflict with common system shortcuts, implement
configurable key bindings as first mitigation. If tray diverges structurally
between platforms, document the delta — acceptable, not a blocker.

---

#### Spike S2 — Whisper Accuracy Under Tabletop Conditions

**Question:** Does whisper.cpp medium produce usable transcription from a laptop mic
in a live gaming environment?

**This is the highest-risk spike. Run first.**

**Deliverable:** CLI script. Accepts pre-recorded WAV. Runs full pipeline:
resample → normalize → VAD → (skip speaker check) → Whisper → keyword match.
Reports metrics.

**Test matrix — record all conditions separately:**

| Condition | Description |
|-----------|-------------|
| Baseline | GM narrates alone, quiet room |
| Crosstalk | GM narrates, 3 players talking/laughing simultaneously |
| Music bleed | GM narrates, music from same laptop speaker at session volume |
| Character voices | GM does NPC voices (pitch/accent variation) |
| Code-switching | Greek narration with English game terms interspersed |
| Quiet narrator | GM with naturally soft voice, same noisy conditions |

**Pass criteria:**
- [ ] Keyword detection rate (crosstalk condition): ≥ 75%
- [ ] Keyword detection rate (baseline): ≥ 85%
- [ ] False positive rate: ≤ 1 per 10 minutes of non-trigger narration
- [ ] P95 latency (end of trigger phrase → Whisper complete): ≤ 10 seconds
  *(Note: this is detection latency, not transition latency — the 8–12s buffer
  means Whisper fires 8–12s after speech begins, not 2s. The audio starts
  crossfading immediately when the keyword is detected within that buffer.
  The user-visible latency budget in §18 is measured differently.)*
- [ ] Greek code-switching: detection rate ≥ 60%

**Fail paths:**
- 60–75% detection rate: enforce VAD pre-filtering, tighten vocabulary to top 5
  phrases per mood only, reduce to short-segment mode (flush at 4s, accept
  higher error rate)
- < 60%: evaluate Whisper large-v3; consider real-time streaming via
  whisper.cpp's `stream` example as an alternative to batch inference

---

#### Spike S3 — FMOD Crossfade Quality

**Question:** Does FMOD produce seamless crossfades and SFX layering?

**Pass criteria:**
- [ ] 10/10 non-musician listeners cannot identify crossfade start/end
- [ ] Loop endpoint is genuinely gapless (no click, no silence)
- [ ] SFX layered over high-energy combat track: no clipping
- [ ] Trigger-to-audible latency: ≤ 200ms
- [ ] FMOD indie license permits redistribution in commercial indie app: confirmed

**Fail path:** Platform-native audio engine (Core Audio / WASAPI). Add ~3 weeks.
Document decision.

---

#### Spike S4 — ONNX Stack Resource Profile

**Question:** Can Silero VAD + Resemblyzer + Whisper medium all run simultaneously
on 8 GB RAM / no GPU without causing audio dropout or OOM?

**Test on minimum spec hardware (8 GB RAM, no GPU, ~2019–2020 laptop).**

**Pass criteria:**
- [ ] Combined peak RAM (all three models loaded): ≤ 4 GB
- [ ] VAD inference per frame (30ms): ≤ 5ms
- [ ] Resemblyzer inference per segment: ≤ 200ms
- [ ] CPU utilization during Whisper inference: does not cause FMOD audio dropout
- [ ] emotion2vec base (additional, if run): combined peak RAM ≤ 5.5 GB

**Fail path:** emotion2vec deferred. If Whisper + VAD + Resemblyzer exceeds 4 GB,
evaluate Whisper small model. Spike failure does not block MVP.

---

### Phase 1 — MVP Build

Build in order. Each module's acceptance criteria must pass before the next begins,
except where noted as parallelizable.

---

#### M1 — Application Shell

**Parallelizable with:** M3a

**Scope:**
- Tauri 2.x project with workspace structure per §2
- System tray icon: 7 mood color states (hex values from §13), tooltip shows mood name
- Global hotkeys: Next, Shift, Hold/Lock — all configurable via config table
- Two UI modes: `Setup` (full window) and `Session` (tray + popover only)
- `StartSession` / `EndSession` transitions — no detection wiring yet
- IPC event stubs: emit events with correct payload shapes, frontend subscribes
- Two-phase startup orchestration per §9 — Phase 1 completes in M1
- AppState, SessionState structs defined (§13)
- DB connection pool init + migration runner + schema_migrations enforcement
- Structured logging: `tracing` crate, JSON format, rolling file sink,
  max 10 MB/file, 5 file rotation, console sink in debug builds only
- App config CRUD (SQLite, not filesystem)
- Mic permission check at startup — Mac entitlement + Windows silence test
- `startup-phase-changed` IPC event emitted correctly for both phases

**Do not build in M1:** audio, detection, any meaningful UI beyond navigation shell

**Acceptance criteria:**
- [ ] Hotkeys fire from any foreground app: Windows 10, macOS 12 Intel, macOS 14 ARM
- [ ] StartSession hides window, shows tray only — window recoverable only via EndSession
- [ ] Tray icon color update via IPC: ≤ 100ms from event emit to visible change
- [ ] Phase 1 startup: ≤ 3 seconds cold start on 2020-era hardware
- [ ] DB migrations: idempotent (run 3× in sequence — no errors, no duplicate data)
- [ ] `schema_migrations` table correctly tracks applied versions
- [ ] Log file created, JSON lines, rotation confirmed after artificial size limit
- [ ] Mic permission check: correctly reports Granted/Denied/Unknown on both platforms
- [ ] Windows silence test: distinguishes silence-from-permission-block vs quiet-room

---

#### M2 — Audio Engine

**Depends on:** M1

**Scope:**
- `AudioEngine` trait with full method set
- `FmodEngine`: FMOD initialization, music channel, SFX channel, crossfade,
  ducking, gapless loop, `Standard` and `MusicalBreath` transition types
- `SilentEngine`: records all method calls, asserts on call sequences in tests,
  produces no audio output — **used in all automated tests**
- Crossfade type selection logic: `MoodCategory::transition_to()` (defined in §13)
- Volume ducking: music → 40% during SFX, 200ms ramp up after SFX completes
- Error handling: missing file → skip + log; FMOD init failure → SilentEngine fallback
- `track-changed` IPC event on every track transition

**Acceptance criteria:**
- [ ] SilentEngine used in all unit tests — zero tests require audio hardware
- [ ] Standard crossfade: 10/10 non-musician listeners cannot identify transition
- [ ] MusicalBreath crossfade: dip audible but feels like a deliberate musical pause
- [ ] Gapless loop: zero audible gap in 100 consecutive loop iterations
- [ ] SFX over combat track: no clipping (measured, not just perceived)
- [ ] FMOD init failure: falls back to SilentEngine, logs at ERROR, no crash
- [ ] 4-hour continuous playback: RAM growth ≤ 10 MB, zero audio dropout

---

#### M3a — Audio Analysis Tool

**Parallelizable with:** M1, M2 (standalone Python, no Tauri dependency)
**Must complete before M3b** — produces the content DB seed data

**Scope:**
- CLI: `python tools/audio_tagger/main.py --input <dir> --output <manifest.json>`
- Per-track: librosa feature extraction (BPM, RMS energy, key estimate,
  spectral centroid, duration)
- Rule-based mood classifier: implement exactly the rules in §13, no ML,
  no additional features, heuristic thresholds adjustable only with test data
- Output: `manifest.json` array matching Track schema (§13), genre_tags as array
- `--accuracy-report`: compare predicted vs `ground_truth.json`, output per-category
  precision/recall/F1
- Handle corrupt files: log warning, skip, continue — never crash on bad input
- Recursively process folders, accept: MP3, FLAC, WAV, OGG, M4A

**Acceptance criteria:**
- [ ] 100-track library processed in ≤ 3 minutes on any modern machine
- [ ] Broad mood accuracy ≥ 80% on 50-track labeled test set (fixture)
- [ ] Output JSON validates against Track schema
- [ ] Corrupt files: skipped with warning, processing continues
- [ ] genre_tags output is always a JSON array, never a string

---

#### M3b — Content Database & BYOM

**Depends on:** M1 (DB infrastructure), M3a (manifest)

**Scope:**
- Full schema from migrations/001_initial.sql (§13)
- `track_genres` junction table operations in `db/genres.rs`
- Bundled library seed from manifests on first launch
- BYOM import: drag-and-drop → copy to app-managed dir →
  run audio tagger → insert to DB
- Track review UI: grouped by predicted mood, drag-to-reassign, Approve All
- Library screen: list, filter (mood/genre/source), search, 3s preview,
  per-track mood edit, remove BYOM tracks
- Attribution display: all bundled tracks with `attribution_required` shown
  with source URL
- Attribution export: text file listing tracks played in last session

**Do not build:** subcategory tag UI (Pro, post-MVP), filename parsing heuristics

**Acceptance criteria:**
- [ ] BYOM 100-track import: review screen ready in ≤ 5 minutes
- [ ] Drag-to-reassign persists across app restart
- [ ] Genre filter uses JOIN on `track_genres`, not JSON parsing or LIKE
- [ ] Removing BYOM track: DB record deleted + copied file deleted from app dir
- [ ] Attribution export: lists only tracks actually played (joined with session log)
- [ ] All bundled tracks seeded with `mood_sub` set (even though UI hides it)

---

#### M4 — Keyword Detection Pipeline

**Depends on:** M1, M2, M3b
**This is the product. Allocate the most time here.**

**Scope:**
- CPAL audio capture: configurable input device, opens at native rate
- Resampler (rubato): native rate → 16kHz mono
- Channel downmix: stereo → mono before resample
- All bounded channels per §4 constants
- Normalizer: RMS to -16 dBFS per segment
- Silero VAD (ort): frame-by-frame, stateful, pre/post padding
- Segment buffer: 8–12s window, flush conditions per §5
- Resemblyzer (ort): d-vector extraction + cosine similarity
- whisper-rs: medium model, auto language, word timestamps, spawn_blocking
- Keyword engine: fuzzy Levenshtein matching, vocabulary from DB
- Detection FSM: exact spec from §14 — all states, all transitions, all timeouts
- Session logger: DetectionEvent for both AUTO and CORRECTION types (§13)
- Vocabulary hot-reload: `notify` watcher on `content/keywords/`, reload on
  file change between sessions (not mid-session)
- Detection thread supervisor: restart on failure, max 5, then manual-only mode
- `detect-mode-changed` IPC event when entering/leaving manual-only fallback

**Do not build:** emotion2vec (V1.x), subcategory matching (Pro), speaker diarization

**Acceptance criteria:**
- [ ] Keyword detection rate (crosstalk, group session): ≥ 75%
- [ ] False positive rate: ≤ 1 per 10 minutes
- [ ] Speaker verification rejects player voices: ≥ 80% precision (4-person test)
- [ ] LOCKED state: auto-detection suspended, all hotkeys functional
- [ ] Cooldown: auto re-trigger suppressed for 30s after mood shift
- [ ] FSM LOCKED: exits to IDLE unconditionally on toggle (not to previous state)
- [ ] FSM LISTENING timeout (30s): exits to IDLE, logged as Timeout event
- [ ] FSM TRANSCRIBING timeout (15s): exits to IDLE, logged as Timeout event
- [ ] Detection thread: restarts automatically on panic (3s backoff, ≤ 5 times)
- [ ] After 5 restarts: manual-only mode, IPC event emitted, no crash
- [ ] Both AUTO and CORRECTION detection events logged with all non-null fields
- [ ] AUTO event uncorrected for > 60s: implicit_positive flag set in DB
- [ ] Vocabulary hot-reload: change a .json file, restart session, new keywords active
- [ ] 4-hour session: RAM growth ≤ 25 MB in detection components

---

#### M5 — Voice Training & GM Profile

**Depends on:** M1, M4 (Resemblyzer ORT session scaffolded)
**Parallelizable with:** M4 late-stage

**Scope:**
- Voice enrollment: guided UI, 7 passages, ~60s each, plays matching music
  during recording (demo IS the setup — GM hears the product while training it)
- 7 passages committed to `content/passages/en/` and `content/passages/el/`
  — see passage requirements in §13
- Resemblyzer d-vector extraction, stored in encrypted profile DB
- emotion2vec embeddings extracted from enrollment audio and stored
  even in V1 (null in active use — pre-populated for V1.x activation)
- Profile storage: AES-256-GCM, key from OS keychain via `keyring` crate
  (macOS Keychain, Windows DPAPI). Document as "AES-256-GCM with OS-managed
  key storage" in all user-facing text. Never call it "convenience encryption."
- Consent UI: two explicit checkboxes before any audio is recorded.
  `speaker_verification_consent` required. `vocal_analysis_consent` optional.
  Consent record written to DB before enrollment begins.
- Profile management: re-train, delete (full wipe, confirmed), profile stats display

**Do not build:** subcategory training (Pro, 20 passages), profile sync/export (Phase 2)

**Acceptance criteria:**
- [ ] Full enrollment completes in ≤ 3 minutes
- [ ] Speaker verification: GM vs 3 others, ≥ 80% precision identifying GM
- [ ] `vocal_analysis_consent = false`: no emotion2vec ORT session created (verified
  by process inspection, not UI state — no ort session in memory)
- [ ] Consent record written before any audio recorded (verified: DB record exists
  before CPAL capture starts for enrollment)
- [ ] Delete profile: `profiles/{gm_id}/` directory empty after deletion,
  `gm_consent` record deleted, verified by directory listing + DB query
- [ ] Profile DB: cannot be opened with sqlite3 CLI (encryption verified)
- [ ] App fully functional with keyword detection only when vocal_analysis_consent
  is false — no degraded UX, no error state, no warnings shown to user

---

#### M6 — Session Interface

**Depends on:** M1–M5

**Scope:**
- StartSession validation: ≥ 1 track per mood category (warn if any empty),
  GM profile exists, consent record valid, detection_ready phase complete
- StartSession: starts audio capture, starts detection pipeline, transitions to
  Session mode
- Tray icon: pulse animation during crossfade (500ms ease in/out, 2 cycles),
  solid during LOCKED, grey during startup Phase 2, full color when ready
- Mood popover: 7 mood buttons (3×3 grid, 7 filled + 1 spacer + End Session),
  current mood highlighted, track name, volume slider (music only), Skip Track,
  Lock/Unlock, End Session
- Popover: opens without focus steal, closes on click-away or Escape
- Transition type routing per `MoodCategory::transition_to()` rules
- Error display in popover: single status line only, no modal, no popup
- EndSession: stops detection, stops audio, writes session end record, transitions
  to Setup mode

**Acceptance criteria:**
- [ ] 4-hour session test: zero crashes, zero panics, zero audio dropout
- [ ] Popover open/close: foreground app retains focus before and after (measured)
- [ ] All 7 mood buttons + 3 hotkeys produce correct audio transitions
- [ ] Tray color updates: ≤ 100ms from mood change event
- [ ] StartSession fails gracefully with no profile: error shown in setup window,
  no crash, enrollment flow offered
- [ ] StartSession fails gracefully with zero tracks: error shown, library flow offered
- [ ] StartSession blocked during startup Phase 2 (button disabled until detection_ready)

---

#### M7 — Setup Window

**Depends on:** M3b, M5
**Parallelizable with:** M6 late-stage

**Scope:**
- Navigation: Library, Voice, Session Settings, About (left sidebar)
- Library screen: full M3b spec
- Voice screen: full M5 spec
- Session Settings:
  - Hotkey customization (all 3, conflict detection against common system shortcuts)
  - Input device dropdown (CPAL device list)
  - Output device dropdown
  - Test mic button (real-time VAD activity bar — green when speech detected)
  - Detection confidence threshold slider (0.50–0.95, default 0.75)
  - Detection cooldown slider (10s–120s, default 30s)
  - Language selection (English / Greek / Both)
  - SFX global toggle
  - Whisper model size selector (small / medium / large-v3)
    - Show RAM estimate next to each option
    - Warn if large-v3 selected on < 12 GB RAM system
  - Reset to defaults button (requires confirmation)
- About: app version, full attribution list for bundled library, privacy summary
  ("Your voice is processed locally and never transmitted"), privacy policy link
- All settings: auto-save on change (no Save button), debounced 500ms

**Acceptance criteria:**
- [ ] New user: full setup (import + enrollment + settings review) ≤ 10 minutes
- [ ] Setup window locked/greyed during active session (not hidden — locked)
- [ ] All settings persist across app restart
- [ ] Hotkey conflict detection: warns on Cmd+Space (Mac), Win+key combos (Windows)
- [ ] Test mic button: VAD bar responds to speech within 500ms
- [ ] Model size change: requires app restart (warn user, offer restart button)

---

### Phase 1.x — Vocal Classifier Activation

**Entry condition:** V1 shipped. ≥ 5 sessions logged per early-access GM.
Spike S4 passed. All below is scaffolded in Phase 1 — this phase activates it.

**Scope:**
- Load emotion2vec_base.onnx into ORT session (gated by vocal_analysis_consent)
- Extract embeddings during session (parallel to Whisper path)
- Per-GM mapping layer: `Linear(768, 7)` in Rust (ndarray or candle)
  trained from 7 enrollment samples as seed, updated post-session
- Post-session background training on accumulated DetectionEvents
- Dual-signal fusion per §5 V1.x addition
- `implicit_positive` flag on AUTO events uncorrected for > 60s used as positive
  training examples (GM implicitly confirmed the detection was correct)
- Background model update: runs after EndSession, ≤ 30 seconds on any target hardware
- Profile reset option: wipes mapping layer weights, resets to enrollment-only baseline

---

## 13. Data Schemas — Canonical Definitions

### SQLite — Content Database

```sql
-- ============================================================
-- File: src-tauri/src/db/migrations/001_initial.sql
-- ============================================================

-- Migration tracking — MUST be first in every migration file
CREATE TABLE IF NOT EXISTS schema_migrations (
    version     TEXT PRIMARY KEY,
    applied_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Core tracks table (no genre column — genres are in track_genres)
CREATE TABLE IF NOT EXISTS tracks (
    id                   TEXT PRIMARY KEY,   -- UUIDv4
    filename             TEXT NOT NULL,
    filepath             TEXT NOT NULL,      -- absolute path in app-managed directory
    title                TEXT,
    artist               TEXT,
    source               TEXT NOT NULL       -- 'bundled' | 'byom' | 'commissioned'
                         CHECK (source IN ('bundled', 'byom', 'commissioned')),
    license_type         TEXT NOT NULL,      -- 'CC0' | 'CC-BY-4.0' | 'CC-BY-SA-4.0' | 'PROPRIETARY'
    commercial_ok        INTEGER NOT NULL    CHECK (commercial_ok IN (0, 1)),
    attribution_required INTEGER NOT NULL    CHECK (attribution_required IN (0, 1)),
    attribution_text     TEXT,               -- full attribution string for display
    mood_broad           TEXT NOT NULL
                         CHECK (mood_broad IN ('Calm','Tension','Combat','Mystery','Travel','Celebration','Sorrow')),
    mood_sub             TEXT,               -- null display for free tier; pre-set on all bundled tracks
    bpm                  REAL,
    energy               REAL,               -- RMS energy, normalized 0.0–1.0
    key_estimate         TEXT,               -- e.g. 'G major', 'D minor'
    spectral_centroid    REAL,               -- Hz
    duration_seconds     REAL,
    is_bundled           INTEGER NOT NULL    CHECK (is_bundled IN (0, 1)),
    is_byom              INTEGER NOT NULL    CHECK (is_byom IN (0, 1)),
    created_at           DATETIME NOT NULL   DEFAULT CURRENT_TIMESTAMP
);

-- Genre junction table (replaces genre_tags TEXT column)
-- A track with genres ["fantasy", "horror"] has two rows here
CREATE TABLE IF NOT EXISTS track_genres (
    track_id  TEXT NOT NULL REFERENCES tracks(id) ON DELETE CASCADE,
    genre     TEXT NOT NULL,
    PRIMARY KEY (track_id, genre)
);
CREATE INDEX IF NOT EXISTS idx_track_genres_genre ON track_genres(genre);

CREATE TABLE IF NOT EXISTS sfx (
    id                   TEXT PRIMARY KEY,
    filename             TEXT NOT NULL,
    filepath             TEXT NOT NULL,
    label                TEXT NOT NULL,
    source               TEXT NOT NULL,
    license_type         TEXT NOT NULL,
    commercial_ok        INTEGER NOT NULL    CHECK (commercial_ok IN (0, 1)),
    attribution_text     TEXT,
    category             TEXT NOT NULL,      -- 'door'|'combat'|'weather'|'crowd'|'magic'|'nature'
    confidence_threshold REAL NOT NULL       DEFAULT 0.85,  -- higher than mood: false SFX is jarring
    cooldown_seconds     REAL NOT NULL       DEFAULT 30.0,
    is_bundled           INTEGER NOT NULL    CHECK (is_bundled IN (0, 1)),
    created_at           DATETIME NOT NULL   DEFAULT CURRENT_TIMESTAMP
);

-- SFX trigger keywords (separate table for clarity and queryability)
CREATE TABLE IF NOT EXISTS sfx_keywords (
    sfx_id  TEXT NOT NULL REFERENCES sfx(id) ON DELETE CASCADE,
    phrase  TEXT NOT NULL,
    PRIMARY KEY (sfx_id, phrase)
);

CREATE TABLE IF NOT EXISTS keywords (
    id               TEXT PRIMARY KEY,
    language         TEXT NOT NULL,          -- 'en' | 'el'
    genre            TEXT NOT NULL,          -- 'fantasy' | 'horror' | ...
    version          TEXT NOT NULL,          -- vocabulary file version string
    phrase           TEXT NOT NULL,
    trigger_type     TEXT NOT NULL
                     CHECK (trigger_type IN ('mood_shift', 'sfx_trigger')),
    target_mood      TEXT,                   -- required if trigger_type = 'mood_shift'
    sfx_id           TEXT,                   -- required if trigger_type = 'sfx_trigger'
    base_confidence  REAL NOT NULL,
    description      TEXT,
    UNIQUE (language, genre, phrase)
);
CREATE INDEX IF NOT EXISTS idx_keywords_lang_genre ON keywords(language, genre);

CREATE TABLE IF NOT EXISTS sessions (
    id          TEXT PRIMARY KEY,            -- UUIDv4
    started_at  DATETIME NOT NULL,
    ended_at    DATETIME,
    genre       TEXT NOT NULL,
    gm_id       TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS session_tracks_played (
    session_id   TEXT NOT NULL REFERENCES sessions(id),
    track_id     TEXT NOT NULL REFERENCES tracks(id),
    played_at    DATETIME NOT NULL,
    play_order   INTEGER NOT NULL,
    PRIMARY KEY (session_id, play_order)
);

CREATE TABLE IF NOT EXISTS config (
    key        TEXT PRIMARY KEY,
    value      TEXT NOT NULL,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT OR IGNORE INTO config (key, value) VALUES
    ('hotkey_next',                    'ctrl+right'),
    ('hotkey_shift',                   'ctrl+up'),
    ('hotkey_lock',                    'ctrl+space'),
    ('detection_confidence_threshold', '0.75'),
    ('detection_cooldown_seconds',     '30'),
    ('language',                       'en'),
    ('sfx_enabled',                    'true'),
    ('whisper_model_size',             'medium'),
    ('speaker_similarity_threshold',   '0.75'),
    ('audio_output_device',            'default'),
    ('audio_input_device',             'default'),
    ('compute_profile_override',       'auto');

-- Record this migration as applied
INSERT OR IGNORE INTO schema_migrations (version) VALUES ('001');
```

```sql
-- ============================================================
-- File: src-tauri/src/db/migrations/002_corrections.sql
-- ============================================================

CREATE TABLE IF NOT EXISTS detection_events (
    id                TEXT PRIMARY KEY,      -- UUIDv4
    session_id        TEXT NOT NULL REFERENCES sessions(id),
    timestamp         DATETIME NOT NULL,
    event_type        TEXT NOT NULL
                      CHECK (event_type IN ('auto', 'correction', 'suppressed', 'timeout')),
    detected_mood     TEXT,                  -- mood system had before this event
    triggered_mood    TEXT,                  -- mood system shifted to (auto events)
    corrected_mood    TEXT,                  -- mood GM set (correction events)
    whisper_transcript TEXT,
    keyword_matched   TEXT,
    confidence        REAL,
    vocal_embedding   BLOB,                  -- null in V1; 768-dim float32 in V1.x
    implicit_positive INTEGER NOT NULL       DEFAULT 0  -- set to 1 after 60s uncorrected auto event
                      CHECK (implicit_positive IN (0, 1))
);
CREATE INDEX IF NOT EXISTS idx_detection_events_session ON detection_events(session_id);
CREATE INDEX IF NOT EXISTS idx_detection_events_type    ON detection_events(event_type);

INSERT OR IGNORE INTO schema_migrations (version) VALUES ('002');
```

```sql
-- ============================================================
-- File: src-tauri/src/db/migrations/003_detection_events.sql
-- ============================================================
-- Reserved for detection_events schema evolution.
-- Add here when V1.x activates vocal_embedding column population.
INSERT OR IGNORE INTO schema_migrations (version) VALUES ('003');
```

### Encrypted Profile Database (per GM, at profiles/{gm_id}/profile.db)

This is a standard SQLite database encrypted with AES-256-GCM before writing to disk.
The encryption key is stored in the OS keychain via the `keyring` crate.

```sql
CREATE TABLE IF NOT EXISTS speaker_embedding (
    id            TEXT PRIMARY KEY DEFAULT 'primary',
    embedding     BLOB NOT NULL,        -- Resemblyzer d-vector: 256 × f32 = 1024 bytes
    enrolled_at   DATETIME NOT NULL,
    model_version TEXT NOT NULL         -- resemblyzer.onnx SHA-256 hash used for enrollment
);

CREATE TABLE IF NOT EXISTS mood_samples (
    mood         TEXT PRIMARY KEY,      -- one of seven MoodCategory values
    enrolled_at  DATETIME,
    -- V1: these are null but the row exists (created during enrollment)
    e2v_embedding BLOB,                 -- emotion2vec 768-dim f32 embedding, null in V1
    mapping_seed  BLOB                  -- initial mapping layer seed from enrollment, null in V1
);

CREATE TABLE IF NOT EXISTS vocal_mapping (
    -- V1.x: the per-GM linear mapping layer weights (768 → 7)
    id              TEXT PRIMARY KEY DEFAULT 'current',
    weights         BLOB,               -- 768×7 f32 matrix, row-major
    bias            BLOB,               -- 7 f32 bias vector
    trained_at      DATETIME,
    session_count   INTEGER NOT NULL DEFAULT 0,
    correction_count INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS gm_consent (
    gm_id                        TEXT PRIMARY KEY,
    speaker_verification_consent INTEGER NOT NULL CHECK (speaker_verification_consent IN (0,1)),
    vocal_analysis_consent       INTEGER NOT NULL CHECK (vocal_analysis_consent IN (0,1)),
    consent_version              TEXT NOT NULL,  -- version string matching displayed consent text
    consented_at                 DATETIME NOT NULL,
    last_reviewed_at             DATETIME
);
```

### Rust Types (src-tauri/src/state.rs)

```rust
use serde::{Deserialize, Serialize};

/// All seven mood categories. This enum is the canonical reference.
/// Any string representation in DB, IPC, or config must match these exact names.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "PascalCase")]
pub enum MoodCategory {
    Calm,
    Tension,
    Combat,
    Mystery,
    Travel,
    Celebration,
    Sorrow,
}

impl MoodCategory {
    pub const ALL: [MoodCategory; 7] = [
        Self::Calm, Self::Tension, Self::Combat, Self::Mystery,
        Self::Travel, Self::Celebration, Self::Sorrow,
    ];

    /// Hex color for tray icon. These are the canonical values — match in CSS too.
    pub fn tray_color(self) -> &'static str {
        match self {
            Self::Calm        => "#F5A623",
            Self::Tension     => "#546E7A",
            Self::Combat      => "#C0392B",
            Self::Mystery     => "#6C3483",
            Self::Travel      => "#27AE60",
            Self::Celebration => "#F1C40F",
            Self::Sorrow      => "#2980B9",
        }
    }

    /// Which crossfade type to use when transitioning TO self FROM `from`.
    pub fn transition_from(self, from: MoodCategory) -> CrossfadeType {
        use MoodCategory::*;
        matches!(
            (from, self),
            (Calm, Combat) | (Combat, Calm)
            | (Sorrow, Celebration) | (Celebration, Sorrow)
            | (Calm, Tension) | (Tension, Calm)
            | (Combat, Sorrow) | (Sorrow, Combat)
        )
        .then_some(CrossfadeType::MusicalBreath)
        .unwrap_or(CrossfadeType::Standard)
    }

    /// Mood "distance" for logging/analytics. Not used for transition type selection.
    pub fn distance(self, other: MoodCategory) -> u8 {
        // Approximate emotional distance: 0 = same, 1 = adjacent, 2 = opposite
        use MoodCategory::*;
        if self == other { return 0; }
        matches!(
            (self, other),
            (Calm, Combat) | (Combat, Calm)
            | (Sorrow, Celebration) | (Celebration, Sorrow)
        )
        .then_some(2)
        .unwrap_or(1)
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum CrossfadeType {
    Standard,       // symmetric: 2000ms fade-out overlapping with 2000ms fade-in
    MusicalBreath,  // 800ms fade-out → 500ms near-silence → 1500ms fade-in
}

impl CrossfadeType {
    pub fn total_duration_ms(self) -> u32 {
        match self {
            Self::Standard       => 4000,   // overlapping, net = ~2000ms
            Self::MusicalBreath  => 2800,
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct SessionState {
    pub id: String,                         // UUIDv4, set on StartSession
    pub active: bool,
    pub current_mood: Option<MoodCategory>, // None until first track loaded
    pub current_track_id: Option<String>,
    pub current_track_title: Option<String>,
    pub detection_active: bool,
    pub locked: bool,
    pub in_cooldown: bool,
    pub last_auto_shift_at: Option<i64>,    // Unix timestamp milliseconds
    pub detection_mode: DetectionMode,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Default)]
pub enum DetectionMode {
    #[default]
    Full,          // VAD + speaker + Whisper + keywords (+ emotion2vec in V1.x)
    KeywordOnly,   // speaker verification failed to init or consent declined
    Manual,        // detection thread in failed state — hotkeys only
}

/// Channel capacity constants — all bounded channels use these values.
/// Changing these requires performance testing on minimum spec hardware.
pub mod channels {
    pub const AUDIO_FRAME: usize    = 100;  // VAD frame ring buffer (~3s of frames)
    pub const SPEECH_SEGMENT: usize = 10;   // Assembled segments awaiting Whisper
    pub const DETECTION_EVENT: usize = 50;  // Events from FSM to session logger
    pub const AUDIO_CMD: usize      = 20;   // Commands to FMOD engine
    pub const IPC_EVENT: usize      = 50;   // Events to frontend
}

/// ComputeProfile: detected once at startup, never changed during a session.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ComputeProfile {
    Full,       // Whisper medium + emotion2vec (V1.x)
    Standard,   // Whisper medium only
    Minimal,    // Whisper small only
}
```

### TypeScript Types (src/lib/types.ts)

```typescript
// ============================================================
// Canonical TypeScript types — mirrors src-tauri/src/state.rs
// Any change to Rust types must be reflected here and vice versa.
// ============================================================

export const MOODS = ['Calm', 'Tension', 'Combat', 'Mystery', 'Travel', 'Celebration', 'Sorrow'] as const;
export type MoodCategory = typeof MOODS[number];

export const MOOD_COLORS: Record<MoodCategory, string> = {
  Calm:        '#F5A623',
  Tension:     '#546E7A',
  Combat:      '#C0392B',
  Mystery:     '#6C3483',
  Travel:      '#27AE60',
  Celebration: '#F1C40F',
  Sorrow:      '#2980B9',
} as const;

export type CrossfadeType = 'Standard' | 'MusicalBreath';
export type DetectionMode = 'Full' | 'KeywordOnly' | 'Manual';
export type StartupPhase = 'initializing' | 'ui_ready' | 'detection_ready' | 'error';
export type MicPermission = 'granted' | 'denied' | 'unknown';

export interface SessionState {
  id: string;
  active: boolean;
  currentMood: MoodCategory | null;
  currentTrackId: string | null;
  currentTrackTitle: string | null;
  detectionActive: boolean;
  locked: boolean;
  inCooldown: boolean;
  detectionMode: DetectionMode;
}

export interface Track {
  id: string;
  filename: string;
  title: string | null;
  artist: string | null;
  source: 'bundled' | 'byom' | 'commissioned';
  licenseType: string;
  commercialOk: boolean;
  attributionRequired: boolean;
  attributionText: string | null;
  moodBroad: MoodCategory;
  moodSub: string | null;
  genres: string[];            // from track_genres JOIN
  durationSeconds: number | null;
  isBundled: boolean;
  isByom: boolean;
}

export interface GmProfile {
  gmId: string;
  enrolledAt: string;
  sessionCount: number;
  correctionCount: number;
  speakerVerificationConsent: boolean;
  vocalAnalysisConsent: boolean;
}

export interface AppConfig {
  hotkeyNext: string;
  hotkeyShift: string;
  hotkeyLock: string;
  detectionConfidenceThreshold: number;
  detectionCooldownSeconds: number;
  language: 'en' | 'el' | 'both';
  sfxEnabled: boolean;
  whisperModelSize: 'small' | 'medium' | 'large-v3';
  speakerSimilarityThreshold: number;
  audioOutputDevice: string;
  audioInputDevice: string;
}

// ============================================================
// IPC Event Payloads — emitted from Rust via app_handle.emit()
// ============================================================

export interface StartupPhaseChangedPayload {
  phase: StartupPhase;
  micPermission?: MicPermission;
  error?: string;
}

export interface MoodChangedPayload {
  mood: MoodCategory;
  trackId: string | null;
  trackTitle: string | null;
  transitionType: CrossfadeType;
  triggeredBy: 'auto' | 'manual';
  keyword?: string;
}

export interface TrackChangedPayload {
  trackId: string;
  trackTitle: string | null;
  mood: MoodCategory;
}

export interface SessionStateChangedPayload {
  state: SessionState;
}

export interface DetectionModeChangedPayload {
  mode: DetectionMode;
  reason: string;         // human-readable explanation for status display
}

export interface DetectionErrorPayload {
  code: string;
  message: string;
  recoverable: boolean;
}

export interface ImportProgressPayload {
  processed: number;
  total: number;
  currentFile: string;
}

export interface EnrollmentProgressPayload {
  mood: MoodCategory;
  step: number;
  total: 7;
}
```

### Keyword Vocabulary File Format (content/keywords/*.json)

The `validate-keywords` binary validates this schema. Files not matching it
fail CI.

```json
{
  "language": "en",
  "genre": "fantasy",
  "version": "1.0.0",
  "created_at": "2025-01-01",
  "description": "English fantasy keyword vocabulary — launch vocabulary",
  "triggers": [
    {
      "id": "en_fantasy_combat_initiative",
      "phrases": [
        "roll for initiative",
        "roll initiative",
        "initiative"
      ],
      "trigger_type": "mood_shift",
      "target_mood": "Combat",
      "base_confidence": 0.90,
      "notes": "High confidence — unambiguous combat start signal"
    },
    {
      "id": "en_fantasy_sfx_sword",
      "phrases": [
        "sword strikes",
        "blade connects",
        "steel rings out"
      ],
      "trigger_type": "sfx_trigger",
      "sfx_id": "sfx_sword_clash_01",
      "base_confidence": 0.85,
      "notes": "SFX threshold is higher than mood threshold — false SFX is jarring"
    }
  ]
}
```

### Audio Mood Classifier Rules (tools/audio_tagger/classifier.py)

**Implement exactly these rules. No ML. No additional features.**
Heuristic thresholds may be adjusted only with documented test data justification.

```python
from dataclasses import dataclass
from typing import Literal

MoodStr = Literal['Calm', 'Tension', 'Combat', 'Mystery', 'Travel', 'Celebration', 'Sorrow']

@dataclass
class AudioFeatures:
    bpm: float               # tempo in BPM
    energy: float            # RMS energy, normalized 0.0–1.0
    key: str                 # e.g. 'G major', 'D minor', 'C# minor'
    spectral_centroid: float # Hz, typical range 500–5000

def classify_mood(f: AudioFeatures) -> MoodStr:
    """
    Rule-based broad mood classifier.
    Achieves ~80% accuracy on labeled test set — acceptable for review-and-confirm UX.
    Do not add ML here. The review step exists precisely because this isn't perfect.
    """
    is_minor = 'minor' in f.key.lower()
    is_major = not is_minor

    high_energy  = f.energy > 0.65
    medium_energy = 0.35 <= f.energy <= 0.65
    low_energy   = f.energy < 0.35

    fast_tempo   = f.bpm > 120
    medium_tempo = 80 <= f.bpm <= 120
    slow_tempo   = f.bpm < 80

    bright = f.spectral_centroid > 2500   # Hz
    mid    = 1200 <= f.spectral_centroid <= 2500
    dark   = f.spectral_centroid < 1200

    # Ordered by specificity — most specific first
    if high_energy and fast_tempo:
        return 'Combat'

    if high_energy and is_major and bright:
        return 'Celebration'

    if low_energy and slow_tempo and is_minor and dark:
        return 'Sorrow'

    if low_energy and is_minor and not slow_tempo:
        return 'Tension'

    if low_energy and slow_tempo and bright and is_minor:
        return 'Mystery'       # eerie brightness in slow minor

    if low_energy and slow_tempo and is_major:
        return 'Calm'

    # Medium energy / medium tempo — assume journey/montage
    return 'Travel'
```

### Voice Training Passage Requirements

7 passages per language. Committed to `content/passages/{lang}/{mood}.md`.

Each passage must:
- Be 130–150 words (approximately 60 seconds at natural narration pace)
- Feel like authentic GM narration for its mood — evocative, not a phonetics exercise
- Pull genuine vocal delivery: a flat, emotionless reading produces a bad speaker profile
- Include sentence length variation: Combat uses short, punchy sentences;
  Calm uses longer, flowing ones — this is what creates the vocal differentiation
- NOT contain explicit trigger keyword phrases (enrollment passages are not for
  training the keyword engine; they are for voice profiling)
- Greek passages: written natively from Greek gaming table experience,
  not translated from English. The rhythm and idiom must feel natural to
  a Greek GM narrating in Greek.

---

## 14. Detection FSM — Complete Specification

**Implement this exactly. Every state. Every transition. Every timeout.**

### States

```
IDLE           — Default. Detection running but no active speech segment.
LISTENING      — VAD has detected speech. Accumulating segment. Awaiting speaker check.
TRANSCRIBING   — Speaker verified. Segment forwarded to Whisper. Awaiting result.
TRIGGER        — Keyword matched with sufficient confidence. Executing mood/SFX action.
LOCKED         — Manual hold active. Detection suspended. Entered from any state.
```

### Transitions

```
IDLE
  ── VAD: speech_start ──────────────────────────────────────────► LISTENING
  ── hotkey_lock ────────────────────────────────────────────────► LOCKED

LISTENING
  ── speaker_verify: reject ─────────────────────────────────────► IDLE
     [log: SpeakerRejected]
  ── segment < 0.5s AND VAD silence ────────────────────────────► IDLE
     [log: nothing — too short to be meaningful]
  ── segment OK AND VAD silence ─────────────────────────────────► TRANSCRIBING
     [forward segment to Whisper]
  ── TIMEOUT: 30 seconds in LISTENING ──────────────────────────► IDLE
     [log: Timeout { state: "Listening" }]
     [reason: speaker verification stalled or segment never ended]
  ── hotkey_lock ────────────────────────────────────────────────► LOCKED
     [in-flight segment discarded]

TRANSCRIBING
  ── whisper_complete: no match ─────────────────────────────────► IDLE
     [log: LowConfidence if confidence exists, else NoMatch]
  ── whisper_complete: match, confidence < threshold ────────────► IDLE
     [log: LowConfidence { keyword, confidence }]
  ── whisper_complete: match, confidence ≥ threshold,
     cooldown active ──────────────────────────────────────────► IDLE
     [log: CooldownActive { keyword, mood }]
  ── whisper_complete: match, confidence ≥ threshold,
     cooldown inactive ────────────────────────────────────────► TRIGGER
     [log: nothing yet — log in TRIGGER]
  ── TIMEOUT: 15 seconds in TRANSCRIBING ───────────────────────► IDLE
     [log: Timeout { state: "Transcribing" }]
     [reason: Whisper inference stalled]
     [action: the spawn_blocking task for Whisper is NOT cancelled — it continues
      and its result is discarded if FSM has moved on]
  ── hotkey_lock ────────────────────────────────────────────────► LOCKED
     [in-flight Whisper task continues and result discarded]

TRIGGER
  ── mood_shift_applied ─────────────────────────────────────────► IDLE
     [log: DetectionEvent { type: "auto", triggered_mood, keyword, confidence }]
     [action: start cooldown timer (30s default)]
     [action: start implicit_positive timer (60s) — if no correction
      arrives within 60s, set implicit_positive = true on this event]
  ── sfx_applied ────────────────────────────────────────────────► IDLE
     [log: DetectionEvent { type: "auto", sfx_id, keyword, confidence }]
  ── hotkey_lock (during brief TRIGGER processing) ─────────────► LOCKED
     [mood shift still applied — lock takes effect for subsequent detections]

LOCKED (overlaid — can be entered from any state via hotkey_lock)
  ── hotkey_lock (toggle off) ───────────────────────────────────► IDLE
     [NOT to previous state — always to IDLE]
     [reason: in-flight state at lock time is undefined and stale;
      IDLE is always safe; if GM unlocks mid-session, a clean slate is correct]
  ── VAD, speaker, Whisper, keywords continue running in LOCKED state
     [reason: collecting data while locked, especially vocal embeddings]
     [all TRIGGER transitions are intercepted → log Suppressed, go to IDLE]
  ── hotkey_next: functions normally (skip track in current mood)
  ── hotkey_shift: functions normally (open mood picker, manual set)
  ── NO timeout in LOCKED state — can stay locked indefinitely
```

### Correction Events (Manual Hotkey Shifts)

When the GM uses hotkey_shift to correct the mood (not while LOCKED — that's
just navigation):

```
GM presses hotkey_shift
  → Detection FSM NOT involved (FSM is in IDLE during this)
  → Open mood picker in popover
  → GM selects mood
  → Call set_mood(target_mood)
  → FMOD crossfade (transition type per MoodCategory::transition_from())
  → Log: DetectionEvent {
        type: "correction",
        detected_mood: current_mood_before_correction,
        corrected_mood: target_mood,
        whisper_transcript: last_transcript (if within 30s, else null),
        keyword_matched: last_keyword (if within 30s, else null),
        vocal_embedding: current_embedding (if V1.x active, else null)
    }
  → Update SessionState.current_mood
  → Reset cooldown timer (correction counts as a mood event)
  → Emit mood-changed IPC event
```

### Cooldown Timer

- Duration: 30 seconds (configurable 10s–120s via config table)
- Starts: after any auto mood shift (TRIGGER → IDLE transition)
- Resets: on any correction (GM manual shift)
- During cooldown: `in_cooldown = true` in SessionState; IPC event emitted
- Logging: if keyword fires during cooldown, log CooldownActive event (training data)
- **Cooldown does not apply to manual hotkeys** — the GM can always override

### Implicit Positive Timer

After every AUTO detection event (type: "auto"):
- Start a 60-second timer associated with that event's ID
- If a CORRECTION event fires within 60s → the auto event was wrong → timer cancelled
- If 60s passes with no correction → set `implicit_positive = true` on the event
- This provides positive training labels for the vocal classifier without requiring
  explicit GM confirmation

---

## 15. IPC & Event Contracts

### Tauri Commands (src-tauri/src/commands/)

All use `#[tauri::command]`. Return `Result<T, String>` where `String` is a
user-facing error message in English. All registered in `commands/mod.rs`.

```rust
// session.rs
start_session(gm_id: String, genre: String) -> Result<SessionState, String>
end_session(session_id: String) -> Result<(), String>
set_mood(session_id: String, mood: MoodCategory) -> Result<(), String>
next_track(session_id: String) -> Result<(), String>
toggle_lock(session_id: String) -> Result<bool, String>  // returns new locked state

// library.rs
import_tracks(file_paths: Vec<String>) -> Result<Vec<Track>, String>
get_tracks(filter: TrackFilter) -> Result<Vec<Track>, String>
get_tracks_by_genre(genre: String, mood: Option<MoodCategory>) -> Result<Vec<Track>, String>
update_track_mood(track_id: String, mood: MoodCategory) -> Result<(), String>
remove_byom_track(track_id: String) -> Result<(), String>
get_attribution_report(session_id: String) -> Result<String, String>
preview_track(track_id: String) -> Result<(), String>
stop_preview() -> Result<(), String>

// profile.rs
get_profile(gm_id: String) -> Result<Option<GmProfile>, String>
start_enrollment(gm_id: String, consent: ConsentPayload) -> Result<(), String>
record_passage(gm_id: String, mood: MoodCategory) -> Result<EnrollmentResult, String>
complete_enrollment(gm_id: String) -> Result<GmProfile, String>
cancel_enrollment(gm_id: String) -> Result<(), String>
delete_profile(gm_id: String) -> Result<(), String>
retrain_speaker(gm_id: String) -> Result<(), String>

// config.rs
get_config() -> Result<AppConfig, String>
update_config(key: String, value: String) -> Result<(), String>
reset_config() -> Result<AppConfig, String>
get_audio_input_devices() -> Result<Vec<AudioDevice>, String>
get_audio_output_devices() -> Result<Vec<AudioDevice>, String>
check_mic_permission() -> Result<MicPermission, String>
request_mic_permission() -> Result<MicPermission, String>
get_compute_profile() -> Result<String, String>
```

### Tauri Events (Rust → Frontend)

**Use `app_handle.emit(event_name, &payload)` — Tauri 2.x API.**
`emit_all()` is removed in Tauri 2.x. Never use it.

```
"startup-phase-changed"      StartupPhaseChangedPayload
"session-state-changed"      SessionStateChangedPayload
"mood-changed"               MoodChangedPayload
"track-changed"              TrackChangedPayload
"detection-mode-changed"     DetectionModeChangedPayload
"detection-error"            DetectionErrorPayload
"enrollment-progress"        EnrollmentProgressPayload
"import-progress"            ImportProgressPayload
"vocabulary-reloaded"        { language: string, genre: string, trigger_count: number }
```

### Frontend Event Subscription (src/lib/events.ts)

```typescript
// All event subscriptions go through this file — never listen directly in components.
import { listen } from '@tauri-apps/api/event';
import type { UnlistenFn } from '@tauri-apps/api/event';

export async function onStartupPhaseChanged(
  cb: (payload: StartupPhaseChangedPayload) => void
): Promise<UnlistenFn> {
  return listen<StartupPhaseChangedPayload>('startup-phase-changed', e => cb(e.payload));
}

export async function onMoodChanged(
  cb: (payload: MoodChangedPayload) => void
): Promise<UnlistenFn> {
  return listen<MoodChangedPayload>('mood-changed', e => cb(e.payload));
}

// ... one function per event — same pattern throughout
```

---

## 16. Coding Conventions

### Rust

**Error handling:**
```rust
// Library errors (types that other modules use): thiserror
// Application/command errors: anyhow for context, surface as String to IPC

// NEVER in production code:
some_result.unwrap()
some_result.expect("message")
panic!("message")
todo!()
unimplemented!()

// ALWAYS:
some_result?                                    // propagate
some_result.map_err(|e| e.to_string())?        // at IPC boundary
some_result.unwrap_or_default()                // explicit fallback
```

**Async discipline:**
```rust
// CPU-bound work runs on spawn_blocking — never block the tokio runtime directly
let result = tokio::task::spawn_blocking(move || {
    // Whisper inference, ORT inference, heavy processing
    heavy_computation(data)
}).await??;

// I/O is always async — no std::fs in async context
tokio::fs::read_to_string(path).await?;
```

**Logging — structured with context:**
```rust
// Not: tracing::info!("mood changed to {:?}", mood);
// Yes:
tracing::info!(
    session_id = %session.id,
    mood = ?new_mood,
    triggered_by = "auto",
    keyword = %matched_keyword,
    confidence = confidence,
    "mood shift triggered"
);
```

**State — immutable updates:**
```rust
// SessionState is cloned on modification, not mutated behind a mutex on hot paths
// Changes propagate through channels, not shared references
// Arc<RwLock<SessionState>> for the authoritative copy in AppState
// Local copies on each thread for read-only access during processing
```

**Module visibility:**
```rust
// pub(crate) for anything used only within src-tauri
// pub only for items exposed to the Tauri command layer or integration tests
// All internal implementation details: private (no pub)
```

### TypeScript/Svelte

**Strict types — no escape hatches:**
```typescript
// tsconfig: "strict": true, "noImplicitAny": true
// Never: let x: any; let x as any; @ts-ignore
// Always: proper typing or explicit unknown with type guards
```

**Store discipline — single writer:**
```typescript
// Each store has ONE module responsible for writing to it
// sessionStore: updated only by events.ts event handlers
// Components only read stores, never write them
// Use derived() for computed values — never compute in component templates
```

**Component rules:**
```typescript
// Never call invoke() directly in a .svelte file
// Always: import { startSession } from '$lib/invoke'; then call startSession()
// Reason: typed wrappers catch API changes at compile time; raw invoke() does not
```

**Styling:**
```typescript
// Tailwind utility classes only — no inline styles, no <style> blocks except
// for Tailwind @apply directives
// Color values from MOOD_COLORS constant — never hardcode hex in templates
// All CSS custom properties for dynamic values (tray color updates, etc.)
```

### Python (Tools Only)

```python
# Type hints on every function signature
# mypy --strict must pass

# Sidecars are gone — but this applies to tools:
# Logging: use standard logging module, configure at main() entry
# Errors: never let unhandled exceptions crash silently
# All errors produce a non-zero exit code and a descriptive stderr message
```

### General Conventions

**Naming:**
```
Rust:        snake_case functions/variables, PascalCase types, SCREAMING_SNAKE_CASE constants
TypeScript:  camelCase functions/variables, PascalCase types/interfaces, SCREAMING_SNAKE_CASE constants
Svelte:      PascalCase component files, camelCase events
SQL:         snake_case table and column names
Files:       snake_case Rust, kebab-case TypeScript/Svelte
```

**No magic numbers — ever:**
```rust
// Not: if confidence > 0.75 {
// Yes:
const DEFAULT_CONFIDENCE_THRESHOLD: f32 = 0.75; // config override available
if confidence > config.detection_confidence_threshold {
```

**Documentation:**
```rust
/// All public functions have doc comments.
/// Detection pipeline functions: full docs including failure modes.
/// One-liner acceptable for trivial getters.
/// Include example inputs/outputs for non-obvious functions.
```

**Commit format:**
```
<module>: <imperative description>

Examples:
  detection: add implicit positive timer for uncorrected auto events
  audio: fix MusicalBreath silence dip clipping on macOS CoreML
  db: add schema_migrations table to 001_initial migration
  m4: enforce bounded channel capacity on audio frame ring buffer
```

---

## 17. Testing Strategy

### Principles

- **SilentEngine is the only audio backend in automated tests.** Zero tests require
  audio hardware. A test that fails on CI because it needs speakers is a broken test.
- **Real-world session tests cannot be automated.** They are documented in
  `tests/fixtures/real_world_log.md` and are gating criteria for phase transitions.
- **Integration tests use real SQLite (in-memory).** No mocking the database.
- **ORT models in CI use stub files** (created by `scripts/download_models.sh --ci-stub`).
  Tests that require model inference use pre-computed fixture outputs.

### Unit Tests (`#[cfg(test)]` in each module)

| Module | What to Test |
|--------|--------------|
| `detection/keyword.rs` | Fuzzy match on 50 positive + 50 negative cases from fixture |
| `detection/fsm.rs` | Every state transition including all timeout paths |
| `audio/crossfade.rs` | All 42 MoodCategory→MoodCategory transition type selections |
| `audio/normalizer.rs` | RMS normalization output level, clamp behavior |
| `audio/resampler.rs` | Output sample count correct for various input rates and lengths |
| `detection/fsm.rs` | LOCKED exits to IDLE unconditionally (not to previous state) |
| `detection/fsm.rs` | Cooldown prevents re-trigger; manual hotkey bypasses cooldown |
| `db/tracks.rs` | genre filter uses JOIN, not LIKE or JSON parsing |
| `db/mod.rs` | Migration idempotency: run 3× in sequence, no errors |
| `state.rs` | MoodCategory::transition_from covers all 42 mood pairs |
| `profile/consent.rs` | Consent flag enforcement: false → ORT session not created |

### Integration Tests (tests/integration/)

```rust
// pipeline_test.rs
// Feeds pre-recorded audio fixtures through the full pipeline using fixture outputs
// for ORT models. Tests the wiring, not the model accuracy.

// fsm_test.rs
// Drives the FSM through complete session scenarios using mock channels.
// Tests: happy path, LOCKED behavior, timeout recovery, supervisor restart.

// keyword_engine_test.rs
// Tests fuzzy matching with simulated Whisper transcription errors
// (common substitutions: "initiative" → "initiative" OK, "in it at ive" → should match)

// audio_engine_test.rs
// Tests SilentEngine call recording: correct crossfade type selected,
// correct transition fired for each mood pair.
```

### Real-World Session Tests (Required for Gate G1)

All results documented in `tests/fixtures/real_world_log.md` with date, hardware,
conditions, and raw numbers.

| Test | Conditions | Metric to Record |
|------|-----------|-----------------|
| Solo baseline | Petros narrates 60 min alone | Detection rate, false positive rate |
| Group session | 4 people, GM narrates full session | Detection rate, speaker rejection rate, false positive rate |
| Greek session | Full session Greek + English code-switching | Greek keyword detection rate |
| Blind session | 90 min, no manual interventions | Auto-transition count, rate correct / incorrect by post-session review |
| Quiet narrator | GM with naturally soft voice | Does normalization suffice, or is VAD threshold too aggressive? |

---

## 18. Performance Budgets

**Blocking defects if missed. Not aspirational targets.**

| Metric | Budget | Failure Threshold | Measurement |
|--------|--------|-------------------|-------------|
| Phase 1 startup (UI ready) | ≤ 3s | > 5s | Stopwatch, 5 trials on 2020-era hardware |
| Phase 2 startup (detection ready) | ≤ 15s | > 25s | Same hardware |
| Keyword detection rate (noisy) | ≥ 75% | < 65% | Real-world group session test |
| Keyword detection rate (baseline) | ≥ 85% | < 75% | Real-world solo test |
| False positive rate | ≤ 1 / 10 min | > 3 / 10 min | Real-world group session, 60 min |
| Greek keyword detection rate | ≥ 60% | < 50% | Real-world Greek session |
| Speaker verification precision | ≥ 80% | < 70% | 4-person group session |
| FMOD trigger → audible latency | ≤ 200ms | > 500ms | FMOD callback timing, 20 samples |
| SFX trigger latency | ≤ 500ms | > 1s | End-to-end, measured at speaker output |
| Session RAM at 1 hour | ≤ 3.5 GB | > 5 GB | Activity Monitor / Task Manager |
| Session RAM growth/hour | ≤ 25 MB | > 100 MB | 4-hour integration test, delta |
| Crossfade imperceptibility | 10/10 non-musicians | < 8/10 | Informal listen test |
| BYOM 100-track import | ≤ 5 min | > 15 min | Stopwatch, 3 trials, take max |
| DB query p99 | ≤ 50ms | > 200ms | Timing wrapper in DEBUG build |
| Vocabulary reload | ≤ 1s between file change and active | > 5s | notify trigger → keyword active |
| Idle RAM (no session) | ≤ 150 MB | > 300 MB | After Phase 2 startup complete |

---

## 19. Phase Gates

**Hard stops. All items must be checked before proceeding.**

### Gate G0 — Spike Completion

All results documented in `tests/fixtures/real_world_log.md`.

- [ ] S1: Tauri shell pass criteria met; platform deltas documented
- [ ] S2: Whisper accuracy measured under all test conditions; raw numbers recorded;
       decision on buffer size, model size, and vocabulary size documented
- [ ] S3: FMOD crossfade quality confirmed; license terms confirmed; OR
       fallback decision documented with timeline impact
- [ ] S4: ONNX stack resource profile measured on minimum spec hardware;
       emotion2vec decision (activate on schedule vs defer) documented
- [ ] Decision matrix applied: all framework/library choices confirmed or revised
- [ ] `CLAUDE.md` updated if any locked decisions changed as a result of spikes

### Gate G1 — MVP Complete

- [ ] All module acceptance criteria satisfied (M1–M7), each checked off
- [ ] All performance budgets met (§18), each measured and recorded
- [ ] All real-world session tests completed, results in `real_world_log.md`
- [ ] Constraints P1–P4 verified: code review signed off by checklist
  - P1: no `File::create` in audio path (grep + CI lint)
  - P2: no HTTP client imports in audio/ or ml/ modules (cargo deny)
  - P3: emotion2vec ORT session creation gated by consent flag check in Rust (code review)
  - P4: consent record existence checked before session start (unit test)
- [ ] Constraint D3 verified: both AUTO and CORRECTION events logged across all code paths
- [ ] `schema_migrations` table present; all 3 migrations recorded correctly
- [ ] Attribution metadata complete for all bundled tracks (100% of rows checked)
- [ ] BYOM import, voice enrollment, and profile delete tested end-to-end:
       Windows 10/11 and macOS 12+ Intel, macOS 14+ ARM
- [ ] Mac notarization: `.dmg` passes Gatekeeper on a clean Mac with no developer tools
- [ ] Windows signing: `.msi` installs without SmartScreen warning on clean Windows 10/11
- [ ] Auto-update mechanism tested: simulate a version bump, verify silent delivery
- [ ] itch.io page: screenshots, session demo GIF, attribution, free/Pro feature list
- [ ] GDPR DPIA drafted (internal) — external legal review scheduled (pre-EU launch)
- [ ] Privacy policy published at a stable URL, linked from About screen

### Gate G1.x — Vocal Classifier Ready

- [ ] Gate G1 complete
- [ ] ≥ 5 real sessions logged per early-access GM, `detection_events` table populated
- [ ] Spike S4 passed OR hardware-gated activation plan documented
- [ ] emotion2vec ONNX export validated (SHA-256 of model file matches expected)
- [ ] Per-GM mapping layer: background training completes in ≤ 30s on minimum hardware
- [ ] Dual-signal fusion: detection rate improves over keyword-only baseline (measured)
- [ ] Implicit positive labels: verified correctly set after 60s uncorrected auto events
- [ ] Consent check: emotion2vec activated only for GMs with vocal_analysis_consent = true

---

## 20. Open Questions Register

Track here. Raise a GitHub issue before acting on any of these.
Do not make unilateral decisions on open questions.

| ID | Question | Context | Priority |
|----|----------|---------|----------|
| OQ-01 | Whisper model update strategy | New models release regularly. In-app download? Manual? Update manifest? | Medium |
| OQ-02 | Character voice handling | Heavy NPC voices (pitch shift, accents) may confuse Resemblyzer. What's the right similarity threshold default and how do we tune it? | High — test in S2 |
| OQ-03 | Multi-GM session profiles | Architecture supports it (profile per gm_id) but the UX flow is undefined. Out of MVP scope — do not add logic, just don't preclude it. | Low |
| OQ-04 | Bluetooth audio latency | BT adds 100–300ms to audio output. Does this break the "immediate" crossfade promise? Document as limitation or handle? | Medium |
| OQ-05 | Detection event retention policy | `detection_events` grows unboundedly. Cap at N events per session? Purge sessions older than N days? Needs a decision before V1.x training begins. | High — decide before G1.x |
| OQ-06 | App name / branding | Placeholder `ttrpg-companion` throughout. Itch.io slug, bundle ID, and marketing need a real name. | High — before any public release |
| OQ-07 | Whisper large-v3 for Greek | If S2 shows medium insufficient for Greek code-switching, large-v3 requires ~6 GB RAM, exceeding 8 GB minimum target. Decision may require raising minimum spec. | High — depends on S2 |
| OQ-08 | GDPR DPIA scope | External legal review needed before launch for EU users (Petros is EU-based — all users are EU users for GDPR purposes). Budget €500–1,500. Find AI Act / GDPR specialist, not a generalist. | Critical — blocks EU launch |
| OQ-09 | Vocabulary hot-reload during session | Currently: reload between sessions. Should reload be allowed mid-session for power users? Risk: inconsistent detection state. | Low |
| OQ-10 | emotion2vec model download UX | ~350 MB download. Progress UI? Resume on failure? SHA-256 verification? Delta updates for model revisions? | Medium — design before G1.x |

---

*CLAUDE.md v2 — Synchronized with MVP Strategy v3.*
*Incorporates: ONNX-native ML stack, Vite+Svelte (not SvelteKit), track_genres*
*junction table, schema_migrations table, bounded channels, two-phase startup,*
*FSM timeout transitions + LOCKED→IDLE invariant, full audio pipeline with*
*resampling/normalization/buffering, Tauri 2.x emit() API, Mac entitlements,*
*Windows mic permission detection, CI/CD pipeline, dual event logging (auto +*
*corrections), vocabulary hot-reload, GDPR-appropriate encryption framing.*
*Version: 2.0*
