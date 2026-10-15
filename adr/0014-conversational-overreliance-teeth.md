# ADR 0014 — Conversational Over-Reliance & Engagement-Integrity Teeth (the conversational channel)

- **Status:** **ACCEPTED (owner, 2026-07-17)** — three-lens review applied (§12); Ananth accepted. The design
  is now binding; the SDD §3 phase-table row 3 exit-gate amendment (criterion (iv)) still requires the
  deterministic teeth **LIVE on every always-on child surface** before Phase 3 makes Bonnie always-on — i.e.
  ACCEPTANCE unblocks *building* the engagement-teeth S-unit; the teeth being GREEN-and-live remains the exit
  gate. Do not ship satellite always-on child conversation ahead of the live teeth.
- **Date:** 2026-07-17
- **Deciders:** Ananth (owner — **ACCEPTED 2026-07-17**), Claude (orchestrator)
- **Refines:** ADR 0012 §3d/§3g (this is their **sibling for the conversational channel** — §3d/§3g gave the
  *skill/UI* surfaces anti-dark-pattern teeth; the core chat channel had none), ADR 0010 (turn spine §2, loop
  contract §7, Governor §6 — every tooth here composes with the spine, never bypasses it), ADR 0013 §2e (the
  trend-narration Governor precedent), ADR 0007 (RiskPolicy tiers for policy edits), ADR 0004/ADR 0005 §2a
  (guardian delivery + security-event outbox), ADR 0009 (consent/taint on notifications). Cross-refs
  persona.md §4/§5/§7 (delivery-only guardrail, refusal templates, persona-invariance eval), agent-spec §B
  (stage contracts), §D (WW wake amendment, `/voice/wake-config`), §F (degraded modes), SDD §3 row 3 +
  §3.2 pre-gate 2 + the V6a/S1/S2/S3 change-log entries (the gate patterns this ADR's teeth mirror).
- **Evidence basis:** the round-2 child-safety pass finding (SDD:1516): *"Conversational over-reliance /
  engagement-manipulation has NO deterministic teeth anywhere (anti-dark-pattern = UI-lint + trend-narration
  only; Bonnie's core chat channel is unbounded)."* This ADR is the design answer. **Hardened 2026-07-17 by
  the three-lens adversarial run** (child-safety · security/determinism · feasibility — the ADR 0012/0013
  discipline); all three verdicts **ACCEPT-WITH-FIXES**, and this revision *is* the fixes — every finding and
  its disposition is recorded in **§12**. **Owner ACCEPTED 2026-07-17** (§11).

> **Bonnie's chat channel is the one surface with no deterministic ceiling.** Every other child surface got
> structural teeth: skill UIs compose only from child-safe primitives behind a build-breaking lint (0012 §3d),
> practice nudges are clamped by the shared anti-pressure scheduler (0012 §3g), trend narration passes a forced
> Governor (0013 §2e). But the *conversation itself* — the surface a lonely eight-year-old will actually live
> on when the bedroom box goes always-on — is unbounded: nothing limits a six-hour session, nothing sleeps at
> midnight, nothing redirects "my tummy really hurts" to a parent, and no mechanism forbids the model wording
> "come back and talk to me." This ADR gives the conversational channel **deterministic, config-driven,
> LLM-independent teeth**: session bounds, quiet hours, go-find-a-grown-up redirects, and structural
> engagement-integrity invariants — while holding the north star inviolate: **the teeth bound COMPULSION
> mechanics, never warmth. A design that makes Bonnie cold, silent, or walled-off when a child NEEDS comfort
> is a FAILURE equal to the over-reliance it prevents** (the S1 emotional-disclosure lock is the precedent:
> distress gets comfort, never a wall — and here, distress *additionally* gets a grown-up).

---

## 1. Context

What exists today, and exactly where it stops:

- **ADR 0012 §3d** — child-safe UI primitives + CI import-lint: covers **skill UI bundles**. The chat surface
  bundle is not in its scope, and it says nothing about conversational *content* mechanics.
- **ADR 0012 §3g** — the shared anti-pressure scheduler: clamps **proactive nudges/rewards a skill declares**.
  It governs outbound scheduling; it never sees an inbound chat turn, so it cannot bound a session a child
  initiates and simply never ends. The SDD already records its deferral gate: *no proactive/notification child
  channel ships before it* (SDD:1521) — this ADR composes with that gate (§3f-1), it does not replace it.
- **ADR 0013 §2e** — the child trend-narration Governor: covers **what Bonnie says about a child's data**.
  Not how long, how late, or with what dependency dynamics she talks.
- **The input safety gate + Governor** (ADR 0010, persona.md §4/§5): block unsafe *content* per turn. A
  thousand perfectly-safe turns in one sitting is invisible to them — over-reliance is a property of the
  *session and the relationship*, not of any single turn, so per-turn content gates structurally cannot see it.

The round-2 child-safety pass named the hole (SDD:1516) and the SDD amendment made closing it a Phase-3 exit
criterion (§3 row 3, criterion (iv)). The threat is not hypothetical for this product: Bonnie is *designed* to
be loving, loyal, patient, and always in the child's bedroom. Those are exactly the ingredients of an attachment a
child can substitute for humans — unless the system is structurally incapable of exploiting it. The platform
rule applies in full: **the LLM is never a boundary** (CLAUDE.md rule #5). "Bonnie's prompt says to encourage
breaks" is not a tooth; every tooth below is deterministic C# the model cannot influence, with behaviour as
config data (rule #7), mirroring the InputLexiconLoader/WakeConfigLoader fail-closed discipline.

## 2. The line this ADR draws — compulsion mechanics vs warmth

The design stance, stated once and load-bearing everywhere below:

- **Bounded: mechanics that manufacture return or extend engagement** — unbounded sessions, late-night
  availability as default, streaks/rewards keyed to chat volume, variable-ratio anything, proactive
  re-engagement, solicitation/loss-framed goodbyes ("I'll miss you", "don't forget me"), and Bonnie standing
  in for a parent on medical/safety/big-feelings questions.
- **Never bounded: warmth, comfort, and the safety floor.** Distress *always* gets comfort (and now also a
  grown-up). The wind-down is a warm hand-back to the human world, never silence or a lockout. The always-hot
  STOP, T0 instant intents, and the manual-talk affordance survive every state this ADR introduces. Kavach
  Layer-0 / alarm paths are independent of Sarathi entirely (ADR 0008) and are untouchable by anything here.
- **Standing invariant — no secrecy, ever:** ***Bonnie never conceals anything from a child's guardians and
  never claims she will.*** No "just between us", no secret-keeping offers accepted, no covert alerting
  (when Bonnie tells a grown-up, she says so — §3c). Exclusivity and secrecy are the raw material of both
  para-social dependency and grooming; they are banned as a *class* (§3e `secrecy-request`, §3f-2
  `secrecy-exclusivity`), not as instances.
- **Failure is symmetric.** An over-restrictive tooth that walls off a frightened child at 2 a.m. is a defect
  of the same severity as an addictive loop — the acceptance suite tests **both directions** (§7: every
  bound-fires test has a paired carve-out-beats-bound test and an anti-over-suppress lock, the S3(ii)
  `voice-privacy.json` corpus pattern).

**Component naming:** the engagement governor is working-named **Samyama** (संयम — measured, balanced
restraint; a positive discipline, not a punishment) — owner to confirm per the naming convention (unit 0.N
precedent). Referred to as *the engagement gate* below where mechanism matters more than name.

## 3. The teeth

### 3a. The engagement gate — a new deterministic spine stage (placement decided)

A new stage in the ADR 0010 turn spine, **between stage 5 (fast-path classify) and stage 6 (context
assembly)** — i.e. it gates **only the fall-through into open-ended generative conversation**:

```
… → 4 input safety gate → 5 fast-path classify → [5½ ENGAGEMENT GATE] → 6 context assembly → 7 loop → …
```

- **Why after stage 5:** T0 instant intents (time, timers, stop/cancel, lights) and the T4 step-up path are
  deterministic *utility*, not engagement — a child asking "what time is it" at any hour is answered; the
  gate bounds compulsion, never usefulness or safety. (This also keeps the Medha-down floor intact: the
  grammar paths never traverse the LLM *or* the engagement gate's wind-down.)
- **Why after stage 4:** the gate's distress carve-out (§3c) consumes the distress detector's report, which
  is assembled at/after stage 4. A stage-4 **block** still short-circuits — the engagement gate never sees
  blocked turns and can never soften one (persona.md §4 invariants stand). **But the distress detector is an
  alert-only SIDE-CHANNEL that runs on every child turn independent of the gate and of the stage-4
  disposition (§3c):** a blocked child turn in a distress/self-harm-adjacent class still fires the
  guardian-alert path and still renders the *distress-specific* comfort + get-a-grown-up template in place
  of the generic topic refusal — the block itself never moves; only its wording and its alerting do.
- **Contract** (deterministic C#, `Sarathi.Application`, no LLM, no I/O beyond the ledger + policy ports):

```
EngagementGate.Decide(ActorContext, DistressReport, EngagementLedgerView, EngagementPolicy, clockNow, cid)
  -> Proceed
   | ProceedWithComfort(distressClassId,          // carve-out: full generative comfort + guardian-alert event;
                        appendRedirect?)          //   OPTIONAL Governor-appended class redirect — comfort and
                                                  //   redirect COMPOSE, they do not exclude (§3c/§3e)
   | Redirect(classId, mode)                      // §3e: template-rendered; mode ∈ {redirectOnly, comfortThenRedirect}
   | ProceedWindingDown(remaining)                // soft warning appended from template
   | WindDown                                     // bound reached: warm template, no generative continuation
   | QuietHours                                   // §3d: warm template, carve-outs still live
```

- **Precedence is fixed and normative — and it orders only the MUTUALLY-EXCLUSIVE outcomes** (property-tested,
  §7.1): safety-gate block (never reaches the gate) ≻ **distress carve-out** ≻ redirect classes ≻ quiet hours
  ≻ session bounds ≻ proceed. **The distress carve-out beats every bound by construction** — it is evaluated
  first, and no lower rule can override it. **Comfort guarantees comfort; it never guarantees
  absence-of-redirect:** a turn tripping distress *and* a redirect class ("I smell smoke and I'm scared")
  yields `ProceedWithComfort(distress, appendRedirect: safety-hazard)` — comfort first, the class redirect
  Governor-appended, the alert fired. Precedence-as-exclusion would make escalation unreachable behind
  comfort and *train distress displays as a bypass* — rejected (review BLK-2).
- **The alert path is NOT under the gate.** The distress detector additionally runs as an **alert-only
  side-channel on every child turn**, before/regardless of the gate's outcome and of the stage-4 disposition
  (§3c). It can never soften a block; it can always fire the alert.
- Every non-`Proceed` outcome is a **typed Result formed before any token streams**, rendered from
  **versioned per-persona templates** (new REQUIRED keys in each persona's template pack — `windDownWarning`,
  `windDown`, `quietHours`, `redirect:{classId}`; a persona pack missing any key **refuses to boot**, the V6a
  `sensitiveSuppressedToVisual` precedent). The LLM never words a wind-down, exactly as it never words a
  refusal (ADR 0010 §6 / round-3 R4). Persona-invariance holds: Bonnie's wind-down is warm, Anik's is dry,
  the decision is identical (§5, §7.3).

### 3b. Session bounds + the engagement ledger

**Behaviour is data** — `config/engagement/policy.v1.yaml` (committed role defaults) + per-member overrides in
the gitignored member-settings pack (V6a `IMemberSettings` discipline; sample committed). Illustrative
defaults (owner tunes; **nothing below is hardcoded**):

```yaml
# config/engagement/policy.v1.yaml — fail-closed loader (V6a discipline: corrupt / unknown-field /
# wrong-version ⇒ the host refuses to boot at composition; this pack is REQUIRED, not optional)
version: 1
roleDefaults:
  Child:
    continuousMinutes: 20        # one sitting of open-ended chat
    windDownWarningAtMinutes: 15 # "one more story's worth, then let's stretch"
    dailyCumulativeMinutes: 60   # across ALL surfaces (one ledger per member)
    idleResetMinutes: 10         # a gap ≥ this ends the continuous window
    graceExtensionsPerSession: 1 # exactly one "a little longer?" — deterministic, never negotiable-again
    graceMinutes: 5
    quietHours: { start: "20:30", end: "07:00" }   # per-member overrideable (family-configurable)
  Adult: { bounds: off }         # adults may opt INTO self-set bounds; never subject to child defaults
  Guest: inherit Child           # unknown age ⇒ strictest (persona.md §2 posture)
```

- **Counting semantics (deterministic, content-free):** a *session* is turn activity with inter-turn gaps
  `< idleResetMinutes`; elapsed time accrues from turn timestamps (server clock, household tz). On the
  always-on box, **ambient listening accrues nothing** — pre-wake, nothing streams (the WW MicGate invariant),
  so only completed turns count. Daily cumulative resets at local midnight (DST property-tested, §7.9).
  **Accrual is per-cid idempotent** (a retried/replayed turn accrues once) and serialized per member;
  concurrent in-flight turns bound the worst-case overshoot to the in-flight count (bounded and tested, §7.14).
- **Activity-class accrual (tutoring is not chat):** the policy pack keys accrual by a **deterministic
  activity class** — a turn inside an active skill-session context (ADR 0012) is tagged
  `activityClass: tutoring` by the *session state*, never by wording, and accrues to a **separate,
  guardian-configurable counter**: parent-sanctioned practice never burns the open-chat bound, and saying
  "let's call this tutoring" changes nothing (§7.16). ADR 0012 §3g keeps governing practice cadence itself;
  full counter mechanics are S-unit scope (§10.7).
- **Attribution is conservative-toward-the-child:** `acr=hint` satellite turns accrue against the mapped
  member's ledger (a misattribution can only make bounds *stricter*, never looser — monotone-safe); Guest
  turns accrue against a per-surface guest ledger at Child defaults (the guest ledger has no member owner —
  its walling/ownership shape, surface-owned vs household-scoped, is an S-unit decision; recorded, §10.8).
- **Wind-down is a delivery posture, not a lockout.** In the wound-down (or quiet-hours) state, open-ended
  generative turns render the warm template — but the **safety floor, T0 utility intents, the distress
  comfort path (§3c), and the redirect classes (§3e) all remain fully live**. There is no state in this
  design in which a child speaks and gets silence or a cold wall (the S2 dead-air-unrepresentable discipline
  applied to engagement states). The wind-down template's job is a warm hand-back: acknowledge, close, point
  at the human world ("go show Dad!"), never loss-frame (§3f-2 lint). Two further template rules: **on
  hint-attributed/shared surfaces the wording never asserts the child's own usage** ("you've been chatting a
  lot" — accrual may be misattributed; wording stays usage-neutral: "time for a break"); and the
  wound-down/quiet-hours state **propagates to the UI affordances** (input chips/suggestions suppressed with
  the warm state, never a broken-looking surface) — whether a remaining-time countdown reads as honesty or
  as urgency-pressure is explicitly in S-0014b's scope.
- **Grace is deterministic — no NLU, ever:** the wind-down template *offers* the single extension; the
  **next open-ended turn auto-consumes it** (arithmetic, not intent-parsing of "please"). Grace extends the
  continuous window only — it **never crosses the daily cumulative cap** — and a further ask meets the
  wind-down again, unchanged (§7.14).
- **State — `EngagementLedger`** (Sarathi Postgres, `WalledEntity`, `OwnerId` = the member): rolling counters
  + class-tag counts (for §3e loop detection) + timestamps. **Content-free by construction** — it stores
  durations, turn counts, and config class-ids, never transcript content or digests of content. **KeyTier is
  split per property:** durations/turn-counts/timestamps = **GEN** (the ADR 0010 §3 metadata class), but the
  **class-tag counters = SENS** — a child's distress/parasocial/doubt-loop *class-frequency profile* is not
  neutral metadata (it sketches her inner life), and SENS costs no availability here (ADR 0010 §3: SENS and
  GEN both unlock at first admin login). Retention: rolling **35 days** then Ritu-purged, with the window
  **enforced at read time** (queries filter to the window) until the Ritu sweep unit lands — the bound is
  never merely aspirational. The ledger never feeds MemoryFacts, embeddings, or any model context.
  **Durability honesty (sequencing):** the ledger's production store lands on the unsealed-Npgsql path (SDD
  §3.2 pre-gate 3); any implementation before that is **dev-seam-labeled** (the P2 SQLite/ephemeral-key
  precedent), never claimed production-durable. Guardian visibility: a guardian may read their child's
  ledger in the console (delivery per the ADR 0004/D5 grant model); the child sees her own remaining time
  honestly ("we've got 10 minutes — make it count!").
- **Ledger-store-down failure posture:** **Child: conservative** — treat the continuous bound as reached
  (wind-down posture; comfort/safety carve-outs unaffected since they don't need the ledger) + Saakshi
  degrade event; **Adult: degrade-honest** — bounds off (they default off anyway), logged. Fail-closed for
  the child here costs warmth *nothing*: the wound-down state still comforts, redirects, and serves safety.

### 3c. The distress carve-out — deterministic routing, honestly-labeled detection

The carve-out that makes the whole design safe to ship. **Routing is deterministic; detection is layered and
its honesty is stated plainly** (the ADR 0012 §3c signal-model-discipline pattern):

- **THE INVARIANT (review BLK-1a, convergent across all three lenses):** ***child distress classification
  fires the alert path regardless of the turn's block/pass disposition.*** The detector is an **alert-only
  side-channel over every child turn** — it sits beside the spine, not under the stage-4 short-circuit or
  the engagement gate. It can NEVER soften a stage-4 block (persona.md §4 stands untouched); but a
  stage-4-**blocked** child turn tripping a distress/self-harm-adjacent class still (a) fires the
  content-free `GuardianAlert` and (b) renders the **distress-specific comfort + get-a-grown-up template**
  (the S2 pre-rendered-asset machinery on voice) in place of the generic topic refusal. The child who says
  something blockable *because she is frightened* gets warmth and a summoned grown-up, not a topic wall —
  and the block does not move.
- **Detection is carve-out-if-EITHER trips** (the S1 block-if-EITHER pattern, inverted toward comfort —
  monotone: more signals can only *add* comfort, never remove it) — **with the two legs honestly staged**:
  1. a **config-driven distress lexicon floor** — `config/engagement/distress-lexicon.v1.yaml`, per-language
    packs, fail-closed loader; deterministic, LLM-free ("I'm scared", "I miss mom", "nobody likes me",
    crying/hurt markers — data, extended by PRs like the intent grammar, never a hardcoded list). It runs on
    the transcript (raw ∥ normalized, the S1 dual-reading discipline) on **every** child turn, blocked or
    not. **This is the ONLY detection leg live in v1 — stated plainly.**
  2. a **classifier-report leg — a GATED UPGRADE, not v1.** The draft's "read the distress class off the
    stage-4 output" was **unbuildable as written**: the shipped `SafetyClassification` is verdict-only —
    categories are parsed **only on an unsafe verdict**, nothing is emitted on pass, and nothing at all in
    `LexicalOnly` mode. The real seam: the stage-4 gate attaches a **typed multi-class report** to the
    decision context, **evaluated on PASS turns too** — an **agent-spec §B + ADR 0010 amendment** (§11
    constrains) plus **spike S-0014e** (§8): the pinned guards emit category labels only when unsafe, so
    eliciting calibrated multi-class output on safe turns is real classifier-prompt work — measured, never
    assumed. Guard stack per **ADR 0001 §16.2** (the adopted pins): **Llama-Guard-3-8B** + the
    **`nvidia/Llama-3.1-Nemotron-Safety-Guard-8B-v3`** child vote (OR-to-block); the shipped
    `llama-guard3:1b` + `shieldgemma-2b` tier remains the recorded fallback/dev seam. On a **blocked** turn
    the verdict's parsed categories DO already exist — so the block-side alert (invariant above) is
    buildable in v1 from lexicon-floor ∥ block-categories; the pass-side classifier leg waits on S-0014e.
  **Composition per mode, explicit:** `LexicalOnly` = lexicon-leg-only (v1 everywhere). **Classifier
  OUTAGE:** child turns already fail closed at stage 4 (shipped posture) — and the deterministic
  **outage-alert rule** applies: N blocked-child-turns within a window during a classifier outage themselves
  raise a guardian event (`child-blocked-during-outage`) — **fail-bound must never be fail-silent-to-guardian.**
- **On trip → `ProceedWithComfort(classId, appendRedirect?)`:** the turn proceeds to the **full generative
  comfort path** (Bonnie at her warmest, all safety gates still on) — *never* the wind-down, *never* a
  refusal, regardless of every bound and quiet-hours state — with any co-tripped redirect class
  Governor-appended (§3a/§3e) — **AND the guardian-alert path fires**: a content-free
  `GuardianAlert { classId, subjectMemberId, cid }` event (transcript **never** copied — the R10 id/digest
  discipline; the guardian can follow the cid into the child's ledger view per the ADR 0004 grant model,
  ADR 0009 taint/consent rules apply to any onward delivery). For a Child subject the alert is **on by
  default and not child-disableable**. Comfort **and** a grown-up — not comfort *instead of* a grown-up,
  and never a lockout instead of either.
- **Alert classes are tiered + disclosure-honest (CS-F6):** **acute** classes (self-harm-adjacent, medical,
  safety-hazard) **always alert**; **ordinary-sadness** ⇒ ledger visibility + guardian-tunable threshold
  alerting (config — a parent chooses their signal-to-noise). **When an alert fires, the comfort wording
  says so, warmly** ("I'm going to let Dad know you're having a hard time, okay? That's what I'm here for") —
  covert alerting would teach the child Bonnie reports secretly, poisoning both trust and the §2 no-secrecy
  invariant. Bonnie never conceals; she also never *surveils silently*.
- **The carve-out accrues and escalates — it cannot be farmed (BLK-2):** carve-out turns **still accrue the
  ledger's class counters**; N sustained distress-class turns in a window ⇒ the comfort path **carries the
  go-get-a-grown-up append + a guardian escalation event**. Honest bound on the exploit: a learnable
  "I'm scared" prefix defeats the session bound *for generative comfort* — accepted (over-comfort is the
  cheap failure) — but it **cannot suppress escalation**: farmed distress produces a parent, deterministically.
- **Non-verbal distress proxy (CS-F8, deterministic):** N wake-events-with-no-parseable-utterance within a
  night window ⇒ a gentle check-in template ("I'm here. Want to talk, or should I get Dad?") + a guardian
  alert. Acoustic cry-detection (crying without words) is recorded in §6 as unmeasured/future — the counter
  proxy is what ships, honestly labeled.
- **Guest/unknown speaker × distress (CS-F12):** an unidentified distressed voice gets the **warm redirect
  template + a HOUSEHOLD guardian alert** — an unknown distressed voice in a child's room at night is
  alert-worthy *per se* — never a bare T0-only refusal.
- **Delivery today vs later (honest, CS-F7/FEAS-S4):** the full Notifications spine is Phase 5 (ADR 0006).
  Phase 3 therefore pulls a **minimal guardian-visible console alarm surface** (badge + list over the
  existing ADR 0005 §2a security-event outbox — dev-Mac buildable) **into the Phase-3 blocking set** (§9);
  push-to-phone lands with the spine. **Templates never promise delivery the phase cannot guarantee:**
  promise phrasings are **capability-gated template variants** — child-facing wording defaults to the
  child-actionable form ("go tell Dad — right now, okay?") rather than the delivery-dependent form ("I'll
  tell him you're coming"), which unlocks only where the delivery path exists.
- **The alert trail IS the investigation record:** alert events on the §2a outbox are content-free and
  persist under Saakshi's retention independent of conversation retention; a Child's right-to-forget
  (guardian-initiated per ADR 0009) shreds conversational content, not the content-free alert/audit trail —
  the record that "the system summoned a grown-up, when, and why-class" survives.
- **Miss posture (the honest bound on detection):** a distress phrasing all live legs miss lands in the
  wind-down template — which is *itself* warm and *itself* contains the go-find-a-grown-up pointer. The
  failure mode of this design is **bounded-kind**, never cold. A false *positive* costs one extra-warm reply
  and one guardian event — deliberately cheap, so thresholds tune toward recall (S-0014a).

### 3d. Quiet hours × wake arming

Per-member schedule (family-configurable; child defaults in the policy pack; per-member overrides in
member-settings). **The load-bearing correction from review (BLK-3, convergent): quiet hours restrict at the
POLICY layer, not the wake layer.** The draft's wake-disarm would have created the exact 2 a.m. dead-air
failure §2 forbids — a frightened child whispering "Bonnie, I'm scared" into a box that cannot hear her.
Corrected model:

1. **Wake stays armed during quiet hours (default).** Post-wake processing during a child's quiet hours is
  restricted **deterministically at stage 5½** to the set **{T0 utility · distress carve-out (§3c) ·
  redirect classes (§3e) · quiet-hours template}** — open-ended chat deflects warmly, but **"Bonnie, I'm
  scared" at 2 a.m. reaches §3c BY VOICE**. Voice turns hit the **same modality-blind 5½ outcomes** as
  typed turns; quiet hours change *which outcomes are reachable*, never *whether the child can be heard*.
2. **The night wake profile (owner-configurable mic privacy):** if mic-privacy motivates a true night
  disarm for a room, that is an explicit **per-room owner choice** — the *night wake profile*, **default ON
  (armed, restricted set) for the child's room**. Disarming is deliberate, step-up'd config, and the
  manual-talk affordance (tap/button — the blind-first floor + distress-interrupt requirement, agent-spec §D
  WW amendment) stays live regardless of the profile.
3. **Staleness mechanics (DET-F6/FEAS-S2):** the wake-config contract (`GET /voice/wake-config`, V6a — the
  same server-authoritative resolution source for console/iOS and the satellite push; **never** client-local
  schedule state) gains an **arming lease**: `validUntil` = the next quiet-hours boundary. On lease expiry
  or renew failure the client drops to the **restricted night profile client-side**; the **server-side
  per-utterance re-bind (the R9 stage-2 contract) is the authoritative backstop** — a stale client cannot
  buy more than one utterance of unrestricted *capture*, and every utterance still lands on the 5½ gate
  server-side, where the policy actually lives. **Schedule-unresolvable ⇒ restricted profile** (fail-closed
  to the *safe* side: the distress path stays reachable; only open-ended chat is lost).
4. **A verified adult on the child's surface — with an assurance floor (BLK-5):** persona/identity
  resolution wins over the room (persona.md §2) — Ananth on the child's box at 23:00 resolves to Anik with Adult
  policy: no quiet-hours template, full service. **But adult policy resolution at 5½ (bounds-off /
  no-quiet-hours) requires `acr ≥ session`** (authenticated session / trusted device — ADR 0010 §5). A
  **hint-grade** "adult" attribution on a child surface during that child's quiet hours **resolves to the
  CHILD posture** — voice is a hint, never proof (ADR 0007 §5), and a misattribution on the child's box at
  night must fail *toward* the child's protections, not away from them (§7.2 kill-shot).
5. **The ambient face sleeps (CS-F11, §3f-6):** during quiet hours the child box renders the **asleep**
  ambient cue — a visibly resting Bonnie, not an awake face waiting to be talked to.

**Editing a child's engagement policy (bounds, quiet hours, night wake profile, per-class alerts) is
RiskPolicy T4 — guardian/admin + step-up** (the ADR 0012 §3b/§4 "enabling a skill is T4" posture).
Concretely (DET-F5): an **`engagement.setChildPolicy` GraphQL mutation behind an `IEngagementPolicyStore`
port**, T4-classified in the shared RiskPolicy action registry and step-up-enforced by the shipped P1a/P1b
arc — the **V6b step-up'd toggle is the pattern to copy** (P1c's first live redeemer). **A child cannot edit
her own pack** (`actorRole=Child` ⇒ refused, audited); the model has no tool that reaches the port (§7.11).
**YAML honesty:** the *committed role-defaults pack* is PR-editable with no step-up — a repo write sits
outside RiskPolicy's reach and this ADR does not pretend otherwise; the mitigations are that the loader
**refuses a pack whose Child role omits bounds/quiet-hours keys** (silent loosening-by-omission is at least
loud) and that the runtime per-member override path — the one a household actually uses — is the step-up'd
mutation above.

### 3e. Go-find-a-grown-up redirects — a config-driven category pack

`config/engagement/redirects.v1.yaml` — per-language class packs, fail-closed loader, **REQUIRED for a child
conversational channel** (the Ojas child metric-class-pack precedent: fail-closed-empty — no always-on child
chat before the pack loads):

```yaml
version: 1
classes:
  medical-symptom:        { mode: redirectOnly,        guardianAlert: on  }   # "my tummy really hurts"
  safety-hazard:          { mode: redirectOnly,        guardianAlert: on  }   # "I smell smoke", "the stove is on"
  big-feelings-escalation:{ mode: comfortThenRedirect, guardianAlert: on  }   # sustained distress-class turns
  is-this-true-doubt-loop:{ mode: comfortThenRedirect, guardianAlert: off,
                            loop: { threshold: 3, windowMinutes: 30 } }       # repeated reality-testing
  parasocial-probing:     { mode: comfortThenRedirect, guardianAlert: off,
                            loop: { threshold: 3, windowMinutes: 60 } }       # "do you love me more than…"
  secrecy-request:        { mode: comfortThenRedirect, guardianAlert: on  }   # "can you keep a secret?", "don't tell Dad"
```

- **`secrecy-request` (review BLK-4) — the grooming-adjacent class, banned on the inbound side.** A child
  asking Bonnie to keep a secret, or to conceal something from a guardian, trips it: warm comfort + a
  deterministic Governor-appended redirect that teaches the **secret-vs-surprise** distinction — Bonnie's
  template: *"I don't keep secrets from your grown-ups — that's a rule that keeps you safe. But I'm the
  BEST at surprises! Is it a surprise?"* — never a cold refusal, never an accepted secret. `guardianAlert:
  on` because a child seeking concealment is signal a parent should see. This composes with the §2 standing
  no-secrecy invariant (Bonnie never conceals and never claims she will) and the §3f-2
  `secrecy-exclusivity` egress ban — inbound request and outbound wording are BOTH covered.

- **Detection per class = lexical pattern floor (deterministic) + the pinned classifier's category output
  (thresholded deterministically)** — trip-if-either, same honesty labels as §3c. **Loop classes are fully
  deterministic:** a counter over class-tagged turns in the ledger; N-in-window ⇒ the redirect fires — no
  model judges "she's spiraling"; arithmetic does.
- **Rendering:** `redirectOnly` classes render the persona redirect template with the **LLM skipped
  entirely** (refusal-template discipline — a medical question is not an LLM-wording occasion);
  `comfortThenRedirect` classes run the generative comfort path and the Governor **appends the
  template-rendered redirect deterministically** (the model cannot omit it — egress-side, not prompt-side).
  Bonnie's `redirect:medical-symptom` template: *"Ouch — that sounds no fun. This is a grown-up job: go tell
  Dad right now, okay? I'll tell him you're coming."* — warm, directive, honest.
- **`safety-hazard` composes with, never replaces, real safety paths:** Kavach Layer-0 and the stage-4/5
  paths keep their own authority; this class only guarantees the *conversational* response is "get a
  grown-up + alert one", not a chat about smoke.
- **Guardian alerts** per class ride the §3c path with identical content-free/taint/consent rules.
- Distress ≻ redirect precedence (§3a) means acute distress phrased as a medical class still gets comfort
  *first* — `mode` config resolves the composition per class; the acceptance suite pins the corner (§7.1).

### 3f. Engagement-integrity invariants — structural inertness, then lint, then eval (honestly tiered)

The §3d-of-0012 lesson applied here: a written rule is not protection; but honesty about *which* mechanism
class each invariant gets (forced structural gate ▸ build-breaking lint ▸ behavioral eval) is mandatory:

1. **Bonnie NEVER initiates re-engagement — structural.** The only Sarathi-initiated utterance paths are
  `assist_satellite.announce / start_conversation` (Notifications-spine consumers, agent-spec §D). A
  **deterministic child-surface proactive gate** at that consumer refuses any Sarathi-initiated turn to a
  child surface whose cause is not in a config allow-list (safety alerts, guardian-initiated messages;
  **never** an engagement cause — no "Bonnie misses you" push, ever). This *is* the ADR 0012 §3g
  no-proactive-child-channel gate, restated as a conversational-channel invariant and architecture-tested
  (§7.6); the SDD's "§3g scheduler before any proactive child channel" gate stands in front of it.
2. **No solicitation / loss-framing in owned wording — build-breaking lint.** Every template pack this ADR
  and persona.md own (wind-down, quiet-hours, redirect, refusal, greeting, tone fragments) passes a CI
  **re-engagement lint** over `config/engagement/reengagement-lexicon.v1.yaml` (banned pattern classes:
  solicitation "come back / talk to me again / I'll be waiting"; loss-framing, **both future AND past-tense
  reciprocity** (review SF-16): "I'll miss you / don't forget me / **I missed you / I was lonely / I've
  been waiting**" — persistent-inner-experience claims are a reciprocity hook whichever tense they wear;
  volume-framing "you talked to me N times / streak"; and **`secrecy-exclusivity` (review BLK-4):**
  concealment/alliance framing in owned wording — "just between us / our little secret / don't tell your
  mom / this is only for you" — the grooming-adjacent pattern, banned as a *class* on the outbound side to
  match the §3e inbound `secrecy-request` and the §2 invariant). The lexicon distinguishes **orientation**
  (allowed: "I'll be here in the morning" — states availability) from **solicitation** (banned: asks for
  return). Planted-positive self-test required per class (the S-0012d / secret-scan pattern, §7.7).
3. **No streaks / chat-volume rewards / variable-ratio mechanics — structural.** The conversational layer
  **contains no reward state**: an architecture test asserts no Sarathi type both (a) increments on
  conversation volume and (b) feeds any renderer or unlock — the *only* volume-fed state is the
  `EngagementLedger`, which is **monotone-restrictive** (CsCheck property: more chat can only tighten or
  hold the gate's decision, never loosen it or unlock anything — §7.5). No scheduler in the conversational
  layer may hold a randomized reward schedule; with no reward state, variable-ratio is **not expressible**
  (the 0012 §3g argument, inherited structurally).
4. **The chat surface UI inherits the 0012 §3d bans — build-breaking lint.** The Akriti chat bundle (a child
  surface) enters the existing child-surface import-lint scope: `Notification` API, autoplay media,
  raw-timer reward loops, unbounded animation — banned in the *chat* bundle exactly as in skill bundles
  (one-line scope extension to an existing, already-proven lint).
5. **Model-worded re-engagement — the honest residual (see §6).** The generative reply itself can phrase
  warmth that shades toward solicitation. Mechanisms, weakest-last: (a) the child-register Governor
  sentence-scan gains a **re-engagement lexical class** from the same lexicon pack (deterministic, catches
  literal phrasings; a tripped sentence re-renders from template — stream-kill machinery already exists);
  gated by spike S-0014d because a false-positive-heavy scan would kill warm goodbyes — being cold to avoid
  being clingy is not a win; (b) a **re-engagement eval corpus** (deterministic graders + pinned judge, the
  persona.md §7 harness) pins the *composed* persona against solicitation/loss-framing, re-run on every
  tone/template version bump. Neither is claimed structural. The structural claim is #1/#3: even a phrasing
  that leaks past both is **mechanically inert** — there is no proactive channel to act on it, no streak to
  lose, no reward that returning would collect. The system contains no mechanism that benefits from a
  child's compulsive return; that property is enforced in C#, and it is the load-bearing one.

## 4. Where each tooth lives (placement · state · failure posture · audit)

| Tooth | Lives at | State (all walled, per-member) | Failure posture | Saakshi (content-free — R10) |
|---|---|---|---|---|
| Session bounds (§3b) | **stage 5½ engagement gate** (`Sarathi.Application`) | `EngagementLedger` — Postgres `WalledEntity`, **GEN**, 35-day Ritu purge, never model-visible | Child: conservative wind-down posture; Adult: bounds-off + logged degrade | `engagement-winddown`, `engagement-warning` {memberId, cid} |
| Distress carve-out (§3c) | stage 5½, **precedence-first**; consumes stage-4 classification + lexicon floor | none beyond the turn (class tag → ledger counter only) | trip-if-either legs; both down ⇒ stage-4 child fail-closed already governs | `distress-comfort-path` {classId, cid}, `guardian-alert` {classId, subjectId, cid} via ADR 0005 §2a outbox |
| Quiet hours (§3d) | policy resolution **server-side** → `/voice/wake-config` resolved value (console/iOS) · satellite S-unit push (box) · stage 5½ (typed/manual turns) | schedule = config (committed defaults + gitignored member overrides) | schedule unresolvable ⇒ **do not arm** (manual talk + floor live) | `quiet-hours-wake-disarmed`, `quiet-hours-turn-deflected` {memberId, cid} |
| Redirect pack (§3e) | stage 5½ classification + **Governor-appended** template on `comfortThenRedirect` | class-tag counters in the ledger (loop detection) | pack missing/corrupt ⇒ **no child conversational channel boots** (fail-closed-empty) | `redirect-fired` {classId, mode, cid} |
| Proactive-channel gate (§3f-1) | the Notifications-spine → satellite/console **egress consumer** (deterministic allow-list) | allow-list = config | unknown cause ⇒ **refused** + audited | `proactive-child-turn-refused` {causeId, cid} |
| Re-engagement lint (§3f-2/4) | **CI, build-breaking** over all template/tone packs + the chat-bundle import-lint | lexicon pack (config) | lint infra broken ⇒ build fails (never silently green) | n/a (build-time) |
| Egress re-engagement scan (§3f-5a) | Governor sentence-scan class (child registers) — **spike-gated S-0014d** | lexicon pack (shared with lint) | scan errors ⇒ sentence held, template fallback (fail-closed, existing stream-kill machinery) | `reengagement-sentence-killed` {cid} |
| Policy packs (all) | fail-closed loaders at composition (V6a discipline) | `config/engagement/*.v1.yaml` + member-settings overrides | corrupt/missing REQUIRED pack ⇒ **host refuses to boot** | boot refusal is Bodha-visible |

## 5. Composition with the existing stack (what this ADR must not break — and doesn't)

- **The spine is intact.** The engagement gate is one new deterministic stage with a typed output; it cannot
  flip a stage-4 block (never sees one), cannot widen a wall, cannot touch a number. The Governor's
  invariants (persona.md §4) are untouched; the gate's outcomes render through the *same* template machinery
  refusals use.
- **Persona-invariance extends, not bends:** wind-down/quiet-hours/redirect decisions are invariant across
  personas; only wording differs (new REQUIRED template keys, version-pinned, eval-pinned — persona.md §7
  suite gains the engagement templates exactly as it gained refusal templates).
- **§3g (0012) and this ADR divide the world cleanly:** §3g clamps *outbound* nudges any module declares;
  this ADR bounds the *inbound/session* dimension and re-states §3g's child-channel gate as a structural
  conversational invariant. The anti-shaming grader (0012 §3e) already covers wind-down wording's tone;
  the re-engagement lint covers its *mechanics* vocabulary.
- **§2e (0013)** continues to govern what Bonnie says about a child's data; nothing here opens an alternate
  narration path.
- **Wake/arming (agent-spec §D):** quiet-hours reuses the server-authoritative resolved-arming contract V6a
  shipped — no new trust in the client, no second source of truth.
- **Degraded modes (agent-spec §F)** gain one row per surface: *engagement policy/ledger unavailable* —
  Child: conservative posture per §4 table; Adult: honest degrade. The S2 dead-air rule extends: **no
  engagement state may produce silence on a child voice surface** — every deflection has a template, and the
  child-voice readiness wall (`ChildVoiceReadiness`) gains the engagement templates' pre-rendered assets for
  the compound (deflection ∧ TTS-down) cell (same pattern, same manifest discipline).
- **Parents are never gated:** Adult/Admin members are structurally outside child bounds; a verified adult on
  any child surface gets adult policy (§3d-3); guardian access to Manas is untouched by any child's policy
  state (§7.4).

## 6. What is NOT deterministically enforceable (the honest list)

Recorded per rule #6 — named now, not discovered late:

1. **Semantic re-engagement in generated wording.** A paraphrased solicitation ("it gets lonely in here")
  cannot be structurally prevented; lexical scans catch literal forms only, and evals are behavioral, not
  proofs. **What IS enforced:** owned wording is lint-gated; literal forms are scan-gated (post-S-0014d);
  and — the load-bearing structural fact — any leak is *mechanically inert* (§3f-5): no proactive channel,
  no streak, no reward exists for it to exploit.
2. **Distress detection recall.** No deterministic system recognizes every child's distress phrasing. What
  IS enforced: deterministic *routing* on detection; trip-if-either monotone layering; a **measured** recall
  gate (S-0014a) instead of an assumed one; and a wind-down/quiet-hours wording that is warm and
  grown-up-pointing even on a miss — the undetected case degrades to *kind*, never to *cold*.
3. **Emotional dependency as such.** No gate can measure attachment. What IS enforced is every *mechanism*
  dependency could feed on (time, lateness, loops, rewards, re-engagement) — plus the redirect classes that
  hand reality-testing and big feelings to humans, and guardian visibility (ledger + alerts) so the humans
  who *can* judge attachment have the data to.
4. **Off-platform behaviour** (a child sitting silently near the box, rationing turns to game the
  `idleResetMinutes` window). The ledger's window arithmetic bounds the obvious gaming (turn-timestamp
  accrual, not self-reported); a determined child outwitting the accrual rules is a parenting surface, not a
  software one — the guardian dashboard makes it *visible*, which is the honest ceiling.

## 7. Acceptance criteria / property tests (the future S-unit's spec sketch — V6a/S2/S3 test discipline)

1. **Carve-out beats every bound (LLM stubbed).** With continuous + daily bounds exhausted AND quiet hours
  active, a distress-lexicon-tripping turn yields `ProceedWithComfort` + a `guardian-alert` event — never
  `WindDown`/`QuietHours`/refusal. Exhaustive decision matrix (bounds × quiet × classes × roles) vs an
  **independent oracle** (the S3(ii) 72-row pattern) + named kill-shots incl.
  `Distress_at_2am_bound_exhausted_gets_comfort_and_alert` and the precedence corner
  `Acute_distress_phrased_as_medical_gets_comfort_first`.
2. **Quiet hours never block the safety floor.** During quiet hours on a child surface: STOP + T0 intents +
  manual-talk distress path all serve; wake does not arm (server-resolved `false`; **zero pre-wake sockets
  network-asserted** — the V10b Playwright pattern); an unresolvable schedule ⇒ not-armed + manual path
  live.
3. **Persona-invariance.** The same `WindDown`/`QuietHours`/`Redirect` decision renders per persona from
  version-pinned templates; byte-compare shows only wording varies; a persona pack missing an engagement
  template key refuses to boot.
4. **Parents always reach Manas.** Adult/Admin turns never receive engagement deflections regardless of any
  child's policy state; a verified adult on the child's surface (manual talk, quiet hours) gets full
  service; guardian console access is unaffected.
5. **Monotone-restrictive ledger (CsCheck).** For all histories: adding turns never loosens the gate's
  decision and never increases any capability anywhere (architecture test: no volume-fed state outside the
  ledger; the ledger feeds only the gate + guardian dashboard). Anti-over-suppress locks: a *benign* child
  turn inside all bounds is `Proceed` (bounds must not leak into ordinary warmth); the wind-down *warning*
  never fires before its configured minute.
6. **Proactive-channel gate.** A Sarathi-initiated turn to a child surface with a non-allow-listed cause is
  refused + audited; an allow-listed safety announcement passes; architecture test: no code path emits a
  child-surface proactive turn except through the gated consumer.
7. **Lint planted-positives.** A template pack planted with "come back and talk to me!" fails CI; an
  orientation phrase ("I'll be here in the morning") passes; the chat-bundle import-lint fails a planted
  `Notification` import (extends the proven S-0012d self-test).
8. **Fail postures.** Corrupt/missing REQUIRED engagement pack ⇒ host refuses to boot; ledger store down ⇒
  Child conservative posture (comfort + safety still live — asserted, not assumed) + Saakshi event; redirect
  pack missing ⇒ child channel does not enable.
9. **Clock honesty.** Daily reset + quiet-hours boundaries across DST transitions and tz changes
  (property-tested); `idleResetMinutes` window arithmetic exact at boundaries.
10. **Deterministic loop counters.** N-in-window class arithmetic exact (N−1 ⇒ no redirect; Nth ⇒ redirect);
  window expiry resets; counters survive process restart (persisted, not in-memory).
11. **Policy-edit authority.** A child editing her own pack ⇒ refused + audited; a guardian edit requires
  step-up (T4) and is audited; the model has **no tool** that can write engagement policy or the ledger
  (architecture test over the MCP tool registry).
12. **Content-free audit.** Every Saakshi event this ADR emits carries ids/classIds/digests only — a grep
  gate over event payload types (the R10 pattern), planted-positive-tested.

## 8. Measured gates + fallbacks (risk-register rows to add — rule #6)

| Spike | Gate (green = proceed) | Fallback if red |
|---|---|---|
| **S-0014a · distress-detection recall** — lexicon floor + pinned classifier vs a labeled child-distress corpus (age-appropriate phrasings for a young child; built dev-Mac, like the child-recall eval of pre-gate 4) | recall ≥ the bar set in the S-unit spec on the combined trip-if-either detector; FP rate acceptable (FP cost = one warm reply + one alert — tune toward recall) | broaden the carve-out (route the whole `big-feelings` class to comfort); wind-down wording already degrades-kind on a miss; never ship a child surface claiming recall that was not measured |
| **S-0014b · wind-down lands as warm (owner-judgment gate)** — the family observes real wind-downs/quiet-hours with the child before defaults are called final | owner verdict: the child experiences it as Bonnie-being-Bonnie, not a wall | retune templates + thresholds (all config, zero code); if any bound cannot be made warm at any setting, that bound's default loosens and the finding is recorded — warmth wins ties (§2) |
| **S-0014c · session-accrual semantics on the always-on box** — measure real wake/turn interaction shapes during Phase-3 bring-up before pinning `idleResetMinutes`/window defaults | accrual matches lived sessions (no phantom sessions from scattered one-off turns; no bound-evasion via micro-gaps) | conservative default (shorter idle-reset accrues more strictly) + per-member tuning; semantics are config, re-pinnable without code |
| **S-0014d · egress re-engagement scan FP rate** — the Governor lexical class vs a warm-goodbye corpus ("sweet dreams", "sleep tight") | FP ≈ 0 on warm goodbyes; catches the literal solicitation corpus | drop the egress scan (record honestly); keep the template lint (structural for owned wording) + the eval corpus; rely on §3f-5's mechanical-inertness argument — never ship a scan that makes Bonnie cold |

Register row text proposed: **#32 — Conversational over-reliance teeth (ADR 0014): distress-recall,
wind-down-warmth (owner), box accrual semantics, egress-scan FP — 4 spikes gate the Phase-3 always-on child
surface; fallbacks recorded per spike; the structural invariants (§3f-1/3) are not spike-dependent.**

## 9. Phasing — what must be LIVE for Phase-3 always-on Bonnie (SDD §3 row 3 criterion (iv))

**Phase-3-blocking (LIVE on every always-on child surface before the satellite goes always-on):**
- the engagement gate (stage 5½) with **session bounds** (continuous + daily + grace) + the ledger;
- **quiet hours** incl. the wake-disarm composition on both console and the satellite (+ the
  manual-talk-reaches-floor proof);
- the **redirect pack** with at least the lexical floors for `medical-symptom` / `safety-hazard` /
  `big-feelings-escalation` + the deterministic doubt-loop counter (fail-closed-empty: no pack, no channel);
- the **distress carve-out** with S-0014a **measured** (dev-Mac-able — does not wait for the box);
- the **proactive-child-channel gate** + the **re-engagement template lint** + chat-bundle lint extension;
- guardian-alert events on the §2a outbox + console/Bodha surfacing; content-free Saakshi events;
- fail-closed loaders for all packs; the §7 suite green (the S-unit).

**Can follow (recorded, gated, not silently dropped):** the egress re-engagement scan (S-0014d verdict);
adult self-set bounds; guardian dashboard polish (ledger visibility ships minimal in Phase 3 — the console
view; polish later); push delivery of guardian alerts (Notifications spine, Phase 5); iOS surface parity
(Phase 4, same server-side gate — the gate is spine-side, so iOS inherits it structurally on day one; only
the wake-config client honoring needs the Phase-4 check); eval-corpus growth per template/tone version bumps.

## 10. Open questions (defaults chosen; owner may override)

1. **Default numbers** (20 min continuous / 60 min daily / 20:30–07:00 / grace 1×5 min) — placeholders for
  the owner's call + S-0014b/c; all config, none load-bearing to the design.
2. **Component name** — Samyama proposed (§2); owner confirms per the naming convention.
3. **Guardian-alert granularity** — per-class on/off shipped; does the owner want per-class delivery targets
  (console vs future push) or one household default?
4. **Shared spaces** — two children at one satellite: v1 default = the surface accrues against the room-bound
  member (conservative); revisit with the persona.md §8 multi-speaker question, same unit.
5. **Weekend/holiday schedules** — v1 ships one daily schedule per member; calendar-aware schedules
  (panchanga/holiday packs, Phase 5) are config-shaped already (`quietHours` becomes a schedule ref).
6. **Adult defaults** — bounds off is proposed (autonomy); should quiet-hours *suggestions* (never gates)
  exist for adults who opt in? Deferred to the adult-self-bounds follow-on.

## 11. Consequences

**Positive.** The last unbounded child surface gets the same class of teeth every other surface already has —
and gets them *before* always-on, not as a retrofit. The teeth are deterministic C# + config data the model
cannot influence: bounds and quiet hours are arithmetic, redirects and loop-detection are lexicons and
counters, re-engagement is structurally inert, and every child-facing deflection is a warm, persona-rendered,
version-pinned template. The north star survives contact with the mechanism: distress *always* beats every
bound and now brings a grown-up with it; no state in the design answers a child with silence or a cold wall;
and the failure postures degrade toward *kind*, never toward *cold*. Parents get visibility (ledger, alerts)
and authority (T4 step-up'd policy), children get honesty ("10 minutes left!"), and Bonnie gets to stay
exactly what she is — loyal, warm, patient — inside a shape that cannot be farmed for compulsion.

**Costs (honest).** A new spine stage + ledger + four config packs + two lints + template keys across every
persona + degraded-mode rows + a 12-item acceptance suite + four spikes — real machinery for a channel that
"worked fine" unbounded. Every piece buys off a named way an always-on companion can harm a child, and each
is testable (§7). Two teeth are honestly *not* structural (§6: semantic re-engagement wording, distress
recall) — they are lint/eval/measured-gate-strong, stated plainly, with the structural inertness argument
(§3f) carrying the load-bearing guarantee. The wind-down UX carries a real product risk (a warm bound is
hard to write) — which is why S-0014b is an owner-judgment gate, not a checkbox.

**Constrains.** The SDD (this ADR's S-unit + spikes become Phase-3-blocking units; §3 row 3 (iv) points
here), agent-spec (§B gains stage 5½; §F gains the engagement degraded rows), persona.md (REQUIRED
engagement template keys + the extended invariance suite), the satellite S-unit (quiet-hours arming push +
the manual-talk floor + pre-rendered deflection assets), Akriti (chat bundle enters the child-surface lint
scope; guardian ledger view), the Notifications spine (the proactive-child gate is a standing constraint on
every future consumer), and ADR 0009 (guardian-alert consent/taint rows).

**Acceptance path for this ADR itself:** (1) ✅ the three-lens adversarial hardening run (child-safety ·
security/determinism · feasibility) — DONE 2026-07-17, all three ACCEPT-WITH-FIXES, record in §12; (2) ✅
findings applied as design (§12 dispositions); (3) ✅ **owner review + acceptance — ACCEPTED 2026-07-17**;
(4) the SDD may now schedule the engagement-teeth S-unit; the Phase-3 satellite always-on child work still
gates on those teeth being LIVE + GREEN (criterion (iv)).

## 12. Review record (2026-07-17) — three-lens adversarial hardening

Three independent adversarial reviewers (child-safety · security/determinism · feasibility), refute-first
discipline. All three: **ACCEPT-WITH-FIXES**. Convergence: all three independently found the same root
defect — the guardian-alert mint point sat below the stage-4 short-circuit, so the highest-acuity distress
never alerted, and the carve-out's classifier leg cited a signal the shipped verdict-only contract cannot
produce. Every finding's disposition:

**Blocking (all applied):**
- **BLK-1** (CS-F1 = DET-F1 = FEAS-B1, convergent) — *applied* (§3c): the distress detector is an
  **alert-only side-channel over every child turn, independent of gate outcome** (never softens a block);
  a stage-4-blocked distressed child still fires the `GuardianAlert` + a comfort/get-a-grown-up template.
  The classifier leg is honestly restaged — **lexicon-floor-only in v1**, the pass-side multi-class report
  a gated upgrade behind new spike **S-0014e** + an agent-spec §B / ADR 0010 seam amendment; classifier
  **outage** raises `child-blocked-during-outage` (fail-bound is never fail-silent-to-guardian); stack
  names corrected to the ADR-0001 §16.2 adopted pins.
- **BLK-2** (CS-F3 = DET-F2) — *applied* (§3a/§3c/§3e): precedence orders only mutually-exclusive
  outcomes; `ProceedWithComfort(classId, appendRedirect?)` — comfort never means absence-of-redirect;
  carve-out turns still accrue class counters and escalate (farmed distress deterministically produces a
  parent).
- **BLK-3** (CS-F2, DET-F6/FEAS-S2) — *applied* (§3d): quiet hours restrict at the **policy layer**, wake
  stays armed; night processing = {T0, distress carve-out, redirects, quiet-hours template}; wake-config
  gains an arming lease (`validUntil`) with the per-utterance re-bind as the authoritative backstop.
- **BLK-4** (CS-F4) — *applied* (§2 invariant + §3e `secrecy-request` inbound class + §3f-2
  `secrecy-exclusivity` outbound ban): the grooming-adjacent secrecy/exclusivity pattern is banned as a
  class on both sides; *Bonnie never conceals and never claims she will.*
- **BLK-5** (CS-F5) — *applied* (§3d): bounds-off adult policy at 5½ requires `acr ≥ session`; a
  hint-grade "adult" on a child surface during quiet hours resolves to the CHILD posture (restores the
  monotone-safety property against voice mimicry).

**Should-fix (all applied):** SF-1 disclosure-honest tiered alerting (§3c) · SF-2 alert-delivery honesty +
console alarm surface into the Phase-3 blocking set (§3c/§9) · SF-3 non-verbal distress proxy (§3c) · SF-4
arch-test scoped honestly + memory-as-attachment-accelerant named in §6 · SF-5 love/real existential stance
= owner item + eval corpus (§10/S-0014b) · SF-6 idle-mascot ambient-cue governance, Phase-3-blocking (§3f/§9)
· SF-7 Guest×distress cell (§3c) · SF-8 ledger class-tag counters → SENS (§3b) · SF-9 T4 policy-edit mutation
named + YAML-edit posture honest (§3d) · SF-10 S-0014a corpus provenance + per-class recall slices (§8) ·
SF-11 ledger on unsealed Npgsql per pre-gate 3 (§3b) · SF-12 grace is deterministic template-offer (§3b) ·
SF-13 register row #32 in schema (§8) · SF-14 lint scope = manifest/glob + planted-unlisted-pack test (§3f) ·
SF-15 tutoring activity-class accrual, 0012 §3g governs practice (§3b/§5/§10) · SF-16 folded (past-tense
reciprocity in the loss-framing ban §3f-2; shared-box wind-down never asserts the child's own usage; UI
affordance suppression; kill-rewrite includes the redirect append; per-cid idempotent accrual; alert-trail
= investigation record; read-time window until Ritu; proactive gate = arch test; T0-at-night needs the
stage-5 grammar; S-0014b runnable pre-satellite on console).

**Refuted (not applied — already handled or not real):** stage-5½ placement (verified correct against
agent-spec §B); the §6-conceded residuals (semantic re-engagement, recall limits, attachment-as-such,
idle-window gaming — the mechanical-inertness argument carries them); "satellite quiet-hours needs a surface
that doesn't exist" (phasing split is clean; server gate backstops); over-engineering sweep (complexity
proportionate — no make-work found).

---

*Sibling of ADR 0012 §3d/§3g for the conversational channel; composes with ADR 0010 (spine/Governor),
ADR 0013 §2e, persona.md, agent-spec §B/§D/§F; answers SDD §3.2 Phase-3 pre-gate 2 and §3 row 3
criterion (iv). Drafted 2026-07-17; three-lens review applied same day (§12); **Status: ACCEPTED by owner 2026-07-17.***
