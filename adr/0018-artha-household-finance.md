# ADR 0018 — Artha (Household Finance) & the Sangam Ingestion Seam

- **Status:** **Accepted (2026-07-19)** (owner: Ananth) — promotes [`../vision/artha.md`](../vision/artha.md)
  (self-labelled *pre-ADR vision / intent*) to a design-of-record. Acceptance **includes the §15 threat model
  dispositions** (all findings F1–F12 dispositioned *adopted* by the owner on 2026-07-19 and folded into the body).
- **Date:** 2026-07-19
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines / depends on:** [ADR 0001](0001-manas-platform-architecture.md) §3 (Steward→Artha rename), §5.2/§5.4
  (walls, LLM-never-a-boundary); **hard** on [ADR 0002](0002-key-hierarchy-and-unlock.md) (Artha = `SENS` tier,
  sealed-until-login), [ADR 0004](0004-walling-maysee-and-sharing.md) (per-user wall, `may-see`, `SharingGrant`),
  [ADR 0007](0007-risk-tiers-and-step-up.md) (money-move tier table T2/T3/T4 + factors + Automation-refusal),
  [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) (7-yr finance retention, consent→revoke→forget),
  [ADR 0013](0013-shared-read-model-and-trend-kernel.md) (Artha consumes the walled trend kernel);
  [ADR 0005](0005-audit-pipeline-and-tamper-evidence.md) (finance audit mandatory),
  [ADR 0008](0008-availability-degraded-mode-and-dr.md) §5/§11 (DR + degraded-mode-for-money),
  [ADR 0003](0003-identity-propagation-and-pki.md) (delegation scopes, step-up factor), [ADR 0006](0006-automation-principal-and-event-bus.md)
  (Automation principal / taint / outbox). Cross-refs [ADR 0010](0010-agent-conversational-core.md) (Sarathi front door).

> **The vision catalogues ~70% of Artha as "already exists in a mature Steward .NET codebase" and defers the
> hard questions to "later ADRs 0002+."** This ADR is that later ADR. It DECIDES the seven sharp edges the
> vision left open — the account/ownership model against the platform wall, Plaid-Production-primary ingestion
> behind a new **Sangam** OAuth-vault seam, money-move authorization bound to the `RiskPolicy` tiers, the
> child wall, finance retention, **whether "re-home the Steward codebase" survives contact with CLAUDE.md's
> quality bar** (it does not, as written), and the compliance line — and it pins the one non-negotiable:
> **money is the highest-stakes boundary; every dollar and every move is deterministic C#, never the model
> (rule #5).** Greenfield confirmed 2026-07-19: no `Artha`/`Sangam`/`Steward` service code exists in `src/`.

---

## 1. Context

Artha is the Manas finance service: it ingests the household's full money picture (transactions, balances,
cards, loans, holdings, bills), runs **deterministic C# math** to protect and inform, and — within explicit,
tiered limits — *acts*. Three things make it the single most dangerous service to get wrong:

