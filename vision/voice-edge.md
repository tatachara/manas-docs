# Voice & Edge — Vision & Feature Charter

> Subsystem: **Voice & Edge tier** (the satellites). Part of the **Manas** platform; a cross-cutting delivery
> surface, not a domain service — it is *how a person talks to Sarathi (the Agent) from a room*, and how a persona
> (Bonnie / Anik) is embodied in the physical home.
> Read [`docs/adr/0001-manas-platform-architecture.md`](../adr/0001-manas-platform-architecture.md) first —
> especially **§5.6 Persona layer** and **§5.7 Voice & edge tier**. This document refines that charter and must
> not contradict it.
>
> - **Status:** Vision (pre-build). Absorbs the Vatara voice-satellite work (myJarvis ADR 0011/0012).
> - **Date:** 2026-07-11
> - **Where it sits:** the satellites are HA-side edge hardware; the brain is central (STT → Sarathi → TTS via
>   **Medha**). Audio never leaves the home. Sarathi, walls, and step-up are unchanged by the surface —
>   a voice turn is just another identified caller.
>
> **Honesty header (the owner's standing instruction):** be brutally realistic. The #1 flagged risk of this
> whole subsystem — a reliable **custom wake word ("Hey Bonnie Blue")** — is **UNPROVEN**, and §4 says so plainly
> with a data plan, fallbacks, and the exact spike that decides it. No "it doesn't work" surprises after we build.

---

## 1. Intent — the role of the voice-edge tier

**The voice-edge tier is the household's ears, mouth, and face in every room.** It is a fleet of small,
always-listening **the voice box** satellites — one per room/zone — that wake on a spoken word, capture
near-field audio, stream it to the central brain over the **Wyoming** protocol, speak the reply through **Piper
TTS**, and render a **persona's animated face** on the box's 320×240 screen. It is the surface that makes Manas
*present*: you speak into the air of a room and the home's shared mind answers, in the voice and character that
belongs to *you* (the child's room speaks as **Bonnie**; the adults' spaces as **Anik**).

It is deliberately **thin at the edge and smart at the centre.** The satellite does exactly three hard-real-time
jobs locally — **wake-word detection**, **audio capture with echo cancellation**, and **playback + face
animation** — because those must be instant and must not depend on the network. Everything cognitive — STT,
Sarathi's reasoning, tool-calls, safety gating, TTS synthesis — happens **centrally on the home server via Medha and
Sarathi**, so the edge stays cheap, replaceable, and never holds a secret or a model that matters. The satellite
is a **peripheral of the brain, not a brain**: it has no independent authority, actuates nothing, decides nothing,
and — per `[SEC-5..7]` carried from Vatara — talks to **exactly one peer (Home Assistant)** on an isolated,
egress-blocked VLAN. A compromised or stolen box yields a microphone on a quarantined network, nothing more.

It is **local-first and private by construction**: wake detection runs on-device (no audio leaves the box until
*after* the wake word), STT/LLM/TTS run on the family's own hardware, and **no child's voice ever reaches a
vendor cloud**. It is **accessible-first**: voice is a primary, first-class way to reach *every* Manas capability
(the platform's voice-complete mandate), not a bolt-on. And it is **honest about its limits**: this exact
hardware has two microphones and no XMOS DSP, so far-field performance is good-not-magical; custom wake words are
an open research risk; and the tier is designed to **degrade gracefully** — a push-to-talk button and a built-in
pretrained wake word keep every box shippable even if custom training never lands.

Rule of thumb: **the box is a face and a pair of ears on a leash; the mind is in the other room.**

---

## 2. Capability areas

**Kind legend:** `[deterministic]` = pure C# / firmware config, no model · `[LLM]` = the central LLM genuinely
helps (via Sarathi) · `[on-device]` = runs on the voice box itself (wake/AEC/VAD/animation) ·
`[integration:name]` = depends on an external system/adapter (HA, Wyoming, ESPHome, Piper…).
**Feasibility:** `now` = buildable on today's stack/hardware · `needs-hardware` = requires a box/mic purchase or
a physical flash to validate · `UNPROVEN` = a real technical risk not yet demonstrated on this hardware — must
be validated by a spike before we design as if it works.
**Priority:** P0 = foundational / first slice · P1 = core value · P2 = later polish or specialised.

### 2.1 The satellite — capture & wake (the ears)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| On-device wake-word (built-in) | microWakeWord on the voice box detects a **pretrained** phrase ("Okay Nabu"/"Hey Jarvis") locally; audio only leaves the box *after* wake. Proven live on the Vatara box. | `[on-device]` `[integration:ESPHome]` | now | P0 |
| **Custom wake-word ("Hey Bonnie Blue")** *(amended 2026-07-11, round 2)* | An on-device custom phrase for the child's box. Vatara proved the *training toolchain* end-to-end but the proof model's **recall was 0.0** — a usable custom word is **not yet demonstrated**. See §4. | `[on-device]` | **UNPROVEN** | P1 |
| Push / tap-to-talk fallback | The box's hardware button and touchscreen start a listen turn with no wake word — the deterministic floor that keeps every box usable regardless of wake-word success. | `[on-device]` `[deterministic]` | now | P0 |
| Near-field audio capture | Dual-mic capture (ES7210 ADC) at 16 kHz mono, the format the STT stack expects. | `[on-device]` `[integration:ESPHome]` | now | P0 |
| Voice activity detection (endpointing) | On-device VAD marks speech start/end so only the utterance streams and the turn ends cleanly. | `[on-device]` | now | P0 |
| Acoustic echo cancellation (AEC) | ESP-SR AFE cancels the box's own speaker playback from the mic → barge-in while Bonnie is talking. **Software AEC on 2 mics** — good, not XMOS-class. Honest limit (§7). | `[on-device]` | now | P1 |
| Stationary-noise suppression / AGC | ESP-SR AFE noise-suppression + auto-gain — partial mitigation for the quiet-child-voice problem seen in Vatara. | `[on-device]` | now | P1 |
| Wake sensitivity / threshold config | Per-box wake threshold + "wake volume" tuning (data, not code) — a child's box may need a lower bar. | `[deterministic]` `[integration:ESPHome]` | now | P1 |
| WavGain / capture normalisation (central) | Central WavGain normalises quiet clips before STT — the proven fix for the child's low mic level (carried from Vatara). | `[deterministic]` | now | P1 |

### 2.2 Transport & the central turn (the leash)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Wyoming protocol transport | Box → HA Assist → **Vatara/Manas Wyoming brain** (one TCP port; stt/conversation/tts services). Wire format pinned to `wyoming` lib; golden-tested in Vatara. | `[integration:Wyoming]` `[deterministic]` | now | P0 |
| HA Assist as the pipeline shell | HA orchestrates wake→stt→intent→tts; **the STT/Sarathi/TTS are Manas'**, not HA's generic ones (child STT repair, Sarathi, Bonnie's persona). | `[integration:HomeAssistant]` | now | P0 |
| Central STT (whisper + repair) | `RoutingSpeechToText`: WavGain → prompted whisper → LLM transcript repair. Per-language routing seam (Kannada later). Runs on Medha's hardware. | `[LLM]` `[deterministic]` | now | P0 |
| Sarathi turn (converse or tool-call) | The transcript enters **Sarathi** exactly as any other channel: identity resolved, safety-gated, either answered or a governed tool called. Voice is not a special cognition path. | `[LLM]` `[deterministic]` | now | P0 |
| Central TTS (Piper per-persona) | Piper synthesises the reply in the speaker's persona voice (Bonnie: natural child-appropriate voice, no pitch/sample-rate trick, see §6 below; Anik adult) *(amended 2026-07-11, round 2)*. Streamed back through HA → box speaker. | `[deterministic]` `[integration:HomeAssistant]` | now | P0 |
| Conversation service ownership | *(amended 2026-07-11, round 2)* The **central turn** described in this section (Wyoming → STT → conversation → TTS) is **Sarathi's** conversation service, not Kaya's ([ADR 0010](../adr/0010-agent-conversational-core.md) §4) — Kaya is reached from it only as a `kaya.*` tool. Step-up park/resume interim + completion cues are **spoken utterances** delivered through this same turn (the agent spec §D path 3). | `[deterministic]` | now | P0 |
| Speaker identity from satellite | `satellite_id` → member → persona/voice/walls (`Wyoming:Satellites:{id}=memberId`). One brain serves every room *personally*. Deterministic mapping now; voiceprint later. | `[deterministic]` | now | P0 |
| Input safety gate on voice | Child-safety classifier (lexical floor → Llama Guard → ShieldGemma child policy) runs on the transcript before any command — same gate as every surface. | `[LLM]` `[deterministic]` | now | P0 |
| Continued conversation | Keep listening for a follow-up turn without re-waking, within a short window (HA Assist "continue conversation"). | `[integration:HomeAssistant]` `[on-device]` | now | P2 |
| On-device timers | Box-local kitchen timers with a screen/tone, surviving a brief brain outage. | `[on-device]` | now | P2 |

### 2.3 Multi-satellite behaviour (a fleet, not one box)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Single-responder arbitration | When several boxes hear the wake word, **exactly one** responds. HA today does **first-to-report-wins** (fastest box), not true nearest/loudest — honest gap (§5). | `[integration:HomeAssistant]` `[deterministic]` | now | P1 |
| Nearest / loudest arbitration | Pick the box the speaker is *actually* nearest to (wake-confidence / RMS energy / presence fusion), not merely the fastest. **Not solved by HA out-of-the-box** — a Manas-side design (§5). | `[deterministic]` | **UNPROVEN** | P2 |
| Barge-in (interrupt playback) | Speak over Bonnie mid-reply and be heard (depends on on-device AEC quality). | `[on-device]` | now | P1 |
| Reply on the box you spoke to | The response (audio + face) returns to the *same* satellite that captured the turn, keyed by `satellite_id`. *(amended 2026-07-11, round 2)* The turn itself is Sarathi's conversation service (§2.2, ADR 0010 §4); a parked step-up's interim/completion cues are spoken utterances returned the same way (the agent spec §D path 3). | `[deterministic]` | now | P0 |
| Room/zone → satellite map | Each box is bound to a room (and, where apt, an occupant + persona); the Kutumba/Kaya topology owns the mapping. | `[deterministic]` `[integration:Kutumba]` | now | P1 |
| Whole-home / targeted announce | Speak a message to one room, a zone, or every box (`assist_satellite.announce`) — the fastest end-to-end audio probe and the notification path. | `[deterministic]` `[integration:HomeAssistant]` | now | P1 |
| Simultaneous independent turns | Two people talking to two different boxes are two independent Sarathi turns (separate identities, separate conversation state) — no cross-talk. | `[deterministic]` | now | P1 |
| Follow-target handoff | A turn that starts on one box and should answer on another (e.g. "read it in the kitchen") — Sarathi-directed, not automatic. | `[LLM]` `[deterministic]` | now | P2 |

### 2.4 Persona on the box (the face & voice)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Per-satellite persona binding | A box carries a persona (the child's box = **Bonnie**): name, voice, tone, safety register, wake word, mascot style — resolved from Kutumba, applied uniformly. Delivery only, never truth/safety/walls. | `[deterministic]` `[integration:Kutumba]` | now | P0 |
| Piper voice per persona | Bonnie (natural child-appropriate voice, A/B-picked; pitch/sample-rate manipulation prohibited — §6, [persona.md](../architecture/persona.md) §1) *(amended 2026-07-11, round 2)* vs Anik (adult). Voice is config/asset, swappable. | `[deterministic]` | now | P0 |
| Mascot face on the 320×240 screen | An animated character (Bonnie the stick-girl in Vatara) drawn on the box's LCD, driven by cue state — listening/thinking/speaking/celebrating/storytelling… | `[on-device]` `[deterministic]` | now | P1 |
| Cue engine (LLM tag → animation) | Sarathi emits an `[anim:x]` tag (or keyword fallback) → resolved against a **data-driven catalog** (`animations.json`, hot-reload) → published to an HA sensor the box subscribes to. | `[LLM]` `[deterministic]` | now | P1 |
| Built-in pipeline screens | ESPHome's stock listening/thinking/replying screens work day one, before custom art. | `[on-device]` `[integration:ESPHome]` | now | P0 |
| Story / long-form narration | Chapters synthesised ahead → streamed to the box's media_player → face animates "storytelling"; voice "stop" works (Vatara's BoxNarrator). | `[deterministic]` | now | P2 |
| Off-hardware face preview | Render the mascot in a browser emulator to iterate art without a flash (Vatara's `box-emulator.html`) — de-risks the flash-only iteration loop. | `[deterministic]` | now | P2 |
| Per-box provisioned persona | Persona/voice/wake are **provisioned per device** (which box is whose) as fleet config — a new box is data + assets, no code. | `[deterministic]` | now | P1 |

### 2.5 Fleet management (provisioning, OTA, health)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| First-flash provisioning | USB flash of the ESPHome firmware, then adopt in HA; box mapped to room/occupant/persona in Manas config. | `[deterministic]` `[integration:ESPHome]` | now | P0 |
| OTA updates | After first flash, iterate firmware/screens over-the-air (~30s), no cable. | `[integration:ESPHome]` | now | P0 |
| Health & connectivity monitoring | Track each box: online/offline, wake-engine alive, last-seen, Wi-Fi RSSI, "stuck in responding" — surfaced to **Bodha** + **Saakshi**. | `[deterministic]` | now | P1 |
| Config-as-data fleet registry | Boxes, their rooms, personas, voices, wake models, thresholds — all **data** (Kutumba/Kaya config), hot-reloadable; no per-box code. | `[deterministic]` | now | P1 |
| Secrets discipline | Wi-Fi/API secrets never hardcoded; **human-typed** (Vatara field note: the agent sandbox redacts SSIDs → a `<redacted>` literal bricked a flash). | `[deterministic]` | now | P0 |
| Isolated-VLAN enforcement | Every box on the untrusted VLAN, egress-blocked, HA-only peer (`[SEC-5..7]`). Fleet health includes network posture. | `[deterministic]` `[integration:network]` | needs-hardware | P1 |
| Graceful offline behaviour | Brain down → box shows an honest "can't reach home" state, on-device timers survive, wake still works locally, no crash-loop (Vatara "stuck responding" lesson). | `[on-device]` `[deterministic]` | now | P1 |
| Fleet self-report to Sarathi | "Which rooms can hear me / is the child's box online?" answered from the live fleet model (Bodha introspection). | `[deterministic]` `[LLM]` | now | P2 |

---

## 3. The hardware reality — the voice box

The exact device, so no feature is invented on silicon that isn't there.

| Component | Spec | Consequence for Manas |
|---|---|---|
| **SoC** | the voice box, dual-core Xtensa LX7 @ 240 MHz, vector/AI instructions | Runs microWakeWord on-device in <~10 ms/inference; **cannot** run STT/LLM/TTS — those are central by necessity, not choice. |
| **PSRAM** | 16 MB Octal PSRAM (+ 512 KB SRAM) | Enough for audio buffers, the AFE pipeline, one wake model, and an LVGL screen — **one** wake model at a time, modest headroom. |
| **Flash** | 8/16 MB | Firmware + screen assets fit; not a media store. |
| **Microphones** | **Two** digital mics (ES7210 ADC) — near-field | **No 4-mic far-field array, no XMOS DSP.** AEC/noise-suppression is **software** (ESP-SR AFE). Good in-room; weaker across-room / noisy / quiet-child than a HA Voice PE (which has an XMOS XU316). This is the honest ceiling on far-field and barge-in. |
| **Speaker** | Mono, via ES8311 codec + power amp | Fine for a persona voice and short narration; not hi-fi music. |
| **Screen** | 2.4″ **320×240** SPI IPS, capacitive touch | The mascot face canvas + tap-to-talk surface. LVGL, cue-driven, OTA-iterable. |
| **Buttons / IMU** | 3 buttons, 3-axis gyro + accel | Hardware push-to-talk (the deterministic fallback); IMU unused for now. |
| **Radio** | Wi-Fi + BLE (the voice box) | Wi-Fi to HA; BLE could later serve as a room-presence proxy (Kaya's occupancy). |
| **Power/USB** | USB-C (⚠ kit cable is **power-only** — use a data cable on the **screen unit's** port) | Vatara field note; costs an hour if forgotten. |

**ESPHome voice-assistant capabilities in 2026 (what the firmware gives us, for free, maintained upstream):**
on-device microWakeWord + VAD, ESP-SR AFE (AEC + noise-suppression + AGC), streaming to HA Assist over the
native API, **media-player ducking** on wake, continued conversation, on-device timers, and stock
listening/thinking/replying screens. **Limits:** two mics + software AEC (above); one on-device wake model
loaded at a time; the box is a satellite, never a standalone assistant; custom microWakeWord models are
"advanced users, expect several runs."

---

## 4. Wake-word feasibility verdict, data/training plan, fallbacks & the spike ⭐

**This is the most important section. The owner's rule: no "it doesn't work" surprises after we build.**

### 4.1 The verdict (blunt)

- **A built-in, pretrained wake word ("Okay Nabu" / "Hey Jarvis") on-device: PROVEN.** Vatara's box completed a
  full real voice turn on a pretrained wake word. This is a `now`, P0 capability. **The product ships on this.**
- *(amended 2026-07-11, round 2 — wake-phrase standardization)* **A custom wake word ("Hey Bonnie Blue")
  on-device: UNPROVEN, and the single biggest risk in this subsystem.** In Vatara we proved the **training
  toolchain end-to-end** — the openWakeWord/microWakeWord
  Docker pipeline resurrected an 8-version-deep 2022 stack and produced a *valid* `.onnx` (loads in onnxruntime,
  correct tensor shape). **But the proof model's Final Recall was 0.0 — it detects nothing.** That was expected
  (1500 synthetic clips, 5000 steps, **zero real child audio**); it proves the *pipeline runs*, not that a
  *usable detector* exists. **We have never demonstrated a working custom wake word on this hardware.**
- **Why custom training commonly fails (industry-honest):** the upstream projects say it plainly — microWakeWord
  is *"intended for advanced users,"* the basic notebook *"will produce a model, but it will most likely not be
  usable,"* and *"training a usable model requires a lot of experimentation."* The failure modes are structural:
  (a) **a child's voice** is the worst case — high pitch, variable prosody, low volume, under-represented in the
  synthetic-speaker pools; (b) **synthetic-only data** doesn't cover the real acoustic target (the child, in her
  room, at her distance and volume); (c) the objective is a **two-sided knife-edge** — you minimise false-accepts
  per hour on ambient noise *and* maximise recall, and the sweet spot is found by hand-tuning class weights and
  thresholds over *many* runs; (d) a 3-word phrase like "Hey Bonnie Blue" (chosen over "Hey Bonnie" precisely to
  cut false wakes) needs the model to learn the *whole* phrase against hard negatives ("Hey Bonnie", "Hey Bunny
  Blue", "Bonnie Blue"). None of this is a toolchain bug we can code around — it is a data-and-tuning problem.

**Design consequence (locked):** **We do NOT design as if custom wake words work until the spike (§4.4) says so.**
Every box ships usable on a built-in wake word + tap-to-talk. "Hey Bonnie Blue" *(amended 2026-07-11, round 2 —
wake-phrase standardization)* is an *upgrade we attempt*, gated on evidence — never a dependency.

### 4.2 The two engines (and why we may need both)

| | **openWakeWord** (host-side) | **microWakeWord** (on-device) |
|---|---|---|
| Runs on | The home server, on streamed audio | The voice box itself, <10 ms, no audio leaves the box pre-wake |
| Difficulty | Easier — mature training, forgiving | Hard — "advanced users, several runs" |
| Privacy | Audio streams to *our own* box (still local) | Best — nothing leaves the box until wake |
| Latency / load | Adds server CPU per always-open stream | Near-zero server load |
| Role in Manas | **Prove the phrase is learnable at all** (faster iteration), and a fallback wake path for non-box surfaces | **The production target** for the box (privacy + load) |

**Strategy:** attempt **openWakeWord first** — it's the cheaper experiment to answer *"is 'Hey Bonnie Blue' even
learnable for the child?"* If openWakeWord can't hit usable recall with real clips, microWakeWord (harder, smaller)
almost certainly can't either, and we stop and ship the fallback. If openWakeWord *does* work, we port the
recipe to microWakeWord for the on-device win.

### 4.3 Data-collection & training plan (what a real "Hey Bonnie Blue" needs) *(amended 2026-07-11, round 2)*

The toolchain exists and is pinned (the durable value from Vatara — the prod run "just works"). The missing
ingredient is **data and tuning**, and specifically **real child audio**.

1. **Positives — synthetic base.** ~30k synthetic "Hey Bonnie Blue" clips via Piper's multi-speaker generator
   (LibriTTS ~900 speakers), the recommended volume (the 1.5k proof run was deliberately tiny). Augment with
   room impulse responses (MIT-RIR), background noise (AudioSet), and gain/pitch/time jitter.
2. **Positives — REAL child clips (the crux).** **Record ~30–50 real "Hey Bonnie Blue" utterances of the child** —
   different distances, volumes, moods, times of day, in *her actual room* — and fold them in (recorder exists:
   `tools/wake/record-clips.py`). **This is the single highest-leverage input; a child wake word without real
   child audio is the known way it fails.** These clips are exactly why training must be **100% local** — a
   child's voice must never touch a cloud trainer. (This aligns with Manas's structural-privacy mandate.)
3. **Negatives — hard + ambient.** ACAV/large negative corpora for general speech + **hard negatives** of the
   near-miss phrases, so the model learns the whole 3-word phrase, not "Bonnie."
4. **Train + tune (the loop that actually matters).** Full config (~30k clips, ~20k steps) on the **the home server GPU**
   (native + fast; the M4 ran emulated + overnight). Then **iterate**: minimise false-accepts/hour on ambient
   first, then push recall via class-weights + detection threshold, **re-testing on held-out real child clips**
   each round. Budget **several full runs**, not one.
5. **Per-child threshold.** Ship the detection threshold as **per-box config** — a child box likely runs a lower
   bar (more sensitive) than an adult's, traded against false wakes. Behaviour is data.

**Realistic expectation to set with the owner:** even done right, a *child* custom wake word is the hard case;
plan for "usable with a slightly lower threshold and the occasional missed/false wake," **not** "flawless like
'Alexa.'" If after several tuned runs recall on real child clips stays low, that is a **legitimate result**, and
we ship the fallback without treating it as failure. (Do **not** cite the old "41% vs 11.6% child-WER" figure —
it was refuted in Vatara's own research verification.)

### 4.4 The spike that decides it (run BEFORE committing the on-device custom-wake design)

> **Spike: "Is 'Hey Bonnie Blue' a viable custom wake word for the child?" — a time-boxed, measured experiment.**

1. **Collect** ~30–50 real "Hey Bonnie Blue" clips of the child (+ ~20 hard-negative clips) with `record-clips.py`.
2. **Train openWakeWord** with the full synthetic set + augmentation + the real clips folded in (local Docker,
   the home server GPU).
3. **Measure on a held-out real-child test set** the two numbers that matter: **recall** (does it wake for her?)
   and **false-accepts per hour** on ambient household audio (does it wake by itself?). *Recall 0.0 = the proof
   state; we need materially better than that on real audio.*
4. **Decision gate:**
   - **Recall usable (say ≥~0.8 on real clips) at an acceptable false-accept rate → GREEN.** Port the recipe to
     **microWakeWord** for on-device, validate on the physical box, ship "Hey Bonnie Blue." *(amended
     2026-07-11, round 2 — wake-phrase standardization)*
   - **Marginal → tune** (threshold, class weights, more real clips) for a bounded number of runs, then re-gate.
   - **Still poor after the budget → RED. Ship the fallback** (built-in wake + tap-to-talk), record the result,
     and revisit only if the upstream tooling improves. **This is an acceptable outcome, decided by data.**

**Until this spike returns GREEN, "Hey Bonnie Blue" is a hoped-for upgrade, not a designed-in assumption.**
*(amended 2026-07-11, round 2 — wake-phrase standardization)*

### 4.5 Fallbacks (the product ships regardless) — ranked

1. **Built-in pretrained wake word** ("Okay Nabu" / "Hey Jarvis") on-device — **proven**, zero training risk.
   The child can use a built-in word; the *persona* is still Bonnie (persona ≠ wake word). **This is the default.**
2. **Tap / push-to-talk** — the hardware button or a big touch target on the 320×240 screen starts a turn with
   **no wake word at all**. Fully deterministic, zero ML risk, and arguably *better* for a young child (explicit,
   no false wakes, no "why won't it hear me" frustration). A strong P0 floor.
3. **Host-side openWakeWord** for non-box surfaces (phone/browser/desk mic) where server load is acceptable.
4. **Revisit custom on-device** only after the spike earns it.

---

## 5. Multi-satellite arbitration design

A house full of always-listening boxes raises real questions. Here is the honest state and the Manas design.

**What HA gives us today (2026):** when several satellites hear the wake word, HA prevents a chorus by letting
the **first satellite to report the wake win** (effectively "fastest device"), and the rest stand down. The reply
returns to the box that captured the turn (`satellite_id`). This is **adequate for a first release** and is a
`now`/P1 capability.

**What HA does NOT give us (be honest):** it is **not** true *nearest*/*loudest* arbitration. "Fastest to report"
correlates loosely with "closest" but can pick the wrong room (a box with a faster network path, or one that
mis-triggers a hair earlier). There is **no built-in "the person is clearly in the kitchen, answer there"**
logic. Treat nearest/loudest as **UNPROVEN / P2**, not a solved feature.

**Manas-side arbitration design (P2, layered on HA, only if the simple path proves annoying):**

- **Short arbitration window.** When box A reports a wake, hold a brief window (tens of ms) for other boxes'
  reports rather than committing to the literal first.
- **Score, don't race.** Rank the reporters by a **wake-confidence + capture-energy (RMS)** score the box already
  computes, so the *loudest/clearest* hearer wins, not merely the fastest reporter.
- **Presence tie-break.** Fuse **Kaya's room-occupancy signal** (mmWave/BLE) — if presence says the speaker is in
  the den, prefer the den box even if the kitchen box scored marginally higher.
- **Sticky conversation.** Once a turn (and any continued-conversation follow-ups) starts on a box, keep it there
  to avoid mid-dialogue room-hopping.
- **Deterministic + config-driven.** The arbiter is pure C# scoring over reported signals — testable, tunable by
  data (weights/window), never an LLM judgment.

**Barge-in & echo:** on-device **ESP-SR AEC** cancels the box's own speaker so a user can interrupt Bonnie
mid-reply. Realistic caveat: with **two mics + software AEC** (no XMOS), barge-in works but is not bullet-proof
in a loud room or at volume — an honest hardware ceiling (§7), acceptable for the home, improvable only by better
hardware (HA Voice PE) later.

**Simultaneous speakers:** two people at two boxes = **two independent Sarathi turns**, each with its own resolved
identity and its own conversation state (Vatara's structural speaker-isolation: `userId` is part of the
conversation key, so no one inherits another's context). Same box, two people talking over each other = one turn,
best-effort STT, and the speaker-of-record is the box's mapped occupant (voiceprint refinement is later, `[P2]`).

**Offline / degraded behaviour:** a satellite must **fail honest, never crash-loop**. Brain unreachable → the box
shows a clear "can't reach home" state, on-device timers keep running, on-device wake still fires (it queues/​drops
the turn gracefully), and it recovers when the brain returns. Directly encodes Vatara's hard-won lesson: a box
**stuck in `responding`** silently stops wake-listening and masks fixes — so the offline state is explicit and
self-clearing (reboot/idle tell), and a failed announcement must **not** retry forever.

---

## 6. Persona-on-device design *(amended 2026-07-11, round 2 — aligns with [persona.md](../architecture/persona.md) §1)*

Sarathi is **one brain wearing a per-member persona** (ADR §5.6). The voice-edge tier is where a persona
becomes *physical* — a voice and a face in a room. **the child's-room box is Bonnie**, the product's north star:
loving, loyal, patient, funny, child-safe.

**Three persona channels, all cue-driven from the centre:**

1. **Voice (Piper).** The reply is synthesised in the speaker's persona voice. *(amended 2026-07-11, round 2 —
   resolved in [persona.md](../architecture/persona.md) §1's favor.)* **Bonnie** uses a **natural,
   child-appropriate Piper voice chosen by A/B listening**; **any pitch/sample-rate manipulation is
   prohibited** (`pitch: 0` enforced by persona validation). The Vatara-era `?pitch=` trick was A/B-rejected as
   "creepy"; the later declared-higher-sample-rate workaround has the same defect and is **retired** — *do not
   re-suggest either*. **Anik** stays adult. Voice is a swappable asset behind config.
2. **Face (the 320×240 mascot).** An animated character on the box LCD — Bonnie the vector stick-girl in Vatara,
   drawn in LVGL, OTA-iterable in ~30s. States: listening / thinking / speaking / celebrating / storytelling /
   idle… A browser **emulator** renders the face off-hardware so art iterates without a flash.
3. **Cue engine (the tie between mind and face).** Each turn Sarathi resolves an animation: the model emits an
   **`[anim:x]` tag**, falling back to **keyword triggers**, else a default — resolved against a **data-driven
   catalog** (`animations.json`, hot-reload; ~25 animations in Vatara). The cue is published to an **HA sensor**
   (`sensor.*_animation`, state = animation id, attrs = speaker) that the box subscribes to via a `homeassistant`
   text sensor. **One brain-side cue vocabulary; every renderer** (console widget, box screen) maps `kind` →
   visuals and may ignore ids it doesn't know. Honest Vatara note: in the tool-calling pipeline the `[anim]`
   instruction can *suppress* a small model's tool call — so in that mode the cue **rides the keyword fallback**,
   not a model tag. Cue derivation must stay robust to the model's quirks.

**Provisioning persona per device (config, not code):** a box's persona = **which member it's bound to**
(`satellite_id → memberId → persona/voice/wake/mascot`), resolved from **Kutumba**. A new persona is **data +
optional voice/mascot assets** — never a code change. **Persona shapes delivery only** — never truth, safety, or
access: the anti-sycophancy rule and the per-user walls bind identically whether Bonnie or Anik is speaking,
and the **input safety gate + output governor** run on the child's turns regardless of how sweet the voice is.

**Safety-first for the child box (non-negotiable):** the child's box converges on the **same `CommandPipeline` input
gate** as every surface (lexical floor → Llama Guard → ShieldGemma child policy), and its replies pass the
deterministic **output governor** before TTS. The persona is the *loving* wrapper; the *guardrails* are
deterministic code underneath. Bonnie being warm never means Bonnie being unsafe.

---

## 7. Integrations & hardware needed

- **the voice box** (`[on-device]` / `needs-hardware`) — the satellite. Multiple units, one per room/zone.
  Two-mic, software-AEC, 320×240 screen (see §3). The proven Vatara platform. Buy N; flash; adopt.
- **ESPHome** (`[integration:ESPHome]`) — the box firmware layer (voice-assistant package: microWakeWord, AFE,
  VAD, ducking, screens), maintained upstream; first flash USB, then OTA. `bonnie-box.yaml` carried from Vatara.
- **Home Assistant + Assist** (`[integration:HomeAssistant]`) — the pipeline shell and the box's single peer.
  Assist orchestrates wake→stt→intent→tts, delegating stt/Sarathi/tts to Manas over Wyoming. HA also owns the
  `assist_satellite.*` services (announce, the fleet's audio probe).
- **Wyoming protocol** (`[integration:Wyoming]`) — the box↔brain transport (one TCP port; stt/conversation/tts).
  Wire format pinned; golden-tested (Vatara `MyJarvis.Wyoming`, to become the Manas Wyoming brain).
- **Medha (LLM gateway)** — the *only* path to STT-repair/LLM/TTS models on the home server, with QoS so the
  **always-listening voice path outranks background batch**. The edge never calls a model directly.
- **Piper (TTS)** — persona voice synthesis (Bonnie: natural child-appropriate voice, no pitch trick — §6;
  Anik adult) *(amended 2026-07-11, round 2)*; a swappable asset.
- **whisper (STT)** — prompted whisper + LLM repair (`RoutingSpeechToText`), per-language routing seam.
- **Kutumba / Kaya** — the box→room→occupant→persona topology and fleet config; Sarathi resolves identity/
  walls from it.
- **Bodha / Saakshi** — fleet health/connectivity + self-report; every voice turn correlation-id'd and audited.
- **Network** (`[SEC-5..7]`, `needs-hardware`) — isolated VLAN, egress-blocked, HA-only peer for every box.
  Assumed by this tier, implemented by platform/ops.
- **Better hardware, later (optional upgrade path, honest):** the **HA Voice PE** (XMOS XU316: hardware AEC +
  far-field beamforming) is the drop-in answer if the BOX-3's two-mic ceiling bites for far-field/barge-in. Same
  Wyoming/Assist path, better ears. Not required to ship; a known lever if room acoustics demand it.

---

## 8. Explicitly out of scope (and why)

- **On-device STT / LLM / TTS.** The voice box physically can't (§3). Cognition is central via Medha. Trying to
  run it on the box is the classic mistake; the box is ears/mouth/face, not a brain.
- **The box actuating anything or holding authority.** A satellite decides nothing, actuates nothing, holds no
  secret that matters. It captures audio and renders a face. All authority (walls, step-up, actuation) is central
  and deterministic — a compromised box is a mic on a quarantined VLAN, nothing more.
- **The wake word as an identity/security factor.** Wake is a *convenience trigger*, not authentication. Identity
  comes from the satellite mapping (later voiceprint); **high-risk actions still require deterministic step-up**
  regardless of who "woke" the box. A child saying an adult's wake word gains nothing.
- **Designing as if custom "Hey Bonnie Blue" works.** *(amended 2026-07-11, round 2 — wake-phrase
  standardization)* Structurally forbidden until the §4.4 spike returns GREEN. The
  product ships on a built-in wake word + tap-to-talk; custom is a data-gated upgrade.
- **Far-field/whole-room magic on this hardware.** Two mics + software AEC ≠ Alexa/XMOS. We set honest
  expectations, mitigate centrally (WavGain, STT repair), and name the HA Voice PE as the hardware upgrade — we
  do **not** promise across-room, loud-room, quiet-child perfection on the BOX-3.
- **True nearest/loudest arbitration as a shipped guarantee.** HA gives "fastest wins"; the scored/presence-fused
  arbiter is a P2 design we build *only if* the simple path annoys — not a day-one promise.
- **Cloud anything.** No cloud wake training (child audio is local-only), no cloud STT/TTS/LLM, no vendor voice
  service. Local-first is a hard platform requirement; the voice tier is one of its sharpest tests.
- **A generic "smart speaker" that free-associates.** The box is a governed surface onto Sarathi, with the same
  safety gate and walls as every surface — not an open-mic assistant that says whatever a model generates.
- **Re-implementing ESPHome/HA plumbing.** We stand on the upstream voice-assistant package (wake, AFE, ducking,
  screens, OTA) and add the persona, the cue engine, and the central brain — we don't fork the firmware layer.
- **Multi-tenant fleet management / productisation.** The tenant seam is present (boxes belong to a home/tenant),
  but selling a managed fleet is deferred per the root ADR.

---

*Sources for the hardware/ecosystem grounding: the voice box hardware overview + dual-mic/16 MB-PSRAM/320×240
specs (espressif/esp-box, Adafruit, CNX-Software, Mouser); ESP-SR AFE/AEC user guide (docs.espressif.com);
ESPHome `micro_wake_word` + on-device ducking (esphome.io); microWakeWord difficulty/"advanced users, most
likely not usable" + false-accept-first tuning (OHF-Voice/micro-wake-word, kevinahrendt.com); HA multi-satellite
"fastest device wins" + on-device microWakeWord approach (home-assistant.io voice_control, smarthomescene.com,
smarthomeexplorer.com). Vatara ground truth: myJarvis ADR 0011 (Wyoming/mascot/box-live field notes), ADR 0012
(speaker resolution & persona trust), and the wake-word memory (`tools/wake/` local Docker pipeline, valid .onnx,
**Final Recall 0.0** proof result). All tailored to the Manas local-first / deterministic / privacy / child-safety
constraints in ADR 0001.*
