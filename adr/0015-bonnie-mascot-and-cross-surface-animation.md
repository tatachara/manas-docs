# ADR 0015 — Bonnie the Mascot & Cross-Surface Animation (web + the voice box)

- **Status:** **ACCEPTED (owner, 2026-07-18)** — with owner riders: (1) the component name **Mukha** (§11) is
  CONFIRMED; (2) the animation subsystem gets **its own ADR (0016 — Mukha)**: the registry must be flexible to
  add new animations, and personas beyond Bonnie (Anik, Vani) may carry their own mascot/animation sets in the
  future — the §10 two-layer split (universal core + per-mascotSet extension) is the seed of that design; (3)
  owner sees a Bonnie concept render before build starts. *(Previously: Proposed, three-lens review applied §13.)* — the mascot was a `P1`/`P2` line in
  `voice-edge.md` §2.4 with the right *skeleton* (closed cue vocabulary, box-is-its-own-frame-renderer, browser
  emulator) but no hardware-budgeted rendering design and no answer to the specific failure that killed the
  predecessor. This ADR is that design. **It must be Accepted before any box mascot firmware is built.**
- **Date:** 2026-07-18
- **Deciders:** Ananth (owner — pending), Claude (orchestrator)
- **Refines:** `voice-edge.md` §2.4 (mascot face, cue engine, off-hardware preview), ADR 0011 §68 + ADR 0012 §3e
  (the **closed cue vocabulary**; "the box's 320×240 face is its own frame renderer and shares only the cue
  vocabulary" — this ADR keeps that and hardware-budgets it), ADR 0010 §D (the `[anim:x]` cue on the egress
  `Final { …, animCue }`), ADR 0014 §3f + unit E5 (idle-face governance — whimsy yes, attention-seeking guilt
  no), ADR 0014 §3d + unit E3 (quiet hours → Bonnie sleeps), persona.md §5 (delivery-only; the mascot is
  never a wall/truth/safety boundary). Cross-refs the risk register (#1 wake, and the new mascot rows below).

> **This ADR exists because a mascot designed without the hardware killed the predecessor.** Ananth's own words
> (2026-07-18): Jarvis "spent months trying to get this to work reliably without success… it was designed very
> poorly without understanding the hardware constraints or the hardware of the voice box," and the concrete
> symptom was: **"when the box was replying it reverted back to the voice box default animation."** That single
> symptom is the whole diagnosis — the stock voice-assistant firmware's display state machine was preempting the
> custom face during the listening/thinking/speaking states. This ADR's #1 decision makes that structurally
> impossible, and every rendering choice below is derived from what a **320×240 SPI display at a 40 MHz pixel
> clock, no GPU** can *actually* do, with a measured spike (on the voice box Ananth now has) whose green gate is
> literally "the face does NOT revert to default during a reply." **This is rule #6 in its purest form: name the
> sharp edge now, gate it, give it a fallback — never discover it after building.**

---

## 1. Context — what exists, what Jarvis got wrong, what changed

**The hardware (measured, not remembered) — the voice box:**
- 2.4″ **320×240 SPI IPS** display (ILI9341-class), capacitive touch; **40 MHz** LCD pixel clock.
- the voice box dual-core Xtensa LX7 @ 240 MHz, **512 KB SRAM**, **16 MB flash**, **16 MB Octal PSRAM**. No GPU.
- Dual mic (ES7210 ADC), **no XMOS DSP** (software AEC), speaker, 3 buttons.
- LVGL is the supported UI stack (BSP allows LVGL buffers DMA-capable or in PSRAM).

**The reality — and the RIGHT bottleneck (hardware review H1, corrected):** a full 320×240 RGB565 frame =
320·240·16 = **1.23 Mbit**; at the ESPHome `ili9xxx` driver's default 40 MHz `data_rate` that is a **~30 fps
theoretical / ~15–20 fps realistic** full-frame ceiling. (The 40 MHz is the driver default, not a silicon
ceiling — the S3 SPI bus can run to ~80 MHz — but it doesn't matter, because **SPI is NOT the binding
constraint.**) A 12 fps face-*region* clip needs ~0.6–1.8 MB/s of SPI against a ~5 MB/s budget — comfortable.
**The real ceiling is the dual-core LX7 CPU DURING A REPLY**, and there is no GPU: ESP-SR's software AEC pins
`feed` to Core 0 and WakeNet `detect` to Core 1 — **AEC-on uses both cores** (measured ~42% of a core per 16 ms
frame), leaving no free core for LVGL software rendering. So the **`speaking` state — animate WHILE the mic AEC
+ TTS playback run — is the single most CPU-contended cell, and it is the exact state Jarvis glitched in.**
**Conclusion, load-bearing: you cannot real-time-*render* a character on this box (that is compute, not
bandwidth), AND animated character motion concurrent with the reply's AEC load is the tightest budget — so the
`speaking` face is designed static/low-frame by default (§5), full motion only where S-0015b proves headroom
with AEC on.** You *can* reliably *play back* baked clips for the cheap (idle/listening/whimsy) states. Jarvis's
error was treating the box like a browser AND vector-drawing during the reply.

**Forensic evidence from the predecessor (myJarvis, read-only — NO Jarvis code enters Manas, per the standing
owner rule `jarvis-test-treasure-trove`).** The failure is not a mystery; the abandoned code shows exactly two
root causes, and the fix here is derived from them:
1. **They built ON the stock ESPHome voice-assistant package that OWNS the screen.** `bonnie-box.yaml` header,
   verbatim: *"ESPHome's official BOX-3 voice-assistant package (mics/codec/display/touch all set up, **with
   built-in listening/thinking/replying screens**). Our overlay adds the … animation cue sensor … the hook for
   Bonnie's custom screens."* Their own backlog names the symptom and root cause: *"[BOX-FLASH] flash-to-default
   + occasional blank. Root cause: the [stock] listening/thinking/replying pages … the stock frame is visible
   for ~1 frame (the flash)."* — **this IS the "reverts to the default animation when replying" symptom**: the
   stock display path repaints for ~1 frame on every phase change. Their attempted fix (*"draw_display is now the
   SOLE panel write, one flush per turn = no flash"*) was marked **"FIXED (code, needs OTA confirm)" — and never
   confirmed on hardware before the project was abandoned.** → §3 chooses the LVGL-custom path where
   `voice_assistant` writes NOTHING to the display (verified feasible, §13), not the stock package; and §9
   S-0015a makes on-real-hardware no-revert the blocking gate they never cleared.
2. **They VECTOR-DREW the character on-device at runtime.** `bonnie-box.yaml`, verbatim: *"the box draws Bonnie
   (the blue stick-girl) in the matching pose, **vector-drawn on-device — no image assets**."* That is why it
   could only ever be a *stick-girl* (procedural on-box compute affords nothing richer) and why it was fragile —
   exactly the compute Jarvis couldn't spare during a reply. → §4 bakes the art into clips (personality in art,
   not runtime rig-eval); the rich cowgirl is bought in flash (§4 budget), not box CPU.