1. **It holds the family's money.** A hallucinated figure erodes trust (rule #1); an un-gated `pay`/`transfer`
   loses real dollars (rule #5). Both must fail at deterministic code, never at model judgement.
2. **It is walled across the two adults AND the child** — Ananth and the other adult are private from each other's
   personal money ([ADR 0004](0004-walling-maysee-and-sharing.md) rows 2–3), and **the child must never see
   household finance at all** (rule #9). A leak here is a family-trust failure, not a bug.
3. **It reaches the outside world** — Plaid, statement files, email, money-movement rails — through
   credentials that must **never enter the model's context** and OAuth grants that must be consented,
   revocable, and forget-scoped ([ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §6).

The vision ([`artha.md`](../vision/artha.md)) is a rich, honest capability catalogue but is explicitly
**pre-ADR** and **deepen-and-extend** — it assumes a "mature Steward .NET solution (~70% exists)" that
re-homes under the Artha name. That assumption collides head-on with CLAUDE.md (TDD-from-commit-#1, test ≥ 10×
prod, no god classes, no cowboy import — the same quality bar that *banned Jarvis code*). This ADR resolves
the collision explicitly rather than letting a wholesale import smuggle un-tested code past the gate.

**Owner decision taken as input (2026-07-19):** the household **will pursue Plaid Production** as the
transaction source. So Plaid Production is the **primary** ingestion path; statement/CSV/OFX/QFX + manual
accounts are the **deterministic degraded fallback** — a source outage or an un-linkable issuer degrades to
manual entry, **never a hard block** (rule #6: named up-front with a fallback, never discovered late).

---

## 2. Decision — summary

Artha is a new walled `SENS`-tier service (`src/Artha/…`, clean-architecture like Abhyasa/Kaya/Sarathi),
built **TDD, unit by unit, from commit #1** — **not** imported wholesale. Concretely this ADR decides:

- **§3 — Sangam is a sibling ADR, not a section here.** This ADR specifies only the *minimal ingestion
  contract* Artha depends on; Sangam's full design (OAuth vault, per-source tiers, connector governance) is
  **ADR 0020** because Sangam is a *platform* service consumed by more than Artha (email, notifications, Plaid).
- **§4 — Account/ownership model.** `SharingScope` is **not** a new kernel concept; it is an **Artha-local
  projection over the platform `SharingGrant`** ([ADR 0004](0004-walling-maysee-and-sharing.md) §5). The child's
  allowance is a **non-Plaid internal `KidLedger`**, guardian-walled.
- **§5 — Ingestion.** Plaid-Production-primary + statement fallback; read-only enforced deterministically;
  idempotency by provider transaction id; `[Encrypted(SENS)]` amounts/masks; consent record per source with
  revoke ⇒ scoped-forget.
- **§6 — Money-move authorization.** The vision's tiered `ActionDispatcher` is **deleted as a parallel scheme**;
  Artha **registers its action tiers and rides the already-built `ToolRiskGate` + `StepUpChallengeAuthority`**
  ([ADR 0007](0007-risk-tiers-and-step-up.md)'s `RiskPolicy` T2/T3/T4 + §7 nonce/hash/TTL) +
  [ADR 0003](0003-identity-propagation-and-pki.md) delegation scopes — it builds **no new step-up mechanism**.
  The only pending shared piece is the real factor verifier (`IStepUpFactorVerifier`, interim
  `DevSharedSecretFactorVerifier`), a phase-shared keystone. Idempotency-key + Kernel outbox give exactly-once;
  **movement rails are OFF by default** behind config — **v1 is read + advise only.**
- **§7 — Walls incl. the child.** Property-tested "wall never leaks"; **the child sees ONLY her allowance ledger.**
- **§8 — Retention.** Ratifies [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) for finance: 7-yr
  transactions, aggregate-forever, consent-revoke-forget.
- **§9 — Steward migration.** **Rewrite-behind-tests by default; port-with-characterization-tests only per
  isolated, deterministic module** — never a wholesale import that cannot satisfy the ratio gate or red-first TDD.
- **§10 — Compliance.** Observation + commoditised consensus only; **never** personalised advice;
  personal-use-only until legal review.
- **§11 — Deterministic boundary.** A `NumericProvenance` guard makes every spoken figure trace to a
  deterministic tool; a prompt-injection on the child's turn attempting `artha.pay`/`move_money` fails at C#
  authorization, never at model judgement.

---

## 3. Sangam — sibling ADR (0020), with the minimal contract pinned here

**Decision: Sangam gets its OWN ADR (0020 — Proposed, to follow); this ADR fixes only the minimal contract
Artha requires.**

*Justification.* Sangam is already named as a **platform** OAuth seam across the design ([ADR 0008](0008-availability-degraded-mode-and-dr.md)
§1 "Sangam (per-user OAuth)"; [`artha.md`](../vision/artha.md) §4/§5). It fronts **email** (Sarathi/Abhyasa
intake), **notification channels** (Twilio/Pushover/Slack/Discord), **and** Plaid — so making it a *section of
the finance ADR* would (a) mis-scope a cross-service platform service inside one domain, and (b) force every
future Sangam consumer to refine a finance ADR. One-decision-per-file ([README](README.md)) says Sangam is its
own decision. What Artha needs to build against is small and stable, so we pin it here as a **consumed
contract** and leave the vault internals to ADR 0020:

**Sangam minimal contract (the seam Artha builds to):**

1. **Per-source OAuth consent.** Each external source (Plaid Item, email, a notification channel) has a
   **consent record** `{ownerId, source, scope, grantedAt, grantedBy}` stored as an audited `WalledEntity`
   ([ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §6). No source is reachable without a live consent.
2. **Read-vs-act are distinct tiers on the connector.** A Sangam connector exposes `read` and (optionally)
   `act` capabilities as **separate grants**. Artha's Plaid connector is granted `read` by default; `act`
   (money movement, §6) is a *separate, off-by-default* grant that itself requires step-up to enable.
3. **Credentials never enter the model's context.** OAuth tokens / Plaid `access_token` / API secrets live in
   the Sangam vault (`MTRUST`-adjacent custody, detail owned by ADR 0020) and are **referenced by opaque
   connector-handle**, never materialised into any MCP tool argument, prompt, trace, or `MemoryFact`. A
   NetArchTest asserts no token type is reachable from the Sarathi/agent assembly (extends [ADR 0004](0004-walling-maysee-and-sharing.md)
   §7 P11 pattern to secret types).
4. **Revoke ⇒ scoped-forget.** Revoking a source's consent stops future ingestion and enqueues the
   [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §2.1 crypto-shred fan-out scoped to that
   source's per-source sub-key — Plaid-sourced data forgotten without touching email/manual data for the same owner.
5. **Metered + capped.** Every acting/costed call passes `ISpendGuard` (the product's hard $25/mo vendor cap);
   fail-closed on cap breach.

> **Cross-link forward:** ADR 0020 owns the vault crypto-custody, token rotation, connector lifecycle, and the
> email/notification consumers. If ADR 0020 does not land before Artha's ingestion unit, Artha builds against
> this contract behind an `ISangamConnector` interface and a test double — the seam is stable enough to do so.

---

## 4. Account & ownership model (reconciled against the ADR 0004 wall)

**Decision: `SharingScope` is an Artha-local projection over the platform `SharingGrant`, NOT a new kernel
concept.** [ADR 0004](0004-walling-maysee-and-sharing.md) §5 already reconciled "Steward's `Personal | Shared`
enum" into the single platform primitive and said Steward's enum *"becomes a thin projection over it."* This
ADR honours that verbatim:

- **Ownership is `OwnerId`** (the Kernel per-user wall / global query filter). An account, budget, goal, or
  envelope owned by Ananth is invisible to the other adult **by default** ([ADR 0004](0004-walling-maysee-and-sharing.md)
  row 2), even though both are Admin.
- **"Shared" is an explicit, revocable `SharingGrant`**, not a relaxed wall. Artha exposes a convenience
  read-through enum `SharingScope { Personal, Shared }` on ownable objects, but `Shared` is **materialised as a
  household-scoped `SharingGrant`** (`Service=Artha, ObjectType=Account|Budget|Goal, ObjectId=…`) whose scope
  the [ADR 0004](0004-walling-maysee-and-sharing.md) §5 grammar already expresses. `may-see` decides
  visibility; the enum is display sugar. There is **no** Artha-owned wall logic — a NetArchTest asserts Artha
  never implements its own `may-see`.
- **Household net-worth / combined views are opt-in and both-consent** (§7): the aggregate is served only when
  a `SharingGrant` covers each contributing account, and it is subject to the [ADR 0013](0013-shared-read-model-and-trend-kernel.md)
  §3.4 **differencing gate** (household-total − my-own = the other adult's balance is a leak) — so adult↔adult
  household aggregates stay behind that gate, exactly as ADR 0013 requires before the fork ships.
- **The child's allowance is a non-Plaid internal `KidLedger`.** A parent-funded virtual ledger
  (`KidLedger{ childUserId, fundingParentId, balance }` + `KidBucket{Spend|Save|Give}` + `Allowance` +
  `ChoreEarning`), **never a real bank/Plaid account** (real kid-banking is out of scope, §10). It is
  guardian-walled: parents with a `guardianOf` edge ([ADR 0004](0004-walling-maysee-and-sharing.md) §4) see and
  manage it; **the child sees only her own ledger and never any household-finance row** (§7).

*Data-model note.* All entities extend `Manas.Kernel`'s walled base (`TenantId` + `OwnerId` + provenance +
audit interceptor); sensitive fields (`amount`, `mask`, `plaid*Token`) are `[Encrypted]` → resolve to `SENS`
via Artha's default `ITierMap` entry (`Artha → Sensitive`, [ADR 0002](0002-key-hierarchy-and-unlock.md) §4).

---

## 5. Transaction ingestion — Plaid-Production-primary, statement fallback

**Decision: Plaid Production is the primary read source; statement (CSV/OFX/QFX/PDF) + manual accounts are the
deterministic degraded fallback — never a hard block.**

- **Plaid primary.** Incremental cursor-based `/transactions/sync`, balances, Liabilities (APR/min/due),
  and read-only Investments where covered. Ingestion runs behind the **Sangam Plaid connector** (§3) under a
  `read` grant; the `access_token` never leaves Sangam.
- **Read-only-by-default, enforced deterministically.** The Plaid connector is provisioned with **read
  products only**; money movement (Plaid Transfer / Method) is a **separate `act` grant, off by default** (§6).
  Read-only is not a policy string — it is the *absence of an `act` grant* on the connector, checked in C# at
  the Sangam boundary. A `read`-only connector **cannot** emit a transfer instruction.
- **Idempotency by provider transaction id.** Every ingested row is keyed by `(source, providerTxnId)`;
  re-delivery (cursor replay, statement re-import of an already-synced period) is a **no-op upsert**, never a
  duplicate. Pending→posted transitions update in place by the same key.
- **Degraded fallback is first-class.** If Plaid is unreachable, an Item is de-linked, or an issuer is not
  Plaid-reachable, ingestion **degrades to statement import + manual accounts** — deterministic parsers
  (CSV/OFX/QFX exist as a class; PDF metadata extraction; LLM assists *only* when deterministic parse fails and
  its extraction is **re-validated by code**, never trusted raw). The user is told the source is degraded; the
  picture is honestly marked incomplete — **it is never a hard block** (rule #6).
- **SENS encryption of amounts/masks.** `Transaction.amount`, `Account.mask`, and any token reference are
  `[Encrypted]` → `SENS` ([ADR 0002](0002-key-hierarchy-and-unlock.md)); a pre-login read raises
  `TierLockedException` (§12).
- **Consent record per source + revoke ⇒ scoped-forget.** Each Plaid Item / statement source / email source
  carries a [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §6 consent record; revocation stops
  ingestion and enqueues the source-scoped crypto-shred fan-out (ADR 0009 §2.1). **Email intake persists facts
  only, never bodies** ([`artha.md`](../vision/artha.md) §2.1).
- **Ingested string fields are Tainted/untrusted (F3).** Attacker-controllable free-text on an ingested row —
  `memo`, `merchant`, `payeeName`, OCR'd bill text — is marked **Tainted/untrusted** even when the row itself is
  a provenance-clean DB fact. Tainted strings may be **displayed only as quoted untrusted content**; they must
  **never** alter tool routing nor **pre-fill a proposal's payee/amount**. A proposal whose payee/amount derives
  from tainted ingested content is **visually flagged at confirm time** ("suggested from an unconfirmed email")
  and **barred from one-tap approval**. This closes the residual the "a human approves the nonce-bound confirm"
  story otherwise leaves — an injected proposal reaching a fatigued human — and extends the ADR-0013 §2e/§3.5
  child-path taint-hedge to Artha's **adult** narration and §6 proposal path. (This taint treatment is part of
  §13.3's proven bar for enabling rails.)
- **Ingested transactions are Tainted-until-corroborated for actuation (F9).** Idempotency dedups rows but does
  **not authenticate** them; a compromised Sangam connector or MITM'd `/transactions/sync` could return
  attacker-chosen provenance-clean rows. So a freshly-ingested transaction is **Tainted-until-corroborated** for
  any **actuation/proposal** path — Artha **never auto-proposes a pay against an un-reconciled freshly-ingested
  biller**. Plaid **error bodies and webhooks** are scrubbed under the **same no-token-in-context rule** as
  test #15 (§3 credentials-never-in-context). Connector-response integrity + per-account anomaly bounds are
  tracked as an open question (§13).

---

## 6. Money-move authorization — bound to `RiskPolicy`, never a parallel scheme

**Decision: the vision's tiered `ActionDispatcher` (Tier A/B/C) is NOT re-implemented, AND Artha builds NO new
step-up mechanism. Artha registers its action tiers into the tier registry and RIDES the already-built platform
gate — `Manas.Kernel.Risk.ToolRiskGate` + the ADR 0007 §7 `StepUpChallengeAuthority` — with the legacy Steward
tiers folded in exactly as ADR 0007 §2 already specified.**

> **Verified state (2026-07-19, orchestrator confirmed in code).** The step-up infrastructure is already GREEN:
> `Manas.Kernel.Risk.ToolRiskGate` enforces classify→step-up **below both the MCP-tool boundary and the GraphQL
> resolvers**, with Saakshi audit; `StepUpChallengeAuthority` implements the ADR 0007 §7 nonce + exact-action-hash
> + TTL + single-use challenge loop. **Artha does not add to this — it registers `artha.*` action tiers and
> relies on the existing gate + challenge.** The **only** pending shared piece is the real FACTOR verifier
> (`IStepUpFactorVerifier` — today a `DevSharedSecretFactorVerifier`; the real WebAuthn/passkey verifier, design-
> complete via [ADR 0003](0003-identity-propagation-and-pki.md) §6 + [ADR 0007](0007-risk-tiers-and-step-up.md)
> §6/§7, is a **phase-shared keystone built centrally, not by Artha**). So Artha's T4 money-move is: *classified
> → nonce-challenged → enforced by the existing `ToolRiskGate`*; the strong factor is the shared keystone
> (interim = dev shared-secret verifier). This is a dependency Artha consumes, never a mechanism it invents.

- **The C# authorization boundary is the gate, never the model.** Classification runs at Artha's **application
  layer** via the existing `ToolRiskGate` — below both the MCP tool boundary and the GraphQL resolvers
  ([ADR 0007](0007-risk-tiers-and-step-up.md) §8) — on the **verified identity** (never the voice hint). Artha
  **registers** its per-action sensitivity/operationKind into the tier registry (fail-closed at startup — an
  unregistered `artha.*` tool is a boot failure, [ADR 0007](0007-risk-tiers-and-step-up.md) §9 test #7); the
  tier table is already fixed in [ADR 0007](0007-risk-tiers-and-step-up.md) §3 and this ADR consumes it verbatim:

  | Action | Context | → Tier | Factor ([ADR 0007](0007-risk-tiers-and-step-up.md) §4) |
  |---|---|---|---|
  | `artha.get_balance` / `get_budget` | any | **T2** `SensitiveRead` | recent strong-auth; **bare voice ⇒ step-up** |
  | `artha.pay` | in-limit, known payee, authenticated web/trusted-device | **T3** `Actuate` | PIN / one-tap app confirm, action-bound |
  | `artha.pay` | out-of-limit / new payee | **T4** `HighRiskIrreversible` | passkey/Face ID on trusted device, action-bound |
  | `artha.transfer` (external) | any | **T4** | as T4 above |
  | `artha.add_payee` / `artha.trust_payee` / `artha.raise_payee_limit` | any | **T4** `HighRiskIrreversible` | passkey/Face ID on trusted device, action-bound |
  | `artha.enable_act` (turn on `act` on a connector / a rail globally) | any | **T4** `HighRiskIrreversible` | passkey/Face ID on trusted device, action-bound; per-rail, independently revocable |
  | `artha.export` (bulk finance export) | any | **T4** `HighRiskIrreversible` | passkey/Face ID on trusted device, action-bound; walled delivery channel only |

- **Payee-management is a standing capability grant, not a single pay (F1).** `artha.add_payee` /
  `artha.trust_payee` / `artha.raise_payee_limit` classify **T4** — adding or trusting a payee is a *standing*
  grant (every future pay to it rides the grant), so it is ≥ a single pay and demands the T4 action-bound factor,
  and is audited. This closes the payee-registry side-door: a pay to a payee **trusted <N min ago still
  classifies as new-payee T4**, so an attacker who adds a "known payee" cannot reclassify the subsequent pay down
  to T3. These tools are registered in the fail-closed tier registry (an unregistered `artha.*` payee-management
  tool is a boot failure, [ADR 0007](0007-risk-tiers-and-step-up.md) test #7), mirroring §6's "a limit raise is
  itself T4" and the ADR-0007 §6 soft-disarm-side-door lesson.
- **Enabling `act` is itself ≥ T4 (F4).** Turning on the Sangam `act` grant on a connector — or enabling a
  movement rail globally — is a **standing** capability (every future pay rides it), strictly more dangerous than
  one T4 pay. It classifies **≥ T4**, action-bound, audited, **per-rail, and independently revocable**. This is a
  tier *floor* that the §13.3 owner "proven-bar" decision sits **on top of**, never below.

- **Legacy tier fold-in (no parallel ladder).** Steward Tier A → T3 (app confirm); Tier B → T3 with re-auth,
  escalating to T4 as amount approaches limit; Tier C → T4. The per-payee/daily **limit is a context input** to
  `Classify` (`amount/limit`), not a separate enum — one parameterised ladder ([ADR 0007](0007-risk-tiers-and-step-up.md)
  §2). A **limit raise is itself T4** ([ADR 0007](0007-risk-tiers-and-step-up.md) OQ-4).
- **Delegation scopes + the shared challenge loop.** Artha asserts only the *required* tier; the existing
  `StepUpChallengeAuthority` ([ADR 0007](0007-risk-tiers-and-step-up.md) §7) mints the nonce-bound,
  action-hash-bound, short-TTL, single-use challenge, and Dwarpal ([ADR 0003](0003-identity-propagation-and-pki.md))
  supplies the delegation scope. A captured "approve" cannot be replayed against a different amount/payee or
  later — this anti-replay is **already built**, not Artha's to design. The strong FACTOR behind the challenge
  is the phase-shared `IStepUpFactorVerifier` keystone (interim `DevSharedSecretFactorVerifier`; real
  WebAuthn/passkey planned this phase, not yet started) — Artha depends on whichever verifier the platform provides.
- **Idempotency-key + outbox = exactly-once, no double-pay.** Every actuation carries an `idempotencyKey` and
  rides the **Kernel outbox** (`src/Platform/Manas.Kernel/Outbox/OutboxDispatcher.cs`, exists). A retry after a
  network blip **redeems the same key** and does not double-execute. This is the "no silent double transfer"
  guarantee, mechanised, not asserted.
- **Idempotency-key derivation is pinned — no silent suppressed payment (F6).** The money-move idempotency-key
  derivation is **pinned** so that a retry of the *same* intent collapses **and** two *genuinely distinct* pays
  can **never** share a key: the key binds `(payee, amount, currency, time-window)` (not a collision-prone
  `correlationId + intent-signature` alone, cf. [ADR 0005](0005-audit-pipeline-and-tamper-evidence.md) §6). Two
  distinct real bills that would otherwise collide must **each execute** — a second real pay must never return a
  stored prior result and silently skip execution (a bill the family believes paid going unpaid). Both directions
  are tested (§12): retry collapses; distinct intents never collide.
- **A queued proposal is re-classified at approval/execution, never at enqueue (F7).** A `pay`/`transfer`
  enqueued for human approval carries **no frozen tier**. It is **(re)classified at approval/execution time**
  against **live** limits + payee state via `ToolRiskGate` at the boundary ([ADR 0007](0007-risk-tiers-and-step-up.md)
  §8), so an in-limit/known-payee proposal that **drifts** out-of-limit or onto a new-blocklist between enqueue
  and approval **escalates T3→T4 or refuses** — never approves a stale T3. The nonce + action-hash bind the
  *re-classified* action, so a captured approval cannot ride the old classification.
- **Movement rails OFF by default — v1 is read + advise only.** Plaid Transfer (own-account ACH sweeps) and
  Method (third-party biller ACH) are **config-gated OFF until proven** and require the separate Sangam `act`
  grant (§3). **v1 ships with no money movement enabled** — Artha reads, detects, plans, and *proposes*; a
  proposed `pay`/`transfer` is **enqueued for a human** to approve via the nonce-bound confirm, exactly as
  [ADR 0007](0007-risk-tiers-and-step-up.md) §8 allows a rule to *request* but never *complete*.
- **Automation can never self-satisfy a money move.** An ECA rule ([ADR 0006](0006-automation-principal-and-event-bus.md))
  running as `actorRole = Automation` is **refused all money movement, always** ([ADR 0007](0007-risk-tiers-and-step-up.md)
  §8, even from a fully-Trusted trigger, regardless of taint) — it may only enqueue a human-approval request.

> **Shipping precondition — real chained audit sink (F5), stated like the §13.10 factor-verifier keystone.**
> Artha's money path — **including the T2 read authorization and the §12.2 injection refusal** — ships **only
> after** the real chained Saakshi sink replaces `NoopSaakshiSink` on the live turn path (closing the SDD §5
> blocker). Test #14 (every money-move authorization and wall-denial emits a tamper-evident Saakshi event) and
> test #2 (the audited deny) must run against the **real** sink, **never Noop** — otherwise the finance-audit
> guarantee would be claimed green against a no-op hash-chain, a rule-#4 honesty gap. This is a dependency Artha
> consumes (like the factor-verifier keystone), not a mechanism it invents.

- **Currency/unit is a first-class, checked dimension (F8).** Every amount is `(minorUnits:int, currencyCode)`
  **end-to-end** — integer minor units, never float. Tier classification's `amount/limit` comparison
  ([ADR 0007](0007-risk-tiers-and-step-up.md) §3) runs in **one normalized comparison currency**, so a foreign
  `unofficial_currency_code` amount can never slip under a USD limit as spuriously "in-limit T3" (nor the
  reverse). The human confirm **always names the currency** ("Send US$4,000", never a bare "Send 4000"); a
  mixed-currency aggregate either **refuses** or converts with a **labeled rate + as-of time**. (The household is
  USD/en-US — [household-locale-usd] — but Plaid is multi-currency-capable, so currency is checked, never
  inferred; cf. the ₹-mockup error on record.)

---

## 7. Walling across members — including the child

**Decision: Artha delegates every visibility decision to `may-see` ([ADR 0004](0004-walling-maysee-and-sharing.md)),
and adds a property-tested "wall never leaks" suite; the child sees ONLY her allowance ledger, never household finance.**

- **Adults cross-walled on personal money.** Ananth and the other adult are walled from each other's personal accounts,
  transactions, budgets, and goals ([ADR 0004](0004-walling-maysee-and-sharing.md) row 2) even as Admin; a
  shared object is visible only via an explicit `SharingGrant` (row 3). Property tests (FsCheck/CsCheck over
  generated actors/owners/grants) assert the **negative** across all five stores — the leak must be
  *impossible*, not *usually blocked* ([ADR 0004](0004-walling-maysee-and-sharing.md) §7 P1–P14 extended to
  Artha's entity set).
- **The child sees only her allowance ledger.** The child wall is the load-bearing rule #9 guarantee:
  `may-see(Child, householdFinanceRow) = FALSE` ([ADR 0004](0004-walling-maysee-and-sharing.md) row 6 — a child
  never sees an adult's rows), and the child's own reads are scoped to her `KidLedger`/`KidBucket`. A property test
  asserts **no query issued as the child ever returns a household `Account`/`Transaction`/`Budget` row**, under any
  role, grant, or guardian edge. Guardians (Ananth/the other adult, via `guardianOf` edges) see and manage her ledger;
  child→adult is always nothing ([ADR 0004](0004-walling-maysee-and-sharing.md) row 6).
- **Household-aggregate views are opt-in / both-consent** and pass through the [ADR 0013](0013-shared-read-model-and-trend-kernel.md)
  §3.4 differencing gate + k-anonymity floor before any adult↔adult combined total is served — so a "household
  net worth" tile cannot become a side-channel to a partner's private balance.
- **Trends via the walled kernel only.** Artha's spend/net-worth/cashflow trends are computed through the
  [ADR 0013](0013-shared-read-model-and-trend-kernel.md) `WalledSeries` substrate (never hand-rolled, never raw
  SQL — P15), so the wall holds *at the aggregate* before any figure reaches the LLM.

---

## 8. Retention — ratify ADR 0009 for finance

**Decision: Artha adopts the [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §3 retention matrix
for finance unchanged.**

- **Finance transactions: 2 years hot → 7 years warm (compacted), then purge** — the tax/audit statute window
  ([ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §3). Balances/trends are kept as **aggregates
  forever** (the analytical value is in the trend, not the raw row).
- **Forever-aggregates cannot retain a reconstructable residual of forgotten transactions (F10).** The
  "aggregate-forever" retention must not become a back-door copy of forgotten rows. A forever-aggregate is
  either **coarse enough to be non-identifying** (down-sampled / k-anon so **no single forgotten transaction is
  reconstructable** from it) **or** is itself **shredded and rebuilt on forget** (yielding a hole, per
  [ADR 0013](0013-shared-read-model-and-trend-kernel.md) §3.6). A test asserts a forgotten source's
  transactions **cannot be reconstructed** from any retained aggregate (§12).
- **Consent-revoke-forget** is the [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §6 mechanism:
  revoking Plaid consent stops ingestion **and** runs the source-scoped crypto-shred fan-out (§2.1) for
  Plaid-sourced data only — email/manual/health data for the same owner is unaffected (ADR 0009 test #10).
- **Export precedes forget** (ADR 0009 §6): a machine-readable export of the owner's SoR finance facts is
  offered before shred — "forget" is never silent data loss to the user.
- **Finance export is T4 over a walled channel (F11).** Any bulk finance export — the export-before-forget above
  **and** any ad-hoc "export my finances" — is a single **bulk SENS egress** and classifies **T4**
  (`artha.export`, §6 tier table): step-up + audited, delivered **only over a walled channel** (never spoken to a
  satellite, never clear-emailed). An injection/social-engineering "export my finances" cannot egress the whole
  SoR without the T4 action-bound factor.
- **The Ritu sweeper** ([ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §3.1) runs the finance
  purge nightly, policy-as-data (`RetentionPolicy` table), audited per batch, idempotent.

---

## 9. Steward migration — the quality decision (a sharp edge, decided explicitly)

**Decision: REWRITE-behind-tests by default; PORT-with-characterization-tests only for an isolated,
deterministic module, one module at a time — and NO wholesale import that cannot satisfy the ratio gate or
red-first TDD.**

The vision's "~70% already exists, re-home the Steward solution" is the single most dangerous line in it,
because it invites exactly the move CLAUDE.md forbids and that *banned Jarvis code*: dumping un-tested,
possibly-god-class production code into `src/` and back-filling green. That is a rule #2/#3 violation
(test ≥ 10× prod from commit #1; TDD red→green→refactor; no test-relaxing; no cowboy import). So:

- **Default — rewrite behind tests.** Each capability (recurring detection, missed-payment radar, fee detector,
  cashflow forecaster, debt payoff, envelopes, …) is re-specified as a small unit with a **failing test first**,
  implemented clean behind an interface, DI-registered. Steward's source is treated as a **read-only reference
  spec** (like the Jarvis eval treasure-trove: mine the *behaviour and edge cases*, not the code).
- **Port-with-characterization-tests — a narrow, gated exception.** Where a module is (a) **pure/deterministic
  math**, (b) **already well-factored** (no god class, no hidden I/O, no wall/security logic), and (c) carries
  or can be given a **characterization-test corpus that pins its exact numeric outputs**, it may be *ported*
  under those characterization tests **as the red gate** — the port is "green" only when the pinned outputs
  match and the 10× ratio holds for that module. **Anything touching walls, step-up, money movement, or the
  LLM boundary is rewrite-only** (never ported) — those are the crown-jewels and get clean-room TDD.
- **The ratio gate is per-unit and non-negotiable.** A module (ported or rewritten) that would drop the
  repo-wide test ≥ 10× prod ratio (CI-tracked) **fails the build** (CLAUDE.md rule #3,
  testing-gate-policy). There is no "import now, test later."
- **No god classes.** Steward's `AdvisorService + 17 tools` and `ActionDispatcher` are re-expressed as small
  classes behind interfaces (rule #8), not lifted as-is.

> This is the honest answer to "70% exists": the *knowledge* is 70% done; the *shippable, tested, walled code*
> starts at zero and is earned unit by unit. Claiming the code is 70% done would be a rule #1 lie.

---

## 10. Compliance line

**Decision: Artha is software, not a licensed advisor. It surfaces observation + the user's own opportunity-cost
math + commoditised industry consensus — always labelled — and NEVER personalised financial/investment/tax/legal
advice. Personal-household use only until legal review.**

- **No personalised advice.** No "buy/sell security X," no allocation prescription, no market timing, no
  personalised tax or insurance advice ([`artha.md`](../vision/artha.md) §6). Investment surfaces are read-only +
  observation ("your allocation drifted from *your* target"; "idle cash below HYSA yield, here is the annual
  opportunity cost"); the compliance hard-line from the vision §2.9 stands.
- **Not a bank / not a tax preparer / no kid neobank.** Artha never custodies funds; it *initiates* movement
  (when enabled, §6) through licensed partners between the family's own accounts / known billers. Tax features
  **organise and estimate** and produce a CPA-ready export; the family or their CPA files. The child's money is an
  **internal teaching ledger**, never a real card (real kid-banking / Greenlight-style is out of scope).
- **Personal-use-only until legal review.** The household default is conservative; **productisation requires
  counsel** (RIA/Circular-230/FCRA/money-transmission surfaces) — flagged in Open questions and consistent with
  [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §8 and [ADR 0001](0001-manas-platform-architecture.md) §12.

---

## 11. Deterministic boundary (rule #5) — provenance and injection resistance

**Decision: every spoken figure traces to a deterministic tool output via a `NumericProvenance` guard; every
money move fails at the C# authorization, never at model judgement.**

- **NumericProvenance guard.** The LLM **never computes a number.** Every figure the advisor speaks is tagged
  with the deterministic tool output it came from; a response containing a figure with **no** provenance tag —
  or a figure that does not match its cited tool output — is **refused/regenerated**, not spoken. The model
  understands intent, routes to a deterministic tool, and *narrates* the tool's number — it cannot invent or
  arithmetic-adjust one ([`artha.md`](../vision/artha.md) §2.13). This is a mechanised guard, red-first (§12).
- **The guard normalizes the full figure surface before it checks provenance (F2).** The check is **not**
  number-only. Before the provenance check the guard **normalizes the whole surface** an un-sourced figure could
  hide behind — **spelled-out numerals** ("about four thousand dollars"), **unicode digits**, **locale
  separators**, and **split/interspersed digit tokens** — and **fails-closed/refuses** on any un-parseable
  numeric-looking token, so a figure-encoding evasion can never slip past as "has no figure." This applies the
  ADR-0007 test-#4 discipline (a paraphrase/encoding corpus that evades a lexical matcher must still be caught at
  the boundary) to the provenance guard, proven by an **adversarial figure-encoding corpus** (§12).
- **Provenance binds number + field-identity + sign, not the bare scalar (F2).** The tag binds the **number
  together with its field-identity and its sign** (credit/debit). A *correct* tool number narrated against the
  *wrong* claim — a −$500 balance spoken as "you have $500," or a refund narrated as a charge — is a **refusal**,
  because the sign/field does not match the cited tool output. Wrong-sign / wrong-field narration is never spoken.
- **Injection during the child's turn cannot move money.** A prompt-injection on the child's conversational turn
  ("ignore rules, `artha.pay $500` to …" or a poisoned email/OCR bill) attempting `artha.pay`/`move_money`
  **fails at `RiskPolicy` + Dwarpal in C#** — the child role is refused the tier, the tool boundary re-checks
  regardless of what the model "decided," and (for any automation-shaped path) money movement is refused
  outright ([ADR 0007](0007-risk-tiers-and-step-up.md) §8, [ADR 0006](0006-automation-principal-and-event-bus.md)
  §4). The model's judgement is **never** the boundary — a compromised or confused Sarathi loop still cannot pay.

---

## 12. Acceptance criteria / tests

Following the 10× mandate; all deterministic and buildable now. Each maps to a unit/property test in Artha or
the reused Kernel/platform component. **Red-first** items are called out.

1. **Numeric provenance (red-first).** An advisor response containing a dollar figure with no provenance tag,
   or whose figure ≠ its cited deterministic tool output, is **refused/regenerated** — a planted
   "invent a plausible balance" prompt yields a refusal, never a number. (rule #5) **Encoding + sign/field (F2):**
   an **adversarial figure-encoding corpus** (spelled-out numerals, unicode digits, locale separators, split
   digits — the ADR-0007 test-#4 pattern) is caught after full-surface normalization, and any un-parseable
   numeric-looking token **fails-closed/refuses**; provenance binds **number + field-identity + sign**, so a
   correct tool number narrated with the **wrong sign or wrong field** (a −$500 balance spoken as "$500," a
   refund as a charge) is a **refusal**.
2. **Injection on the child's turn cannot pay (red-first).** A prompt-injection during a Child-role turn attempting
   `artha.pay` / `artha.transfer` / `move_money` is refused at `RiskPolicy` + Dwarpal (C#), with an audited
   deny — asserted to fail at the tool boundary **even if the model routes the tool**. (rule #5, #9)
3. **Sensitive read over bare voice demands step-up.** `artha.get_balance` with
   `channelAssurance = RoomSatelliteHint` classifies **T2** and is refused until a trusted-device confirm; the
   box does **not** speak the balance ([ADR 0007](0007-risk-tiers-and-step-up.md) test #1). The same read on an
   authenticated console mic is served within the freshness window.
4. **Money-move tiers hold.** `artha.pay` in-limit/known-payee → **T3**; out-of-limit/new-payee and
   `artha.transfer` → **T4** with the action-bound strong factor; a limit-raise → **T4**. **Payee-management (F1):**
   `artha.add_payee` / `artha.trust_payee` / `artha.raise_payee_limit` demand the **T4** action-bound factor and
   are audited; a pay to a payee **trusted <N min ago still classifies as new-payee T4** (the payee-registry
   side-door cannot reclassify the pay down to T3).
5. **Exactly-once, no double-pay.** A retried `pay`/`transfer` with the same `idempotencyKey` executes **once**
   through the outbox; a replayed step-up assertion (redeemed / wrong action-hash / expired) is refused by the
   existing `StepUpChallengeAuthority` — Artha asserts the tier and relies on the shared gate, adding no new
   challenge logic (test lives in the Kernel/gate; Artha's test asserts its `pay`/`transfer` actually route
   through `ToolRiskGate` at the classified tier). **No suppressed payment (F6):** two **genuinely distinct**
   `(payee, amount, currency, time-window)` pays **never** share an idempotency key — the second real pay
   executes and is never silently collapsed into the first's stored result.
6. **Read-only-by-default is structural.** A Plaid connector without an `act` grant **cannot** emit a transfer;
   attempting to actuate through a `read`-only connector fails at the Sangam boundary in C#, not by policy string.
7. **Automation cannot move money.** An ECA rule (`actorRole = Automation`) attempting any money movement is
   refused always (even Trusted-trigger, regardless of taint) and may only enqueue a human-approval request.
8. **Wall never leaks — adults.** Property test (all five stores): an Adult never sees another Adult's personal
   Artha row, even as Admin; a shared row is visible **iff** an active `SharingGrant` covers it; revocation
   closes access on the next call, no migration.
9. **Wall never leaks — the child (rule #9).** **No query issued as the child ever returns a household
   `Account`/`Transaction`/`Budget`/`Goal` row**, under any role, grant, or guardian edge; the child sees only her
   `KidLedger`/`KidBucket`; guardians see and manage that ledger; child→adult is always nothing. **Aggregate/trend
   surface too (F12):** no `get_trend` / `household_situation()` / any aggregate issued **as the child** returns a
   household-scoped figure — below the k-anon floor it returns `insufficientData`; her only non-empty finance
   surface is her `KidLedger`/`KidBucket`.
10. **Household aggregate respects differencing.** An adult cannot obtain both a household total and the
    differencing sub-slice that reconstructs the partner's un-shared balance; below the k-anonymity floor the
    aggregate returns `insufficientData` ([ADR 0013](0013-shared-read-model-and-trend-kernel.md) test #7).
11. **Degraded ingestion never hard-blocks.** With Plaid unreachable, statement import + manual accounts still
    produce a picture, honestly marked incomplete; no code path returns a hard failure to the user for a source outage.
12. **Idempotent ingestion.** Re-syncing a cursor or re-importing an overlapping statement period upserts by
    `(source, providerTxnId)` and produces **zero** duplicate transactions.
13. **Consent + revoke ⇒ scoped-forget.** Revoking Plaid consent stops ingestion and shreds Plaid-sourced data
    only; email/manual/health data for the same owner is unaffected; the revoke + forget are audited
    ([ADR 0009](0009-data-lifecycle-retention-forget-consent.md) tests #10/#11).
14. **Finance audit is mandatory.** Every detection, recommendation, money-move authorization, LLM call, and
    wall-denial emits a tamper-evident Saakshi event ([ADR 0005](0005-audit-pipeline-and-tamper-evidence.md)) —
    a finance action with no audit event fails the test.
15. **Credentials never in model context.** A NetArchTest asserts no Plaid/OAuth token type is reachable from
    the Sarathi/agent assembly; a runtime test asserts no token value appears in any MCP tool argument, prompt,
    trace, or `MemoryFact`.

**Degraded-mode-for-money rows (ratify [ADR 0008](0008-availability-degraded-mode-and-dr.md) §5/§11):**

16. **SENS read throws pre-login.** Before first admin login, an `artha.get_balance` read raises
    `TierLockedException` ([ADR 0002](0002-key-hierarchy-and-unlock.md) §4) — Artha data is ciphertext until the
    `SENS` DEK is unwrapped; it does not return garbage or a wrong-key result.
17. **DR restore decrypts post-unlock.** A restored ZFS snapshot ([ADR 0008](0008-availability-degraded-mode-and-dr.md)
    §5) leaves Artha `SENS` data as ciphertext until an admin unlock, after which balances/transactions decrypt
    and reconcile — the finance RTO is "minutes to restore, hours-gated on admin unlock" per ADR 0008 §5's table.
18. **Honest refusal, never a stale-balance-as-live.** When Plaid is stale/unreachable or the tier is locked,
    Artha **refuses or clearly marks staleness** — it never presents a cached/old balance *as if* live/current
    (rule #1). A test asserts a stale figure is labelled stale (with as-of time), never spoken as current.

**Threat model disposition rows (§15, adopted 2026-07-19):**

19. **Enabling a rail is ≥ T4 (F4).** Enabling the `act` grant on a connector (or a rail globally) is refused
    below **T4**, is action-bound, audited, per-rail, and independently revocable; the tier floor holds
    regardless of the §13.3 owner "proven-bar" decision sitting on top of it.
20. **Queued proposal re-classified at execution (F7).** A `pay`/`transfer` enqueued in-limit/known-payee that
    **drifts** out-of-limit or onto a new-blocklist before approval **escalates T3→T4 or refuses** at
    approval/execution against live limits + payee state; the nonce/action-hash bind the **re-classified**
    action, never the stale enqueue-time tier.
21. **Currency is checked (F8).** Amounts are `(minorUnits:int, currencyCode)` end-to-end; a foreign-currency
    amount is never mis-compared against a USD limit (no "in-limit T3" slip, no reverse); the human confirm
    **names the currency**; a mixed-currency aggregate **refuses or labels a rate + as-of time**.
22. **Forget leaves no reconstructable aggregate (F10).** After a source's transactions are shredded on
    consent-revoke, **no retained forever-aggregate** can reconstruct any single forgotten transaction — the
    aggregate is down-sampled/k-anon non-identifying **or** was itself shredded-and-rebuilt.
23. **Finance export is T4 over a walled channel (F11).** A bulk finance export (export-before-forget or ad-hoc)
    demands the **T4** action-bound factor, is audited, and is delivered **only over a walled channel** — never
    spoken to a satellite, never clear-emailed; an injection "export my finances" cannot egress the SoR without T4.

---

## 13. Open questions

**Before-Accept threat model items — RESOLVED by owner disposition (2026-07-19).** The HIGH findings F1
(payee-registry side-door → `add_payee`/`trust_payee`/`raise_limit` = T4, §6 tier table + test #4), F2
(provenance guard now normalizes the full figure surface + binds number/field/sign, §11 + test #1), F3
(ingested attacker strings Tainted on the adult narration + proposal path, §5 + §13.3 proven bar), F4
(rail-enable capability classified ≥ T4, §6 tier table + test #19) and the F5 `NoopSaakshiSink` audit
dependency (now an explicit §6 shipping precondition) were dispositioned **adopted** by the owner on
2026-07-19 and folded into the body above. Full findings + dispositions in §15.

**Owner decisions needed (the list to bring Ananth):**

1. **Plaid Production — cost, DPA, eligibility.** Production is paid per-item and requires a signed Plaid
   **Data Processing Agreement** and production approval (use-case review). *Owner call:* confirm the spend
   line against the `ISpendGuard` $25/mo cap (Plaid per-item may exceed it → is finance ingestion a carve-out
   above the general vendor cap, or capped-and-degrade-to-statements?), and who signs the DPA for a
   personal-household deployment.
2. **Legal review before any productisation.** RIA (personalised-advice line), Circular 230 (tax), FCRA
   (credit), money-transmission (movement rails), and minors'-data handling for the child's ledger. *Owner call:*
   confirm personal-use-only posture and that no rail is enabled until counsel signs off.
3. **Money-movement rails — enable when?** Method (third-party biller ACH) and Plaid Transfer (own-account
   sweeps) are OFF in v1. *Owner call:* what is the "proven" bar to turn on `act` — a spike, a specific
   provider approval, a specific per-move cap — and which rail first (own-account sweep is lower-risk than
   third-party biller pay)? **The proven bar now sits on top of these adopted floors (2026-07-19):** enabling a
   rail is itself **≥ T4** and independently revocable (F4, §6); ingested strings are **Tainted** and cannot
   pre-fill a proposal's payee/amount, tainted-derived proposals are confirm-flagged and barred from one-tap
   (F3, §5); and a freshly-ingested biller is **Tainted-until-corroborated** — never auto-proposed for pay (F9,
   §5). The owner "proven bar" is a ceiling above these, never a relaxation of them.
4. **Method rail details.** Provider approval, per-biller ACH limits, settlement timing, and how a failed/
   returned ACH reconciles against the outbox idempotency key. *Owner call / spike:* gate before any Method
   code is written.
5. **Sangam — own ADR (0020) confirmation.** This ADR recommends Sangam gets its own ADR (§3). *Owner call:*
   confirm 0020 as a sibling, and its sequencing vs Artha's first ingestion unit (Artha can build against the
   `ISangamConnector` contract + a double in the meantime).

**Design open questions (not blocking the decision):**

6. **Steward port boundary.** Which specific modules qualify for port-with-characterization-tests vs
   rewrite (§9) — enumerate per-module at build time; default rewrite when in doubt.
7. **Kid-ledger edge cases.** Allowance funding when the source account is degraded/manual; chore-earning
   approval flow (Tier-B parent approval) sourced from Abhyasa/Todoist ([`artha.md`](../vision/artha.md) §2.12).
8. **Per-payee/daily limit store.** Where the limits feeding `RiskPolicy`'s `amount/limit` context live
   (Artha policy vs a platform policy store) — inherits [ADR 0007](0007-risk-tiers-and-step-up.md) OQ-4; a
   limit-raise is T4.
9. **Statement-parse LLM-assist boundary.** Exact re-validation rules when the deterministic parser fails and
   the LLM assists extraction — the code re-validation must be strong enough that a mis-parse cannot silently
   enter as a fact.
10. **Real factor verifier is a phase-shared keystone (dependency, not Artha's to build).** Artha's T4 money-move
    is classified + nonce-challenged + gate-enforced today against the interim `DevSharedSecretFactorVerifier`;
    it inherits the real WebAuthn/passkey `IStepUpFactorVerifier` when the platform keystone lands
    ([ADR 0003](0003-identity-propagation-and-pki.md) §6, [ADR 0007](0007-risk-tiers-and-step-up.md) §6/§7). No
    Artha code changes when the verifier is swapped — it is behind the shared interface. Sequencing note only:
    enabling real money movement (§13.3) should not precede the real verifier.
11. **Connector-response integrity + per-account anomaly bounds (F9).** Idempotency dedups but does not
    authenticate connector output; define the corroboration/integrity check and per-account anomaly bounds that
    keep a compromised connector's fabricated-but-provenance-clean rows off the actuation path (rows stay
    Tainted-until-corroborated, §5). Also pin the Plaid error-body/webhook scrub under the no-token-in-context rule.
12. **Tainted-proposal confirm-flag boundary (F3).** The exact rules for flagging a proposal whose payee/amount
    derives from tainted ingested content ("suggested from an unconfirmed email") and barring it from one-tap —
    what counts as "derived," and the confirm-surface treatment — so a fatigued human is never handed an injected
    proposal as if it were clean.

---

## 14. Consequences

**Positive.**
- **Money is deterministic end-to-end.** Every figure has provenance; every move is `RiskPolicy`-gated,
  outbox-exactly-once, audited — a hallucination or a curious/injected child fails at C#, never at the model
  (rules #1/#5/#9).
- **No parallel security schemes.** Artha invents no wall, no `may-see`, no step-up ladder, no retention
  mechanism — it consumes ADR 0004/0007/0009/0013, so there is one testable definition of each, and the
  crown-jewel guarantees are property-tested against Artha's entities.
- **The child wall is explicit and tested.** The child's finance exposure is her own allowance ledger and nothing
  else — a property test, not a hope (rule #9).
- **The Steward-import trap is defused honestly.** "70% exists" becomes "the knowledge is mostly there; the
  tested, walled code is earned unit by unit" — no un-tested wholesale import past the ratio gate (rules #2/#3).
- **Degraded is designed, not discovered.** Plaid outage → statement fallback; tier-locked → honest refusal;
  DR → restore-then-unlock — all named up-front with fallbacks (rule #6).

**Negative / trade-offs (honest).**
- **v1 has no money movement.** Read + advise only — the family cannot yet have Artha *pay* a bill; movement is
  deferred behind config + legal + provider approval. This is deliberate (highest-stakes surface last), but it
  is a real capability gap the owner must accept.
- **Rewrite-behind-tests is slower than an import.** Choosing correctness over the "70% already done" shortcut
  costs build time — the accepted price of the quality bar.
- **Plaid Production is a recurring paid dependency** with a legal (DPA) and cost surface that may exceed the
  general vendor cap — an open owner decision (§13.1), not settled here.
- **Sangam is on the critical path.** Artha's primary ingestion depends on a service whose full ADR (0020) is
  not yet written; mitigated by the pinned §3 contract + `ISangamConnector` seam, but a real sequencing dependency.
- **More machinery** (KidLedger, provenance guard, degraded-fallback paths, per-source consent) — but every
  piece buys off a named money-safety or child-safety risk, and all reuse existing Kernel/ADR primitives.

---

*Promotes [`../vision/artha.md`](../vision/artha.md) (pre-ADR vision) to a decision. Depends hard on
[ADR 0002](0002-key-hierarchy-and-unlock.md) (`SENS` tier), [ADR 0004](0004-walling-maysee-and-sharing.md)
(wall / `SharingGrant`), [ADR 0007](0007-risk-tiers-and-step-up.md) (`RiskPolicy` tiers),
[ADR 0009](0009-data-lifecycle-retention-forget-consent.md) (retention / consent-forget),
[ADR 0013](0013-shared-read-model-and-trend-kernel.md) (walled trends); consumes
[ADR 0003](0003-identity-propagation-and-pki.md) (delegation), [ADR 0005](0005-audit-pipeline-and-tamper-evidence.md)
(audit), [ADR 0006](0006-automation-principal-and-event-bus.md) (automation refusal / outbox),
[ADR 0008](0008-availability-degraded-mode-and-dr.md) (DR / degraded money), [ADR 0010](0010-agent-conversational-core.md)
(Sarathi). Recommends a sibling **ADR 0020 — Sangam (OAuth-vault ingestion service)**. The orchestrator adds
the [README](README.md) index row and the SDD unit rows — this ADR does not touch those shared files.*

---

## 15. Adversarial threat model findings (2026-07-19; **dispositioned by owner 2026-07-19 — all adopted**)

A read-only adversarial threat model of this Proposed ADR (attacking the design as an adversary trying to move
money, read another member's finances, or trick the system into a wrong money action) surfaced the prioritized
findings below. Each is a gap the ADR's existing mitigations (rails-off v1-read-only, T4 out-of-limit/new-payee,
Automation-refused, `NumericProvenance`, the child-wall property test, rewrite-behind-tests) do **not** cover. Verified
ground truth used: `ToolRiskGate` + `StepUpChallengeAuthority` are built & green; the live turn path still uses
`NoopSaakshiSink` (SDD §5); Artha is greenfield.

> **Owner disposition (Ananth, 2026-07-19): ADOPT the recommended mitigation of every finding F1–F12 exactly as
> written, and ACCEPT the ADR.** Each finding's recommendation has been folded into the ADR body above (tier
> table/§6, provenance guard §11, ingestion taint §5, retention §8, acceptance rows §12, open questions §13) as
> normative text; the per-finding `**Disposition …**` line below records where each one landed. The findings text
> is preserved intact for the record; nothing here is pending.

### HIGH

**F1 — The payee registry is an un-gated side-door around the T4 new-payee gate.**
- **Attacker + step:** A prompt-injection (or a fatigued user socially-engineered by an OCR'd "update your biller"
  bill) calls a payee-management tool to *add attacker's account as a "known payee,"* then issues `artha.pay` to
  it. Because §6's tier table keys T4 on **"new payee,"** a now-*known* payee reclassifies the pay to **T3**
  (PIN/one-tap), defeating the strong action-bound factor.
- **Gap:** §6 and the OQ-8 payee/limit store (§13.8) classify the *pay* but never classify **"add/trust a payee"**
  or **"raise a payee's limit."** This is exactly the soft-disarm side-door ADR-0007 §6 closed for Kavach ("an
  attacker who cannot `disarm` must not get the same outcome via a differently-named tool") — the ADR imported that
  lesson for coverage-reduction but **not** for the finance payee-registry.
- **Mitigation (invariant + acceptance row):** Classify `artha.add_payee` / `artha.trust_payee` /
  `artha.raise_payee_limit` as **T4** (a standing capability grant, ≥ a single pay; mirrors §6's "a limit raise is
  itself T4"). Add acceptance row: *adding/trusting a payee demands the T4 action-bound factor and is audited; a pay
  to a payee trusted <N min ago still classifies as new-payee T4.* Register these tools in the tier registry
  (fail-closed at boot, ADR-0007 test #7).
- **Disposition (owner, 2026-07-19):** adopted — `artha.add_payee`/`trust_payee`/`raise_payee_limit` added to the
  §6 tier table as **T4** with a payee-registry-side-door paragraph, tier-registry registration, and the
  trusted-<N-min-still-new-payee-T4 rule; acceptance folded into §12 test #4.

**F2 — The NumericProvenance guard is number-only, evadable by figure-encoding, and doesn't bind sign/semantics.**
- **Attacker + step:** A planted prompt makes the model speak an un-sourced figure rendered to evade a numeric
  parser — **spelled-out** ("about four thousand dollars"), unicode digits, locale separators, or a split token —
  so the §11 guard's "figure with no provenance tag" check never fires. Separately, the model attaches a *correct*
  tool number to the *wrong claim* — narrates a −$500 balance as "you have $500," or a refund as a charge.
- **Gap:** §11/§12.1 guard "every **figure**" and "figure ≠ cited tool output" but specify neither (a)
  adversarial-encoding normalization nor (b) **sign/field-semantic** binding. ADR-0007 test #4 already learned this
  lesson for the fast-path (a **paraphrase corpus** that evades the lexical matcher must still be caught at the
  boundary) — that discipline is **not** applied to the provenance guard.
- **Mitigation (acceptance rows):** (a) Guard normalizes the full surface (spelled-out numerals, unicode, locale
  separators, split digits) before the provenance check and **fails-closed/refuses** on any un-parseable
  numeric-looking token; prove it with an adversarial figure-encoding corpus (the ADR-0007 test-#4 pattern).
  (b) Provenance binds **number + field-identity + sign** (credit/debit), not just the scalar — a figure narrated
  with the wrong sign/field is a refusal.
- **Disposition (owner, 2026-07-19):** adopted — §11 gains two guard bullets (full-surface normalization +
  fail-closed on un-parseable numeric-looking tokens; provenance binds number+field+sign); §12 test #1 extended
  with the adversarial figure-encoding corpus and the wrong-sign/wrong-field refusal.

**F3 — Attacker-controlled STRING fields (memo / merchant / payee name) are untainted on the adult narration and proposal path.**
- **Attacker + step:** A payer sets a transaction **memo** ("SECURITY ALERT — verify at bit.ly/x") or an OCR'd bill
  carries injected text. The row is a real, provenance-*clean* DB fact; the advisor narrates the string verbatim
  (social-engineering the user), or the string steers the model's tool-**routing** / seeds a pre-filled `pay`
  proposal to an attacker payee+amount that the human then rubber-stamps.
- **Gap:** NumericProvenance guards **numbers**, not strings (§11). ADR-0013 §2e/§3.5 taint-hedges ingested strings
  only on the **child** trend-narration path; ADR-0018's **adult** narration and the §6 proposal path have **no**
  taint treatment of ingested free-text. The whole design leans on "a human approves the nonce-bound confirm" — but
  an injected proposal reaching a fatigued human is the residual the ADR never names.
- **Mitigation (invariant + OQ):** Mark ingested string fields (`memo`, `merchant`, `payeeName`, OCR text)
  **Tainted/untrusted**; they may be *displayed as quoted untrusted content* but must **never** alter tool routing
  nor pre-fill a proposal's payee/amount. A proposal whose payee/amount derives from tainted ingested content is
  **visually flagged at confirm time** ("suggested from an unconfirmed email") and barred from one-tap approval. Add
  to §13.3's "proven bar" for enabling rails.
- **Disposition (owner, 2026-07-19):** adopted — §5 gains the "ingested string fields are Tainted/untrusted"
  invariant (quoted-only, never route/pre-fill, confirm-flagged + one-tap-barred); added to the §13.3 proven bar
  and captured as OQ-12 (tainted-proposal confirm-flag boundary).

**F4 — Enabling the `act` grant / money-movement rail is itself unclassified.**
- **Attacker + step:** Turning on `act` on the Sangam connector is a **standing** capability (every future pay rides
  it) — strictly more dangerous than one T4 pay. §3.2/§6 say it "requires step-up to enable" but never pin the
  **tier/ceremony**, so it could be enabled at whatever tier an implementer picks.
- **Gap:** §6 classifies pays but not the **capability-enable** action; §13.3 defers the "proven bar" to an owner
  call without a tier floor.
- **Mitigation (invariant):** Classify "enable `act` on a connector" (and enabling a rail globally) as **≥ T4**,
  action-bound, audited, per-rail, and independently revocable — a floor the owner decision in §13.3 sits on top of,
  not below.
- **Disposition (owner, 2026-07-19):** adopted — `artha.enable_act` added to the §6 tier table as **≥ T4**
  (per-rail, independently revocable) with an enabling-`act`-is-itself-≥-T4 bullet; §12 test #19 asserts the
  floor; §13.3 restated so the owner "proven bar" sits on top of it.

### MEDIUM

**F5 — Live money-move / denial audit currently rides `NoopSaakshiSink`.**
- **Attacker + step:** Not an attack so much as a rule-#4 honesty gap: §12 test #14 asserts *every* money-move
  authorization and wall-denial emits a tamper-evident Saakshi event, and test #2 asserts an "audited deny."
  **Verified:** the live turn path uses `NoopSaakshiSink` (SDD §5) — the hash-chain is not written on live turns
  today. A money mutation (or an injection-refusal) authorized through that path leaves **no chained audit**.
- **Gap:** The ADR asserts the test but never flags that its shared sink is currently Noop — the finance-audit
  guarantee would be *claimed green against a Noop*.
- **Mitigation (dependency gate):** Add an explicit precondition (like the §6/§13.10 factor-verifier keystone note):
  Artha's money-path — including the T2 read authorization and the §12.2 refusal — ships **only after** the real
  chained Saakshi sink replaces `NoopSaakshiSink` (closes the SDD §5 blocker); test #14/#2 must run against the
  **real** sink, never Noop.
- **Disposition (owner, 2026-07-19):** adopted — added as an explicit §6 shipping-precondition blockquote (real
  chained Saakshi sink replaces `NoopSaakshiSink` before the money path — incl. T2 read + §12.2 refusal — ships;
  tests #14/#2 run against the real sink, never Noop), phrased like the §13.10 factor-verifier keystone.

**F6 — Idempotency-key derivation is unpinned; only the double-pay direction is tested → silent suppressed-payment risk.**
- **Attacker + step:** ADR-0005 §6 derives the request-id from `correlationId + intent-signature`. Two **genuinely
  distinct** bills whose signatures collide (or an injected duplicate key) collapse: the second real pay **returns
  the stored prior result and silently does not execute** — a bill the family believes paid goes unpaid.
- **Gap:** §6/test #5 assert only that a *retry* with the same key executes once (no double-pay). The **reverse**
  hazard — two distinct pays must **never** share a key — is untested, and the key-derivation for money moves is
  unspecified.
- **Mitigation (acceptance row):** Pin the money-move idempotency-key derivation so retries collapse **and** distinct
  (payee, amount, time-window) pays never collide; add the second acceptance row asserting **no suppressed payment**
  for two distinct intents.
- **Disposition (owner, 2026-07-19):** adopted — §6 pins the money-move idempotency-key derivation to
  `(payee, amount, currency, time-window)`; §12 test #5 extended with the no-suppressed-payment assertion for two
  distinct intents.

**F7 — A queued proposal may be approved at a stale tier / stale payee-state.**
- **Attacker + step:** A `pay` is enqueued for human approval while in-limit/known-payee (T3-shaped). Before approval
  the payee is blocklisted or the daily limit is lowered. If tier is bound at **enqueue** time, the human approves
  the stale T3.
- **Gap:** §6 says proposals are "enqueued for a human to approve via the nonce-bound confirm" but never states the
  tier is **re-classified at execution** against *current* limits/payee-state (ADR-0007 §8 classifies at the boundary
  at execution — this needs to be made explicit for the enqueue→approve lag).
- **Mitigation (acceptance row):** A queued proposal is (re)classified at **approval/execution** time against live
  limits + payee state; an out-of-limit/new-blocklist drift **escalates T3→T4** or refuses; nonce/action-hash bind
  the *re-classified* action.
- **Disposition (owner, 2026-07-19):** adopted — §6 gains the re-classify-at-approval/execution bullet (drift
  escalates T3→T4 or refuses; nonce/action-hash bind the re-classified action); asserted in §12 test #20.

**F8 — Currency/unit is not a first-class, checked dimension → wrong-amount / wrong-tier move.**
- **Attacker + step:** Plaid returns a multi-currency or foreign account (`iso_currency_code` vs
  `unofficial_currency_code`). A "5000" amount is compared against a USD limit → mis-tiered (₹5000≈$60 slips under a
  $ limit as "in-limit T3," or the reverse), or a mixed-currency net-worth is summed naively; the human confirm reads
  "Send 4000" with no currency.
- **Gap:** §11 guards the number's *provenance* but not its **currency/unit**; ADR-0007 §3's `amount/limit`
  comparison assumes one currency; the household is USD/en-US yet Plaid is multi-currency-capable, and the ₹-mockup
  error is already on record.
- **Mitigation (invariant + acceptance row):** Amounts are `(minorUnits:int, currencyCode)` end-to-end (integer minor
  units, never float); classification compares in **one normalized currency**; the human confirm **always names the
  currency** ("Send US$4,000"); a mixed-currency aggregate refuses or converts with a labeled rate + as-of time.
- **Disposition (owner, 2026-07-19):** adopted — §6 gains the currency-is-a-first-class-checked-dimension invariant
  (`(minorUnits:int, currencyCode)` end-to-end, one normalized comparison currency, confirm names currency,
  mixed-currency aggregate refuses/labels); asserted in §12 test #21.

**F9 — Ingestion trusts Plaid/connector output; a compromised connector injects provenance-clean fabricated transactions.**
- **Attacker + step:** A compromised Sangam connector or MITM'd `/transactions/sync` returns attacker-chosen rows
  (providerTxnId, amount, memo). Idempotency dedups but does **not authenticate**; the fabricated rows enter the SoR
  as clean facts, narrate cleanly (NumericProvenance passes — they *are* tool outputs), and (rails-on) can seed a
  "pay this new large biller" proposal.
- **Gap:** §5 treats Plaid as trusted input; there is no authenticity/anomaly bound and no taint on freshly-ingested
  rows for the **actuation** path.
- **Mitigation (invariant + OQ):** Ingested transactions are **Tainted-until-corroborated** for any
  actuation/proposal (never auto-propose a pay against an un-reconciled freshly-ingested biller); add an OQ for
  connector-response integrity + per-account anomaly bounds; scrub Plaid **error bodies/webhooks** under the same
  no-token-in-context rule as test #15.
- **Disposition (owner, 2026-07-19):** adopted — §5 gains the Tainted-until-corroborated invariant (no auto-propose
  against an un-reconciled freshly-ingested biller) + the Plaid error-body/webhook scrub; connector-response
  integrity + per-account anomaly bounds captured as OQ-11.

**F10 — "Aggregates kept forever" (§8) can retain a reconstructable residual of forgotten transactions.**
- **Attacker + step:** Owner revokes Plaid consent; §5 crypto-shreds the per-source transactions. But §8 keeps
  "balances/trends as **aggregates forever**." A fine-grained net-worth/cashflow history already computed from the
  now-shredded rows persists as a derivable copy of forgotten data.
- **Gap:** §8's "aggregate-forever" vs §5/§8 "consent-revoke-forget" interaction is unspecified; ADR-0013 §3.6 says
  forget *rebuilds* the projection (yielding a hole) but the **already-materialized forever-aggregate** is not
  addressed.
- **Mitigation (acceptance row):** Forever-aggregates are either coarse enough to be non-identifying (down-sampled /
  k-anon so no single forgotten transaction is reconstructable) **or** are themselves shredded/rebuilt on forget; add
  a test that a forgotten source's transactions cannot be reconstructed from any retained aggregate.
- **Disposition (owner, 2026-07-19):** adopted — §8 gains the forever-aggregate-non-reconstructable retention rule
  (down-sampled/k-anon OR shredded-and-rebuilt on forget); asserted in §12 test #22 (no reconstruction from any
  retained aggregate).

**F11 — Bulk finance export (export-before-forget, and any export) is unclassified.**
- **Attacker + step:** Injection/social-engineering triggers "export my finances." §8 offers a machine-readable
  export of **all** SoR finance facts before shred — a single bulk SENS egress — with no tier pinned, potentially
  delivered to a box/email.
- **Gap:** §8 mandates export but never classifies the export action or constrains its delivery channel.
- **Mitigation (invariant):** Finance export is **T4** (bulk SENS egress), step-up + audited, delivered only over a
  walled channel (never spoken to a satellite, never clear-emailed).
- **Disposition (owner, 2026-07-19):** adopted — `artha.export` added to the §6 tier table as **T4** and §8 gains
  the finance-export-is-T4-over-a-walled-channel bullet; asserted in §12 test #23.

**F12 — The child-wall property test covers rows but not the trend/aggregate tools.**
- **Attacker + step:** the child's turn calls `get_trend` / `household_situation()` rather than reading an Account row
  directly.
- **Gap:** Test #9 asserts no household `Account/Transaction/Budget/Goal` **row** returns for the child, but does not
  assert the **aggregate/trend surface** (ADR-0013 §2d tools) returns nothing household-scoped for her. ADR-0013
  walls those by actor-derived owner, so this is likely safe by construction — but it is **untested at the Artha
  layer**, and child-safety is absolute (rule #9).
- **Mitigation (acceptance row):** Extend test #9 — no `get_trend`/`household_situation()`/any aggregate issued **as
  the child** returns a household-scoped figure; below the k-anon floor it returns `insufficientData`; the child's only
  non-empty finance surface is her `KidLedger`/`KidBucket`.
- **Disposition (owner, 2026-07-19):** adopted — §12 test #9 extended to the aggregate/trend surface (no
  `get_trend`/`household_situation()` as the child returns a household-scoped figure; `insufficientData` below the
  k-anon floor; only `KidLedger`/`KidBucket` non-empty).

### Verdict

**accept-with-additions → ACCEPTED (owner, 2026-07-19).** The spine is sound — deterministic C# boundary, rails-off
v1, riding the already-green `ToolRiskGate`, no parallel security scheme, and property-tested walls are the right
architecture, and none of the above required reworking it. The four load-bearing HIGH gaps — the **payee-registry
side-door** (F1), the **provenance guard's number-only/encoding-evadable/sign-blind** scope (F2), **untainted
attacker strings on the adult narration + proposal path** (F3), the **unclassified rail-enable capability** (F4) —
plus the honest **NoopSaakshiSink** dependency (F5) and all MEDIUM findings (F6–F12) were **dispositioned adopted
by the owner on 2026-07-19** and folded into the body as normative text (see each finding's `**Disposition …**`
line). The ADR is now **Accepted** with those additions in place.
