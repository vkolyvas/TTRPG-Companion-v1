# TTRPG Companion App — Strategic Development Summary v3

---

## PRODUCT CONCEPT

A real-time audio companion app for tabletop RPG sessions. It runs in the background on the GM's laptop — the laptop already at the table — and listens to the GM narrating scenes. It automatically matches ambient music and sound effects to the current moment through a combination of keyword detection and vocal mood analysis. GMs can also supply their own music library.

The product is an **audio intelligence layer** — not a music library, not a VTT, not a prep tool. It takes what's happening at the table and makes it sound right, in real time, without the GM having to stop narrating.

**Platform:** Desktop application (Windows + Mac). The laptop-at-the-table is already the norm for most GMs — running PDFs, notes, or reference material. This app activates hardware that's already present and idle.

---

## DESIGN MANTRA

**"The GM is narrating, not operating software."**

Every feature, every control, every screen is evaluated against this single principle. If it requires the GM to stop talking to their players and think about the app, it's wrong. If it can be done with a keystroke while mid-sentence without switching windows, it's right.

---

## MVP DEFINITION — LOCKED

The MVP is the audio intelligence core. Nothing more. Everything else is expansion.

**The MVP is:**
- Mood-aware music playback that responds to what's happening at the table
- Keyword detection (Signal 1) for automatic track/mood shifting via on-device Whisper
- Vocal mood analysis (Signal 2) via emotion2vec foundation model — ships as speaker verification in V1, vocal classifier in fast follow (see Dual-Signal Architecture)
- Smart crossfading between tracks
- Sound effects triggered by keywords (toggleable)
- Voice training as core onboarding: GM reads mood-specific passages, app builds speaker profile and vocal mood baseline
- Bring-your-own-music (BYOM) support with automatic broad mood tagging via audio analysis
- A bundled starter library (CC-licensed + commissioned originals), curated via audio analysis tool
- Fantasy genre first
- English and Greek language support at launch
- Modular, language-agnostic keyword architecture designed for future localization
- Three global hotkeys for session control (next, shift, hold/lock)
- **Vocal delivery threshold detection during voice training** — determines whether the GM's narration style carries enough dynamic range for autonomous mood detection
- **Mode A (Autonomous):** App drives mood transitions automatically. GM corrects with hotkeys. Default for GMs who pass the vocal threshold.
- **Mode B (Collaborative):** App suggests moods via a persistent mood palette; GM confirms with a single keypress or numpad key. Default for GMs below the vocal threshold, or any GM who prefers manual control.
- System tray presence with mood-color indicator — no visible window during play

**The MVP is NOT:**
- AI-generated visuals
- Battlemaps
- Adventure module upload
- Session zero ritual (beyond voice training + genre selection)
- Campaign memory
- Player-facing access
- Nuanced subcategory mood tags (Pro tier, post-validation)
- Tablet or mobile version (Phase 2)

**The MVP success test:** Does it create the moment where a GM tells another GM "you need to try this thing"? That moment comes from the music shifting on its own and feeling right. Everything else is enhancement.

---

## PRODUCT IDENTITY & POSITIONING

- **Target audience:** Kitchen table GMs who already have a laptop at the table — running PDFs, notes, D&D Beyond — and want low-friction atmosphere enhancement without investing in additional hardware
- **Market gap:** VTT-invested GMs are already catered to; physical table GMs are not. The laptop-at-the-table GM is the underserved segment with the lowest adoption barrier.
- **Core value proposition:** "You're not bringing a new device to the table. You're activating the one that's already there." One more app on the laptop, running invisibly in the background.
- **Product type:** GM-facing desktop background app, running on the existing laptop at the table
- **Form factor:** System tray icon (Windows) / menu bar icon (Mac) during sessions. Pre-session setup in a full window. During play, the app is essentially invisible — the GM never switches away from their module PDF.
- **Trust philosophy:** "Your table stays your table" — all audio and voice processing happens locally on-device. Voice data is never transmitted. Vocal profiles are stored encrypted on the local machine only.
- **Tablet vision (Phase 2):** The tablet becomes the premium, purpose-built companion experience once the desktop product is validated. Touch interface, ambient color wash, dedicated form factor. The desktop version is the product; the tablet version is the aspiration.

---

## ARCHITECTURE & TECHNOLOGY

### Platform & Framework
- **Target:** Windows + Mac desktop, any laptop from approximately 2020 onward
- **Framework:** Tauri (Rust-based, lightweight, native binaries, web frontend for setup UI, much smaller than Electron)
- **Distribution:** itch.io (primary — TTRPG community already lives here), direct download from website, potentially Steam
- **No App Store dependency:** Direct distribution eliminates review friction and platform fees

### Core Engine Stack
- **Speech-to-text:** whisper.cpp with medium model (769M parameters, ~1.5 GB GGML, ~2.6 GB RAM). Runs comfortably on any target laptop. Medium model selected specifically for Greek language accuracy — smaller models show noticeable degradation for Greek.
- **Audio engine:** FMOD (industry standard game audio, free for indie under revenue threshold) or platform-native (Core Audio on Mac, WASAPI on Windows). Must support gapless crossfading, ambient loop layering, one-shot SFX on top, volume ducking, and responsive transitions.
- **Speaker verification:** pyannote.audio or Resemblyzer — lightweight speaker embedding models for GM voice identification.
- **Vocal mood analysis:** emotion2vec+ base (~90M parameters) as frozen backbone, with lightweight per-GM mapping layer. (See Dual-Signal Architecture for phased implementation.)
- **Audio analysis:** librosa for feature extraction (tempo, energy, key, spectral profile) — powers automatic mood tagging of both bundled and BYOM content.

### Detection Pipeline
Audio capture → Voice Activity Detection (is anyone speaking?) → Speaker Verification (is it the GM?) → **Two parallel paths:**
- **Signal 1 (Keywords):** Whisper transcription → keyword matching → mood trigger
- **Signal 2 (Vocal mood):** emotion2vec embeddings → per-GM mapping layer → mood classification

