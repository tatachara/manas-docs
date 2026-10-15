# ADR 0013 — Shared Read-Model & Trend Kernel (a common, WALLED way for LLMs to consume aggregation & trends)

- **Status:** **Accepted (Ananth, 2026-07-12)** — **HARDENED** by three independent adversarial runs (feasibility
  read against the *shipped* `Manas.Kernel`; wall-integrity; child-safety + LLM-consumption) before acceptance.
  The child-narration teeth (§2e) and the differencing gate (§3.4) are load-bearing for the child. `D-0013-taint`
  (§5.1) **resolved to option (a) — conservative wholesale-`Tainted`** (Ananth, 2026-07-12). Same pattern as
  ADR 0012.
- **Date:** 2026-07-12
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** ADR 0001 §5.8 (Interconnection / read side), §2.4/§2.5 (LLM-never-a-boundary; no super-key),
  ADR 0004 (walling / may-see truth table + per-store walls + `SharingGrant` — the load-bearing dependency),
  ADR 0005 (audit — now extended to a positive trend-read event), ADR 0006 (taint), ADR 0009 (Derived data /
  forget), ADR 0012 (the shared-engine pattern **and its child-safety teeth**: anti-shaming grader §3e,
  anti-pressure scheduler §3g, `unjudgeable` discipline), ADR 0011 (Akriti co-consumer). Cross-refs persona.md
  §2/§4/§5/§7 (identity-bound persona, delivery-only, cross-persona isolation eval), agent-spec §C/§D
  (`dataSubjectIds` forget fan-out; how the agent pulls context), abhyasa.md §2.1/§2.4/§2.9, ojas.md §2.0/§2.2/§2.9.
- **Evidence:** three adversarial hardening runs on the 2026-07-12 draft. Verdicts:
  **FEASIBLE-after-fixes / SECURE-after-fixes / NOT-SAFE-as-written** — this revision *is* the fixes. The
  feasibility run read the shipped Kernel source (`ManasDbContext`, `MaySee`, `WallScopeResolver`, `WalledEntity`,
  `Taint`), so the corrections below are grounded in real code, not prose.

