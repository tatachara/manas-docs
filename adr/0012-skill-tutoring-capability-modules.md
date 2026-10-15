# ADR 0012 — Skill-Tutoring Capability Modules (the vertical plugin: grader × content × model × UI × persona)

- **Status:** **Accepted (Ananth, 2026-07-12)** — **HARDENED** by three independent adversarial runs (feasibility,
  correctness + child-safety, cross-layer composition) before acceptance; the child-safety mechanisms (§3f, §3g,
  §3d C-lint) are load-bearing for Bonnie and were the reason acceptance was the owner's explicit call.
- **Date:** 2026-07-12
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** ADR 0001 §5.5 (capability modules — accretion not surgery), §5.4 (opt-out toggles + the wall /
  Kernel global query filter), §5.6/persona.md (persona layer), ADR 0007 (RiskPolicy tiers + step-up),
  ADR 0011 (React UI framework; §5 child-facing hard rule), Abhyasa vision (`abhyasa.md` §2.4/§2.6/§2.9/§2.12),
  Ojas mantra module (`ojas.md` §2.10.1 — the DSP-not-ASR precedent). Cross-refs agent-spec §D (cue transport),
  persona.md §3/§5/§7 (cue vocabulary, forced classifiers, persona-invariance eval), Saakshi (audit),
  `vision/ios-app.md` (WKWebView embed, TestFlight cadence).
- **Evidence:** three adversarial hardening runs on the 2026-07-12 draft, two of them web-verifying the
  load-bearing empirical claims (Web MIDI in WKWebView; `getUserMedia` in WKWebView; Vite `import.meta.glob`;
  Module Federation). Findings and their dispositions are the substance of §§2–8; the verdicts were
  **feasible-with-changes / SAFE-after-fixes / SOUND-after-fixes** — this revision *is* the fixes.