Both signals weighted and combined. Keyword signal is primary in V1; vocal signal weight increases as per-GM profile matures through session corrections.

### Hardware Self-Regulation
The system detects available hardware and adjusts automatically. Powerful desktop: runs full dual-signal pipeline. Older laptop: disables vocal classifier, falls back to keyword-only detection. Future constrained tablet: runs speaker verification only with keyword detection. Graceful degradation, not feature fragmentation.

---

## DUAL-SIGNAL DETECTION ARCHITECTURE

### The Core Innovation
The product doesn't just listen to what the GM says — it listens to how they say it. Two independent signal layers combine to make smarter mood decisions than either alone.

**Signal 1 — Keywords (what the GM says):** Whisper transcribes audio, keyword engine spots trigger words and phrases mapped to mood categories. Language-dependent. This is the system from the original design.

**Signal 2 — Vocal characteristics (how the GM sounds):** Prosodic analysis classifies the GM's vocal delivery — pitch, pace, volume dynamics, energy, spectral profile — into mood categories. Language-agnostic. This is the new layer enabled by voice training.

**Combined signal example:** Keywords detect "tavern" → probably calm. But the GM's vocal characteristics match their trained "tension" profile → this tavern isn't safe. The combined signal produces a more accurate mood classification than either input alone.

### Phased Implementation — Path B (LOCKED)

**V1 (MVP launch):**
- Speaker verification solves "who is speaking" — filters player noise, reduces false positives, lowers Whisper compute load
- Voice training session collects all seven mood samples, stores in data structure ready for classifier
- Every session silently logs vocal features (emotion2vec embeddings) alongside keyword-selected moods and GM manual corrections
- Building labeled dataset from real sessions, real GMs, real narration — not scripted training passages
- Detection relies on keywords (Signal 1) with speaker verification as a quality gate

**V1.x (fast follow):**
- Vocal mood classifier ships when accumulated session data supports it
- emotion2vec base as frozen backbone, lightweight per-GM mapping layer trained on accumulated real-session data
- Dual-signal weighting activated — both keywords and vocal mood contribute to detection
- Confidence weighting between signals shifts dynamically as vocal profile matures

### emotion2vec Foundation Model

emotion2vec+ is an open-source universal speech emotion representation model. Selected as the foundation for Signal 2 instead of building from scratch.

**Key properties:**
- Three sizes: seed (tiny), base (~90M params), large (~300M params). Base selected for balance of quality and compute.
- Nine emotion categories: angry, disgusted, fearful, happy, neutral, sad, surprised, other, unknown
- Language-agnostic by design: trained across Mandarin, French, German, Italian, and others. Works for English and Greek out of the box.
- Outputs 768-dimensional embedding capturing emotional texture of speech
- State-of-the-art performance across multiple languages and recording conditions

**Mapping to seven moods:**

| emotion2vec output | Maps to mood(s) | Notes |
|---|---|---|
| Angry | Combat / Tension | Context from keywords disambiguates |
| Fearful | Tension | Strong signal for dread, suspense |
| Sad | Sorrow | Direct mapping |
| Happy | Celebration | Direct mapping |
| Neutral | Calm / Travel | Default resting state |
| Surprised | Mystery / Combat | Context-dependent |
| Disgusted | Tension / Sorrow | Weaker signal, context-dependent |

**Architecture:** emotion2vec base runs as frozen backbone. A lightweight per-GM mapping layer sits on top, trained from emotion2vec embeddings to the GM's seven mood categories. Not training the 90M param model — fine-tuning a thin personalization layer with seven labeled samples per GM as starting point, then accumulating real session data.

**Caveat:** All speech emotion models are trained on acted emotion in clean conditions. GM narration at a table with ambient noise, character voices, and performative-but-not-acted delivery is a different domain. The model gives a massive head start but won't be perfect without adaptation — exactly why voice training, real-session data collection, and continuous learning matter.

### Continuous Learning (LOCKED)

The vocal classifier improves per-GM, session by session, through correction-based learning.

**Training data generation during live sessions:**
- GM does nothing after mood selection → implicit positive label (system was correct)
- GM hits "shift" to correct mood → explicit label ("I was speaking like THIS, mood should be THAT")
- GM hits "next" within same mood → weaker signal (mood right, track wrong — not a vocal training event)
- Each session produces dozens of labeled data points from natural gameplay

**Post-session background update:**
- Lightweight mapping head retrains on accumulated data (NOT emotion2vec foundation — that stays frozen)
- New labeled samples added to accumulated profile
- Quick retrain of small neural network — seconds on any laptop
- Older data weighted down gradually, never discarded entirely
- Reset option available if GM style changes or profile becomes corrupted

**Product story:** First session is rough — the system knows the GM only from a 60-second voice training sample per mood. By session 3–4, it has heard "something terrible is about to happen" dozens of times, and corrections have taught it exactly what that sounds like. By session 10, it rarely gets the mood wrong. This is non-transferable value — switching to a competitor means starting over with a system that doesn't know your voice.

**Tier differentiation reinforced:**
- Free tier: keyword detection only, no vocal learning, static accuracy
- Pro tier: vocal classifier that improves over time, compounding personalization
- Continuous learning only applies to Signal 2 (vocal); keywords don't need per-GM adaptation

---

## VOICE TRAINING

### Core Concept
Voice training is the first thing the GM does after installing the app. It serves triple duty: speaker identification, vocal mood baseline, and product onboarding.

### Setup Flow
1. Install and launch
2. **Voice enrollment:** "Let's get to know your voice. Read these passages aloud." Seven mood-specific passages, each approximately 60 seconds. The app captures the GM's vocal fingerprint and mood-specific delivery characteristics simultaneously.
3. During training, the app plays sample music matching each mood — the GM *experiences* the product while teaching it their voice. The setup IS the demo.
4. "Great. I'll be listening for you during your sessions. Let's set up your music."
5. BYOM import (or proceed with bundled library)