> **The point (owner's words, 2026-07-12):** *"a common way LLMs can consume secure walled aggregation and
> trends data."* Today Artha (finance), Abhyasa (tutoring), and Ojas (health) will each **hand-roll**
> trend/time-series/aggregation. This ADR makes it **one deterministic substrate** with two consumers: the
> **premium Akriti charts** (ADR 0011) and **the LLM/persona layer** (so the agent learns household trends over
> time). The non-negotiable: **the wall is enforced deterministically over walled data, BEFORE any trend reaches
> the model** (rule #5 — the kernel computes; the LLM only narrates).

> **What the hardening changed (read this first).** The number layer is sound — per-owner + guardian→child ride
> the *shipped* owner-granular wall correctly, and the determinism boundary holds. But three lenses converged on
> two honest gaps in the draft:
> 1. **The aggregate guarantee was *asserted, not mechanized*.** The kernel took a plain `IEnumerable` — it could
>    not tell a walled query from a hand-built cross-owner list, re-creating the "forgettable helper" defect ADR
>    0004 killed, one layer up. **Fix: a `WalledSeries` type a raw `List` cannot satisfy + a NetArchTest (P15).**
> 2. **The draft imported the *wall* from ADR 0004 but forgot the *teeth* from ADR 0012**, on the one surface
>    where they were invented — *a child hearing her own trends narrated by the LLM*. A correctly-walled downward
>    trend is still shaming; determinism protects the *number*, not the *wording*. **Fix: a Child-Facing
>    Trend-Narration Governor (§2e) composing already-invented mechanisms.**
> Plus: **differencing** (household-total − my-own = the other person — found *independently* by two runs) gates
> the deferred fork; **per-store** aggregate walls (the EF filter is Postgres-only); **revocation-aware**
> materialization; a **read-audit** event; and honest corrections (percentile has no EF translation → app-side,
> no raw SQL; per-row taint doesn't exist on the substrate yet → a decision before build).

---

## 1. Context

Three services need the same machinery: bucket a metric over time, aggregate it, compute a trend/delta/
moving-average, rank change. Abhyasa's grade trend (SDD unit 1.4), Artha's spend/net-worth trends, Ojas's
correlation surfacing are the *same shape* over different domains. Built three times they diverge — most
dangerously **in how the per-owner wall is applied to an aggregate**: a `SUM`/`AVG`/`COUNT` can silently fold in
a row the viewer may not see, or *leak a hidden value by differencing two aggregates the viewer may see.*

The owner's reframe raises value and stakes together: the substrate should be **a common, secure, walled
interface the LLM consumes** for household situational awareness. That moves the read-model **in front of the
LLM**, so the wall, taint, **and child-safety** must hold *at the aggregate and at the narration*, not just at
the row.

## 2. Decision — a shared read-model & trend kernel in `Manas.Kernel`

A small deterministic library every service consumes (the ADR 0012 shared-engine pattern), one common contract,
and — new after hardening — a forced narration governor. Five parts (2a–2e):

### 2a. Deterministic trend/aggregation primitives (C#, no LLM) — computed **app-side over walled rows**
Time-bucketing (day/week/month/term), aggregation (sum/avg/min/max/count/**percentile**), trend/delta/
moving-average/change-ranking, gap/null handling with an explicit `insufficientData` marker. Pure and
deterministic (same inputs → same trend), unit-testable on a dev Mac, **can never fabricate** (honesty rule for
numbers). Services supply their domain series; the kernel supplies the math.

**Feasibility corrections (F2, F6 — verified against EF Core):**
- **Percentile / window-shaped primitives have NO EF Core translation** (`PERCENTILE_CONT`, `OVER(...)` are
  untranslatable through EF Core 10). They are computed **app-side in C# over the materialized walled row set** —
  which is exactly what the deterministic primitives are for.
- **Raw-SQL / Dapper / `FromSqlRaw` / DB materialized-view aggregation is BANNED in the trend kernel and in every
  service's trend adapter** — raw SQL bypasses the EF global query filter entirely (a wall leak by construction).
  Enforced by a **NetArchTest ("P15")** extending ADR 0004 §7's P11 to the trend namespaces.
- **`insufficientData` is computed over the *per-viewer walled* row count, never the true count** (W8) — so
  "walled-out" is indistinguishable from "genuinely absent." No existence oracle.
- **On-demand first, but measured (F6).** On-demand is viable because ADR 0009 §3 bounds the hot window
  (health down-sampled after warm; finance 2-yr hot; grades low-volume). Spike **S-0013d** measures app-side
  bucket+MA+percentile+change-rank latency on the home server over realistic per-domain hot-window counts; the
  measured crossover decides where (if anywhere) a materialized read-model is required. The materialized store's
  forget/revocation-awareness (§3.6, §3.7) is designed now (seam reserved), built only if the measurement demands.

### 2b. The walled projection contract — the wall lives in the **walled DbSet**, and the kernel takes only a `WalledSeries`
The honest location of the guarantee (F4, W1): **the wall is enforced by each service's own walled `DbSet`** (the
shipped `ManasDbContext` global query filter, ADR 0004 §2.1), and the kernel math is **wall-agnostic pure math**
over whatever series it is handed. Therefore the guarantee cannot be "the kernel walls the aggregate" — the
kernel can't know if its input was walled. It must be made structural:

- **`WalledSeries` is a distinct type a raw `List`/`IEnumerable` cannot satisfy at compile time**, obtainable
  **only** from a walled repository, and it **carries the `ActorContext` it was filtered under**. The primitives
  accept `WalledSeries`, never a bare enumerable (W1 — convention → mechanism).
- **Owner-granular is the real guarantee (F1).** The shipped filter is *owner*-granular: `OwnerId == self OR
  OwnerId ∈ VisibleOwners`, where a grant makes a whole owner visible. So a DB-side aggregate over the walled
  DbSet is wall-safe **at whole-owner granularity** — exactly right for per-owner (self) and guardian→child
  (both whole-owner). A **row-scoped** aggregate (the deferred adult↔adult grant, ADR 0004 §5) **cannot** be a
  DB-side `GROUP BY` — the row-narrowing (`HasActiveGrantCovering`) must run app-side over the narrowed set
  *before* aggregation. Stated plainly, and it is another reason the adult↔adult fork is deferred (§3.4).
- **A `WalledSeries` never spans stores whose aggregate-wall is unproven (W3).** The EF filter is Postgres-only;
  Neo4j/Qdrant/MinIO/Redis have their own per-store walls whose *aggregate* completeness (e.g. Cypher `count()`
  over every bound node) is itself an ADR 0004 §8 open question. A trend series may be sourced **only** from a
  store with a proven walled aggregate primitive; Neo4j-sourced trends are **gated on resolving ADR 0004 §8**.
- **NetArchTest (P15):** the kernel math symbols are unreachable from any non-`WalledSeries` input, and every
  service's trend-adapter input originates from the walled DbSet (no raw SQL, no cross-service source).

### 2c. One common trend/report shape (GraphQL) — consumed by BOTH the UI and the agent
A single `TrendSeries` / `TrendPoint` / `AggregateSummary` GraphQL type (`unit`, `window`, `confidence`,
`insufficientData`, `taint`, `dataSubjectOwnerIds` — §2d/§3) that Akriti renders and the agent reads. One shape,
two consumers — no shown-vs-believed drift. **Nullability discipline (F5):** an `insufficientData=true` series
with null points is a valid, non-throwing shape on *both* consumers. **Delivery is query/tool by default;** if a
live-updating chart subscription is added it inherits the R12 synchronous-authz + `[Subscribe(With=)]` pattern
(tech-stack §3a) verbatim — otherwise the two HC16 subscription footguns don't apply.

### 2d. The agent trend-feed — walled tools, **actor-derived owner**, forget-stamped
The agent consumes trends through **walled tools** — `get_trend(metric, window)`, `household_situation()` —
returning the §2b projection. Hardened:
- **No caller/LLM-supplied owner (W7).** Trend tools take **no** `owner`/principal argument; the owner-set is
  **derived inside the walled repo from the verified `ActorContext`** (self ∪ may-see; guardian scope from
  resolved `guardianOf` edges). The LLM never selects whose trend to pull (rule #5).
- **Forget-stamping (C4).** Every trend tool result stamps its **data-subject owner id(s)** into the turn's
  `dataSubjectIds` (agent-spec §C R1) — for `household_situation()` the *full* set of owners whose trends it
  returned. Without this, a trend about the child pulled on Ananth's turn and cached as a Sarathi `MemoryFact` would
  **survive the child's right-to-forget**. This makes trend-derived memory forget-reachable.
- **`household_situation()` is coarse + non-sensitive-class only (C5).** It returns coarse summaries; the
  **extra-strict classes** (Ojas mood/journal/cycle; clinical/disability-adjacent; and any child-subject
  sensitive trend) are **excluded from the ambient overview** and require an explicit, purpose-specific
  `get_trend` — may-see ≠ "belongs in an ambient digest" (ojas.md §2.8). A **deterministic no-unprompted-
  child-trend rule** applies: a child-subject trend is narrated pull-on-explicit-context only, **never
  volunteered** by the agent.
- **Machine/automation principals** (ADR 0004 rows 9/10) get an empty trend unless a machine-grant covers the
  owner; the projection runs the walled filter for that principal, not a human's.

### 2e. NEW — the Child-Facing Trend-Narration Governor (the teeth the draft forgot)
Determinism protects the *number*; the *harm to a child lives in the wording*, and the general agent's
`get_trend` narration path does **not** pass through a skill's persona overlay, so ADR 0012's teeth do **not**
auto-apply. So every **child-facing** trend narration is routed through a forced, deterministic, content-aware
per-edge gate (analogous to persona.md §4's Governor, but trend-content-aware), composing already-invented
mechanisms — a **gate with stream-kill to a template**, not "narrate warmly":

- **Anti-shaming / anti-comparison grader (reuse ADR 0012 §3e)** — forbids "down again"/bare-negative verdicts,
  peer/sibling/"other kids" comparison, and external anchors. The base classifiers (Llama-Guard/ShieldGemma)
  catch *toxic*, not *shaming* — this is why a dedicated grader is required (C1).
- **Anti-pressure / bounded-nudge scheduler (reuse ADR 0012 §3g)** — any practice/study-inducing trend nudge is
  clamped (caps, grace-days, quiet-hours, no "you broke your streak"); no variable-ratio reward (C1).
- **Narration-faithfulness check (reuse ADR 0012 `unjudgeable` discipline)** — the narrated direction/strength
  must match the `TrendSeries` `confidence`/`insufficientData`; an "improving!" over `insufficientData`/`flat`/
  low-confidence is a **stream-kill** to a template ("not enough data yet"). Stops over-claiming a child's
  trajectory (C6).
- **Health-class governor (reuse Ojas §2.0/§2.2)** — for any health-class metric, compose Ojas's
  wellness-not-medical stamp, correlation-not-causation stamp, and red-flag/crisis routing; plus a
  **child-mode metric-class filter**: no weight/diet/body/clinical trends are narratable to a child
  (ojas.md §2.9 body-positive-only) (C2).
- **Taint-aware child-facing suppression (extend ADR 0006 from actuation to assertion)** — tainted/unconfirmed
  source rows (email/OCR-derived, abhyasa §2.1 "never silently authoritative") do **not** enter a child-narrated
  trend until confirmed; if surfaced at all the narration is **forced to hedge** ("from an unconfirmed email")
  (C3).
- **Shared-surface child-presence rule (C8)** — a sensitive child-subject trend narrated on a shared/box surface
  (e.g. Ananth→Anik on the child's room box) must respect possible child presence: suppress-to-visual or force
  child-safe framing, composing with the speak-sensitive-aloud opt-in (agent-spec §D).

## 3. The wall & security model (foregrounded — this is the product)

- **Per-owner by default** *(owner decision 2026-07-12)*. Adults are walled from each other (ADR 0004 rows 1–2)
  even as Admin. Walling is by the **verified speaker's `ActorContext.UserId`**, decided by the Kernel *before*
  the Governor. **Persona is delivery-only and is NEVER a wall (C7)** — the draft's phrase "persona may-see" was a
  category error (persona.md §5 shows persona influence on the wall = *none*) and is deleted; an implementer must
  wall by `ActorContext`, not by persona.
- **Guardian → child is the sanctioned cross-person view, and it is FREE** *(owner decision 2026-07-12)*. It
  already exists — ADR 0004 §3 row 4: an Adult with a Kutumba `guardianOf` **edge** over a Child has
  `may-see = TRUE` (via the *edge*, not the Admin role). A guardian's aggregate covers **only** children they hold
  edges to; non-guardian adult → child = nothing (row 5); child → adult = nothing (row 6). **No new "guardian
  role" — guardianship is the existing edge.**
- **3.4 · The deferred fork — adult↔adult household aggregates — is gated on DIFFERENCING (W2 = C9, found
  independently by two runs).** "An aggregate can't fold in a row you may-not-see" is *necessary but not
  sufficient*: a hidden **value** leaks by subtracting two aggregates the viewer legitimately sees —
  **household-total − my-own = the other adult's spend**; the same de-anonymizes a cohort of one (a scoped
  guardian shown "average across my children" over a single child *is* that child's grade). Not live in the
  shipping scope (per-owner + guardian-sees-all has no partially-visible cohort), but it goes live the moment a
  scoped `SharingGrant`, a scoped guardian (§5.1a), or a hide-individuals aggregate ships. **Before that fork
  ships:** the grant primitive must be **differencing-aware** (a grant over an aggregate either also authorizes
  each addend, or the kernel **refuses to serve both the total and the differencing sub-slice to the same
  actor**), plus a **k-anonymity floor** (`insufficientData` when the contributing-owner count < k). Asserted now
  (§4) so the fork cannot ship the leak.
- **3.5 · Taint — and an honest substrate gap (F3, W6, C3).** Aggregate taint is **max over the contributing
  rows, monotonic across windowing** (a value derived through a tainted intermediate stays tainted even after
  that bucket ages out of a moving-average window — no laundering). **But per-row taint does NOT exist on the
  shipped `WalledEntity`** — `Taint` currently rides the *event/outbox* row (ADR 0006 §3), not the persisted
  source row. **DECIDED (Ananth, 2026-07-12) — `D-0013-taint` = option (a), conservative wholesale-`Tainted`:** a
  trend is marked `Tainted` wholesale if its source table can hold model-derived/ingested values (coarse, safe,
  "highest-taint-wins"), **no new per-row column**. Option (b) — a persisted per-row provenance column — is
  deferred; add it only if a specific domain later needs row-level precision (a targeted schema change then). This
  keeps unit 1.3a from freezing an unused column into every source table. Backstop that holds regardless: ADR
  0006 §4 carves
  money-movement and disarm/unlock out of unattended execution *entirely, regardless of taint* — a laundered
  "clean" trend still cannot move money or unlock a door; residual risk is Tier-B actuation only.
- **3.6 · Derived-data lifecycle (ADR 0009 §134).** Trends are **Derived** — rebuildable; right-to-forget on a
  source **sheds/rebuilds** the projection, and (C4) the agent's trend-derived `MemoryFact`s are forget-reachable
  via `dataSubjectIds` stamping (§2d).
- **3.7 · Materialized projections must be REVOCATION-aware, not just forget-aware (W4).** A cached "the child's grade
  trend" tile computed while a `guardianOf` edge existed must **not** keep serving after the edge is removed
  (custody change / child turns 18) or after a `SharingGrant` is revoked/expires. ADR 0004's "no code change
  closure" holds only for the *live* path. So a materialized trend is a `may-see` cache: it is **invalidated on
  Kutumba edge change and grant revoke/expiry**, or the wall is **recomputed live-on-read even when the numbers
  are materialized**. A `WalledProjectionStore` shape (one-store-one-wall, ADR 0004) is specified before any §5.2
  materialization ships.
- **3.8 · Trend reads to the LLM are AUDITED (W5).** ADR 0005 audits writes and treats ambient scoped reads as
  silent — but a trend fed to the LLM is high-value aggregated PII, and a differencing sweep (§3.4) otherwise
  leaves *zero trace*. A **positive `TrendReadEvent`** rides the ADR 0005 §2a security-event transport carrying
  `{actor, ruleOwnerPrincipal?, metric, window, resolved-owner-set, taint, cid}` (id + digest only, per the
  Sarathi rule) — the trail of *what the agent surfaced about whom* (including *what Bonnie surfaced about the child*)
  and the hook that makes a differencing sweep detectable.
- **Determinism as the boundary (rule #5).** The kernel computes every number in C#; the LLM only narrates.
  Perturbing the model never changes a `TrendPoint` (§4). But note the honest split (mirrors ADR 0012 §7.3):
  determinism guarantees the **number**; the **narration** is guaranteed by the §2e Governor, not by determinism.

## 4. Acceptance criteria / property tests

1. **Wall holds at the aggregate (owner-granular).** A viewer's trend over a multi-owner metric returns only
   whole-owners they may-see; a sibling's/other-owner's rows never enter `SUM`/`AVG`/`COUNT` — architecture +
   property test over a seeded multi-owner set, computed as an EF query over the walled DbSet.
2. **Structural walling (W1).** Kernel primitives accept only `WalledSeries`; a NetArchTest (P15) proves the math
   is unreachable from a bare `List`/`IEnumerable` and that every trend-adapter input originates from the walled
   DbSet (no raw SQL, no cross-service source). A planted raw-`List` call **fails the build**.
3. **Owner-granular boundary is explicit (F1).** A property test seeds a multi-owner set, gives the actor a
   one-object `SharingGrant`, and asserts a DB-side `Sum` over the walled DbSet would fold the grantor *wholesale*
   — locking the rule that grant-scoped aggregates run app-side over `HasActiveGrantCovering`, never SQL.
4. **Per-viewer projection.** Same query as two actors → two correctly-walled projections regardless of persona
   (extends persona.md §7 cross-persona isolation to the trend tools); no actor obtains another's series.
5. **No fabrication + faithful narration.** `insufficientData` over the *walled* count → explicit marker, never a
   fabricated trend (deterministic). The §2e narration-faithfulness check **stream-kills** any narrated
   direction/strength that exceeds `confidence`/`insufficientData` (C6).
6. **Determinism boundary.** Perturbing any LLM output never changes a `TrendPoint`/`AggregateSummary`.
7. **Differencing gate on the fork (W2/C9).** A scoped total-grant must **not** allow reconstruction of an
   un-granted addend: the actor cannot obtain both the total and the differencing sub-slice; a cross-owner
   aggregate below the k-anonymity floor returns `insufficientData`.
8. **Taint (D-0013-taint).** Aggregate taint = max over contributing rows, monotonic across windowing; a trend
   from a tainted source is marked tainted end-to-end and cannot actuate a Tier-A automation; a
   tainted/unconfirmed row is **suppressed or hedged** in a child-narrated trend (C3).
9. **Forget + revocation.** Forgetting a source row sheds/rebuilds the projection **and** invalidates any
   materialized tile; a trend-derived `MemoryFact` about the forgotten subject is shredded via `dataSubjectIds`
   (C4); removing a `guardianOf` edge or revoking a grant closes access to **live and materialized** projections
   immediately (W4).
10. **Guardian access (owner decision).** Guardian → their child returns the walled series (row 4); non-guardian
    adult → nothing (row 5); child → adult → nothing (row 6); multi-child aggregate only over held edges.
11. **Child-narration teeth (C1/C2, BLOCKER-level).** A correctly-walled *downward* child trend is narrated
    **without** shaming/comparison/pressure (anti-shaming grader + anti-pressure scheduler fire); a health-class
    metric narration composes Ojas's governor; weight/diet/body/clinical trends are **refused** to a child; the
    grader is proven to catch planted shaming/comparison phrasings **with the number correct** (the harm is the
    wording, not the number).
12. **No unprompted child trend / overview scoping (C5).** `household_situation()` never returns extra-strict or
    child-subject sensitive classes; a child-subject trend is never volunteered unprompted.
13. **Read-audit (W5).** Every agent trend pull emits a `TrendReadEvent` `{actor, metric, window, owner-set,
    taint, cid}`; a differencing sweep is reconstructable from the audit.
14. **One shape, two consumers.** Akriti and the agent read the identical `TrendSeries`, including the
    `insufficientData=true`/null-points shape, with no drift (shared-contract test).
15. **Actor-derived owner (W7) + no-oracle insufficientData (W8).** Trend tools reject any caller-supplied owner;
    `insufficientData` is computed over the per-viewer walled count so walled-out ≡ absent.

## 5. Open questions / decisions

- **RESOLVED (owner, 2026-07-12):** per-owner by default; guardian→child by default via the existing ADR 0004
  row-4 `guardianOf` edge (no new machinery, no new "role"). Adult↔adult aggregate deferred behind an explicit
  `SharingGrant`, and now additionally **gated on the differencing defense (§3.4)** before it ships.
- **5.1 · RESOLVED (Ananth, 2026-07-12) — `D-0013-taint` = option (a), conservative wholesale-`Tainted`.** A
  trend is tainted wholesale if its source table can hold model-derived/ingested values; **no per-row provenance
  column** is added now (option (b) is deferred to a targeted future schema change if a domain needs row-level
  precision). Rationale: over-caution on taint never causes a leak or a false claim to a child; it only
  occasionally labels a clean trend "unconfirmed" (harmless), and it avoids freezing an unused column into every
  source table at 1.3a. (§3.5.)
- **5.1a · Scoped/typed guardians (flagged ADR 0004 refinement, NOT blocking):** a *limited* guardian
  (tutor → Abhyasa only) composes a `guardianOf` edge with the existing `SharingGrant` scope grammar — and, note,
  activates the §3.4 differencing risk (a scoped guardian is a partially-visible cohort). A small ADR 0004
  amendment **if/when** wanted; parent-guardian-sees-all stands now.
- **5.2 · Materialized vs on-demand:** on-demand first (spike S-0013d decides the crossover); a materialized
  `WalledProjectionStore` is forget- *and* revocation-aware (§3.7) if built.
- **5.3 · Cross-domain correlation** (Ojas sleep-vs-spend-vs-mood spanning services): in scope here only as
  **walled, per-owner** correlations over that owner's own multi-domain series (still database-per-service — the
  owner's services each hand their own walled series); genuinely cross-*owner* correlation stays Interconnection's
  job (ADR 0006). Health correlations carry Ojas's correlation-not-causation stamp (§2e).

## 6. Required on-box / build spikes (risk register — before build)

| Spike | Gate (green = proceed) | Fallback if red |
|---|---|---|
| **S-0013a · owner-granular wall boundary** | property test locks that a DB-side aggregate over the walled DbSet is owner-granular; grant-scoped aggregates are forced app-side | keep adult↔adult deferred; document the boundary |
| **S-0013b · `WalledSeries` + P15 NetArchTest** | the lint fails a planted raw-`List`/`FromSqlRaw` trend call (self-test, like secret-scan) | do not ship any trend adapter until the mechanism catches a planted positive |
| **S-0013c · child-narration Governor vs a shaming corpus** | the anti-shaming/anti-comparison + faithfulness graders refuse a corpus of shaming/comparison/over-claim narrations **with the number correct** | no child-facing trend narration ships until green |
| **S-0013d · the home server app-side latency** | bucket+MA+**percentile**+change-rank p95 within the interactive budget over realistic per-domain hot-window counts | materialize the heavy classes (revocation-aware, §3.7) |

## 7. Consequences

**Positive.** One deterministic trend substrate instead of three; the aggregate wall is a **mechanism**
(`WalledSeries` + P15), not a convention; the agent gains **secure, walled, audited** household situational
awareness; **child-facing narration has teeth** (the §2e Governor) on the surface where a child hears her own
trends; Akriti and the agent share one shape; determinism, forget, and revocation hold by construction.

**Costs (honest).** More machinery than the draft implied — the `WalledSeries` type, the P15 test, the §2e
Governor (reusing ADR 0012/Ojas mechanisms), the read-audit event, per-store aggregate walls, revocation-aware
materialization, and the taint substrate decision. Every piece buys off a *named, adversarially-found* leak or
child-harm. Four spikes gate the build (§6).

**Sequencing (act on this now).** **Abhyasa's trend is SDD unit 1.4.** Insert **unit 1.3a** (the kernel:
primitives + `WalledSeries` + `TrendSeries` shape + P15) **before 1.4**, so Abhyasa is its first consumer. Wire
the §2d agent-feed tools **and** the §2e narration Governor in the phase the agent gains real context (≈1.5+) —
the Governor is a prerequisite for **any** child-facing trend narration. `D-0013-taint` (§5.1) is decided before
1.3a freezes source schema.

**Constrains** `Manas.Kernel` (primitives + `WalledSeries` + the P15 rule), Abhyasa/Artha/Ojas (consume the
kernel; never hand-roll trend math, raw-SQL aggregation, or re-implement the aggregate wall), Akriti (render the
shared shape), the agent layer (walled tools, actor-derived owner, forget-stamped, narration through §2e), ADR
0005 (the new `TrendReadEvent`), and — when the fork ships — ADR 0004 (the differencing-aware grant).

**Verdict of the three runs:** FEASIBLE / SECURE / SAFE **after these fixes** — applied here as design
(mechanisms, not prose). **Ananth accepted it on 2026-07-12** with `D-0013-taint` = conservative wholesale; §2e
and the §3.4 differencing gate are now binding on the trend kernel, its agent-feed, and the deferred fork.

---

*Refines ADR 0001 §5.8; depends hard on ADR 0004 (walling + `SharingGrant`) and ADR 0006 (taint); extends ADR
0005 (trend-read audit); trends are ADR 0009 Derived data; reuses ADR 0012's child-safety teeth and Ojas's
wellness governor; Akriti (ADR 0011) is a co-consumer. Hardened + Accepted 2026-07-12 (`D-0013-taint` =
conservative wholesale-`Tainted`).*