3. **What they got right (adopt it):** a **30-cue closed vocabulary** (`animations.json`: speaking, listening,
   thinking, storytelling, yawning, sleeping, eating, watering, peepee, poopy, dancing, singing, celebrating,
   surprised, hugging, searching, doctor…) and a **real off-hardware SDL emulator** (*"[BOX-EMU] DONE — a real,
   rerunnable satellite emulator … captures the SDL frame headlessly to a BMP"*). The cue *design* was never the
   failure; the *rendering* was. This ADR keeps the closed-vocabulary + emulator ideas and fixes the rendering.
   (Governance note: their set included a `sad-gentle` idle — precisely the state §7 gates as a potential
   attention-hook; comfort-in-response is fine, sad-as-idle is not.)

**What's already decided right (keep it):**
- **Closed cue vocabulary** (ADR 0012 §3e; `src/Apps/Manas.Console/src/lib/cues.ts`): a turn emits ONE cue from a
  *fixed shared set* — no bespoke avatar states, so there is never a simultaneous-cue conflict. (Today it's only
  3 cues; this ADR expands it — see §6 — while keeping it CLOSED.)
- **The box is its own frame renderer, sharing ONLY the cue vocabulary** (ADR 0011 §68 / ADR 0012 §3e:330): the
  web renders with Rive; the box plays its own baked frames; they share the *cue names*, not the `.riv` file.
- **Off-hardware browser emulator** (`voice-edge.md` §2.4:114, Vatara's `box-emulator.html`): iterate the art in
  a browser without a flash — de-risks the flash-only art loop. **Keep and make it first-class.**
- The cue transport: Sarathi emits `[anim:x]` → `Final.animCue` (`EgressEvent.cs:64`, `null` today) → web
  consumes it (`cues.ts` `resolveCue`); the box path is an HA sensor the firmware subscribes to (§8).

**What changed (2026-07-18, owner):** the character is upgraded from "Bonnie the stick-girl" (`voice-edge.md`
§2.4:110) to a **rich, funny cowgirl** with a personality-loop of whimsical states. That richness raises the art
budget but — because the personality lives in **pre-baked art**, not a runtime engine — it does **not** raise the
box's per-frame compute. The constraint is **flash/PSRAM budget for the clip library**, which is a measured gate
(§9), not a hope.

---

## 2. The character — Bonnie (owner-specified)

**Visual identity (owner, 2026-07-18):** a little **cowgirl** — **red hair in a single Jessie-style braid over
the shoulder with a BLUE RIBBON bow** (owner refinement, 2026-07-18 night: "the Jessie look from Toy Story" —
*inspired-by, never a copy*: Bonnie keeps her own blue-shirt/yellow-pants identity), **blue shirt**, **yellow
pants**, **cowgirl costume** (hat, boots). **Concept direction APPROVED by the owner (green light, 2026-07-18
night) on the v2 sheet at `docs/design/bonnie/bonnie-concept-v2.{html,png}`** — the character-spec seed the M2
artist bakes from. This is Bonnie's canonical look on **every** surface (web + box); it is
per-persona config (the child's box = Bonnie), never hardcoded, resolved from Kutumba (persona.md §5, `voice-edge.md`
§2.4). Bonnie is the north star (CLAUDE.md Part V) — the art direction is warm, expressive, funny, never creepy
(no uncanny realism; the pitch-shift "creepy voice" ban of §6 has a visual analogue — see §7).

**The state set (the personality loop) — three families:**
1. **Conversation cues** (driven by the turn lifecycle, deterministic): `idle`, `listening`, `thinking`,
   `speaking`, `celebrating`, `try-again`, `storytelling`, and the **honest-degraded** states the S2 voice matrix
   already needs — `cant-hear` (mic/STT down) and `need-a-grown-up` (the pre-rendered refusal/box-death cue).
2. **Personality / idle whimsy** (owner-specified, delightful): **poop break**, **reading**, **queasy / getting
   vomity**, **thirsty**, **drinking water**, **getting sleepy**, **bored**, plus the whimsy the predecessor's
   30-cue set already proved worth having (**yawning**, **eating a snack**, **dancing**, **humming/singing to
   herself**, **searching/looking for something**, **stretching**, **waving hello**, **surprised**). These play
   in the ambient loop when Bonnie is not in a turn — Bonnie *living her little life*. They are *whimsy* — see the
   §7 governance line that separates whimsy from an attention-hook. **⚠ HARD CONSTRAINT (owner, 2026-07-18): the
   box's MEMORY is the ceiling on how many of these ship** — each state is a baked clip competing for the 16 MB
   flash budget (§4/§9 H3). The whimsy set is therefore **prioritized and incremental**: a v1 core ships first,
   the rest land as the measured flash budget (S-0015b) allows. More states is an art+budget question, never a
   compute question (§4).
3. **Circadian** (composes with E3 quiet hours, deterministic): **sleeping at night**, **waking in the morning**.
   Bonnie genuinely sleeps during the child's quiet-hours window and wakes when it ends — the animation reads the same
   `QuietHoursPolicy` (E3) that already governs the surface. No new clock, no LLM.

Each state is a **closed-vocabulary cue** (§6) mapping to a baked clip per surface. The set is versioned data
(`animations.json`), grows by art PRs, and is **reviewed** (no cue is a dark pattern — §7).

---

## 3. THE load-bearing decision — Manas owns the display; the face never reverts

**Decision (non-negotiable; this is the Jarvis fix): on the box, Manas' cue state machine is the SOLE authority
over the screen. Bonnie's animation is never preempted by the voice-assistant firmware's default display state —
NOT during listening, NOT during thinking, and CRITICALLY NOT during TTS playback.**

- The predecessor used the stock voice-assistant UI (ESPHome/`wyoming-satellite`'s built-in display), which owns
  the screen during its own `listening → thinking → replying` states and **reverts to its default face** — that
  is the "reverted to the voice box default animation when replying" symptom, verbatim.
- **The fix:** the box runs a **custom Manas display component** (ESPHome custom component / LVGL, or a
  purpose-built firmware) in which the Manas cue state is the *only* writer to the framebuffer. The
  voice-assistant pipeline's display hooks are **disabled/overridden** — the pipeline still does wake/capture/
  playback, but it **does not touch the screen**. TTS playback and the "speaking" face are driven by the SAME
  Manas cue state, so audio and face are one coherent state, never two fighting owners.
- **Acceptance (the spike, §9): S-0015a — the face does NOT revert to a default during a full reply.** Green =
  across a wake→capture→think→speak→idle round trip on the real voice box, the screen shows ONLY Manas cues, with
  the `speaking` clip playing throughout TTS, and returns to Bonnie's idle (never a stock face). This is the
  literal thing Jarvis never proved; nothing else in this ADR ships until it is green.
- The web surface has no competing owner (it's our React app), so this decision is box-specific; but the web and
  box share the cue state so both are always showing the same Bonnie doing the same thing.

---

## 4. The rendering approach — personality in the art, not a runtime engine

**Decision: both surfaces play PRE-BAKED animation clips selected by cue. Neither surface renders the character
procedurally from a rig at runtime on the box.** The art is authored once (by an artist / an asset pipeline) into
per-cue clips; the runtime just plays the clip for the current cue.

- **Box renderer:** each cue → a baked clip = an ordered set of frames, **stream-decoded per-clip from flash**
  and pushed (only the animated *region* — the face box, not the full 320×240) over SPI at a budgeted fps (target
  ~12 fps for the cheap states; **`speaking` static/low-frame, per §1/H2**). **Leading codec candidate (hardware
  review H6): a custom RLE region frame-player streaming from flash** — LVGL `animimg` holds all of a clip's
  frames resident in RAM (it multiplies the memory pressure below), so it is the *fallback*, not the default;
  full-screen GIF decode is excluded. **No procedural rig, no vector tessellation, no per-frame compositing on
  the box** — that is the compute Jarvis burned on a stick-girl. *(Correction, H3: clips stream-decode from the
  16 MB flash; they are NOT held decoded in the 16 MB PSRAM — PSRAM hosts the audio buffers, the AFE working set,
  and the LVGL draw buffers.)*
  - **Flash budget — the arithmetic (H3, the number that must be respected NOW, not "discovered" in the spike):**
    a full RGB565 frame = 150 KB; a ~160×160 face region ≈ 50 KB. For ~20 clips (the §2 set + `[anim:x]`
    variants) at 12 fps × 2 s = 24 frames: **uncompressed full-frame ≈ 72 MB (4.5× over 16 MB flash) — a
    non-starter; uncompressed face-region ≈ 24 MB — still over.** Only **compressed (RLE/PNG, ~3–6× on flat
    cartoon art) AND region-limited AND/OR short clips** fits (~5–8 MB). So **compressed + region is the
    BASELINE, not the fallback** — and the whimsy count is memory-bounded (§2 owner constraint): v1 core states
    first, the rest as the budget allows.
- **Web renderer (Akriti):** richer — **Rive** (`RiveAvatar.tsx`) with the honest **Motion `FallbackAvatar`**
  already in place (`cues.ts`). Higher resolution, smoother, can be procedural — the browser can afford it. It
  consumes the exact same cue enum, so the web and box are always the same Bonnie in the same state.
- **One character spec, two renderers, shared cue vocabulary** (keeps ADR 0011 §68 / 0012 §3e): the `.riv` is NOT
  shipped to the box; the box has its own baked frames. What is shared is the **cue names + the character design
  spec** (the canonical Bonnie look + a per-cue behavior description the artist bakes on each surface). A
  **golden-frame check** (§9) asserts the two renderers agree on identity per cue (same costume, same state read).
- **Art iteration without a flash:** the **browser emulator** (§2.4:114) renders the box's *baked clips* at the
  box's resolution/fps so art can be iterated and reviewed (by Ananth, screenshot-loop — the standing premium-UI
  process) **without reflashing**. The flash step is only for the final on-hardware confirmation. This directly
  attacks the second Jarvis time-sink (flash-only iteration).

---

## 5. State authority — who picks the cue, and coherence

The current cue is a **single deterministic value** owned by Manas, resolved in strict precedence (no two owners,
so no glitch). **The precedence mirrors ADR-0014 §3a's gate precedence — distress beats everything, including
sleep (review finding D, the fix for the 2 a.m. regression):**

1. **Distress / safety comfort — ALWAYS wins, including at night (composes with E1 + ADR-0014 §3d).** Whenever the
   E1 distress carve-out fires — including the **non-verbal night proxy** (crying / repeated wake-with-no-utterance
   at night, ADR-0014 §3c CS-F8), which forms **no active turn** — the face shows the **comfort / `need-a-grown-up`**
   cue, NOT `sleeping`. **Invariant: distress beats sleep — a distressed child is NEVER shown a sleeping Bonnie.**
   The comfort face is reachable from the **alert-only side-channel**, not only from an active turn (so the
   non-verbal night proxy reaches it). Acceptance: `Distress_at_night_wakes_the_face_never_sleeping` (the
   ADR-0014 §7.1 kill-shot analogue).
2. **Turn lifecycle (deterministic):** an active turn drives `listening → thinking → speaking → (celebrating |
   try-again | comfort | need-a-grown-up) → idle`, from the turn spine's stage + the Governor result. `speaking`
   is bound to TTS playback (§3; static/low-frame per §1/H2). Degraded states (`cant-hear`, `need-a-grown-up`)
   come from the S2 voice-degrade matrix — the face has an honest state for **every** degraded cell (never a
   frozen/blank screen — the box-death → child-comprehensible-state Phase-3 exit criterion).
3. **Circadian / quiet-hours (E3):** ONLY when no turn is active AND no distress (items 1–2 did not fire) and the
   surface is in the child's quiet-hours window → `sleeping`; at the window's end → `waking` (once) → `idle`.
   Deterministic, from the SAME `QuietHoursPolicy` E3 already governs (no second copy — review finding).
4. **LLM `[anim:x]` flavor (advisory, bounded — review finding E):** within `speaking`/`storytelling`, Sarathi MAY
   emit an `[anim:x]` tag to pick an *expressive* variant **from the affect-neutral, `llmSelectable:true` subset
   only** (storytelling gestures — never the bodily-need / circadian / forlorn states, which the model could pair
   with a warm goodbye to evade the S-0014d word-scan). `cues.ts` fail-closed: unknown OR non-selectable tag →
   the deterministic cue. The model flavors a narrow set; it never controls a boundary or a parasocial cue.
5. **Ambient idle loop (deterministic scheduler) — PRESENCE-BLIND and HISTORY-BLIND (review finding B/G,
   structural):** when no turn is active and not in quiet hours, a deterministic scheduler rotates the whimsy
   states (`reading`, `thirsty`, `drinking water`, `bored`, `poop break`, `getting sleepy`…). **The scheduler
   takes NO input from mic energy, presence detection, or per-child arrival-time — its rotation is fixed/random
   from config, so it can never manufacture a "perks up when you arrive / droops when you leave" return-hook via
   timing** (a face driven by the bedroom mic would also violate the WW MicGate "nothing pre-wake" invariant).
   Architecture-tested like ADR-0014 §3f-3 (no-reward-state).

**The whole screen is one owner (Manas), one current cue, strict precedence — so there is never a "two animations
fighting" glitch.** This is the same single-owner discipline that killed the Jarvis symptom at §3, applied to cue
selection.

---

## 6. The closed cue vocabulary — expanded, still closed

`animations.json` (versioned data, hot-reload, per-surface baked-clip refs) defines the closed set. The **shared
enum** (web `cues.ts` + the box firmware + `animations.json`) is expanded from today's 3 to the §2 state set. The
**closed** property is load-bearing: only these cues exist, both renderers implement all of them, an unknown cue
fails closed to `idle` (web) / the safe default (box), and a fail-closed loader refuses a pack that references a
cue a renderer doesn't have (a box that can't render a cue must never be sent it — the missing-clip analogue of
the S2 no-dead-air rule). Adding a state = an art PR that bakes the clip on **both** surfaces + adds the enum
member + passes the §7 governance review. No code change, no `.riv` surgery for a new *variant*.

**§6.1 — the BINDING (the mappings; review F3/F5/F6/F7 — "a closed set of names with an unspecified binding IS
the composition, and today it's safety-wrong").** The current web `cues.ts` collapses **all ~12 `RefusalKind`s
to `try-again`** — so today a self-harm disclosure (`ChildDistressComfort`) would render a *retry-nudge* face
(F3, a rule-#9 defect). The expansion is not enum-only; it ships the NORMATIVE binding table:
- **A first-class `comforting` cue** (F3): `ChildDistressComfort → comforting` (a warm, gentle, present face —
  never `try-again`, never `idle`). The safety-cue **fail-closed target is a neutral-concerned face, NEVER
  `idle`/happy** — a distressed child must never see a cheerful fallback.
- **The `RefusalKind → cue` table** (F6): `EngagementWindDown → getting-sleepy`, `QuietHours → sleeping`,
  `GrownUpRedirectMedical|Hazard → need-a-grown-up`, `ChildDistressComfort → comforting`, `ChildTrendNarration
  → thinking`, `SafetyBlock/TierRefusal/WallDenial/ToolRefused → a calm can't-do face`, `Locked/
  DependencyUnavailable → the box-fault face`. No kind silently degrades to `try-again`.
- **The S2 degraded-cell → cue table** (F7): every `agent-spec` §F cell gets a face — TTS-down → "voice
  resting", STT/mic-down → `cant-hear`, responder/Medha-down → a `sleepy-thinking` face, safety-models-down →
  the child-nap face, **and `need-a-grown-up` is SPLIT from a `box-fault` cue** (a broken box and a "this is a
  grown-up question" are different drawings — F7 overloading fix).

**§6.2 — the AMBIENT cue channel (F1, BLOCKING).** `Final.animCue` is **turn-terminal** — it cannot carry a
*between-turns* cue (sleeping/waking/whimsy), so the web today has **no channel for an ambient cue at all** and
the "same Bonnie both surfaces" claim is false for every non-turn state. Fix: add a standalone **`AmbientCue {
cue }`** push to the egress (decoupled from `Final`) + a matching web subscription; the box's HA-sensor path
already carries standing cues. One **server-side cue authority** publishes to BOTH transports (F4 — no
client-side scheduler, no split-brain; the idle scheduler runs server-side).

**§6.3 — LOCALLY-generated fault face (F2, BLOCKING).** All box cues route over the HA sensor — but the state
that matters most, **HA/server unreachable (box-death)**, is exactly when that channel is DOWN, so the box can
never be *told* to show the fault face. Fix (firmware requirement + spike gate): **the box renders the
`box-fault`/"can't reach home" face LOCALLY on loss-of-cue-channel** (a heartbeat timeout on the HA sensor),
never depending on the dead channel to deliver its own death notice. This is the box-death →
child-comprehensible-state Phase-3 exit criterion, made real.

**§6.4 — fail-closed mechanism, stated honestly (F9).** "Unknown cue fails closed to `idle`" is only true on the
**server** (the cue authority clamps to the safe default). On the web, `FallbackAvatar.POSE` is compile-
exhaustive (a new enum member without a POSE is a **build break**, not a runtime fallback) and `RiveAvatar`
no-ops a `.riv` missing the state (caught by the ADR-0012 §4 `.riv` build-check). Three mechanisms, one runtime
fail-closed (server) — stated so no one relies on a fallback that isn't there.

---

## 7. Governance — delightful, never a dark pattern (composes with E5 / ADR 0012 §3e)

The funny states are the *point* — they make Bonnie feel alive and loved. But an always-on face in a child's
bedroom is exactly where a mascot becomes a Tamagotchi guilt-engine, and unit **E5** already banned that class in
the idle-face governance. The line, stated once:

- **ALLOWED (whimsy — self-contained personality):** poop break, reading, thirsty, queasy, getting sleepy,
  sleeping, waking, a *silly* bored shrug, celebrating a turn. These are Bonnie *being herself*; they ask nothing
  of the child.
- **BANNED by construction (attention-seeking / guilt / solicitation):** any idle state that **pleads, sulks,
  looks lonely/abandoned, or lures the child back** — "I'm so bored without *you*", sad-puppy-come-play,
  visible "perks up when you walk by and droops when you leave." `bored` ships as a *self-amused* shrug, NOT a
  guilt look. This is the **E5 idle-cue governance**: the closed cue set is reviewed (owner + the review lens)
  and no cue may be an engagement hook; it composes with ADR 0012 §3g (no proactive child channel) — the face
  never *initiates* a bid for attention.
- **No uncanny/creepy visual** (the visual analogue of §6's retired pitch-shift ban): warm, cartoon, clearly a
  friendly drawing — never a realistic or unsettling render.
- **Quiet hours = genuinely asleep** (E3): during the child's night window the face sleeps; it does not glow, pulse,
  or invite. A pre-literate child seeing Bonnie asleep is a *cue to sleep*, not a companion still awake.

**Honest gate (review A — do NOT over-claim E5).** E5's `reengagement-lint` scans **text wording** under
`config/personas/**` + `config/engagement/**` — it **cannot** see that a *baked clip's art/motion* (drooping
eyes, a lonely posture, a perk-then-droop) is a guilt look, and `animations.json` isn't in its manifest. So the
**load-bearing gate is OWNER ART SIGN-OFF against a per-cue governance rubric** (is the resting posture neutral?
does the clip resolve on-screen *without* the child? no perk/droop keyed to arrival? not frightening to a
pre-literate child in the dark? — review H). A text-lint MAY cover cue *names/descriptions* (add
`config/animations/**` to the E5 manifest if so), but it is honestly labeled as **NOT** covering the art. The
**structural backstops** (these are real, in code): the face has **no proactive channel** (§5, presence-blind/
history-blind scheduler, arch-tested); and **bodily-need states resolve themselves on-screen** (review C) —
**Bonnie drinks her own water, naps herself; the child is NEVER the agent who fixes her need** (no touch
"feed/give-water" affordance — open-Q4). Whimsy that Bonnie resolves alone is delight; a need only the child can
satisfy is a Tamagotchi care-loop, banned.
- **Fright/dark review (H):** the `queasy/vomity` clip and the night face get an explicit "not frightening to a
  pre-literate child" art-review criterion; the quiet-hours night state is preferably **fully dark (screen
  off/minimal)** rather than a lit sleeping face a dark-adapted child could find unsettling (this also composes
  with the S-0015b static/off `speaking` budget).
- **Web renderer parity (review F):** the governance bans are an acceptance item on **both** renderers; the web
  Rive is constrained to **cue-driven state only — no pointer/scroll/presence inputs** (a Rive state machine
  that "watches the cursor" is an un-cued needy behavior with no box analogue). S-0015e asserts the Rive machine
  exposes no input outside the cue enum.

---

## 8. The cue → surface transport

- **Web:** unchanged and already built — `Final.animCue` on the GraphQL egress; `resolveCue` (`cues.ts`) fail-
  closed; RiveAvatar / Motion fallback. This ADR only *expands the enum* and *authors the `.riv`* (the SDD's
  "real .riv + backend cue emission = later" becomes now-ish).
- **Box:** Sarathi publishes the current cue to an **HA sensor** (`voice-edge.md` §2.4:111); the box's Manas
  display component subscribes and plays the cue's baked clip. The turn-lifecycle cues are emitted by the same
  spine that streams the reply, so the cue and the audio arrive coherently on the *same* satellite (keyed by
  `satellite_id`, §2.2). **The cue channel is delivery-only** (persona.md §5) — it carries a cue name, never
  content, never a boundary; a lost/garbled cue degrades to `idle`/safe, never to a wrong action.
- **Firmware base:** the search shows `wyoming-satellite` is superseded by **ESPHome / OHF `linux-voice-
  assistant`** for new features (media player, start/continue conversation, display). **The firmware base is a
  spike decision (S-0015c)** — ESPHome custom display component vs a purpose-built firmware — gated on which one
  lets Manas fully own the screen (§3) while keeping the Wyoming/HA voice pipeline. The requirement drives the
  choice, not the other way round.

---

## 9. Measured spikes + fallbacks (rule #6 — the anti-Jarvis)

**Nothing about the box mascot is designed as if it works. Each unknown is a spike with a green gate and a
recorded fallback, run on the voice box Ananth now has.**

| Spike | Green gate (proceed) | Fallback if red |
|---|---|---|
| **S-0015a · No-revert-during-reply** (THE Jarvis test) — **✅ GREEN 2026-07-18 ~23:29 EDT, OWNER-OBSERVED on the real voice box**: spike firmware (the satellite firmware spike tooling, the `draw_display` remove-and-replace — stock pages structurally unreachable, verified in the resolved config) ran a full wake→think→speak→idle cycle; the owner watched ONLY Manas states the whole way, SPEAKING held through the entire TTS reply, boot shows the Manas frame (owner: "green, forest green"); audio side cross-verified by the pipeline trace (03:28:57Z "What is the current time?" → "11:29 PM"). Mechanism finding recorded: `!extend` on a script CONCATENATES (stock body would run first = a guaranteed 1-frame flash); `!remove`-then-redefine is the only true replacement. **Box-side mascot work is UNBLOCKED; S-0015b (fps under reply load) is now the next box spike — the spike pages were static text, fps was deliberately NOT exercised.** | Across a full wake→think→**speak**→idle round trip on the real box, the screen shows ONLY Manas cues; the `speaking` clip plays throughout TTS; returns to Bonnie idle — never a stock/default face | If the firmware can't cede the screen: a purpose-built firmware where Manas owns the panel outright (drop the stock display component entirely); worst case, a **static Bonnie pose** held during replies (no motion) — coherent and never-reverting beats glitchy motion |
| **S-0015b · Frame/clip budget + fps UNDER REPLY LOAD** (H1/H2/H3) | (a) the cheap states (idle/listening/whimsy) play at ≥ ~12 fps on the region; (b) **the `speaking` clip's fps is measured WITH AEC enabled + TTS playing (barge-in on)** — full motion only if there is headroom, else static/low-frame is the accepted baseline; (c) the **compressed + region-limited** clip set for the v1 states fits 16 MB flash with headroom | `speaking` = static-or-2-frame (the H2 baseline — coherent, never-reverting beats glitchy); face-only region; fewer frames/clips; fewer whimsy states in v1 (conversation + sleep first); 2-frame blink loops |
| **S-0015c · Firmware base** | ESPHome **LVGL custom display component** (verified §13: `voice_assistant` writes nothing to the display) — or `linux-voice-assistant` — lets Manas fully own the screen (S-0015a) AND keeps the Wyoming/HA voice pipeline + OTA. **Do NOT build on the stock BOX-3 voice-assistant package with built-in screens (the Jarvis trap).** | Purpose-built firmware (more work, full control); recorded as the fallback, not discovered late |
| **S-0015d · Art-without-flash loop** | The browser emulator (the predecessor's `[BOX-EMU]` idea, kept) renders the box's baked clips at box res/fps + captures a frame headlessly so Ananth reviews art via the screenshot loop with no reflash | Accept a slower flash-to-review loop for final art only; keep the emulator for iteration |
| **S-0015e · Golden-frame cross-surface identity** (H5 — demoted) | **Manual owner art review per cue is the BASELINE** (web Rive vs box baked are different renderers — a pixel/perceptual diff is fuzzy; do not sink eng time into a fragile identity classifier); an automated "costume tokens present" check is nice-to-have only | Manual review (already the baseline) |
| **S-0015f · OTA of the clip-library partition** (H4, new) | A full clip-set update OTAs to the box without bricking, within an acceptable window (the multi-MB clip library breaks voice-edge §2.5's "~30 s OTA" — that line is for small firmware, not an art payload) | Ship art on a separate flash partition updated out-of-band; accept a slower art-OTA than the firmware-OTA; reconcile the "~30 s" claim in voice-edge §2.5 |

Proposed **risk-register rows #33–#34**: **#33 — box mascot reliability (the Jarvis scar):** *High · 🔴 until
S-0015a/b green on the real voice box* — validation = the spikes above; fallback = static-pose Bonnie + face-only
animation, which ships regardless. **#34 — mascot art production** (a real cowgirl-Bonnie clip set is an art
deliverable, not code): *Med · 🟡* — validation = the emulator loop + owner art review; fallback = a simpler v1
clip set (the whimsy states land incrementally). Neither blocks the voice path (voice ships with a minimal face).

---

## 10. Phasing — what's dev-Mac/emulator vs on-the voice box

- **Now, dev-Mac/emulator (no box needed):** expand the closed cue vocabulary + `animations.json` schema; author
  the web Rive Bonnie (cowgirl) + the Motion fallback for the full state set; wire `Final.animCue` emission from
  the spine (turn-lifecycle → cue, deterministic; `[anim:x]` advisory); the E3 quiet-hours → `sleeping`/`waking`
  binding; the E5-style cue governance review; the browser emulator rendering box-resolution baked clips.
- **On the voice box (Ananth has it) — the spikes:** S-0015a (no-revert), S-0015b (budget/fps), S-0015c (firmware),
  S-0015e (golden-frame). **S-0015a is the gate for everything box-side.**
- **The voice path does not wait on the mascot:** per `voice-edge.md`, the box ships a *minimal* coherent face
  first — but the minimal set is **conversation + the SAFETY-DEGRADED cues** (`idle`, `listening`, `speaking`,
  `sleeping`, **`comforting`, `need-a-grown-up`, `cant-hear`, `box-fault`**), NOT just idle/listening/speaking/
  sleep (review J): the whimsy can land incrementally, but the safety/degraded faces are **Phase-3-blocking**
  (box-death → child-comprehensible-state; the E1 comfort face) and ship in v1. The rich whimsy set lands
  incrementally behind the emulator loop.
- **Persona split (review F8):** the closed vocabulary is TWO layers — a **universal core** (conversation +
  degraded + circadian, which EVERY persona/mascotSet implements) + a **per-mascotSet whimsy extension** (Bonnie
  has the cowgirl whimsy; a dignified adult Anik need not render a poop-break; Guest gets the core only). Each
  layer stays closed; §6's "all renderers implement all cues" applies per-layer, resolving the Bonnie-only-
  whimsy vs shared-vocab tension.

---

## 11. Open questions (owner may override)

- **Art production:** who bakes the cowgirl-Bonnie clips — a commissioned artist, a generated-then-cleaned
  pipeline, or Ananth? (Affects §9 #34. The *architecture* here is art-source-agnostic; the emulator loop is the
  review surface either way.)
- **Component name:** the mascot/animation subsystem wants a Sanskrit name per the naming convention (unit 0.N
  precedent) — e.g. **Mukha** (मुख, "face") — owner to confirm.
- **How many whimsy states in v1** (S-0015b budget-dependent) and their exact behaviors — the funny set is
  owner-curated (this is the delight; Ananth designs the jokes).
- **Touch interactions with the face** (tap Bonnie → she reacts) — in scope for the mascot or a later polish unit?

## 12. Consequences

**Positive.** The one subsystem that killed the predecessor gets designed hardware-first: the exact "reverts
during reply" failure is made structurally impossible (§3, single screen owner), the character's richness is
bought with *art*, not box compute (§4), and every box-side unknown is a measured spike with a fallback that
ships regardless (§9) — so there is no path to "we built it and it doesn't work." Bonnie becomes a warm, funny,
genuinely-alive companion whose sleep, waking, and whimsy compose with the deterministic policy layer already
built (E3 quiet hours, E5 idle governance).

**Costs (honest).** Real art production (a cowgirl-Bonnie clip set is weeks of asset work, not a code unit) and
custom box firmware (S-0015c) — both named as risks with fallbacks, neither blocking voice. The closed cue
vocabulary must be implemented on *both* renderers for every state (the discipline that prevents glitches has a
per-state cost). The mascot is `P1`/`P2` — it ships after the voice path, incrementally.

**Constrains.** `voice-edge.md` §2.4 (mascot rows point here), the SDD (the mascot S-units + the S-0015 spikes
become Phase-3 items; box-side gated on hardware), persona.md (Bonnie's canonical visual identity), Akriti (the
Rive Bonnie + the emulator), and the risk register (#33/#34).

**Acceptance path for this ADR itself:** (1) ✅ three-lens adversarial hardening — DONE 2026-07-18 (§13); (2) ✅
findings applied to the design (§13 dispositions); (3) ⏳ owner review + acceptance — OWED; (4) only then does
the SDD schedule the mascot units, and **S-0015a is green on the real voice box before any box mascot firmware is
built on**.

## 13. Review record (2026-07-18) — three-lens adversarial hardening

Three independent adversarial reviewers, refute-first. The **hardware-feasibility** lens WEB-VERIFIED the core
claim and confirmed it (ACCEPT-WITH-FIXES); the **child-safety/governance** and **cross-surface composition**
lenses each found real defects (ACCEPT-WITH-FIXES / not-ready). **No REDESIGN — every finding was a localized doc
fix, all applied above.** The single vindicating result: the §3 "Manas owns the screen, never reverts" fix is
**real** — verified against ESPHome docs (`voice_assistant` writes nothing to the display; LVGL owns it), and
confirmed by the myJarvis forensics (they built on the stock package that owns the screen — §1). The through-line
of the fixes: the ADR had the right *hardware* spine but (a) modeled SPI when the real limit is CPU-during-reply,
and (b) expanded the cue *names* without specifying the *bindings* — which was safety-wrong (distress → a
retry-face; sleep outranking distress).

**Hardware (H):** H1 fps limit is CPU-during-reply not SPI → §1 rewritten, S-0015b measures fps with AEC+TTS live
· H2 `speaking` static/low-frame is the baseline not worst-case → §1/§5/§9 · H3 flash math (uncompressed 4.5×
over; compressed+region is baseline) → §4 · H4 add S-0015f OTA-of-clip-library + fix "~30s OTA" → §9 · H5 golden-
frame manual baseline · H6 RLE region player is the codec default · H7 40MHz is the driver default not silicon.
All applied.

**Child-safety (BLOCKING D, A + should/notes):** **D — distress must beat sleep** (a crying child at 2 a.m. must
never see a sleeping Bonnie; the E1 non-verbal night proxy forms no turn) → §5 precedence reordered to
distress ≻ everything, comfort face reachable from the alert-only side-channel, invariant + kill-shot added.
**A — the §7 E5-lint gate can't see baked art** → §7 rewritten: owner art sign-off is the load-bearing gate, E5
honestly labeled as text-only. B/G presence-blind/history-blind scheduler (§5, arch-tested) · E LLM restricted to
an affect-neutral `llmSelectable` subset (§5) · F web Rive cue-driven-only (§7) · J minimal-v1 includes the
safety-degraded cues (§10) · C bodily-needs resolve on-screen, no feed/water touch (§7) · H fright/dark art
review + night face fully-dark (§7) · I touch inherits the no-variable-ratio ban (open-Q4). All applied.

**Cross-surface (BLOCKING F1, F2, F3 + should):** **F1 — the web has no ambient-cue channel** (`Final.animCue`
is turn-terminal) → §6.2 adds the `AmbientCue` push + one server-side cue authority. **F2 — the box can't be
told to show the fault face over the dead channel** → §6.3 the box renders the fault face LOCALLY on
loss-of-cue-channel. **F3 — `ChildDistressComfort` renders as `try-again` today** (rule #9 defect) → §6.1 adds a
first-class `comforting` cue + the `RefusalKind → cue` table, safety-cue fail-closed to neutral-concerned never
`idle`. F4 single cue authority (§6.2) · F5 distress suppresses ambient sleep/whimsy (§5, = D) · F6 the ~12→1
collapse → the §6.1 binding table · F7 every §F degraded cell gets a face + split `need-a-grown-up` from
`box-fault` (§6.1) · F8 persona split: universal core + per-mascotSet whimsy (§10) · F9 fail-closed mechanism
stated honestly (§6.4) · F10 delivery-only VERIFIED-OK (the one invariant the ADR got fully right). All applied.

---

*Sibling of `voice-edge.md` §2.4; composes with ADR 0010 (cue on the egress), ADR 0011/0012 §3e (closed cue
vocabulary, box-is-its-own-renderer), ADR 0014 §3d/§3f + units E3/E5 (quiet-hours sleep, idle-face governance),
persona.md (delivery-only). Drafted 2026-07-18 as the hardware-first answer to the predecessor's months-long
mascot failure; awaiting adversarial hardening + owner acceptance.*
