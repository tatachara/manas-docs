# Manas — Service Blueprint (worked example: Abhyasa)

- **Status:** Design (sketches, not code) · **Date:** 2026-07-11 · with
  [kernel-spec](kernel-spec.md) + [ADR 0001](../adr/0001-manas-platform-architecture.md) + [abhyasa vision](../vision/abhyasa.md).

> The **canonical shape every service follows**, made concrete with Abhyasa (the first vertical slice). Copy
> this structure for Kaya/Kavach/Artha/Ojas. Design sketches only — implementation is TDD, 10×, after the
> walking skeleton.

## 1. The four layers (dependencies point inward only)

```
Abhyasa.Domain          entities (WalledEntity), value objects, pure rules. NO EF, NO LLM, NO transport.
Abhyasa.Application     use-cases; ports (IQueries/ICommands + analyzer/responder/notification ports); Result<T>.
Abhyasa.Infrastructure  adapters: EF Core (OWN Postgres DB), Neo4j (skill/curriculum graph), the deterministic
                        analyzers, the child-safe LLM adapter (behind a port), Notifications adapter.
Abhyasa.Api             GraphQL subgraph (over app services + DTOs) + the MCP tool server + minimal REST (health).
```

Owns its own datastores + connection strings + `appsettings` (ADR §5.4 isolation). No other service reads its DB.

## 2. Data model (Domain — all user-scoped types derive from `WalledEntity`)

| Entity | Key fields | Notes |
|---|---|---|
| `Child` | ownerId(parent), displayName, gradeLevel | the learner (a household member) |
| `Subject` | childId, name, standardsRef | Math, Reading, … |
| `Assessment` | childId, subjectId, kind(test/quiz/hw), score, maxScore, weight, term, date | a recorded grade |
| `Assignment` | childId, subjectId, title, dueAt, status | homework/tasks |
| `Activity` | childId, kind(sport/music/…), title, cadence | extracurricular |
| `Goal` | childId, subjectId?, target, dueAt, progress | goals/streaks |
| `ImprovementArea` | childId, subjectId, signal, severity, suggestion, detectedAt | **derived, deterministic** |
| `StudySession` / `ReadingLog` | childId, subjectId?, minutes, at | practice/reading tracking |

Graph (Neo4j): `Child`→`Subject`→`Skill`→`Standard` relationships for curriculum/skill mapping and prerequisite
chains. Postgres is the system-of-record; Neo4j holds the relationships.

## 3. Application — use-cases + ports (CQRS-lite; deterministic core)

- **Commands:** `RecordAssessment`, `AddAssignment`, `CompleteAssignment`, `LogStudy`, `LogReading`, `SetGoal`.
- **Queries:** `GetSubjectTrend`, `GetImprovementAreas`, `GetUpcoming`, `GetReportCardSummary`.
- **Ports:** `IAbhyasaQueries`, `IAbhyasaCommands`, `IImprovementAnalyzer` (deterministic), `IChildSafeResponder`
  (LLM, behind a port), `INotificationPort`.
- **Deterministic vs LLM split (the rule):**
  - **Deterministic C#:** all recording, per-subject/skill **trend math**, weighted averages, streaks,
    **improvement-area detection + ranking**, study planning, spaced-repetition scheduling. This is ~all of it.
  - **LLM (thin skin, child-safe):** *only* phrasing the encouragement ("You've leveled up in fractions —
    want to try word problems next?") and *optional bounded* concept explanation — hard-fenced from doing the
    homework. Never computes a grade, never a boundary.

## 4. GraphQL subgraph (sketch — resolvers over app services, walled)

```graphql
type Query {
  child(id: ID!): Child                      # walled: only the caller's tenant + allowed children
  subjectTrend(childId: ID!, subjectId: ID!, term: Term): Trend
  improvementAreas(childId: ID!): [ImprovementArea!]!
  upcoming(childId: ID!): [Assignment!]!
  reportCardSummary(childId: ID!, term: Term!): ReportCardSummary   # LLM-framed, deterministic data
}
type Mutation {
  recordAssessment(input: RecordAssessmentInput!): RecordResult!
  completeAssignment(id: ID!): Assignment!
  logReading(input: LogReadingInput!): ReadingLog!
  setGoal(input: SetGoalInput!): Goal!
}
type Subscription { improvementAreaRaised(childId: ID!): ImprovementArea! }   # via Redis backplane
```

Composed into the one Manas HotChocolate-16 schema as a module (no federation — tech-stack D3, revised 2026-07-11); every resolver runs over the application services (never EF
entities), inside the Kernel wall.

## 5. MCP tools (Sarathi-facing — the same app services, identity-carrying)

| Tool | Purpose | Kind |
|---|---|---|
| `abhyasa.record_grade` | record an assessment by voice/chat | deterministic write |
| `abhyasa.get_trend` | a subject/skill trend | deterministic read |
| `abhyasa.suggest_improvements` | ranked improvement areas + child-safe framing | deterministic + LLM-framed |
| `abhyasa.plan_study` | build/adjust a study plan | deterministic |
| `abhyasa.log_reading` | log reading minutes | deterministic write |

Each tool carries the caller's `ActorContext`; writes go through the wall + audit + attribution automatically
(Kernel). Sarathi (the Agent) auto-discovers these (capability-module extensibility, ADR §5.5).

## 6. Interconnections

`Abhyasa → Ojas` (sustained late-night study / low mood → a gentle wellness nudge) · `→ Notifications`
(assignment reminders, streak nudges) · `→ Kutumba` (which child, roles) · `→ Sangam` (Canvas token / email
grade-digest import) · `→ Saakshi/Bodha` (audit + self-awareness). All via governed tools/events — never a
shared DB.

## 7. Tests (the 10× estate for this service)

- **Unit** (bulk): trend math, improvement-area detection/ranking, streaks, planning — pure, no LLM/DB.
- **Integration:** EF + real SQLite, the GraphQL subgraph, Sangam import parsers (Canvas/email-digest).
- **Contract:** each MCP tool matches its published schema + behavior.
- **Property:** invariants — "a query never returns another child's rows"; "an improvement area is deterministic
  given the same inputs."
- **Regression:** one locking test per fixed bug.
- **Eval:** the child-safe responder tone (never does the homework; encouraging; age-appropriate) — judged.

## 8. The new-service checklist (copy for every service)

- [ ] 4 layers; user-scoped entities derive from `WalledEntity`; own DB + connection strings + appsettings.
- [ ] Commands/Queries ports; deterministic core; LLM only behind a port for framing.
- [ ] GraphQL subgraph over app services; MCP tools carrying `ActorContext`.
- [ ] `[Encrypted]` on sensitive fields; wall + audit + attribution inherited from Kernel (nothing hand-rolled).
- [ ] Domain events → outbox → event bus for interconnections.
- [ ] `Services:Abhyasa:Enabled` opt-out; Bodha reports it; Saakshi logs it.
- [ ] Test estate ≥ 10× (unit/integration/contract/property/regression/eval).