### Training Passage Design (Critical)
- Passages must be written natively in each language, not translated. A Greek calm tavern passage should feel like something a Greek GM would actually narrate, using phrasing and rhythm natural to Greek storytelling.
- Passages must evoke genuine vocal performance. A flat, grocery-list-style passage produces a bad vocal profile. An evocative fantasy narration passage pulls authentic delivery from the GM.
- Petros can write Greek passages from personal experience at Greek game tables — a genuine advantage.

### Data Collected During Training
- Speaker verification embedding (voice fingerprint for GM identification)
- Seven mood-labeled audio samples (one per broad mood category)
- emotion2vec embeddings extracted from each sample (stored for future classifier training)
- All stored locally, encrypted at rest, never transmitted

### Pro Tier Extension
Pro users train approximately 20 subcategory profiles instead of 7. The system learns to distinguish "tavern warmth" from "sacred reverence" within the calm category, because the GM narrated them differently during training. The upgrade feels like unlocking the app's ability to understand the GM better — psychologically powerful conversion trigger.

---

## LIVE SESSION INTERFACE

### Vocal Delivery Threshold — Mode Routing (MVP)

During voice training, the app measures energy variance across the seven mood passages. If the GM's vocal delivery shows insufficient dynamic range across moods — meaning their tension, combat, and calm passages are prosodically indistinguishable — the system flags this immediately rather than silently routing them into an experience that will underperform.

**Threshold reveal moment (critical UX):** The app does not present this as a failure. After training completes, if variance is below threshold: *"Your narration style is measured and controlled — a lot of experienced GMs narrate this way. The app works best when you're in the driver's seat for mood calls. Here's how your session will look..."* Then shows Mode B interface before the first session, not during it.

GMs who pass the threshold enter **Mode A**. GMs below threshold enter **Mode B**. Both modes can be manually overridden in pre-session setup regardless of training result — some GMs will prefer Mode B even with expressive delivery because they want direct control.

### Mode A — Autonomous

The app drives. Mood transitions happen automatically when keyword confidence is sufficient. The three hotkeys are the GM's correction interface:

1. **Next** (`Ctrl+→` or configurable) — Different track, same mood. The app picked the right mood but the wrong track. Crossfade to new track from the same pool. GM doesn't break narration. Nobody at the table notices.

2. **Shift** (`Ctrl+↑/↓` or mood-cycle key) — Different mood entirely. The app misread the scene. GM taps the correct mood. Transition uses a designed crossfade: faster fade-down, brief "musical breath" dip, new mood fades in.

3. **Hold / Lock** (`Ctrl+Space`) — Stay put, stop making decisions. The app is being too active in a moment where the GM wants atmospheric neutrality. One press freezes current playback and suspends detection. Another press releases.

### Mode B — Collaborative

The app suggests. The GM confirms. A persistent but minimal mood palette is visible during the session — seven mood buttons in a horizontal strip, current active mood gently pulsing. The keyword engine still runs and highlights what the app *thinks* the mood should be, but does not auto-transition. The GM taps to confirm or override with a single click or keypress.

**Numpad support (optional, power-user):** Mode B GMs can map the seven moods to numpad keys 1–7. Suggested spatial layout:

```
7 [Sorrow]    8 [Mystery]   9 [Tension]
4 [Travel]    5 [Calm]      6 [Combat]
             2 [Celebration]
```

Calm anchors at 5 (center key). Escalation moves up and right. Numpad 0 = Hold/Lock — the largest key, hardest to miss under pressure. After a few sessions this becomes pure muscle memory: the GM calls mood shifts by feel without looking away from the table. This is not a hardware recommendation — it's a mapping the app supports for GMs who already own a numpad or macropad. No specific hardware is required or suggested.

**Mode B value vs manual Syrinscape:** The keyword engine still surfaces suggestions, pre-loads the right mood pool, auto-triggers SFX, and handles all crossfading and library intelligence. The GM makes one call per scene transition rather than operating a full audio UI. It's a co-pilot model, not manual control.

### Detection Philosophy (Unchanged)

**The app errs on the side of staying put.** If confidence is below threshold, it does nothing. A GM will forgive an app that's a bit slow to react — they'll just press a hotkey. An app that constantly fidgets between moods feels broken even if each individual decision was defensible. Calm, steady, occasionally smart is infinitely better than hyperactive and frequently wrong.

Think of it like a good session musician: sit in the pocket, only step forward when sure it's the right moment.

### Session Mode UI

During active play, the app's visible footprint is minimal:
- **System tray icon (Windows) / menu bar icon (Mac)** showing current mood via color (warm amber for calm, red for combat, blue-grey for tension, etc.)
- Click to expand a small popover with: mood buttons for direct selection, now-playing info, volume control
- **No main window during sessions.** When the GM hits "Start Session," the setup window minimizes or closes entirely. The app becomes only the tray icon plus hotkeys.
- This enforced minimalism is the desktop equivalent of the tablet's physical constraint — it prevents feature creep from making the live interface complex

### Pre-Session Setup Window

All complexity lives here:
- BYOM import and mood tagging (review auto-tagged broad categories, adjust as needed)
- Voice training and profile management
- Keyword sensitivity and confidence threshold adjustment
- Genre pack and library management
- Model size selection (small for older machines, medium default, large for power users)

### Power User Trap Protection
On desktop, the temptation to add panels, settings, and visualizations is strong — there's always room for one more window. The session mode enforces discipline by becoming a different UI state. When the session starts, the app disappears. Power user features go into pre-session setup or they don't go in at all. The live interface stays sacred.

---

## MOOD CATEGORIES

### Free Tier — Seven Broad Categories

