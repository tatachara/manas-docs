# Ojas — Family Health & Wellness

> **Vision document** for the **Ojas** service of the **Manas** platform. Ojas (ओजस्, *radiant
> vitality / vigour*) is the family's health record and gentle coach. This document defines its
> intent, capability areas, data model, integrations, cross-service interconnections, and explicit
> non-goals. It refines — and must not contradict — **ADR 0001** (the platform root). Where this
> doc and ADR 0001 disagree, ADR 0001 wins.
>
> - **Status:** Vision (pre-ADR). Feature list is a menu, not a commitment; phasing is in §7.
> - **Date:** 2026-07-11
> - **Household:** Ananth and the other adult (adults, admins) · the child.

---

## 1. Intent

**Ojas is the family's private health record and gentle wellness coach — for both body and
mind.** It continuously and losslessly ingests **Apple Health** (via the Manas iOS bridge) and
lets each family member add vitals by voice or hand — blood pressure, blood glucose, weight,
temperature, sleep, mood. It records **everything historically** in an append-only store, runs
**deterministic** trend analysis to surface *meaningful* changes (not noise), and offers
**general-wellness** guidance across nutrition, fitness, sleep, and — as a flagship area —
**mental wellness**: mood tracking, guided breathing, meditation timers, gratitude journaling,
and a pluggable **Sanskrit mantra-chanting** coaching module. It tracks the whole family: adult
vitals, women's-health cycles, and pediatric growth/vaccines for the child.

Ojas holds **the most sensitive data on the platform**, so it is the **strictest** expression of
Manas's principles:

1. **Local-first and private by construction.** Health data and all inference live on the
   the home server; nothing leaves the home. HealthKit data flows *in* through the iOS bridge and never
   back out to any cloud.
2. **Deterministic C# is the engine; the LLM is a thin skin.** All tracking, aggregation,
   trend/meaningful-change detection, reminder scheduling, adherence math, and ranking are
   deterministic, exhaustively-tested C#. The LLM only *understands intent*, *explains*, and
   *frames coaching in a warm voice*. **The model never computes a health number, never decides
   a threshold, and is never a safety boundary.**
3. **Per-person walls, hardened.** Health data is walled per member at the data layer (EF Core
   global query filter + `Manas.Crypto` field-level encryption). A curious child or a bad prompt
   fails at the wall, deterministically. Sensitive reads/actions require **step-up** (§14 of ADR 0001).
4. **Wellness, never medicine.** *Every* output is **general wellness information, never
   diagnosis, treatment, therapy, or personalized medical/psychological advice.** Ojas is not a
   licensed clinician and says so. Anything high-stakes — alarming vitals, a crisis signal,
   medication dosing, a child red-flag — **defers to a human clinician or a helpline** via a
   deterministic guardrail (§2 *Safety & Guardrails*, §6). This framing is load-bearing, not
   boilerplate.
5. **Composable and inspectable.** Ojas is independently toggleable (`Services:Ojas:Enabled`).
   Every derived insight is traceable to its source data (Saakshi), and every member has
   right-to-inspect and right-to-forget over their own record.

The product edge: a family health record that is **genuinely private**, **honest about its
limits**, **historical** (so it sees trends a snapshot app can't), and **coaching for the mind as
seriously as the body** — the mantra module being the concrete proof of the platform's pluggable
capability-module design.

---

## 2. Capability areas

**Legend**

- **Kind** — `[deterministic]` pure C#, no model · `[LLM]` model behind a Medha port (explain/
  coach/converse only) · `[integration:name]` needs an external system/bridge.
