# Abhyasa — the children's learning steward (vision)

> **Status:** Vision / intent draft · **Date:** 2026-07-11 · **Service:** `src/Services/Abhyasa`
> **Parent ADR:** [`docs/adr/0001-manas-platform-architecture.md`](../adr/0001-manas-platform-architecture.md)
> Abhyasa (अभ्यास — *dedicated practice & study*) is a Manas domain service. It honours the platform
> principles: **local-first & private**, **deterministic C# is the bulk / the LLM is a thin child-safe skin**,
> **per-user walls that survive the agent boundary**, **child-safety is paramount**, **accessible (voice-complete,
> WCAG 2.2 AA)**, and **capability modules that grow by accretion, not surgery**.

Primary user: **the child**, with the **parent (Ananth / the other adult) as guardian/admin**. Built member-agnostic
so it extends to more children without a rewrite.

---

## 1. Intent

> **Abhyasa is the children's learning steward.** It **continuously records grades, assignments, tests, reading,
> and extracurricular progress**; tracks **every subject and skill over time**; and **surfaces concrete,
> encouraging areas of improvement** and next steps — turning scattered school data into a clear picture of where
> a child *thrives* and where a gentle nudge helps. **All tracking, trend detection, ranking, and
> improvement-area detection are deterministic C#**; the LLM only *frames the encouragement* and *optionally
> explains a concept* in a child-safe, motivating voice. It **helps a child learn — it never does the homework**.

Design commitments that shape every feature below:

- **Deterministic truth, LLM delivery.** A grade average, a downward trend, a "weakest skill" ranking, a streak,
  a due-date reminder — all computed by tested C#. The model never invents a number, a grade, or a standard. It
  receives the deterministic result and phrases it warmly. (ADR §2 driver #4 / §5.2.)
- **Encouragement, never shaming.** The coaching voice is the child's assistant persona (Bonnie): loyal, patient,
  funny, growth-mindset. It celebrates effort and progress, frames weak spots as "next adventures", and **never
  compares one child to another** or delivers a bare negative verdict.
- **Guardian-governed, child-facing-safe.** Parents own configuration, see everything, and set goals; the child
  sees an age-appropriate, positive subset. Sensitive actions (adding a portal credential, changing goals,
  right-to-forget) are **admin + step-up gated**. Grades are **per-child walled** — one sibling never sees
  another's grades.
- **Tutoring is bounded.** The optional concept-explanation capability *scaffolds understanding* (explains,
  gives a worked analogous example, asks a guiding question). It is **hard-fenced from producing the answer to a
  graded assignment** (§6).
- **Realistic about school data.** Automated import from school systems ranges from *feasible* (Canvas observer
  token) to *practically blocked* (PowerSchool, most districts' Google Workspace). Abhyasa is **excellent with
  manual/voice entry and report-card import first**, and treats every portal integration as a *best-effort
  enhancement* behind Sangam, never a dependency.

---

## 2. Capability areas

`Kind` legend: **[deterministic]** = pure C#, no model · **[LLM]** = child-safe model via Medha (framing/explanation
only, over a deterministic result) · **[integration:name]** = needs an external connector (via Sangam).
`Feasibility`: **now** = buildable on the Manas foundation today · **needs-3rd-party** = depends on an external
API/grant/school cooperation we don't control.

### 2.1 Grade, assignment & test recording

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Manual grade entry | Parent/child records a graded item: subject, title, type (test/quiz/homework/project/participation), score, max, weight, date, term. Voice ("the child got 18 out of 20 on her spelling test") or form. | [deterministic] | now | P0 |
| Assignment log | Record assignments with due dates, status (assigned/in-progress/submitted/graded/missing), and the resulting grade when known. | [deterministic] | now | P0 |
| Term & subject model | Configurable academic years, terms/semesters, subjects, and per-subject grading scheme (points, percentage, letter, standards-based, weighted categories). Behaviour-as-data — no hardcoded grade scales. | [deterministic] | now | P0 |
| Grade-scale normalisation | Convert letter/percentage/points/4.0-GPA/standards levels to a common internal scale so trends compare fairly across subjects and schools. | [deterministic] | now | P0 |
| Voice capture of a grade | "Bonnie, log that I got an A-minus on my science project" → parsed to a structured item, read back for confirmation before commit. | [LLM] parse + [deterministic] record | now | P0 |
| Report-card import (PDF/image) | Ingest an end-of-term report card (PDF or photo) → OCR → deterministic parse of subject/grade rows → proposed entries the parent confirms. Report cards are the highest-value *bulk* import that needs no portal. | [deterministic] OCR+parse (+ optional [LLM] layout assist) | now | P1 |
| Emailed-grade digest capture | Many schools email grade/progress updates. Parse those emails (via Sangam→email) into proposed grade entries for confirmation. Often more reliable than a portal API. | [integration:Sangam/email] + [deterministic] parse | needs-3rd-party | P1 |
| CSV / spreadsheet import | Import a gradebook export or a parent's own tracking sheet. | [deterministic] | now | P1 |
| Duplicate/conflict guard | Detect a re-imported or re-typed item (same subject+title+date) and merge rather than double-count. | [deterministic] | now | P1 |
| Correction & history | Edit a mis-entered grade; the change is audited (Saakshi) and the item keeps a full value history. | [deterministic] | now | P1 |

### 2.2 School-portal integration (import) — *realistic feasibility*

> **Honest assessment:** parent-facing programmatic access to school grade systems is **inconsistent and often
> blocked**. Abhyasa treats these as *opt-in enhancements per family*, gated behind Sangam grants and admin
> step-up, and **degrades to manual/report-card/email import** whenever a portal is unavailable. Never a hard
> dependency.

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| **Canvas** import (observer/parent) | Pull courses, assignments, due dates, and grades for a linked child using a **parent-generated Canvas access token** or observer enrolment. The most tractable portal: Canvas exposes grades to observers via REST, where the institution enables token generation. | [integration:Sangam/Canvas] | needs-3rd-party | P2 |
| **Google Classroom** import (student scope) | With the *child's own school Google account*, request the student read scopes (own coursework + grades). **Caveat:** post-2024 Google Workspace admin controls block unverified third-party apps by default — many districts will *deny* the OAuth grant. Guardian scope reveals guardians, **not** grades. | [integration:Sangam/Google Classroom] | needs-3rd-party | P2 |
| **PowerSchool** import | PowerSchool's API needs a **district-installed plugin + ClientID/Secret** we cannot obtain as a family; the parent portal has no open API. **Not feasible** without district cooperation. Fallback: emailed-grade parsing, report-card import, or manual entry. Documented as *not planned* until a district grant exists. | [integration:Sangam/PowerSchool] | needs-3rd-party (usually blocked) | P2 (deferred) |
| Import reconciliation | Any portal/email import produces *proposed* entries reconciled against existing manual entries (dedupe, conflict flag) and confirmed by the parent before they count. Import is never silently authoritative. | [deterministic] | now | P1 |
| Connection health & staleness | Bodha-style self-report: which imports are connected, last successful sync, and a clear "portal unavailable — using manual entries" state. | [deterministic] | now | P2 |

### 2.3 Trend & skill analysis (deterministic analytics)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Per-subject running average | Weighted/unweighted average per subject per term, plus all-time, from recorded items. | [deterministic] | now | P0 |
| Per-subject trend over time | Direction and slope of a subject's scores across a window (improving / steady / slipping), with a minimum-sample-size guard so one bad quiz isn't over-read. | [deterministic] | now | P0 |
| Per-skill / per-standard tracking | Roll grades up to skills/standards (e.g. "fractions", "reading comprehension", "multiplication facts") via a configurable item→skill tag map, and trend each skill. | [deterministic] | now | P1 |
| Category breakdown | Performance by item type (tests vs homework vs projects vs participation) to distinguish "knows it but misses homework" from "struggles with the material". | [deterministic] | now | P1 |
| Consistency / volatility metric | Flag erratic vs steady performance per subject (variance), which changes the coaching (routine vs concept help). | [deterministic] | now | P2 |
| Missing/late-work signal | Count and trend missing/late assignments — often the real lever, distinct from low scores. | [deterministic] | now | P1 |
| Effort vs outcome view | Combine practice/study logs (§2.6) and reading logs (§2.7) against outcomes to show effort, not just results. | [deterministic] | now | P2 |
| Term-over-term summary | Compare this term to last per subject/skill; feeds the report-card summary (§2.11). | [deterministic] | now | P1 |

### 2.4 Improvement-area detection & suggestions (the core intent)

> **This is the heart of the service.** Detection and ranking are **100% deterministic** — a transparent,
> testable, configurable rubric, *not* the model's opinion. The LLM only turns the ranked result into an
> encouraging sentence with concrete next steps drawn from a deterministic action library.

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Improvement-area ranking | Rank subjects/skills needing attention from a weighted, config-driven rubric: level below a threshold, negative trend, high volatility, missing-work rate, term-over-term drop. Fully deterministic and explainable ("flagged because: trend −8% over 5 items"). | [deterministic] | now | P0 |
| Strength detection | Symmetrically surface where the child excels — coaching leads with strengths (growth mindset), never only deficits. | [deterministic] | now | P0 |
| Concrete next-step library | A config-driven catalogue of specific, age-appropriate actions mapped to a detected pattern (e.g. missing-homework → a checklist routine; weak fractions → 3 practice sets + a spaced-repetition deck; slipping reading → +10 min/day log). Deterministic selection; no invented advice. | [deterministic] | now | P0 |
| Encouraging framing | Phrase the ranked area + chosen next steps in Bonnie's warm, growth-mindset voice, over the deterministic result. Anti-shaming, effort-focused, never comparative. | [LLM] | now | P0 |
| "One thing this week" nudge | Pick the single highest-leverage, achievable action for the week so a child isn't overwhelmed. Deterministic pick; LLM phrasing. | [deterministic] pick + [LLM] phrasing | now | P1 |
| Parent-vs-child framing split | The parent view shows the full ranked rubric with reasons; the child view shows one or two positively-framed focuses. Same deterministic result, two audiences. | [deterministic] + [LLM] | now | P1 |
| Suggestion feedback loop | Parent/child marks a suggestion helpful / done / not-now; future ranking and phrasing adapt (deterministic weighting, Steward/Vatara-style learning). | [deterministic] | now | P2 |

### 2.5 Homework & study planning + reminders

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Homework/assignment tracker | Central list of what's due, when, and status; the child can check items off by voice. | [deterministic] | now | P0 |
| Due-date reminders | Timely, escalating reminders ("science project due in 3 days / tomorrow / today") via the platform notification path; quiet-hours aware. | [deterministic] (durable scheduler) | now | P0 |
| Study-plan generation | From upcoming due dates + detected weak areas + available time blocks, generate a suggested weekly study plan (which subject, when, how long). Deterministic scheduling; LLM only narrates it kindly. | [deterministic] plan + [LLM] narration | now | P1 |
| Calendar sync | Surface assignments/study blocks on the family calendar (via the platform calendar/Sangam), and read school events if available. | [integration:calendar] | needs-3rd-party | P2 |
| Homework-time focus mode | Optional tie-in to Kaya (Vatara): a "homework time" scene (lights, do-not-disturb) — cross-service, opt-in. | [integration:Kaya] | now (if Kaya enabled) | P2 |

### 2.6 Practice tracking & spaced repetition

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Practice/study session log | Record a practice session (subject/skill, minutes, self-rated or quiz-scored). Voice or form. | [deterministic] | now | P1 |
| Spaced-repetition decks | Lightweight flashcard decks per skill (e.g. multiplication facts, spelling, vocab) scheduled by a proven deterministic algorithm (SM-2/Leitner). Parent- or child-created; content is data. | [deterministic] | now | P1 |
| Practice reminders | Schedule review sessions when a card/skill is due; quiet-hours aware. | [deterministic] | now | P1 |
| Mastery estimate | Deterministic per-skill mastery from practice accuracy + recency (not a grade) — feeds improvement ranking and "what to review". | [deterministic] | now | P2 |
| Auto-generate a deck from a weak area | When a skill is flagged weak, offer a starter practice deck; *content curated deterministically from a local bank or parent-added*, LLM may only help *phrase* a card from parent-provided facts (never invent quiz answers unchecked). | [deterministic] + bounded [LLM] | now | P2 |

### 2.7 Reading logs

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Reading log | Record books/sessions: title, minutes, pages, date; supports school reading-log requirements. Voice-friendly ("I read Charlotte's Web for 20 minutes"). | [deterministic] | now | P1 |
| Reading streaks & totals | Minutes/books per week/month, streaks, and progress toward a reading goal. | [deterministic] | now | P1 |
| Reading-log export/print | Produce the format a school wants (weekly log sheet) for submission. | [deterministic] | now | P2 |
| Book suggestions (bounded) | Suggest next reads from a *curated, parent-approved, age-appropriate local list* filtered by level/interest — deterministic filter; **no open web fetch of book content to a child**, no unbounded LLM recommendations. | [deterministic] | now | P2 |

### 2.8 Extracurricular & activity tracking

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Activity registry | Track sports, music, art, clubs: schedule, attendance, milestones (belt/level, piece learned, position). Behaviour-as-data — any activity type. | [deterministic] | now | P1 |
| Practice/attendance log | Log a practice or session and its focus/outcome (e.g. "piano — 30 min, learned the B-section"). | [deterministic] | now | P1 |
| Milestone & achievement timeline | A celebratory timeline of achievements across academics + activities (recital, medal, level-up). | [deterministic] | now | P2 |
| Activity reminders & schedule | Reminders for practices/games/lessons; surfaced on the family calendar. | [deterministic] (+ calendar) | now | P2 |
| Balance view | Show academics/activities/reading balance over a week so coaching can flag over- or under-load (deterministic; framed gently). | [deterministic] + [LLM] framing | now | P2 |

### 2.9 Goals, streaks & motivation

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Goal setting | Parent (and age-appropriately the child) sets goals: outcome ("bring math to a B"), habit ("read 20 min/day"), or activity ("practice piano 4×/week"). Progress computed deterministically from logged data. | [deterministic] | now | P1 |
| Streak tracking | Habit streaks (reading, practice, homework-on-time) with forgiving rules (grace days) so a single miss isn't crushing. | [deterministic] | now | P1 |
| Progress & celebration | Deterministic progress toward goals; the assistant celebrates milestones warmly (LLM phrasing) and marks them on the timeline. | [deterministic] + [LLM] | now | P1 |
| Reward/token system (optional) | Optional parent-defined points/rewards for goals/streaks — rules are data, fully parent-controlled, deterministic. | [deterministic] | now | P2 |
| Anti-pressure guardrail | Deterministic caps so streaks/goals stay encouraging, not anxiety-inducing (no "you broke your streak" shaming; frame as "let's restart"). | [deterministic] + [LLM] | now | P1 |

### 2.10 Curriculum / standards mapping

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Standards catalogue | Load a standards set (Common Core, NGSS, or a state framework) as **data** — these are freely available, publicly published datasets. Map subjects/skills to standard codes. | [deterministic] | now (open datasets) | P2 |
| Grade↔standard alignment | Tag graded items and skills to standards so trends roll up to "which standards are strong/weak". | [deterministic] | now | P2 |
| Coverage view | Show which standards for the child's grade have evidence vs none yet (informational, not authoritative). | [deterministic] | now | P2 |
| Standards-based report translation | Translate a school's standards-based grades (e.g. "3 = meeting") to the internal scale and back for display. | [deterministic] | now | P2 |

### 2.11 Report-card & summary generation

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Term summary (deterministic) | Compute a per-term rollup: subject averages, trends, top strengths, top improvement areas, missing-work count, reading/practice totals, goals met. | [deterministic] | now | P1 |
| Narrative report-card summary | Turn the deterministic rollup into a warm, parent-facing narrative ("the child's reading grew steadily; math homework consistency is the next focus"). LLM over the computed facts — no invented numbers. | [LLM] over [deterministic] | now | P1 |
| Child-facing "my term" recap | A celebratory, age-appropriate recap for the child, leading with wins and one friendly next focus. | [LLM] over [deterministic] | now | P2 |
| Printable/exportable report | Export the summary (PDF) for records or to share with a tutor/teacher (parent-initiated). | [deterministic] | now | P2 |

### 2.12 Gentle tutoring (concept help — bounded)

> **Charter: help the child *learn*, never *do the assignment*.** This capability explains a concept, gives a
> worked *analogous* example, and asks a guiding question. It is a config-toggle (`Abhyasa:Tutoring:Enabled`),
> child-safety-gated on every turn, and hard-fenced from producing graded answers (§6).

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Concept explanation | Explain an age-appropriate concept ("what's a common denominator?") in kid-friendly language with an analogy. | [LLM] (child-safe, Medha) | now | P2 |
| Worked *analogous* example | Demonstrate the *method* on a **different** example than the child's actual problem, then hand the method back. | [LLM] bounded | now | P2 |
| Socratic guiding question | Ask a next-step question ("what do you think we multiply first?") to prompt the child's own reasoning. | [LLM] bounded | now | P2 |
| "Do-my-homework" refusal | Detect and warmly decline requests to produce the graded answer; redirect to explanation/practice. Deterministic + safety-gate backstop, not left to model goodwill. | [deterministic] gate + [LLM] redirect | now | P0 (if tutoring on) |
| Safety-gated every turn | Every tutoring turn passes the platform child-safety input/output gates (Llama-Guard / ShieldGemma-class), carried from Vatara. | [deterministic] gate + [LLM] | now | P0 (if tutoring on) |
| Escalate-to-parent | If the child is stuck/frustrated or asks something out of scope, suggest asking the parent/teacher rather than over-reaching. | [deterministic] + [LLM] | now | P2 |

---

## 3. Data model sketch

Postgres is the structured system-of-record; every entity carries `TenantId` + `OwnerId` (the child) with the
Kernel's global query filter (per-child wall) and audit interceptor. Blobs (report-card PDFs/photos) go to
MinIO; embeddings (semantic notes/decks) to Qdrant; the child↔skill↔standard relationships to Neo4j
(`IKnowledgeGraph`). Nothing here is read by another service directly — only via Abhyasa's GraphQL/MCP API.

**Core entities**

- **Student** — a child profile (id, name, grade level, school, assistant persona). Distinct from the platform
  Member/Kutumba record; references it. Wall owner for all their data.
- **AcademicYear / Term** — configurable year → terms/semesters, with start/end and status.
- **Subject** — per student per year: name, grading scheme (points/percent/letter/standards/weighted),
  weight categories, teacher (optional). Behaviour-as-data.
- **GradeItem** — the atom: subject, term, title, type (test/quiz/homework/project/participation),
  score, maxScore, weight, letter/level (if applicable), normalised-score, date, source
  (manual/voice/import:canvas/import:email/report-card), plus provenance (`CreatedBy{User,Service}`) and a
  value-history trail.
- **Assignment** — title, subject, assignedDate, dueDate, status (assigned→in-progress→submitted→graded→missing),
  linked GradeItem when graded.
- **Skill / Standard** — a taggable skill ("fractions") and/or a standard code (Common Core/NGSS/state). A
  configurable **ItemSkillMap** links GradeItems/practice to skills; a **SkillStandardMap** links skills to
  standards.
- **PracticeSession** — skill, minutes, accuracy/score, date, source.
- **SrsCard / SrsReview** — spaced-repetition deck cards + review history (SM-2/Leitner scheduling state).
- **ReadingEntry** — title, minutes, pages, date; optional level/interest tags.
- **Activity** — extracurricular (type, schedule); **ActivitySession** (attendance/practice logs);
  **Milestone** (achievements across academics + activities).
- **Goal** — kind (outcome/habit/activity), target, metric binding, window; progress derived, not stored raw.
- **Streak** — habit binding, current/longest, grace-day rules.
- **Suggestion** — a generated improvement area: rubric reasons (structured), chosen next-step actions, audience
  (parent/child), status (new/helpful/done/dismissed) for the feedback loop.
- **TermSummary** — the deterministic per-term rollup snapshot (append-only per term) that report-card narration
  reads from.
- **ImportConnection** — a per-family portal grant reference (token/scope lives in Sangam's vault, **not** here),
  last-sync status, health.

**Derived/computed (not stored as truth — recomputed, testable):** subject averages, trends/slopes, volatility,
missing-work rates, skill mastery, improvement ranking. Snapshotted into `TermSummary` for history.

**Cross-cutting:** append-only audit (Kernel interceptor) on every write; consent record per import source;
soft-delete/tombstone + export + right-to-forget on all child data; correlation ids threaded to Saakshi
(metadata only — grade *values* redacted/encrypted per §5.4/§15 of the ADR).

---

## 4. Integrations needed

All external reach goes through **Sangam** (OAuth vault + adapters, trust-tiered: read vs act) and the shared
platform spines — never bespoke per-service connectors.

| Integration | Purpose | Via | Realistic status |
|---|---|---|---|
| **Canvas LMS** | Import assignments, due dates, grades (observer/parent token) | Sangam adapter | Feasible where institution enables tokens/observers |
| **Google Classroom** | Import coursework + grades (child's own account, student scope) | Sangam adapter | Often blocked by district Workspace admin controls; opt-in, degrade gracefully |
| **PowerSchool** | Grades import | Sangam adapter | Practically blocked (needs district plugin/ClientID); deferred, manual fallback |
| **Email (emailed grade/progress digests)** | Parse school emails into proposed grade entries | Sangam→email (read tier) | Feasible; often the most reliable "import" path |
| **Family calendar** | Surface assignments/study blocks/activity schedules; read school events | Platform calendar / Sangam | Feasible for surfacing; school-event read varies |
| **OCR (report cards, photos)** | Extract grades from PDF/photo report cards | Local OCR engine behind a port (on-box, private) | Feasible, local, private |
| **Standards datasets** (Common Core / NGSS / state) | Curriculum/standards catalogue | Bundled/loaded data files | Feasible — publicly available open datasets |
| **Medha (LLM gateway)** | Child-safe framing, report narration, bounded concept tutoring | Platform spine | In-house, local, $0 |
| **Notification path** | Reminders (homework, practice, activities) | Platform notifier (+ Kaya if enabled) | In-house |

Non-goals for integration: no Khan Academy student-data API (deprecated/none for third parties), no ClassDojo
API, no scraping of portals that forbid it in ToS. Where an API is blocked, the answer is **manual + report-card
+ email import**, not a fragile scraper.

---

## 5. Interconnections with other services

Abhyasa is one mind with the rest of Manas — all cross-service flow is **governed MCP/GraphQL calls carrying the
acting user's identity**, never shared tables.

- **Sarathi (the Agent, orchestrator).** Exposes Abhyasa MCP tools (`abhyasa.record_grade`, `abhyasa.log_reading`,
  `abhyasa.get_trends`, `abhyasa.get_improvement_areas`, `abhyasa.get_term_summary`,
  `abhyasa.explain_concept`…). The child talks to *Bonnie*; Sarathi routes to Abhyasa carrying the child's verified
  identity — the wall holds even though a child is speaking.
- **Dwarpal + per-user walls.** Grades are the crown jewels of a child's dignity: one sibling **cannot** read
  another's, enforced at Abhyasa's data layer. Guardian (parent) access is an explicit relationship grant, not a
  super-user. Adding a portal credential / changing goals / right-to-forget are **admin + step-up** gated.
- **Kutumba.** Child identity, grade level, guardian relationships, and the tenant seam come from Kutumba —
  Abhyasa is member-agnostic and reads who the children are from there.
- **Medha.** All child-safe framing, report narration, and bounded tutoring run through the shared model set +
  child-safety gates (Llama-Guard/ShieldGemma-class) with priority/QoS.
- **Akriti.** Parent dashboard + child-facing views are Akriti components — accessible (voice-complete, WCAG 2.2
  AA), light/dark, one product feel.
- **Saakshi + Bodha.** Every interaction audited (grade *values* redacted/encrypted); Bodha answers "which
  imports are connected / is Abhyasa healthy / why did you suggest that?" (rubric reasons are inspectable).
- **Kaya (Vatara), optional.** A "homework time" or "reading time" scene (lights/DND) — opt-in, only if Kaya is
  enabled; graceful absence otherwise.
- **Ojas (health), optional.** A gentle, walled correlation ("focus tends to dip on low-sleep days") is a
  *future, consent-gated, coarse* signal — health data stays strictly in Ojas; Abhyasa would request a derived
  hint via API, never read Ojas's store. Deferred; noted as a seam only.
- **Sangam.** All portal/email/calendar grants live in Sangam's vault; Abhyasa holds only a connection reference.

---

## 6. Out of scope (and why)

- **Doing the homework / producing graded answers.** The bright line. Tutoring explains, demonstrates on an
  *analogous* example, and asks guiding questions; a deterministic gate + child-safety backstop declines
  "just give me the answer". Rationale: integrity, real learning, and the ADR's "LLM is a thin skin, never the
  engine" — we will not build a cheating machine for a child.
- **The LLM computing grades, trends, rankings, or standards.** All numbers/rankings are deterministic C#; the
  model may never invent or alter a grade. (ADR §2 driver #4 / §5.2 — the anti-fabrication core.)
- **Unbounded/open-web content to a child.** No open web fetch, no unfiltered book/video recommendations, no
  child-facing free chat outside the safety-gated persona. Suggestions come from curated, parent-approved local
  lists.
- **Comparing children to each other / ranking siblings or peers.** No leaderboards, no "your brother did
  better". Coaching is intra-child and growth-focused; per-child walls make cross-child comparison structurally
  impossible.
- **Diagnosis or clinical/educational-disability judgments.** Abhyasa surfaces patterns ("consistent difficulty
  with reading fluency") and can suggest *talking to a teacher*; it does **not** diagnose dyslexia/ADHD/learning
  disabilities. High-stakes interpretation defers to humans (mirrors Ojas's wellness-not-diagnosis rule).
- **Authoritative gradebook / school system of record.** Abhyasa is the *family's* private steward, not the
  school's SIS. Imports are *proposed* and parent-confirmed; the school's system remains authoritative for
  official grades.
- **PowerSchool (and similar closed portals) automated import — for now.** No obtainable parent API; building a
  fragile scraper risks ToS violation and breakage. Deferred until a real district grant exists; manual +
  report-card + email import cover the need.
- **Screen-time / device surveillance of the child.** Abhyasa tracks *learning*, not the child's device
  activity or location. Surveillance is neither the intent nor child-respectful.
- **Sharing a child's data outward.** No external sharing beyond a parent-initiated export/print. Local-first,
  per-child walled, right-to-forget honoured — a child's academic record never leaves the home unbidden.

---

*Every capability above is a **module** behind an extension point (`ICapability`/`IAnalyzer`) discovered via DI,
toggleable in config — Abhyasa grows by accretion, not surgery (ADR §5.5). Deterministic detection/analytics get
exhaustive unit tests (the 10× mandate); LLM framing/tutoring is fenced behind Medha and exercised by the eval
harness with child-safety graders.*