| Category | Covers | Example Moments |
|---|---|---|
| **Calm** | Rest, taverns, downtime, peaceful settings | Campfire, inn, safe haven, morning in town |
| **Tension** | Danger, suspicion, things about to go wrong | Stalking enemies, political intrigue, trapped |
| **Combat** | Fighting, chase sequences, physical confrontation | Initiative, battle, pursuit, brawl |
| **Mystery** | Exploration, discovery, the unknown | Entering ruins, strange findings, puzzles |
| **Travel** | Overworld movement, journeys, montage | Road travel, sailing, wilderness, migration |
| **Celebration** | Victory, revelry, social joy | Feast, triumph, festival, reunion |
| **Sorrow** | Loss, death, heavy emotional beats | Character death, tragic reveal, destroyed village |

Sorrow earns its own category because GM need for sad music is specific, frequent, and tonally distinct enough that lumping it into calm or tension creates the "wrong vibe within the right category" problem.

### Paid Tier — Nuanced Subcategories

Each broad category splits into approximately 3–4 subcategories, expanding the taxonomy from 7 moods to roughly 20. This gates *nuance*, not *functionality*. The free user's game still has atmosphere. The paying GM gets an app that listens more carefully.

Examples (fantasy genre):
- **Tension** → dread, suspense, unease, pursuit
- **Combat** → skirmish, epic battle, desperate last stand, duel
- **Calm** → peaceful rest, melancholy reflection, sacred reverence, tavern warmth
- **Mystery** → wonder, foreboding discovery, arcane strangeness

Exact subcategory taxonomy to be refined through playtesting. Guard against taxonomy bloat — if the detection engine can't reliably distinguish between two subcategories, collapse them.

### Natural Conversion Trigger
Free-tier GMs will organically feel the limitations of broad categories. They'll hit "next track" frequently because the bucket is too broad for the moment. When they see the paid tier splits "tense" into exactly the subcategories they've been wishing for, the upgrade feels like the app finally understanding them. Combined with Pro's vocal mood classifier — which physically requires training those subcategories — the upgrade becomes a deeper relationship with the product, not just a feature unlock.

---

## SOUND EFFECTS — MVP (LOCKED)

### Decision
SFX ships in the desktop MVP as a toggleable feature. Any GM already using atmospheric audio runs external speakers — laptop vs tablet makes no meaningful difference to audio output quality. SFX playback is trivial compute; latency concern only affects the detection pipeline, not playback.

### Architecture
SFX triggering is architecturally separate from the music mood system:
- **Music:** Sustained atmosphere. Crossfade between loops. Gradual, ambient. Tolerant of timing — a half-second delay is imperceptible.
- **SFX:** Punctuation. One-shot sounds layered on current playback. Precise timing required (<0.5 seconds). Higher keyword confidence thresholds than mood detection. False positives are much more damaging — a random sword clash during a tender moment is jarring.

Same keyword detection pipeline feeds both systems, but with different output paths and different confidence requirements. SFX defaults to on but can be toggled off globally or per-effect-category in pre-session setup.

### Hardware Self-Regulation
The system detects available compute and adjusts: full feature set on capable hardware, SFX disabled on constrained machines. Like PC games detecting hardware and adjusting settings automatically.

---

## AUDIO ANALYSIS & AUTO-TAGGING SYSTEM (LOCKED)

### Three-Layer Tagging Model

All music — both bundled CC library and BYOM imports — passes through the same tagging system.

**Layer 1 — Broad Mood (automatic):** Seven categories auto-detected from audio features via librosa. Tempo, energy, key, spectral profile analyzed. Rule-based classification achieves approximately 80–85% accuracy. Runs automatically on import for all tracks.

**Layer 2 — Subcategory Mood (manual with auto-suggestion):** ~20 subcategories (dread vs suspense vs unease, etc.) require GM confirmation. System suggests based on deeper audio features, GM has final say. Pro tier feature.

**Layer 3 — Genre Tags (manual, multi-select):** Same track can carry fantasy + horror + space opera tags. Only the GM knows their campaign context. Tags accumulate as the GM uses tracks across different campaigns. Genre is stored as an array, not a string.

### Triple Purpose
This audio analysis tool serves three functions:
1. **Curates the bundled CC library** — pre-tags tracks before they ship with the app
2. **Powers BYOM auto-suggestion** — for Pro users, the system pre-sorts imported music into mood buckets; the GM reviews for obvious mistakes (fast) rather than manually tagging everything (slow)
3. **Informs the crossfade engine** — key, tempo, and spectral profiles enable smarter transitions between tracks (same-key transitions, compatible tempo matching)

### BYOM Onboarding Flow
1. GM imports tracks from device storage (drag-and-drop from file explorer on desktop)
2. App runs Layer 1 auto-analysis, pre-sorts into seven mood buckets
3. GM reviews the sorting — corrects obvious errors, approves the rest
4. Free tier: broad categories only. Pro tier: subcategory suggestions available.
5. Genre tagging happens naturally during campaign selection

---

## AUDIO CONTENT & LICENSING

### MVP Bundled Library Target
- **Music:** Minimum 5 tracks per mood category × 7 categories = **35 tracks minimum** for fantasy
- **Sound effects:** 20–25 high-impact core effects: door, footsteps, sword clash, spell impact, thunder, rain, fire crackling, crowd murmur, horse hooves, monster roar, etc.
- **Plus BYOM** as the pressure release valve for library depth

### Licensing Sources — Researched and Confirmed

**Creative Commons Music:**
- **Kevin MacLeod / Incompetech:** 2,000+ tracks, CC BY 4.0, tagged by genre/feel/tempo. Roll20 has already integrated 1,000+ of his tracks as precedent. Can pull 50–100 fantasy-appropriate tracks. Massive variety, somewhat variable quality.
- **Scott Buckley:** ~100–150 tracks total, CC BY 4.0, higher quality cinematic/orchestral/ambient. 20–40 TTRPG-suitable tracks. Excellent for key mood categories. **Caveat:** Orchestral remixes are NOT CC-licensed — track-by-track verification required.
- **Eric Matyas / Soundimage.org:** 2,500+ tracks, custom license (free with attribution, $30/track for no-attribution). Dedicated fantasy/horror/sci-fi pages. Ogg loops designed for games. Need email confirmation for app bundling but likely fine.
- **OpenGameArt.org:** Smaller scattered collections, some CC0 (no attribution needed). Useful for gap-filling specific mood categories.