- **Feasibility** — `now` (build on today's stack) · `needs-hardware` (a device/wearable/sensor)
  · `needs-3rd-party` (an external API, model, or dataset to acquire/train).
- **Priority** — `P0` first vertical slice / foundational · `P1` core value, soon after · `P2`
  later / depends on P0–P1.

### 2.0 Cross-cutting: Safety & Guardrails (foundational — build first)

The guardrail is **deterministic and runs before and after every coaching output** — it is not
the model's judgment. It is the single most important part of Ojas.

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Wellness-framing governor | Deterministic output gate: every health/nutrition/mental reply is checked for and stamped with wellness-not-medical framing; strips/blocks any diagnostic or dosing language the model emits. Generalises Vatara's `ResponseGovernor`. | [deterministic] | now | P0 |
| Red-flag vitals detector | Rule table over incoming vitals (e.g. BP ≥180/120, glucose <54 or >300 mg/dL, SpO₂ <90%, child fever thresholds). On trip → does **not** diagnose; surfaces "these numbers are in a range that warrants contacting a clinician" + urges care. Thresholds are config data, sourced from public guidance, reviewed by a human. | [deterministic] | now | P0 |
| Crisis / distress signal detector | Deterministic lexical + safety-classifier gate on mood entries, journals, and mental-wellness turns (self-harm, hopelessness, a child in distress). On trip → suppresses coaching, shows helpline resources (e.g. 988/local), and — for a child — flags to an admin per policy. Reuses Vatara's `IChildComfort` + Llama-Guard/ShieldGemma classifiers. | [deterministic]+[LLM classifier] | now | P0 |
| Medical-advice refusal | Requests for diagnosis, prescriptions, dosing, or "should I stop my medication" are deterministically **refused with a referral**, never answered by the model. | [deterministic] | now | P0 |
| Step-up on sensitive read/write | Reading/exporting another member's record, editing medications, or bulk-forgetting requires an auth strong factor (per ADR 0001 §5.3). | [deterministic] | now | P0 |
| Source & disclaimer provenance | Every coaching card carries its data provenance (which readings) and a persistent "general wellness, not medical advice" disclaimer; guidance thresholds cite their public source. | [deterministic] | now | P0 |
| Child-mode strictness | The child's surfaces route through the stricter child-safety policy for all content (no nutrition restriction/diet talk to a child; body-positive, encouraging framing only). | [deterministic]+[LLM] | now | P0 |

### 2.1 Vitals ingest & manual entry

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Apple Health continuous ingest | iOS bridge streams HealthKit samples (steps, HR, resting HR, HRV, sleep, workouts, weight, SpO₂, respiratory rate, glucose, cycle) into Ojas; incremental via `HKAnchoredObjectQuery` anchors; idempotent (`sourceUUID`) so re-sync never double-counts. | [integration:HealthKit] | now (needs iPhone) | P0 |
| Manual vital entry (voice) | "My blood pressure is 128 over 82" → parsed to a typed BP reading with unit validation and step-up-free write to own record. Deterministic parse + validation; LLM only extracts intent. | [deterministic]+[LLM] | now | P0 |
| Manual vital entry (form) | Akriti forms for BP, glucose, weight, temperature, sleep, mood — WCAG 2.2 AA, voice-complete. | [deterministic] | now | P0 |
| Supported vitals set | BP (systolic/diastolic/pulse), blood glucose (+ meal-context tag), body weight, body temperature, sleep (duration/stages), resting & active HR, HRV, SpO₂, respiratory rate, mood/energy. | [deterministic] | now | P0 |
| Unit & range validation | Every reading validated against a plausible physiological range + unit normalisation (mg/dL↔mmol/L, °C↔°F, kg↔lb) on write; implausible → refuse/confirm. | [deterministic] | now | P0 |
| Correlated readings (BP) | BP stored as a correlation of systolic+diastolic+pulse at one timestamp (mirrors `HKCorrelation`). | [deterministic] | now | P0 |
| Bluetooth device import | Direct cuff/scale/glucometer/CGM import beyond what HealthKit relays. | [integration] | needs-hardware | P2 |
| Backfill / historical import | One-time bulk import of a HealthKit export (`export.xml`) into MinIO + parse to history. | [integration:HealthKit] | now | P1 |

### 2.2 Trend analysis & meaningful-change detection

**All deterministic — this is Ojas's analytical core, and no LLM touches it.**

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Time-series aggregation | Daily/weekly/monthly roll-ups (mean, min, max, last, count) per metric per member over the append-only history; multi-threaded batch (`Parallel.ForEachAsync`). | [deterministic] | now | P0 |
| Rolling baselines | Per-metric personal baseline (e.g. 30/90-day trailing mean + IQR) so "change" is relative to *this person*, not a population. | [deterministic] | now | P0 |
| Meaningful-change detection | Flags a change only when it clears a configured effect size / sustained duration (e.g. resting HR up ≥7 bpm sustained 7 days; weight trend ±X% over N weeks) — filters daily noise. Thresholds are config data. | [deterministic] | now | P0 |
| Trend direction & streaks | Deterministic slope/streak detection (improving/worsening/flat) per metric; streak tracking for adherence & activity. | [deterministic] | now | P0 |
| Correlation surfacing | Deterministic correlation across metrics (e.g. sleep vs resting HR, steps vs mood) — reported as *observed association, not causation*. | [deterministic] | now | P1 |
| Anomaly / outlier flag | Statistical outlier detection (robust z-score / MAD) → routes through the red-flag detector, not to diagnosis. | [deterministic] | now | P1 |
| Plain-language trend summary | LLM turns a *computed* trend object into a warm sentence ("your resting heart rate has settled about 5 bpm lower this month"). LLM never produces the number, only the prose. | [LLM] | now | P1 |
| Weekly digest | Scheduled per-member digest of computed trends + one gentle suggestion, gated by the wellness governor. | [deterministic]+[LLM] | now | P1 |

### 2.3 Nutrition guidance & meal planning

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Food / meal logging | Log meals by voice or form; map to a local nutrition database (calories/macros/micros). Deterministic lookup + storage. | [deterministic]+[integration:food-db] | now (needs food dataset) | P1 |
| Nutrient roll-up & trends | Deterministic daily/weekly macro & micronutrient totals and trends per member. | [deterministic] | now | P1 |
| General nutrition coaching | LLM frames *general* healthy-eating guidance grounded in the family's logged data + preferences — never a prescriptive medical diet; child-mode forbids restrictive/diet framing. | [LLM] | now | P1 |
| Family meal planning | Deterministic constraint solver builds a weekly plan from household preferences, dietary tags (veg/allergies), and pantry; LLM narrates & varies. | [deterministic]+[LLM] | now | P1 |
| Recipe suggestions | LLM suggests recipes fitting the plan + constraints; deterministic filter enforces allergy/veg hard constraints (model can't override a safety constraint). | [LLM]+[deterministic] | now | P2 |
| Grocery list generation | Derive a shopping list from the meal plan; hand off to Kaya/Artha or a list. | [deterministic] | now | P2 |
| Hydration tracking | Log & remind on water intake; simple deterministic goals. | [deterministic] | now | P2 |
| Barcode / photo food capture | Scan packaged-food barcodes or estimate a plate from a photo (VLM). | [integration]+[LLM] | needs-3rd-party | P2 |

### 2.4 Fitness & activity

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Activity ingest | Steps, distance, floors, active/basal energy, exercise minutes, stand hours, workouts from HealthKit. | [integration:HealthKit] | now | P0 |
| Goals & streaks | Per-member activity goals (steps/active-minutes) with deterministic progress, streaks, and nudges. | [deterministic] | now | P1 |
| Workout history & trends | Aggregate workouts by type/duration/intensity over time; trend detection. | [deterministic] | now | P1 |
| Activity coaching | LLM frames encouraging, *general* movement suggestions grounded in trends (never a prescriptive training plan for a condition). | [LLM] | now | P1 |
| Sedentary / move reminders | Deterministic reminders on inactivity windows (config'd, quiet-hours-aware). | [deterministic] | now | P2 |
| Family challenges | Opt-in shared step/activity challenges across members (privacy-respecting: shares only agreed aggregates). | [deterministic] | now | P2 |

### 2.5 Sleep analysis

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Sleep ingest | `sleepAnalysis` stages (awake/REM/core/deep), duration, timing from HealthKit; manual "slept 11–6" entry fallback. | [integration:HealthKit]+[deterministic] | now | P0 |
| Sleep metrics & trends | Deterministic duration, efficiency, consistency (bed/wake regularity), stage mix; personal baselines & trends. | [deterministic] | now | P1 |
| Sleep + vitals correlation | Correlate sleep with resting HR/HRV/mood (observed association only). | [deterministic] | now | P1 |
| Sleep hygiene coaching | LLM frames general sleep-hygiene tips from computed patterns; child-mode = gentle bedtime-routine framing. | [LLM] | now | P1 |
| Wind-down routine | Optional scheduled wind-down (dim lights via Kaya, breathing exercise, quiet mantra) — cross-service. | [deterministic]+[integration:Kaya] | now | P2 |

### 2.6 Medication & appointment reminders & adherence

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Medication & supplement registry | Per-member list: name, dose, schedule, purpose-note. **Records/reminds; never advises on dosing.** Editing requires step-up. | [deterministic] | now | P1 |
| Dose reminders & schedule | Durable scheduler (Quartz/Hangfire) fires timezone-correct reminders via Sarathi/notifications; snooze/skip logged. | [deterministic] | now | P1 |
| Adherence tracking | Deterministic taken/skipped/missed log → adherence % and streaks; gentle, non-judgmental nudges. | [deterministic] | now | P1 |
| Refill reminders | Deterministic countdown from quantity + schedule → refill nudge; optional handoff to Artha/pharmacy list. | [deterministic] | now | P2 |
| Appointment tracking | Store appointments; reminders; optional two-way with the calendar via Sangam. | [deterministic]+[integration:calendar] | now | P1 |
| Interaction check | **Explicitly out of scope as advice** — see §6. At most a deterministic "ask your pharmacist" nudge, never a computed interaction verdict. | [deterministic] | now | P0 (guardrail) |

### 2.7 Symptom & health journaling

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Symptom / health journal | Free-text or voice journal entries, timestamped, per member, encrypted; tag symptoms + severity. | [deterministic] | now | P1 |
| Structured symptom capture | Deterministic capture of common symptom tags + severity/duration for trendable history. | [deterministic] | now | P1 |
| Journal semantic recall | Embed entries (nomic-embed-text, Qdrant, per-user) for "when did I last have this" recall. | [deterministic]+[integration:embed] | now | P2 |
| Reflective summary | LLM summarises a journal window into themes — **not** a diagnosis; routes to red-flag/crisis detectors first. | [LLM] | now | P2 |
| Symptom → care nudge | Persistent or red-flag symptoms → deterministic "worth contacting a clinician" nudge; never a differential. | [deterministic] | now | P0 (guardrail) |

### 2.8 Women's-health tracking

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Cycle tracking | Period start/end, flow, cycle length; deterministic predictions of next period & fertile window from personal history (not a population model). Strictly walled to the member; **not** shared to other members by default. | [deterministic] | now | P1 |
| HealthKit cycle ingest | Ingest menstruation/BBT/cervical-mucus categories from HealthKit where present. | [integration:HealthKit] | now | P1 |
| Symptom & mood overlay | Overlay cycle phase with logged symptoms, mood, sleep for observed patterns. | [deterministic] | now | P1 |
| BBT / fertility signs log | Basal body temperature charting + fertility-sign logging; deterministic, **not** presented as contraception (see §6). | [deterministic] | now | P2 |
| Pregnancy / postpartum mode | Optional phase mode with general-wellness content; explicitly defers clinical questions. | [deterministic]+[LLM] | now | P2 |
| Extra-strict privacy | Cycle data gets the tightest wall + optional PIN-gate even for the owner's quick views; never surfaced in family/shared digests. | [deterministic] | now | P1 |

### 2.9 Pediatric tracking (the child)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Growth tracking | Height/weight/head-circumference over time; deterministic **percentile** placement against a **local copy of WHO/CDC growth charts** (reference data, not a diagnosis). | [deterministic]+[integration:growth-charts] | now (needs chart data) | P1 |
| Growth trend & velocity | Deterministic growth-velocity and percentile-crossing detection → gentle "worth mentioning at the next checkup" nudge, never an alarm/diagnosis. | [deterministic] | now | P1 |
| Vaccination record & schedule | Store administered vaccines; deterministic schedule (config'd from a public immunization schedule) → upcoming/overdue reminders. Records/reminds; the schedule is informational. | [deterministic] | now | P1 |
| Pediatric milestones | Optional developmental-milestone checklist (informational; explicitly "discuss with your pediatrician"). | [deterministic] | now | P2 |
| Child fever / symptom helper | Child temperature logging with **age-aware red-flag thresholds** → strong "contact a pediatrician / after-hours line" nudge; never dosing advice. | [deterministic] | now | P0 (guardrail) |
| Parent-managed, child-visible | The child's record is managed by admins; child-facing surfaces show only encouraging, age-appropriate content (Bonnie persona). | [deterministic] | now | P1 |

### 2.10 Mental Wellness (flagship area)

General wellness for the mind. **Not therapy, not diagnosis.** Every surface here routes through
the crisis/distress detector (§2.0) *before* any coaching, and defers to a helpline/clinician on
any red flag.

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Mood & energy check-in | Fast (<30s) daily mood/energy/stress check-in (scale + optional note), by voice or form; per member, encrypted. | [deterministic] | now | P1 |
| Mood trends & correlation | Deterministic mood trends + correlation with sleep/activity/cycle (observed association only). | [deterministic] | now | P1 |
| Guided breathing | Deterministic timed breathing exercises (box, 4-7-8, coherence) with visual/audio pacing via Akriti + TTS; configurable patterns. No model needed. | [deterministic] | now | P1 |
| Meditation / mindfulness timer | Timed sessions with interval bells, ambient audio, streaks; optional guided scripts. | [deterministic] | now | P1 |
| Guided meditation scripts | LLM generates/narrates a short guided script to a theme (calm, focus, sleep) via Piper TTS; child-safe variant for the child; content pre-governed. | [LLM]+[integration:TTS] | now | P2 |
| Gratitude journaling | Prompted daily gratitude entries; deterministic streaks; gentle reflective LLM prompts. | [deterministic]+[LLM] | now | P1 |
| Reflective / CBT-style prompts | LLM offers **general** self-reflection prompts (reframing, thought-noticing) — explicitly *not* therapy; crisis-gated; hands off to a professional for anything clinical. | [LLM] | now | P2 |
| Mood-aware coaching | Sarathi adapts *tone* (never truth) to a low-mood check-in — warm, patient; anti-sycophancy rule applies. | [LLM] | now | P2 |
| Family calm routines | Shared wind-down / breathing for the household (e.g. bedtime with the child), cross-service with Kaya (dim lights, quiet). | [deterministic]+[integration:Kaya] | now | P2 |

#### 2.10.1 Sanskrit Mantra-Chanting module (the pluggable-capability exemplar)

The concrete demonstration of ADR 0001 §5.5: a **capability module** behind `ICapability`, with
its **own lightweight task-specific model(s)** brokered by Medha, auto-discovered as a Sarathi
tool, and toggleable in config — added by *accretion, not surgery*. It teaches correct
pronunciation, sandhi, rhythm/tempo, and call-and-response, and gives feedback on the user's
chanting.

**Honest feasibility ceiling (researched).** Robust *phoneme-perfect* Sanskrit pronunciation
scoring is **not** reliably solvable on-device today. Sanskrit ASR with Whisper transfer learning
reaches ~**15.4% WER (769M "medium")** and ~**19.9% WER (39M "small")** on read Sanskrit speech
(Vāksañcayaḥ, 78h/27 speakers) — and *chanting* (sustained pitch, elongation, svara) is
**out-of-domain** vs. that read-speech training data, where WER degrades further (~37% OOD).
Retroflexes, breathy consonants, and sandhi are exactly where ASR is weakest. **Conclusion:** the
reliable, feasible core is **deterministic signal processing** (pitch/tempo/timing alignment via
DTW against a reference recording — the approach real apps like HYVE's "Mantra Echo" and Vedic
Voice use, in-browser/on-device), with ASR used only as an *optional, clearly-hedged* word/syllable
detector — never as an authoritative "your pronunciation is wrong" verdict. We coach with
reference audio + objective rhythm/pitch signals + an LLM tutor for *explanation*, and we are
explicit that fine pronunciation judgement needs a human teacher.

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Mantra library | Curated mantras with Devanagari + IAST transliteration + word-by-word meaning + a reference recording per mantra. Data, not code. | [deterministic]+[integration:content] | now (needs content) | P1 |
| Reference playback & slow-mode | Play the reference at normal/half speed; loop lines; call-and-response segmentation (guru line → learner repeats). | [deterministic]+[integration:TTS/audio] | now | P1 |
| Syllable/word highlighting | Karaoke-style highlight synced to the reference timeline (deterministic, pre-aligned). | [deterministic] | now | P1 |
| Rhythm/tempo feedback | DTW-align the learner's audio envelope to the reference → objective tempo/timing/pause feedback ("you rushed the second line"). Signal processing, deterministic. | [deterministic] | now | P1 |
| Pitch / svara feedback | Extract pitch contour (e.g. CREPE/pYIN-class) and compare against the reference's udātta/anudātta/svarita contour → objective intonation feedback. | [deterministic]+[integration:pitch-model] | now (needs pitch lib) | P2 |
| Pronunciation hinting (hedged) | Optional per-syllable ASR/forced-alignment flag of *likely* mismatches, **always** presented as a hint ("this syllable may need the retroflex ṇa") with a "confirm with a teacher" note. Honest about limits. | [LLM/ASR]+[deterministic] | needs-3rd-party (train/fine-tune Sanskrit acoustic model) | P2 |
| Chanting tutor (explanation) | LLM tutor explains meaning, sandhi rules, and *why* a line joins the way it does; answers "what does this mean"; child-safe for the child. Explanation only — not the scorer. | [LLM] | now | P2 |
| Practice tracking | Deterministic streaks, mantra mastery progress, session history — feeds mental-wellness trends. | [deterministic] | now | P1 |
| Call-and-response drill | Timed guru-repeats-learner loop with the rhythm/pitch feedback per line. | [deterministic]+[integration:audio] | now | P2 |

---

## 3. Data model sketch

Deterministic, append-only where it matters; per-user walled; sensitive fields `[Encrypted]`.
Postgres is the system-of-record; Qdrant holds journal embeddings; MinIO holds blobs (HealthKit
exports, reference audio, growth-chart images); Neo4j links members↔metrics↔insights for
cross-domain queries.

**Core entities** (all carry `Manas.Kernel` base: `TenantId`, `OwnerId`, `CreatedBy{User,Service}`,
audit provenance; per-user wall filter applied automatically):

- **`HealthProfile`** — one per member: birthdate, sex-at-birth (for reference ranges/percentiles,
  member-editable), unit preferences, consent flags per source, enabled capability modules.
- **`VitalReading`** *(append-only)* — `id, memberId, metricType, value, unit, takenAt, recordedAt,
  source {Manual|HealthKit|Device}, sourceUUID, mealContext?, note?`. `[Encrypted]` on `value`/`note`.
  `metricType` is an enum-table (BP-systolic, BP-diastolic, pulse, glucose, weight, temperature,
  spo2, respRate, restingHR, hrv, …). BP grouped by a `correlationId`.
- **`SleepSession`** *(append-only)* — `memberId, start, end, stages[], efficiency, source`.
- **`ActivitySample` / `Workout`** *(append-only)* — steps/energy/minutes; workout type/duration/
  intensity.
- **`NutritionEntry`** — `memberId, mealType, items[]{foodId, qty, macros, micros}, loggedAt`.
- **`MoodEntry`** *(append-only)* — `memberId, mood, energy, stress, note?, loggedAt`. `[Encrypted]`.
- **`JournalEntry`** *(append-only)* — `memberId, text (encrypted), tags[], symptomTags[], severity?,
  embeddingRef→Qdrant, loggedAt`.
- **`Medication`** — `memberId, name, dose, schedule (RRULE), purposeNote, active`. Edits step-up'd.
- **`MedicationEvent`** *(append-only)* — `medicationId, scheduledAt, status {Taken|Skipped|Missed},
  actedAt`.
- **`Appointment`** — `memberId, title, whenUtc, location?, calendarRef?`.
- **`CycleEntry`** *(append-only, extra-strict wall)* — `memberId, date, flow?, bbt?, signs[], phase`.
- **`GrowthMeasurement`** *(child)* — `memberId, height, weight, headCirc, measuredAt, percentiles{}`.
- **`VaccinationRecord`** *(child)* — `memberId, vaccine, doseNumber, administeredAt|dueAt, status`.
- **`ChantingSession`** *(append-only)* — `memberId, mantraId, durationSec, tempoScore, pitchScore?,
  completeness, practicedAt`. **`Mantra`** (reference content) — `id, devanagari, iast, meaning,
  segments[], referenceAudioRef→MinIO, tempoTemplate, pitchContourTemplate`.
- **`Insight`** *(derived, append-only)* — `memberId, kind {Trend|MeaningfulChange|Correlation|
  Anomaly|RedFlag}, metricType, computedValue, windowRange, provenance[readingIds], severity,
  createdAt`. **Computed only by deterministic analyzers**; the LLM reads these to phrase, never writes.
- **`Reminder`** — medication/appointment/vaccine/hydration; RRULE + timezone; durable-scheduler backed.
- **`Consent`** *(append-only)* — per member per source (HealthKit, food-db…): granted/revoked, when.

**Derived/analytical (deterministic, not stored as truth):** `Baseline` (per metric/member,
recomputed), roll-up materialised views. **Insights are the only persisted derivations**, always
with `provenance` back to raw readings so Saakshi can explain any card.

**Value objects / rules** (Domain, pure, LLM-free): `Bp`, `Glucose`, `Weight`, `Temperature`,
`BloodPressureCategory` (informational only), `PhysiologicalRange` (validation), `EffectSize`,
`Percentile`, `AdherenceRate`, `CyclePrediction`, `TempoScore`, `PitchScore`. All exhaustively
unit-tested (the 10× estate).

---

## 4. Integrations needed

| Integration | Purpose | Path / notes | Feasibility |
|---|---|---|---|
| **Apple HealthKit** | Primary passive ingest of vitals/activity/sleep/cycle | HealthKit is **iOS-only** → the **`Manas.iOS` bridge** app reads via `HKAnchoredObjectQuery` (incremental anchors, background delivery), Sign-in-with-Apple for identity, pushes to Ojas over **mTLS** on the LAN. Idempotent by `sourceUUID`. **Read-only from HealthKit; nothing written back / out.** Consent recorded per type. | now (needs an iPhone in the home) |
| **HealthKit bulk export** | One-time historical backfill | User exports `export.xml`; stored in MinIO; parsed to `VitalReading` history. | now |
| **Nutrition / food database** | Food → macros/micros lookup | A **local** dataset (e.g. an open food-composition table); no live third-party call → stays private. | needs-3rd-party (acquire dataset) |
| **Growth-chart reference data** | Pediatric percentiles | **Local copy** of WHO/CDC L-M-S growth-chart parameters (public). Deterministic percentile math on-box. | now (data acquisition) |
| **Immunization schedule** | Vaccine due/overdue | Local config of a public schedule (e.g. national immunization schedule); informational. | now (data) |
| **Mantra content + reference audio** | Chanting module | Curated mantra text (Devanagari/IAST/meaning) + reference recordings (licensed/own), stored in MinIO. | now (content acquisition) |
| **Pitch-extraction library** | Chanting pitch/svara feedback | On-device pitch tracker (pYIN/CREPE-class) behind a port. | now (integrate lib) |
| **Sanskrit acoustic model (optional)** | Hedged pronunciation hints | Fine-tuned Whisper-small-class / forced-aligner; **honestly limited** (see §2.10.1). Runs via Medha. | needs-3rd-party (train/fine-tune) |
| **Medha (LLM gateway)** | All explanation/coaching/tutoring + safety classifiers | `IChatClient` behind a port; QoS; spend/trace; safety models (Llama-Guard/ShieldGemma). Never computes health numbers. | now (platform) |
| **Piper TTS** | Guided meditation, breathing pacing, mantra playback, spoken coaching | Existing Vatara Piper adapter, reused via Medha/Kaya audio. | now |
| **Whisper STT** | Voice vital entry, voice journaling, chanting capture | Existing STT seam. | now |
| **Calendar (via Sangam)** | Appointment sync | Sangam connector, governed, read-vs-write tiers. | now (Sangam) |
| **Durable scheduler** | Reminders (meds/appointments/vaccines/hydration) | Quartz/Hangfire, timezone-correct (ADR 0001 §14). | now |

---

## 5. Interconnections with other services

Ojas is one mind with the platform — always via **governed MCP tools carrying the caller's
verified identity**, never a shared table (ADR 0001 §5.2/§5.4).

- **Sarathi (the Agent).** Exposes Ojas MCP tools: `ojas.record_vital`, `ojas.get_trend`, `ojas.get_insights`,
  `ojas.log_mood`, `ojas.start_breathing`, `ojas.log_medication_taken`, `ojas.get_meal_plan`,
  `ojas.start_chanting_practice`, `ojas.journal_add`. Sarathi routes intent and phrases replies;
  the **wellness governor + red-flag/crisis detectors run regardless of the model's output.**
- **Kaya (devices).** Wind-down/calm routines: dim lights, set climate, play mantra/meditation
  audio on a room speaker; morning readiness could adjust based on sleep. Ojas *requests* via
  Kaya's tools; Kaya owns actuation.
- **Kavach (security).** Read-only, opt-in: a night-time medical red-flag *could* inform an alert
  path, but **Kavach's life-safety actions stay deterministic and independent** — Ojas never
  actuates the alarm.
- **Artha (finance).** Refill/appointment costs, HSA/FSA context, insurance-renewal reminders;
  meal plan → grocery budget. Both walled; only agreed aggregates cross, per-user.
- **Abhyasa (school).** The child's sleep/mood/activity trends can gently contextualise learning-energy
  patterns (observed association, never a claim); shared only as coarse, consented signals.
- **Kutumba.** Members, roles (admin vs child), relationships, birthdates → reference ranges,
  percentiles, child-mode routing, who-can-see-whom.
- **Sangam.** Calendar (appointments), optionally a pharmacy/list; governed, read-vs-act tiers.
- **Medha.** All model calls; standardised model set; QoS (a live coaching turn outranks a batch
  digest); spend/trace.
- **Saakshi.** Every ingest, insight computation, coaching turn, and wall denial is traced
  (correlation id) — but health **values are redacted/hashed** in logs, encrypted at rest.
- **Bodha.** Reports Ojas health/topology and can answer "why did you flag my resting HR?" by
  walking the insight's provenance.
- **Dwarpal / Manas.Crypto.** Step-up on sensitive reads/exports/med-edits; field-level encryption on
  all health values; per-user walls at the data layer.

---

## 6. Out of scope (and why)

**The medical-advice guardrail (the central non-goal).** Ojas is **general wellness, not
medicine.** The following are deliberately excluded; each is a deterministic refusal-with-referral,
never an LLM answer:

- **Diagnosis of any condition.** No "you have X." Ojas surfaces *observed data and trends* and,
  on a red flag, urges contacting a clinician. *Why:* diagnosis is a licensed clinical act;
  liability, safety, and the local-first honesty principle forbid it.
- **Treatment, prescriptions, or dosing.** No "take N mg", "stop your medication", "this dose is
  fine." Ojas *records and reminds* medications only. *Why:* dosing is clinical; a model error here
  is dangerous.
- **Drug-interaction verdicts.** At most "ask your pharmacist"; never a computed interaction claim.
  *Why:* incompleteness → false reassurance is harmful.
- **Therapy / clinical mental-health treatment.** Mental wellness is check-ins, breathing,
  meditation, gratitude, general reflection, and chanting — **not** psychotherapy, not a clinical
  assessment, not a substitute for care. Crisis → helpline/clinician, deterministically. *Why:*
  therapy is licensed and high-stakes; a wellness app must not impersonate it.
- **Contraception / fertility as medical guidance.** Cycle tracking is informational; it is **not**
  presented as birth control or fertility treatment (unlike FDA-cleared apps). *Why:* that is a
  regulated medical claim.
- **Emergency response.** Ojas is not an emergency service; red flags point to 911/112/local
  emergency + helplines, and it says so plainly. *Why:* latency and reliability of a home system
  must never sit between a person and emergency care.
- **Authoritative Sanskrit pronunciation grading.** The chanting module coaches with reference
  audio + objective rhythm/pitch signals and *hedged* hints; it does **not** claim
  teacher-grade phoneme judgement. *Why:* ASR for chanted Sanskrit is not reliable enough
  (§2.10.1) — overclaiming would be dishonest.
- **Fabricated numbers or trends.** The LLM never produces a health value or computes a trend;
  those are deterministic. *Why:* the #1 platform risk is an over-trusted model fabricating.
- **Selling/sharing data, cloud analytics, ad targeting.** Structurally impossible: data never
  leaves the box. *Why:* it is the product's core promise.
- **Cross-member snooping.** No member (including a curious child) sees another's record without a
  wall + step-up; cycle/journal data is extra-walled. *Why:* per-user walls are foundational.

---

### Phasing note (aligns with ADR 0001 §11)

**P0 first vertical slice — "BP → trend → wellness nudge, walled + safe":** vitals ingest (manual +
HealthKit), the append-only store + per-user wall + encryption, deterministic trend/meaningful-
change detection, and the **full Safety & Guardrails layer** (wellness governor, red-flag detector,
crisis detector, medical-advice refusal, step-up). This exercises write→read→recommend, the walls,
step-up, and the wellness guardrail on the real foundation. **P1** adds sleep/activity/nutrition
trends, medication & appointment reminders + adherence, mood check-ins, breathing/meditation,
women's-health, pediatric growth/vaccines, and the mantra library + rhythm feedback. **P2** adds
correlations, meal planning, semantic journal recall, guided-meditation generation, pitch/svara
and hedged pronunciation feedback, and device/photo integrations.