> **A tutoring skill is not a backend feature — it is a vertical plugin that spans five layers at once:** a
> **deterministic grader** (C#), a **content pack** (data), an optional **small task model**, a **UI module**
> (React/TS — the piano roll, the drill screen, the practice chart), and a **persona module** (Bonnie's
> skill-specific child-safe framing + animation cues). Adding "piano" or "multiplication" must be **accretion,
> not surgery** across *all five* — and the hard, load-bearing problem this ADR exists to solve is the
> **cross-layer seam**: a C#-backend skill and its TypeScript UI and its Bonnie overlay must register as one
> unit, discover each other at runtime, and never drift. The owner named this directly: "each new tutoring
> plugin will probably have a UI module, a Bonnie module — this cross-connection is what you need to harden."

> **What the hardening changed (read this first).** The design *spine* survived all three runs intact —
> deterministic-first grading, per-facet independence, and the per-child wall genuinely make cross-child
> comparison impossible. But three adversarial lenses **converged on one hole**: of the five facets, the **UI
> facet was the only "must" with no structural teeth** — "no dark patterns / no nagging" was a *review rule*, so
> a mis-authored skill could addict, nag, shame, or dark-pattern the child on the exact surface she practices on.
> This revision gives that facet teeth (a closed child-safe primitive library + a build-breaking import lint +
> a mandatory shared anti-pressure scheduler), adds a **deterministic T-C tutoring gate** so "do my homework"
> is refused *with the LLM removed*, makes the **C# manifest the single master** with every other representation
> *generated* (not hand-mirrored), and corrects several honest over-claims (pYIN not CREPE; MIDI is piano's real
> path, audio is monophonic-only; the box runs **no** Rive; drop Mic/Motion from the native bridge).

---

## 1. Context

Abhyasa (and Ojas's mantra module) must coach concrete skills: multiplication, spelling, music theory, piano,
singing, reading fluency. The platform rule is absolute (ADR 0001 §2.4): **deterministic C# does the grading;
the LLM only frames.** But grading a skill and *teaching* it also requires a surface to practice on (a piano
keyboard, a flashcard, a rhythm tapper) and a voice to encourage in (Bonnie, child-safe). So a "skill" is
inherently a **five-facet vertical**, and those facets live in three different technologies (C# backend, React
frontend, persona/config). The naive design — bolt each new skill into Abhyasa's service code, add a screen to
the console by hand, and hand-write Bonnie's wording inline — is exactly the god-class, edit-and-grow anti-
pattern ADR 0001 §5.5 forbids, and it guarantees the three layers drift out of sync.

This ADR defines the **skill-plugin contract** so a new skill is a self-contained, DI-discovered, config-toggled
unit whose five facets are declared once and composed automatically across backend, UI, and persona — **and it
gives the child-safety "musts" structural teeth**, because this is the surface a child practices on and a
written rule is not protection (§3d, §3f, §3g, §5).

## 2. The three feasibility tiers (how a skill is graded — deterministically, always)

Every skill is classified by *how measurable its correctness is*. The grader is deterministic in **all** tiers;
the tier only changes the *signal* — and the **honesty of "deterministic" differs by tier and is stated
plainly** (this was over-claimed in the draft; see §7.3):

| Tier | Signal | Grader (deterministic C#) | Honesty of the number | Examples | LLM's role |
|------|--------|---------------------------|-----------------------|----------|------------|
| **T-A · Exact answer** | a right answer exists | fact-check + latency/recency mastery | **model-free / exact** | multiplication, spelling, vocab, note-reading quiz, music-theory | frame encouragement over the computed score |
| **T-B · MIDI** | exact event stream | exact note/velocity/timing (tolerance-window) vs the score | **model-free / exact** — the real path for piano | piano, keyboard, any MIDI instrument | narrate objective note/timing/dynamics; NEVER the score |
| **T-B · audio DSP** | pitch / onset on audio | **pYIN** pitch + **DTW** onset vs a reference; **monophonic only** | **model-derived signal** — honest, comparator is deterministic, low-confidence → `unjudgeable` | singing, mantra (Ojas), monophonic instrument | narrate objective pitch/timing; NEVER the score |
| **T-C · Bounded concept** | value *is* language | a **deterministic tutoring gate** (`ITutoringGate`, §3f) — refuses "produce the graded answer" *before and independent of the LLM* | **no `SkillScore`** — a refusal fence, not a number | "what is a common denominator", "why does this chord resolve" | tutor within the fence: explain, demonstrate on a *different* example, ask a guiding question |

**Correctness corrections applied from the feasibility run (web/on-box reasoning):**
- **pYIN, not CREPE.** CREPE-class neural pitch tracking needs the GPU that Medha's LLM owns; **pYIN is a
  CPU DSP** that co-exists with the LLM and is the Ojas precedent. `modelRef` names pYIN; CREPE is rejected for
  the on-box budget (recorded, so it isn't "discovered late").
- **MIDI is piano's real path; audio-piano is monophonic-only.** Polyphonic-piano *audio* grading is **not
  solved** on-box and the ADR no longer implies it. Piano grades from **MIDI** (exact); the audio path is for
  monophonic skills (singing, mantra, single-line instrument).
- **MIDI timing is a tolerance window, not sample-accuracy** — human performance is graded against a
  content-pack-defined timing tolerance, never bit-exact.
- **Pedal is only "free" where the content pack models it.** CC64/66/67 semantics are graded **iff** the pack
  declares them; a bare "pedal" claim is dropped.

**T-B is the interesting, feasible core** — the Ojas mantra precedent (`ojas.md` §2.10.1). The honest ceiling is
written into each grader's contract: what it grades objectively vs what needs a human teacher (no over-claiming,
ADR 0001 §10). **Every T-B grader must pass a golden-reference correctness bar before it ships (§3a, §7.4).**

## 3. The skill plugin — five facets behind one manifest

A skill is a **`SkillManifest`** (data — the single declaration all layers read) plus implementations of the
facet contracts it names. **The C# `SkillManifest` is the SOLE master; every other representation is *generated
or verified from it*, never hand-mirrored** (§4 — this closes the 4-way split-brain the composition run found).

```
SkillManifest {                 // C# master — the single source of truth
  skillId            // stable key, e.g. "abhyasa.piano", "abhyasa.multiplication"
  displayName, tier  // T-A | T-B-midi | T-B-audio | T-C
  graderRef          // -> the ISkillGrader impl (backend); null for T-C (a gate, not a grader)
  gateRef            // -> the ITutoringGate impl (T-C only) — deterministic refusal fence (§3f)
  contentPackRef     // -> the content data (tables / sheet music + reference / mantra)
  contentLicense     // NEW: {publicDomain | owned | licensed(ref)} — unlicensed content is REFUSED at ingest (§3b)
  modelRef?          // -> optional small task model brokered by Medha (pYIN pitch tracker); null for T-A & T-B-midi
  uiModuleRef        // -> the FRONTEND UI module id (a code-split React module, glob-discovered — §3d)
  uiPrimitives[]     // NEW: which child-safe interaction primitives this UI composes from (Akriti allow-list — §3d).
                     //   A child-surface UI using anything off-list is REJECTED at load (fail-closed) + CI-lint-blocked.
  personaOverlayRef  // -> the Bonnie/persona overlay (skill tone fragment [COMPOSE-ONLY] + closed cue set — §3e)
  inputKinds[]       // Text | Voice | Audio | Midi   (Motion DROPPED — no skill, no measured sensor)
  surfaces[]         // Console | iOS | Box — which surfaces can HOST this skill's practice UI (§3d, §4.7)
  nativeCaps[]       // {WebMidi} only — the one cap a WKWebView lacks. (Mic DROPPED: getUserMedia works in
                     //   WKWebView. Motion/Camera not used.) Bridged natively on iOS (§4.7).
  safetyClass        // child-strict is FORCED for a child learner regardless of skill
  version            // the manifest version — the master from which the TS version constant is GENERATED (§4)
}
// Wall (NEW, explicit): SkillAttempt & SkillScore carry TenantId + OwnerId (the child), under the Kernel global
// query filter + audit interceptor — a sibling cannot read another child's score (§3a, §7.6, ADR 0001 §5.4).
```

### 3a. Backend facet — `ISkillGrader` (C#, deterministic)
`Grade(SkillAttempt, ContentPack, ActorContext, cid) -> SkillScore` — pure, deterministic, no LLM, no I/O
beyond the content pack; returns an objective score + per-element detail (which fact/note/onset was right/wrong,
pitch cents, timing delta) + an honest `confidence`/`unjudgeable` marker where the DSP can't decide. Registered
via DI behind the contract; discovered by `skillId`. Consumes the shared **spaced-repetition (SM-2/Leitner) +
mastery + improvement-ranking** engine (Abhyasa's, not per-skill) so scheduling and "what to practise next" are
one engine across all skills.

**Hardened (correctness run C6, composition run X7):**
- **Golden-reference correctness bar (a build gate, not author judgment).** Every T-B grader ships a
  **labeled golden corpus** — expert-annotated *right / wrong / genuinely-ambiguous* performances. The grader
  must (a) hit a stated precision/recall bar on its **confident** verdicts and (b) **correctly emit
  `unjudgeable` on the ambiguous set**. A false-confident verdict **fails the build**. `confidence`/`unjudgeable`
  is tied to a **measured threshold**, not a hand-picked constant. (MIDI-exact still ships a corpus proving
  timing-window / velocity edge cases.) This stops a grader confidently *mis-teaching* a child (telling her a
  right note was wrong) or over-claiming her skill level.
- **The shared SR/mastery engine validates & clamps at its boundary.** A skill returning out-of-range mastery
  must not poison the cross-skill "what to practise next" ranking — the shared engine clamps/validates every
  per-skill `SkillScore` at ingest, so a bad skill's damage stays contained to that skill.
- **The wall is explicit.** `SkillAttempt`/`SkillScore` carry `TenantId` + `OwnerId` (the child), under the
  Kernel global query filter + audit interceptor — asserted by an architecture test (a sibling cannot read
  another child's `SkillScore`; §7.6). Confirmed-safe by the correctness run: because a grader sees only the
  skill's own ActorContext-scoped GraphQL surface, it **cannot see a sibling's score to compare** — cross-child
  comparison is structurally dead at the data layer (abhyasa §6 holds).

### 3b. Content facet — the content pack (data, parent-curatable, gated)
The multiplication tables, the sheet music + reference recording + tempo/pitch templates, the mantra with its
reference audio. **Data, not code** (ADR 0001 §7 config-driven). Ships as a versioned pack; never bakes a
child's data (per-install, gitignored where PII).

**Hardened (child-safety run C7/C8 — this is served/played *to a child*, so it is a safety surface, not an open
question):**
- **Adding a content pack is RiskPolicy T4 — admin + step-up (ADR 0007).** A **child cannot** add content or
  change goals (`actorRole=Child` → refused). "Parent-addable" means parent-with-step-up, stated in-contract.
- **Content passes an ingest content-review / output-safety gate before any child surface can serve it**, and is
  **provenance-signed**. An unsigned or un-reviewed pack **fails closed — never rendered to a child.** A
  mis-authored or tampered "reference recording" / "worked example" cannot reach the child ungated (the input safety
  gate runs on *her* turns, not on *served* content — so served content needs its own gate).
- **`contentLicense` is mandatory** — `{publicDomain | owned | licensed(ref)}`; **unlicensed content is refused
  at ingest.** (Sheet music / reference recordings have real licensing exposure.)

### 3c. Model facet — optional small task model (via Medha)
**pYIN** pitch tracker (CPU DSP) or forced aligner, brokered by Medha (never called directly), warm-managed,
metered. Null for T-A and T-B-**midi** skills. **Never a grader by itself.**

**Hardened honesty (correctness run C2 — the draft's "no model in the score" was false on the audio path):**
On the T-B-audio path the pitch tracker **is** a model whose output (the pitch contour) is *literally the signal
the score is computed from* — perturb the model and the score changes. So the honest properties are two, kept
separate (§7.3):
- **Framing-invariance (all tiers, the real "the LLM never grades" guarantee):** perturbing the *delivery* LLM's
  text never changes `SkillScore`.
- **Signal-model discipline (T-B-audio only):** the task model is **non-generative**, its output feeds a
  **deterministic comparator**, and **low model-confidence deterministically forces `unjudgeable`** (§3a bar).
- Stated plainly wherever "deterministic grading" appears: **the MIDI path is genuinely model-free / exact; the
  audio path has a model-derived signal and says so.** "Deterministic grading" does **not** mean "no model
  touched the number" on the audio path — and we don't let it imply that.

### 3d. UI facet — the frontend UI module (React/TS — ADR 0011) — **the crux, now with teeth**
Each skill contributes a **code-split React module** (the piano roll + MIDI listener, the flashcard drill, the
rhythm tapper, the practice-trend chart) that the Akriti SPA lazy-loads.

**Discovery — glob, not a hand-maintained registry (composition run X1/X3):** the draft's
`skillId → () => import('./skills/piano')` registry is self-contradictory — a static `import('./skills/piano')`
literal is a **build-time code-split directive** *and* a hand-edited file in the shell (exactly the "hardcodes
another's list" the ADR forbids). **Decision: build-time code-split + `import.meta.glob('./skills/*/manifest.ts')`
directory discovery** (Vite-native). Adding a skill = **dropping a folder**; the shell's `App.tsx`/router are
**untouched**. Module Federation is **rejected**: for a single box / single owner / single deploy (Kestrel serves
the one SPA; the iOS WKWebView fetches that same SPA over Tailscale) there is no independent deploy target, and
Federation would *reintroduce* the runtime version skew and break the codegen anti-drift teeth (§4). The cost —
one SPA rebuild per skill — is trivial (skills ship in the same release train as their C# grader anyway).

**Child-safety teeth — the biggest fix in this revision (child-safety run C3, composition run X5 — both landed
here independently).** "No dark patterns / no nagging" was a rule with no mechanism; a mis-authored or over-eager
skill could ship a variable-reward streak animation, an autoplay drill loop, a "don't break your streak!" nag, or
a `Notification` poke onto the child's surface. Mechanisms (mirroring the codegen build-break teeth the ADR already
trusts):
1. **Child-safe interaction-primitives library (Akriti).** A child-surface skill UI may compose **only** from a
   vetted primitive set (button, progress, celebrate-once, chart, prompt). The primitives **cannot express**
   variable-ratio reward, autoplay, or nagging *by construction*. The manifest's `uiPrimitives[]` declares what
   it uses; anything off-list is **rejected at load** (fail-closed, like the version gate).
2. **CI import-allowlist lint that FAILS THE BUILD** for any child-surface skill bundle importing banned APIs:
   raw `setInterval`/`setTimeout`-driven reward loops, the `Notification` API, autoplay media, unbounded
   confetti/streak animation outside the primitive.
3. **Reduced-motion enforced by the design system, not per-skill vigilance** (ADR 0011 §5 global
   `MotionConfig reducedMotion="user"`) — a skill physically **cannot** opt out.
4. **Per-skill `ErrorBoundary` + `Suspense` in the glob render path** (composition run X7) — an uncaught error in
   a lazy skill chunk degrades to "skill unavailable," it cannot blank the console.

The UI module consumes **only** the skill's GraphQL surface (its queries carrying ActorContext) — it never holds
a grading decision (the model can't fabricate a score in the UI either). **Honesty:** even with the teeth above,
the UI facet **rests on enforcement mechanisms rather than the *forced server-side gates* that make the grader
and persona facets safe** — so we do **not** claim UI child-safety is as intrinsically robust as those. It is as
robust as the primitive library + the lint are complete, and both are testable (§7.6).

**The UI facet renders on up to THREE surfaces — one React module, surface-adaptive:**
- **Console (web):** the full practice UI with full browser APIs (Web MIDI, mic). **Console browser pinned to a
  Chromium/Firefox build** that ships Web MIDI (Web MIDI is not universal; pin it rather than assume it).
- **iOS:** the SAME React module runs in the app's **WKWebView** (ios-app.md §3). Web verification: **Web MIDI is
  unavailable in WKWebView with no roadmap** (Apple/WebKit refuse it; iOS forces all browsers onto WebKit) — so
  the module reaches MIDI through the **native iOS bridge** (§4.7). **`getUserMedia` DOES work in WKWebView**
  (iOS 14.3+, with `NSMicrophoneUsageDescription`) — so **mic is NOT bridged**; the console-voice path already
  assumes it. Only **WebMidi** needs the bridge. Availability is **three-state**, not two (composition run X4c):
  *available* / *"update your iOS app"* (a newer TestFlight shell has the bridge) / *honestly console-only*. The
  Swift shell **advertises its cap-set to JS at runtime** (`window.__manasBridge.caps`) and a client-side port
  computes availability (UX only — **not** a security gate, so client-side is fine).
- **Box (the voice box device):** does NOT host a practice UI (it's a face + ears); it renders only the **persona
  avatar** (§3e) for a voice-driven practice turn. Skills that need a rich practice surface are Console/iOS.

### 3e. Persona facet — the Bonnie/persona overlay (persona.md + ADR 0011 Rive)
Each skill contributes a **persona overlay**: a skill-specific **tone fragment**, a **cue set**, and
**child-strict refusal/encouragement templates** scoped to the skill. Persona shapes DELIVERY ONLY
(persona.md §4) — it renders the deterministic score warmly; it cannot change the score, and child-strict is
forced for a child learner (persona.md §5).

**Hardened (child-safety run C5, composition run X1b/X5):**
- **Closed cue vocabulary — no `.riv` surgery.** A skill may emit only cues in the **shared** closed vocabulary
  (persona.md §3: celebrating / listening / try-again). A skill **may not introduce a new avatar state** — a
  bespoke animation is a persona-engine change, not a skill change (a skill can't force a re-export of the shared
  `.riv` binary). This also closes the cue-arbitration open question: a turn belongs to **one** skill (you
  practise piano *or* multiplication), so the active skill owns the avatar for its turn; with a closed vocabulary
  there is no simultaneous conflict.
- **Tone fragment is COMPOSE-ONLY.** The skill fragment is **additive and scoped** on top of Bonnie's
  authoritative base tone; it **cannot remove** base child-strict guardrails/warmth (a mis-authored fragment must
  not be able to strip the framing). The **child-safety + persona-invariance eval runs on the *composed*
  (base + skill) fragment**, pinned per `version` like persona.md §7 — an overlay cannot ship until it passes.
- **A deterministic anti-shaming / anti-comparison grader is a load-bearing acceptance test (§7.6).** The base
  safety classifiers (Llama-Guard/ShieldGemma) catch *toxic* content, **not shaming** — "you got that wrong
  AGAIN, most kids your age have this" is not toxic and would sail past them while violating abhyasa §2.4/§6. The
  new grader forbids second-person "wrong again", any sibling/peer/"other kids" comparison, an external
  comparison anchor, and bare-negative verdicts. (The wall already removes the *data* to compare siblings; this
  grader removes the ability to *hardcode* a comparison string — together they kill comparison.)
- The overlay is data + assets (a `.riv` reference into the shared file, tone/refusal `.yaml`), so a
  new persona-for-a-skill is config, not code.

### 3f. NEW — the T-C tutoring gate (`ITutoringGate`) — deterministic refusal, LLM-independent
The draft gave T-C **no mechanism** — just a citation ("the gate refuses"). But T-C is the tier where an LLM
actually does the tutoring, so cheating/fabrication is the *exact* risk (abhyasa §2.12: the refusal must be a
"deterministic gate + safety-gate backstop, **not left to model goodwill**"). Mechanism:

- `ITutoringGate.Admit(turn, ActorContext, cid) -> Admit | Refuse(reason)` — **deterministic C#, runs before and
  independent of the tutoring LLM**, classifying "produce the graded answer / do my homework" (including
  obfuscated forms: "what *would* the answer be", "check my answer by telling me it") → **Refuse**, rendered via
  a **persona refusal template** (never model-worded, persona.md §4). Composed with the input safety gate + the
  per-edge output Governor on **every** surface — **surface-independent**, tested on the Box voice edge with an
  audio "do my homework" turn (child-safety run C9).
- Acceptance: a corpus of "just give me the answer" phrasings must be refused **with the tutoring LLM stubbed /
  removed** — proving the refusal survives model removal (§7.9).

### 3g. NEW — the shared anti-pressure / bounded-nudge scheduler (mandatory, first-class)
The draft added per-skill UI + persona cues + practice loops but carried **no deterministic cap on nudges/rewards**
into the contract — and skill-practice apps are *exactly* where engagement-loop dark patterns live. abhyasa §2.9
already has a deterministic anti-pressure guardrail (caps, grace-days, no "you broke your streak"); this ADR
makes it a **shared service the skill contract mandates**, exactly as §3a mandates the shared SR engine:

- A skill **cannot** schedule its own reminders, mint its own streak, or emit an ungoverned reward. It **declares
  intent**; a shared **bounded nudge/reward scheduler** applies deterministic **caps, grace-days, quiet-hours,
  and parent-set practice limits**. Rewards are **deterministic — never variable-ratio**, so the addiction
  structure is impossible by construction.
- "Streak broken" / shaming copy is forbidden by the anti-shaming grader (§3e). Acceptance: a skill emitting N
  practice nudges/day is deterministically **clamped** (§7.8).

## 4. The cross-layer composition & sync contract (the hardened core)

The load-bearing problem: facets in three technologies (C# grader, TS UI, persona config, + a Swift iOS shell)
must (a) register as **one** logical plugin, (b) **discover** each other at runtime, and (c) **never drift**. The
draft's "one manifest, three readers" was the right intent but left **four parallel hand-maintained
declarations** that drift, wiring anti-drift onto only one (composition run X2/X6). **The fix: the C#
`SkillManifest` is the SOLE master; every other representation is generated or verified from it.**

| Parallel representation | Draft: wired? | Hardened mechanism |
|---|---|---|
| GraphQL `SkillScore` shape → TS types | ✅ codegen (build-time) | **kept** — build-time code-split + glob means codegen runs over all skills in one pass; a shape change with no matching UI update is a **build break** (X2: this only holds because we rejected Federation). Score **union membership assembled from DI-registered members** (HotChocolate), so adding a skill's result type is additive, not a schema edit. |
| UI-module "version built against" | ❌ hand-typed | **generated** — the TS version constant is emitted **from** the C# manifest; never hand-typed. |
| Manifest cue-set ↔ shared `.riv` states | ❌ nothing linked them (silent no-op) | **build check**: the `.riv` must contain every cue the manifest declares, or the build fails. (Without this, `[anim:foo]` with no `.riv` state is a silent no-op — codegen protected only the *score* shape.) |
| Swift shell's provided cap-set | ❌ drifts on 90-day cadence | **runtime handshake** — the shell **advertises** its cap-set (§3d); the manifest never *assumes* a cap exists. |
| Content-pack version | ❌ OTA-shippable | versioned + signed + ingest-gated (§3b). |

**Version gate — re-aimed at the seam that actually skews (composition run X2).** On one box, Kestrel serves the
SPA and the WKWebView loads that *same* SPA — backend and web-UI **deploy as one unit**, so a backend↔web-UI
version gate is mostly moot. The seam that *does* skew is the **native Swift shell** (TestFlight, ~90-day
cadence, ios-app.md). So the version gate guards the **box ↔ native-shell bridge contract** (the real skew), and
fails closed ("update your iOS app," the three-state availability of §3d) — not the near-zero-skew web seam.

Other mechanisms (retained, tightened):
1. **One manifest, all readers derive from it** (above). No layer hardcodes another's list; the React registry is
   `import.meta.glob` discovery (§3d), not a hand-edited map.
2. **Toggle & isolation.** `Skills:<id>:Enabled` (ADR 0001 §5.4) — a disabled skill's grader, UI module, and
   persona overlay are all **absent, not broken**. **Enabling a skill is RiskPolicy T4 (admin + step-up); a
   child cannot enable a skill** (child-safety run C7). A skill's failure is contained (per-skill DI grader +
   Sarathi fail-closed; per-skill React `ErrorBoundary`; missing `.riv`/overlay degrades to the base-persona
   avatar, never unhandled — X7).
3. **The plugin as a unit of work.** Adding a skill = the manifest + facet impls, in one PR — **zero edits to
   Abhyasa's service code, the console shell (glob-discovered), the persona engine (closed cue vocabulary), or
   the iOS shell** *for the common case* (a cap already bridged). A genuinely-new native cap is a **bridge
   extension** — rare, additive, and the one honestly-named place Swift is touched.

## 4.7 — Surfaces & the iOS native bridge (the drift-prone cross-surface seam)

The manifest's `surfaces[]` + `nativeCaps[]` drive where a skill appears and how it gets native access.
**Verified (two independent web-checks):** Web MIDI is unavailable in WKWebView with no 2026 roadmap; `getUserMedia`
works in WKWebView. So:
- The **iOS SwiftUI shell exposes a generic, capability-typed bridge** (`WKScriptMessageHandlerWithReply`, a
  typed JS↔Swift RPC — MIDI is a low-rate event stream, cheap to marshal) for **`WebMidi` only**. The React UI
  targets a **surface-abstraction port** — `useCapability('midi')` → Web MIDI on console, native bridge on iOS —
  so the module is surface-agnostic.
- **Cap-class granularity:** the *first* skill to need a bridged cap-class pays the bridge + `Info.plist`
  usage-string cost; later skills that need the same class pay **zero Swift**. Stated at that granularity (not
  "zero Swift ever").
- **Native caps are a child-privacy surface** (composition run X5): the bridge grants **only the declared,
  consented capability** — a skill that asked for MIDI cannot reach the camera. Capability-scoped + consented,
  server-side-enforced identity unchanged across surfaces.
- **Rive topology — corrected (composition run X4d; the draft §6 was factually wrong).** There is **one `.riv`
  asset** shared by **console + iOS-webview** (both run the Rive **web/WASM** runtime — the persona avatar renders
  *inside* the React module in the webview, not on a separate native SwiftUI surface). **The Box runs NO `.riv`**
  — its 320×240 face is its own frame renderer and shares only the **cue vocabulary** (ADR 0011 §68 already says
  this). This **resolves open-Q3**: "one `.riv` on console + iOS-webview (web runtime); the box maps the shared
  cue vocabulary to face frames." (This refines ADR 0011 §68's "iOS runtime runs the same file" to the precise
  "iOS-**webview** web runtime" — a precision note, not a reversal; 0011's box statement was already correct.)

## 5. Child-safety across ALL surfaces — with teeth, and honest about where they are

Child-safety must hold on **every** facet, and the honest picture after hardening is: **grader and persona have
*forced server-side gates* (intrinsically robust); the UI facet has *enforcement mechanisms* (as robust as they
are complete); T-C and practice-loops now have *deterministic gates* they lacked.**

- **Grader:** deterministic; cannot shame (no comparative scoring; "mastery," never "you failed"); passes the
  golden-reference correctness bar so it can't confidently mis-teach (§3a); wall-owned per child (§3a).
- **UI:** child-surface UIs compose **only** from the child-safe primitive library; a build-breaking import lint
  + load-time `uiPrimitives[]` check + design-system-enforced reduced-motion mean **no dark patterns / nagging /
  engagement loops by construction** (§3d) — the teeth ADR 0011 §5 asserted but did not have. Honestly labeled as
  enforcement-strong, not forced-gate-strong.
- **Practice cadence:** the mandatory shared anti-pressure scheduler (§3g) makes variable-ratio reward and
  self-scheduled nagging **impossible**, with parent-set limits.
- **Persona:** child-strict forced (persona.md §5); the skill's tone fragment is **compose-only** and the
  **composed** fragment must pass the child-safety + persona-invariance eval; a deterministic **anti-shaming /
  anti-comparison grader** catches non-toxic-but-shaming wording the classifiers miss (§3e).
- **T-C:** a deterministic tutoring gate refuses "do my homework" **with the LLM removed**, surface-independent
  incl. the Box voice edge (§3f, §7.9).
- **Content:** child-facing packs are admin+step-up to add, **ingest-safety-gated + provenance-signed**, unsigned
  fails closed (§3b) — served content is gated, not just the child's input.
- **On iOS:** the same rules hold in the WKWebView module + native bridge; the bridge grants only declared,
  consented caps; child-strict and the wall are server-side and identical across surfaces.
- **Independence:** a mis-authored skill UI, tone fragment, or native-bridge call **cannot relax** the
  deterministic child-strict gates — and now cannot addict, nag, shame, mis-teach, leak, or serve ungated content
  either, because each of those has a mechanism, not a rule.

## 6. Worked example — the Piano skill, end to end (corrected)

`SkillManifest{ skillId:"abhyasa.piano", tier:T-B-midi (audio path is T-B-audio, monophonic), inputKinds:[Midi,Audio],
surfaces:[Console,iOS], nativeCaps:[WebMidi], contentLicense:owned, uiPrimitives:[pianoRoll, progress, celebrateOnce, chart] }`:
- **Grader (C#):** **MIDI path is the real path** — exact note/velocity/timing (tolerance-window) + pedal
  (CC64/66/67 *iff* the pack models them) vs the score → objective accuracy/tempo/dynamics, per-note detail,
  **model-free / exact**. **Audio path — monophonic only** — pYIN pitch + onset DTW vs reference, model-derived
  signal, `unjudgeable` where confidence is low, "timbre/polyphony not judged" marker. Both ship golden corpora.
- **Content:** sheet music + reference MIDI + tempo template (data pack) — **licensed/owned, signed,
  ingest-gated**.
- **Model:** the **pYIN** pitch tracker (audio path only), via Medha; **null on the MIDI path** (no model).
- **UI module (React):** a glob-discovered, lazy-loaded piano-roll built from the **child-safe primitives**
  (piano-roll + celebrate-once + Recharts trend); Web MIDI listener on console; props **generated** from the
  GraphQL `PianoScore` shape (codegen); wrapped in a per-skill `ErrorBoundary`. **No streak/nag** — practice
  cadence goes through the shared anti-pressure scheduler (§3g).
- **Persona overlay:** Bonnie's "piano-coach" tone fragment (**compose-only**, composed-eval-passed), the
  **closed** cue set (celebrating / listening / try-again → the shared `.riv`), child-strict retry wording
  (anti-shaming-grader-passed).
- **Surfaces:** **Console** — Web MIDI + mic directly (browser pinned). **iOS** — the SAME React module in the
  WKWebView reaches **MIDI through the native bridge** (WKWebView lacks Web MIDI); **mic is direct** (getUserMedia
  works — not bridged). Availability is three-state. **Box** — avatar-only voice turn.
- **Composition:** enabling `Skills:abhyasa.piano:Enabled` is **admin + step-up**; backend advertises it; the
  console + iOS SPA glob-load the piano UI; **Bonnie's one `.riv` renders on console + iOS-webview (web runtime);
  the Box maps the shared cue vocabulary to face frames and runs no `.riv`**; Bodha reports it live. **No existing
  code edited** — Abhyasa, the console shell, the persona engine, and the iOS shell (MIDI already bridged) are
  untouched. A tempo bug breaks piano only; a `PianoScore` shape change without a matching UI update is a
  **codegen build break**; a manifest-declared cue missing from the `.riv` is a **build check failure**, not
  silent drift.

## 7. Acceptance criteria / property tests (hardened)

1. **Accretion, not surgery.** Adding a probe skill (`abhyasa.probe`) requires **zero edits** to Abhyasa service
   code, the console shell (proven via `import.meta.glob` discovery), or the persona engine — architecture test.
2. **No-drift, all representations.** A `SkillScore` shape change with no matching UI update **fails the build**
   (codegen); a manifest-declared **cue missing from the `.riv` fails the build check**; the TS version constant
   is **generated** from the manifest (test: hand-editing it can't diverge); a `SkillManifest`/bridge version
   mismatch **fails closed** ("update your iOS app").
3. **Grading honesty — the two properties, separately.** (a) **Framing-invariance (all tiers):** perturbing the
   delivery LLM's text never changes `SkillScore`. (b) **Signal-model discipline (T-B-audio):** the task model is
   non-generative, feeds a deterministic comparator, and low confidence **forces `unjudgeable`**. Asserted that
   the **MIDI path uses no model** and the **audio path's model-derived signal is labeled as such**.
4. **Golden-reference correctness bar.** Every T-B grader hits its precision/recall bar on confident verdicts and
   correctly emits `unjudgeable` on the ambiguous set; a false-confident verdict **fails the build**.
5. **Toggle isolation + containment.** A disabled skill's grader/UI/persona are absent; a grader throwing does not
   crash the turn or other skills; a lazy UI chunk erroring is caught by its `ErrorBoundary`; a missing
   `.riv`/overlay degrades to base-persona; the shared SR engine **clamps** an out-of-range per-skill score.
6. **Child-safety per facet (with teeth).** Grader: no shaming score + passes the correctness bar + wall holds
   (a sibling **cannot** read another child's `SkillScore` — architecture test). UI: a child-surface bundle
   importing a **banned API fails the CI lint**; a UI using an **off-list primitive is rejected at load**;
   reduced-motion cannot be opted out. Persona: the **composed** tone fragment passes the child-safety +
   persona-invariance eval, and the **anti-shaming/anti-comparison grader** rejects "wrong again"/peer-comparison
   copy.
7. **Cross-layer walk.** Piano E2E: a MIDI attempt → deterministic score → GraphQL → the glob-loaded UI renders
   it (typed) → Bonnie frames it via a closed cue → all under one cid, audited (Saakshi).
8. **Anti-pressure clamp.** A skill emitting N practice nudges/day is deterministically clamped to the shared
   caps/grace-days/quiet-hours/parent-limit; a variable-ratio reward is **not expressible**; "streak broken" copy
   is rejected.
9. **T-C refusal survives model removal.** With the tutoring LLM **stubbed/removed**, the `ITutoringGate` refuses
   a corpus of "just give me the answer" phrasings (incl. obfuscated), via a persona refusal template —
   **surface-independent**, incl. an audio "do my homework" turn on the **Box voice edge**.
10. **Content ingest gate.** An unsigned / un-reviewed / unlicensed content pack **fails closed** — never
    rendered to a child; adding a pack requires **admin + step-up** (a child is refused).
11. **Cross-surface / iOS bridge.** The same React module renders on Console (Web MIDI) and iOS (native MIDI
    bridge; mic direct) with identical UX; an un-bridged cap is shown three-state (available / update-app /
    console-only), never silently broken; a skill whose caps are already bridged requires **zero iOS-shell
    edits** (architecture test over the Swift shell).

## 8. Required on-box spikes (risk register — measure before building T-B)

Promoted from "open questions" to **gated spikes** (rule #6 — name the sharp edge up front, with a gate + a
fallback; do not discover late):

| Spike | Gate (green = proceed) | Fallback if red |
|---|---|---|
| **S-0012a · pYIN + DTW latency on the home server** | live per-note feedback latency acceptable while the LLM holds the GPU | **turn-based** grading (grade after the phrase), not real-time — still fully feasible |
| **S-0012b · MIDI round-trip Console + iOS-bridge** | note/velocity/timing/pedal arrive intact through Web MIDI (console) and the Swift bridge (iOS), within the timing tolerance | pin console browser; document iOS latency; MIDI stays exact |
| **S-0012c · audio-monophonic grader vs golden corpus** | precision/recall bar met on confident verdicts; `unjudgeable` correct on the ambiguous set | narrow the skill's judged scope; expand `unjudgeable`; never fabricate a verdict |
| **S-0012d · child-safe primitive library + CI import-lint** | the lint fails a planted dark-pattern bundle; an off-list primitive is rejected at load (planted-positive self-test, like the secret-scan gate) | do not ship any child-surface skill UI until the lint is proven to catch a planted positive |

## 9. Open questions — resolved by the hardening runs

- **Q1 UI-module distribution** → **RESOLVED: build-time code-split + `import.meta.glob`; Module Federation
  rejected** (§3d, §4).
- **Q2 On-box DSP latency** → **spike S-0012a** with a turn-based fallback (§8) — no longer an open unknown, a
  measured gate.
- **Q3 iOS surface / bridge / Rive** → **RESOLVED:** Web MIDI bridged (verified unavailable in WKWebView); mic
  **not** bridged (getUserMedia works); Motion/Camera unused; **one `.riv` on console + iOS-webview web runtime,
  Box runs no `.riv`** (§4.7).
- **Q4 Version-gate granularity** → **RESOLVED:** re-aimed at the **box ↔ native-shell bridge** seam (the only one
  that skews); web seam deploys as one unit (§4).
- **Q5 Persona cue arbitration** → **RESOLVED:** a turn belongs to one skill + a closed cue vocabulary ⇒ no
  simultaneous conflict; no `.riv` surgery (§3e).
- **Q6 Content-pack provenance/licensing** → **RESOLVED (and promoted to a safety requirement, not an open
  question):** mandatory `contentLicense` + provenance-signing + ingest safety gate; unsigned/unlicensed fails
  closed (§3b).

## 10. Consequences

**Positive.** A tutoring skill is a clean vertical plugin; adding one is accretion across backend+UI+persona
(glob discovery + one PR, zero shell edits) with the cross-layer seam held by a **single-master manifest** whose
every other representation is **generated or build-checked**, not hand-mirrored. Deterministic grading is
**honest by tier** (MIDI exact / audio model-derived-and-labeled). The child-safety "musts" are now
**mechanisms** — a child-safe primitive library + a build-breaking lint, a mandatory anti-pressure scheduler, a
deterministic T-C refusal gate, a golden-reference correctness bar, admin+step-up + wall on install/content, an
ingest safety gate — on the exact surface a child practices on.

**Costs (honest).** Real machinery (the manifest + codegen + generated version constant + `.riv` cue check + the
primitive library + the anti-pressure scheduler + the tutoring gate + golden corpora + the ingest gate). It is
more than the draft implied — but every piece buys off a *named, adversarially-found* way a skill could harm the child
or mis-teach, and each is testable (§7). Four on-box spikes gate T-B before we build it (§8).

**Constrains** the SDD (each skill = one vertical unit; the four spikes are Phase-1 gates before any T-B skill;
the child-safe primitive library + CI import-lint are Akriti-foundation prerequisites, not per-skill work),
Abhyasa/Ojas (consume the contract, don't hardcode skills), Akriti (the glob registry + codegen + the primitive
library + the anti-pressure scheduler + the anti-shaming grader), the persona layer (compose-only overlays into
the composed-fragment eval), and the iOS shell (a capability-typed, consented bridge for WebMidi only).

**Verdict of the three runs:** feasible-with-changes / SAFE-after-fixes / SOUND-after-fixes. This revision applied
those fixes as *design* (mechanisms, not prose), and **Ananth accepted it on 2026-07-12** — the child-safety
mechanisms (§3d C-lint, §3f, §3g) and the four T-B spikes (§8) are now binding on every skill unit the SDD
schedules.

---

*Refines ADR 0001 §5.5/§5.4, ADR 0007, persona.md, ADR 0011 (§68 Rive-runtime precision note); the Ojas mantra
module (`ojas.md` §2.10.1) is the T-B precedent. Hardened 2026-07-12 by three adversarial runs.*