MVP target of 35 music tracks is trivially achievable from these sources. Could assemble 100+ fantasy tracks.

**Sound Effects:**
- **Freesound.org:** 714,671 sounds. 72% uploaded in 2025 were CC0 (public domain). Dedicated RPG/fantasy packs from contributors. Sword clashes, doors, footsteps, thunder, fire — all available.
- **itch.io SFX packs:** TomMusic Fantasy 200 SFX Pack — royalty-free. Includes 20+ bow/sword SFX, 20+ spell SFX, 50+ footstep SFX across terrain types, doors/chests/gates, 20+ loopable backgrounds.
- **OpenGameArt.org:** Purpose-built RPG sound packs.

Target of 25 core SFX achievable from Freesound CC0 alone.

### Licensing Traps — Identified and Documented

- **Tabletop Audio:** 450+ ambiences (~75 hours), purpose-built for TTRPGs — seemingly perfect. **BUT licensed CC BY-NC-ND (NonCommercial + NoDerivatives).** Cannot use commercially. Cannot modify for crossfading. Requires separate commercial license negotiation. Approach only with working demo — the dream partnership, not a launch dependency.
- **ZapSplat:** 150k+ items but proprietary license prohibits redistribution in apps. **Skip entirely.**

### Content Database Requirement
From day one, maintain a database tracking per asset: track name, source, license type, commercial use permitted (yes/no), attribution requirements, mood category assignment (Layer 1), subcategory assignment (Layer 2 — pre-assigned even for free tier, ready for instant Pro upgrade value), genre tags (Layer 3), audio analysis metadata (tempo, key, energy, spectral profile). Genre stored as array (multi-select). This database is unsexy but essential.

### Commissioned Originals
Contract 10–15 ambient tracks for critical mood categories as legal bedrock. Atmospheric ambient loops are among the cheaper things to commission. Composers available on Fiverr and in TTRPG community spaces. These tracks provide a guaranteed fallback nobody can revoke or change terms on.

### Protecting Premium Content Value
If BYOM works too well, it weakens incentive to pay for bundled content. Premium content earns its place through:
- **Curated sound effects:** Perfectly timed, tagged, and ready for instant keyword triggering
- **Keyword-specific stingers and transitions:** Designed to work with the detection system in ways personal music can't match
- **Quality floor:** Bundled content is professionally produced and genre-curated. BYOM quality varies.
- **Pre-assigned subcategory tags:** Bundled tracks have subcategories assigned from day one. Upgrading to Pro instantly unlocks finer detection on the entire bundled library — no manual tagging needed.

---

## GENRE PACKS — ONE PRODUCT, MULTIPLE INTELLIGENCE LAYERS

### Structure
- **One app, not separate products.** Same interface, same engine, same everything. Genre determines which content library and intelligence profile loads.
- **Genre packs are content + intelligence bundles.** Each pack ships with: a genre-specific mood subcategory taxonomy, a tuned keyword detection vocabulary, genre-appropriate crossfade and transition behaviors, and curated/licensed audio content.
- **Genre pack as a lens on the GM's existing library.** When the GM buys the horror pack, their existing music gets re-evaluated through a horror lens. A track tagged as "tension/fantasy" might also become "tension/horror." The detection engine gains horror-specific keywords. Same track pool, different genre filters, different detection behavior per campaign. Multi-genre tagging means the GM doesn't maintain separate libraries.

### Genre Sequencing
- **Launch:** Fantasy (biggest market, most codified tropes, most available content, keyword corpus easiest to build)
- **Second:** Horror (Call of Cthulhu / Delta Green community is passionate, underserved, atmosphere-dependent, older, and willing to pay. Product value proposition is strongest where atmosphere matters most.)
- **Later:** Sci-fi, dark/gritty, modern, superheroic — based on community demand

### Genre Intelligence — Why Tropes Make This Work
Genre tropes are predictable by definition. The detection system isn't understanding novel storytelling — it's recognizing patterns repeated across thousands of games. That's a more solvable problem than general narrative comprehension.

**Fantasy knows:** Tavern/inn/hearth/ale/bard → warmth, rest. Door creaks/darkness/decay → dungeon tension. Roll for initiative / creature names → combat transition.

**Horror (second genre) would know:** Silence as active tool — pull music *down*. Slower builds, longer holds on uncomfortable ambience. Sudden loud shifts reserved for rare, deliberate jump scares — GM-triggered only, never auto-detected. Horror pacing is fundamentally different from fantasy pacing; the intelligence layer must reflect this.

### Compounding Content Moat
A carefully tuned detection profile that understands the narrative rhythms of a specific genre is hard-won design knowledge and genuinely difficult to replicate without doing the same deep work. Each genre pack built compounds the product's defensibility.

---

## MULTILINGUAL KEYWORD ARCHITECTURE

### Design Principle
The detection engine is language-agnostic at the vocal layer (Signal 2) and modular at the keyword layer (Signal 1). Each language is a vocabulary file mapping phrases to mood categories and trigger types. Adding a new language = adding a vocabulary file + ensuring Whisper supports that language. The engine itself doesn't change.

### Launch Languages
- **English:** Full keyword vocabulary
- **Greek:** Core keyword vocabulary (30–40 most universal triggers for fantasy)

Greek language support is viable at launch because the medium Whisper model — now comfortably runnable on desktop hardware — handles Greek with solid accuracy, including code-switching between Greek narration and English game terms. The vocal mood layer (Signal 2) works identically in any language.

### Greek-Specific Considerations
- Greek GMs frequently code-switch: narrating in Greek but using English terms for game mechanics. The medium Whisper model handles this.
- Don't translate keywords — capture authentic table language. What Greek GMs actually say, not what a translator would produce.
- **Market opportunity:** No existing TTRPG companion tool supports Greek. First-mover advantage in a tight-knit community where strong recommendations travel fast.
- **Beta strategy:** Greek community as initial beta audience. Small enough for meaningful feedback from a high percentage of users, passionate enough for word of mouth, and Petros has direct cultural access.

### Community-Sourced Localization Pipeline (Post-MVP)

**Methodology:**
1. Develop a questionnaire with ~30 common RPG situations (not phrases to translate)
2. Frame as situations: "Combat is about to start. What words or phrases does the GM typically say at this moment?" — captures official terms, slang, shorthand, and English loanwords simultaneously
3. Include open-ended questions for language-specific triggers not anticipated
4. Deploy to target language TTRPG communities
5. Curate via 1–2 trusted community volunteers per language
6. Build vocabulary file from curated responses

**Strategic benefit:** Localization becomes a community engagement event, not a quiet update. Contributors become emotionally invested advocates before the product even supports their language.

---

## BRING YOUR OWN MUSIC (BYOM)

### Core Concept
The product is an intelligence layer, not a music library. GMs point the app at their existing music collection. The audio analysis system auto-tags tracks into mood categories. The detection engine makes their library smart.

**Legal position:** The app is a tool that organizes and plays local files intelligently. Same legal footing as any music player.

### Onboarding Flow (MVP)
1. Import tracks from device storage (drag-and-drop on desktop)
2. App runs automatic broad mood analysis (Layer 1) — pre-sorts into seven categories
3. GM reviews the auto-sort, corrects obvious errors, approves the rest
4. Free tier: broad categories only. Pro tier: subcategory auto-suggestions available for review.
5. Pre-session only — complexity lives in prep, not play.

### Future Enhancements (Post-MVP)
- **Filename/folder parsing:** Many GMs name files descriptively ("dark_forest_ambient.mp3") or organize by mood folders. Infer rough categorization for free.
- **Full auto-categorization:** All three approaches combined — audio analysis, filename parsing, auto-suggestion. Manual correction always available.

---

## TIER STRUCTURE & FEATURES

### Three Tiers + One-Shot (Simplified)

A niche product for hobbyists needs simple decision-making. Expand tiers only once user behavior data shows what people actually pay for.

**Free Tier**
- Full broad mood category access (7 categories)
- Keyword-triggered track/mood switching (Signal 1 — local Whisper processing)
- Speaker verification (GM voice isolation)
- Manual mood selection and hotkey controls (next, shift, hold/lock)
- Sound effects (toggleable)
- BYOM with automatic broad mood tagging (Layer 1 audio analysis)
- Small bundled starter library
- Static detection accuracy (no per-GM vocal learning)

**Pro Tier (Subscription)**
- Nuanced mood subcategories (~20 subcategories across 7 categories)
- Vocal mood classification (Signal 2 — emotion2vec-based, improves per-GM over time via continuous learning)
- Expanded bundled library with pre-assigned subcategory tags
- Subcategory auto-suggestion for BYOM tracks (Layer 2 audio analysis)
- Genre packs: one included, additional packs at reduced add-on fee
- **Future (post-MVP):** AI-generated atmospheric visuals, battlemap library, adventure module upload, session zero ritual, campaign memory, character leitmotifs

**One-Shot Pass / Pro Trial (Flat Fee)**
- All Pro features for a single session
- Targets: try-before-you-commit, convention GMs, standalone evenings
- On desktop, may take the form of a "Pro trial session" since the free tier is already permanent
- Price point TBD

### Pricing — Deferred
Exact price points deferred until real user feedback. Structural principle locked: free gates broad functionality, paid gates nuance and intelligence depth. The free tier must be genuinely good — it's the growth engine and the word-of-mouth product. Conversion comes from the natural friction of broad categories, not from artificial limitations.

---

## PRIVACY, COMPLIANCE & DATA ARCHITECTURE

### EU AI Act Classification

The vocal mood classifier (Signal 2) constitutes an emotion recognition system under EU AI Act Article 3(39): "an AI system for the purpose of identifying or inferring emotions or intentions of natural persons on the basis of their biometric data."

**Prohibition does NOT apply.** The emotion recognition ban (Article 5(1)(f)) covers workplaces and educational institutions only. Consumer entertainment products are explicitly outside scope. EU Commission guidelines confirm that AI systems detecting customer emotions in commercial contexts do not fall under the ban.

**High-risk classification MAY apply.** Annex III lists "AI systems intended to be used for emotion recognition" as high-risk, with requirements enforceable from August 2, 2026. This triggers potential obligations around risk management, technical documentation, conformity assessment, and EU database registration.

**Article 6(3) exception argument (strong but unvalidated):** The system performs a narrow task (selecting background music), enhances rather than replaces human activity, has zero impact on anyone's rights or safety, the GM has full manual override, and the output is ephemeral entertainment. This argument should be formally documented and reviewed by a lawyer before launch.

**Critical distinction:** The system infers *narrative mood* (what kind of story is being told), not the GM's *actual emotional state* (how they personally feel). A GM may be grinning while narrating a funeral. This is content analysis, not psychological assessment. Strengthens the Article 6(3) position.

**Regardless of high-risk classification outcome, transparency obligations apply (Article 50, enforceable August 2, 2026):** Users must be informed when subjected to an emotion recognition system. This must be disclosed clearly during voice training setup, not buried in privacy policy.

### GDPR Compliance

**Speaker verification = special category biometric data.** Voice processed through technical means for unique identification triggers GDPR Article 9. Requires explicit consent.

**Vocal mood analysis = sensitive processing requiring DPIA.** Even if not technically Article 9 special category data (since the purpose is mood inference, not identification), spectrographic voice analysis for inferring emotional/affective state requires a Data Protection Impact Assessment under Article 35.

**Architecture decisions for compliance (build from day one):**

1. **Explicit, granular consent during voice training.** Before the GM reads the first mood passage:
   - Clear explanation: "This app creates a voice profile to recognize you as the GM. It also analyzes how your voice sounds during narration to match music to your scenes."
   - Separate consent checkboxes for speaker verification and vocal mood analysis
   - The GM must be able to use keyword detection only (no vocal analysis) if they decline mood consent
   - Consent must be freely given — product works meaningfully without vocal analysis (keyword detection is the free tier experience)

2. **All processing local.** Audio never leaves the device. No cloud transmission of voice data. "Your voice never leaves your device, ever." This is both a GDPR simplification and a marketing asset.

3. **No raw audio storage.** Audio processed transiently through Whisper and emotion2vec, converted to abstract numerical features (text transcription, embeddings), then immediately discarded. Embeddings cannot be reversed into voice recordings. Document this in the DPIA.

4. **Portable, deletable vocal profile package.** Speaker verification embedding, emotion2vec mappings, accumulated session corrections — all in one encrypted container per GM. Clear schema, versioned, encrypted at rest.

5. **Right to erasure button.** "Delete my voice profile" wipes everything: speaker embedding, mood mappings, all session correction history. Available from day one. When tablet sync exists (Phase 2), deletion propagates across all synced devices.

6. **DPIA completed before launch.** Documents what data is collected, why, how processed, where stored, what risks exist, and what mitigations are implemented. Can be drafted by the founder; must be reviewed by a lawyer.

### Tablet Sync Strategy (Phase 2 — Designed for Now)

Vocal profiles structured as discrete, exportable packages from day one. When tablet version ships, sync options:
- **Peer-to-peer over local network** (preferred): Desktop discovers tablet on same Wi-Fi, transfers encrypted profile directly. No cloud. "Your voice never leaves your devices."
- **End-to-end encrypted cloud sync** (alternative): Server never sees profile contents. More convenient but adds compliance complexity.

Deletion on one device must propagate to all synced devices.

### Legal Review Budget
Estimated €500–1,500 for specialized AI Act / GDPR legal review before launch. Not an ongoing retainer — a single consultation with a well-prepared compliance package (DPIA, consent flows, Article 6(3) argument, privacy notice). Find a lawyer who specifically knows the AI Act, not just GDPR generalists. Athens has a growing tech law scene.

---

## AI & COMMUNITY POSITIONING

- Two content philosophies within one product — a settings toggle, not separate editions
- Same pricing regardless of AI preference setting
- "AI Off" disables vocal mood analysis entirely, falling back to keyword-only detection
- **Never lead with "AI-powered" in marketing.** Lead with simplicity and the magic of the experience.
- Battlemap library (future) is human-created only — ethical and a community selling point
- **For MVP:** The "AI" is detection and matching — smart search, not generative content. Nobody has a philosophical problem with a smart playlist. Don't call keyword detection "AI" in marketing even if it technically is.
- The vocal mood analysis is a more nuanced positioning challenge. Frame as "learns how you narrate" rather than "AI emotion detection" — technically accurate and community-appropriate.

---

## MONETIZATION & REFERRAL SYSTEM

- Milestone-based referral reward system, not a continuous token economy
- A referral only counts when a purchase is involved — filters casual shares
- Reward is a one-off discount code for subscription, not free access
- Milestones escalate in threshold — first milestone ~3 referrals
- Milestone badges / shareable achievements with TTRPG-themed naming (specifics deferred)
- **Deferred:** Exact milestone thresholds and discount percentages, subscription pricing, one-shot price, creator revenue share percentages

---

## FUTURE FEATURES — POST-MVP ROADMAP

These are explicitly deferred. None block the MVP or should receive development time until the audio intelligence core is validated.

### Phase 1.x — Fast Follow (within weeks of launch)
- Vocal mood classifier activation (emotion2vec-based Signal 2, trained on accumulated session data)
- Dual-signal weighting and confidence calibration
- Continuous learning from session corrections
- **Semantic embedding detection (Signal 1 upgrade):** Replace or augment fuzzy string matching with a lightweight sentence transformer (e.g. all-MiniLM-L6-v2). Encodes transcribed segments into vector space and compares against mood-labeled exemplar sentences. Catches descriptive GM narration that never uses trigger words but semantically means the same thing. Requires compute spike alongside Whisper — validate on target hardware before committing. Not MVP because it adds a third model before the core pipeline is validated in real sessions.
- **Sliding context window:** Maintain a rolling transcript buffer of the last 2–3 minutes. Run periodic semantic analysis across the full window, not just the latest segment. Catches gradual narrative escalation — combat building over five segments, none individually triggering — that single-segment detection misses entirely.

### Phase 2 — Tablet & Visual Layer
- iPad / Android tablet version as premium companion experience (touch interface, ambient color wash, dedicated form factor)
- Profile sync between desktop and tablet
- AI-generated atmospheric background images (ephemeral, mood-setting)
- Human-made isometric battlemaps via Patreon creator partnerships (Czepeku, Heroic Maps, Tom Cartos, Maphammer, Afternoon Maps, Dynamic Dungeons)
- Toggle to disable AI visuals entirely

### Phase 3 — Session Intelligence
- Session zero as core product ritual: GM creates campaign, sets tone/genre; players join via link, choose character themes/leitmotifs; app calibrates to this specific campaign
- **Adventure module upload as keyword expansion engine:** GM uploads their module (PDF or text). Parser extracts location names, NPC names, faction names, and scene descriptors. These become campaign-specific keywords mapped to GM-assigned moods during setup. "The approach to Thornkeep" triggers the right atmosphere without any generic keyword match. Particularly high value for monotone GMs whose narration vocabulary is constrained but predictable and campaign-specific. Requires dedicated spike for PDF parsing reliability before commitment — half-baked module parsing feels broken rather than magical.
- Anticipatory matching — app knows where in the story the session is
- Campaign memory across sessions (character themes, location moods, narrative arc memory)

### Phase 4 — Expansion & Platform
- Full campaign memory, deep homebrew integration, collaborative/player-facing output
- API access for power users
- Actual Play production features
- Foundry VTT integration evaluation (strategic identity tension: only revisit once product-market fit is proven with physical table GMs)

---

## DISTRIBUTION & GO-TO-MARKET

- **Primary channel:** itch.io — TTRPG community already lives here, creator-friendly revenue split, zero approval process, handles payments for Pro tier
- **Secondary:** Direct download from product website
- **Tertiary (evaluate post-launch):** Steam (growing non-game app presence, enormous reach among the gaming-adjacent TTRPG demographic)
- **Greek community as beta launch:** Small enough for meaningful feedback, tight-knit enough for word of mouth, first-mover advantage with Greek language support. Petros has direct cultural access.
- **Creator partners as marketing channel:** Patreon battlemap and audio creators have audiences of exactly the target users
- **Deferred:** Convention strategy, broader community seeding (Reddit, Discord, podcasts), VTT integration, European localization marketing

---

## OPEN QUESTIONS — DEFERRED BUT TRACKED

### Strategic
- **Competitive moat:** What is defensible long-term once Syrinscape, D&D Beyond, or a well-funded startup notices? (Genre intelligence packs + per-GM vocal profiles are the beginning of an answer — both accumulate value that's hard to replicate and costly to abandon.)
- **The magic demo problem:** How do you demonstrate real-time mood detection in marketing without a live session? (Voice training as demo partially solves this.)
- **Name and brand identity:** Nothing decided yet.

### Partnerships
- **Partnership approach sequencing:** Who first — audio creators, battlemap creators, or early adopter GMs?
- **Tabletop Audio commercial license:** Approach Tim with working prototype. "I'm building something that could put your audio in front of a much larger audience." Dream partnership — 450+ purpose-built TTRPG ambiences.
- **Creator value proposition:** Flat-fee licensing for initial library, revenue-share only once traction exists and volume can be delivered.

### Legal
- **Article 6(3) exception formal opinion:** Needs lawyer review before launch
- **DPIA completion and review:** Draft internally, review externally
- **Licensing terms and creator contracts**
- **Content quality control process for creator library submissions**

### Technical
- **Microphone quality validation:** Record a real game session on 2–3 different laptops. Test whisper.cpp medium accuracy in noisy tabletop environment. This is the remaining existential spike — not latency (solved by desktop) but input quality.
- **Character voice handling:** GMs doing heavy voice acting may confuse both speaker verification and vocal mood classification. Hold/Lock is the escape valve, but the threshold tuning matters.
- **emotion2vec integration performance:** Inference speed testing on target hardware, embedding extraction latency.

---

## IMMEDIATE NEXT STEPS — PRE-BUILD CHECKLIST

### Critical Spikes (Run First)
1. **Tauri proof-of-concept:** System tray icon, global hotkeys, audio playback. Confirm the framework handles all three core interaction modes.
2. **Microphone quality testing:** Record a real game session (or simulate with friends) on 2–3 different laptops. Run through whisper.cpp medium. Test keyword detection accuracy in real tabletop chaos. This is the remaining existential risk.
3. **emotion2vec base integration:** Test inference speed. Extract embeddings from sample narration audio. Confirm it runs alongside whisper.cpp without resource contention on target hardware.
4. **Audio engine selection:** FMOD vs native. Build a quick prototype: crossfade between two tracks on a hotkey. Layer a one-shot SFX on top. Does the transition feel seamless?

### Content & Design (Parallel)
5. **Audio analysis tool prototype:** librosa feature extraction → seven-mood rule-based classification. Test against a sample of 50 CC tracks. Measure classification accuracy.
6. **Voice training passage design:** Write seven evocative fantasy narration passages in English and Greek. Must pull genuine vocal performance. Greek passages written natively, not translated.
7. **Fantasy keyword vocabulary:** Draft 15–20 high-confidence trigger phrases per mood category. Test against real GM narration captured through a laptop mic.
8. **Greek keyword vocabulary:** Draft core 30–40 universal fantasy triggers in authentic Greek table language. Test mixed-language detection with medium Whisper model.
9. **Audio licensing audit:** Identify and legally clear 35 music tracks + 25 sound effects. Build content database with per-asset license tracking from day one.
10. **Commission originals:** Contract 10–15 ambient tracks for critical mood categories.

### Compliance (Before Launch)
11. **Draft DPIA:** Document data flows, processing purposes, risks, mitigations.
12. **Draft Article 6(3) exception argument:** Formal document for why the system is non-high-risk.
13. **Design consent flow:** Voice training setup screens with granular consent checkboxes and clear disclosures.
14. **Legal review:** Single consultation with EU AI Act / GDPR specialist. Budget €500–1,500.

---

## DATA MODEL — DAY ONE REQUIREMENTS

Structural decisions that must be correct from the first line of code:

- **Genre:** Array, not string (multi-select support for tracks across genres)
- **Mood subcategory:** Pre-assigned on bundled tracks even for free users (instant value on Pro upgrade)
- **Vocal profile storage:** Speaker verification embedding + emotion2vec embeddings + mapping layer weights + session correction logs, all in one encrypted, portable, deletable container per GM
- **Content database:** Per-asset license tracking, mood tags, subcategory tags, genre tags, audio analysis metadata
- **Session correction logs:** Timestamped records of GM corrections (shift, next) with associated vocal features — training data for continuous learning
- **Consent flags:** Granular per-feature consent stored alongside profile (speaker verification consent, vocal analysis consent)

---

*Document v4 — Incorporates vocal delivery threshold detection, Mode A (Autonomous) / Mode B (Collaborative) session interface split, numpad as optional Mode B power-user enhancement, semantic embedding detection and sliding context window as V1.x candidates, and adventure module pre-loading as Phase 3 feature. Previous: desktop platform pivot, dual-signal detection architecture, emotion2vec foundation model, continuous learning strategy, CC-licensed content library research, SFX inclusion, audio analysis intelligence layer, EU AI Act / GDPR compliance architecture. Living document — to be updated as technical spikes are completed and decisions are validated.*
