# ADR 0020 — Sangam (सङ्गम): OAuth Vault, `act` Grants & the Rail Execution Contract

- **Status:** **ACCEPTED — owner-ratified by Ananth, 2026-08-14 (option card): the seven decided
  positions plus the draft's recommended defaults for the sizing opens (24h/2-cycle corroboration window;
  reconcile cadence decided at its build unit with a measured gate). Ratified from the summarized positions
  with the full text on disk; amendments remain the owner's.**
- **Date:** 2026-08-14
- **Deciders:** Ananth (owner — pending), Claude (orchestrator)
- **Refines:** [ADR 0018](0018-artha-household-finance.md) §3 (`:92-127`, the minimal contract this ADR now
  completes) and §6 (`:208-309`, F1/F4/F6/F7/F9 — the money-move authorization ladder this ADR's execution
  contract must ride, never re-implement) · [ADR 0007](0007-risk-tiers-and-step-up.md) §3/§8 (tier table,
  request-never-complete, `:270`) · [ADR 0002](0002-key-hierarchy-and-unlock.md) §4 (key tiers — the vault's
  custody tier, precedented by the `DWARPAL-BOOT` dedicated-slice pattern, `:90`) · [ADR 0006](0006-automation-principal-and-event-bus.md)
  (outbox exactly-once, Automation refusal) · [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) §6
  (consent/revoke/forget) · [ADR 0005](0005-audit-pipeline-and-tamper-evidence.md) (Saakshi audit, idempotency-key
  chain fields).

**RATIFIED 2026-08-14.** Authored the same day under an earlier owner ruling that green-lit drafting; the pre-ratification banner that stood here for the hours between authorship and the option-card answer is preserved in git history.

> **Greenfield check, 2026-08-14.** `find src/Sangam` → no such directory. No `ISangamActConnector`, no vault
> store, no `act`-capable `ConnectorGrant` minter exists anywhere in `src/`. `ConnectorGrant.Act` is a declared
> enum member with **zero public constructor path to it** (`ConnectorGrant.cs:47-58` — `ReadOnly` is the only
> factory). `MoneyMoveProposal.ApproveAndRefuseExecution` is the **only** domain method that ever touches
> `ProposalState`'s approve/execute transition, and it moves `Proposed` straight to `ExecutionRefused` in one
> save (`MoneyMoveProposal.cs:189-200`); `ProposalState.Approved` is a reserved, never-persisted enum value
> whose doc comment says it is "**reserved for a future rail wave**" (`ProposalState.cs:20-24`). This ADR is
> that wave's design.

---

## 1. Context

Wave 4 shipped Artha's proposal queue honestly inert: a household member can enqueue a `pay`/`transfer`, have it
re-classified live at approval (F7), and watch it land at `ExecutionRefused(NoRailConfigured)` — every dollar
figure real, nothing on the box can move it (the prior wave closed with the queue honestly inert). Three tracked
debts were opened on purpose to name what this ADR must now decide: the F9/OQ-11 corroboration predicate (left
undefined), a doc-comment-only guard against Bill/Loan data seeding a proposal, and the reserved
`ProposalState.Approved` value itself. An owner ruling green-lit drafting
this ADR **in parallel** with the standing-rules/Automation-identity wave, accepting the coordination cost that
the standing-rules→execution seam may force a second design pass here.

Three things make this the single highest-stakes ADR in the set: (1) it is the first design in this repo that
lets a deterministic path **actually move real money outside the household** — every prior gate (T4, nonce,
`ToolRiskGate`) has protected a *proposal*; this protects an *actuation*. (2) The vault holds Plaid access
tokens — real bearer secrets, not the verifier-only material `DWARPAL-BOOT` already accepted auto-unsealing.
(3) A rail can fail **ambiguously** — the call may have succeeded at Plaid while the response never reached
Artha — and the wrong answer here is a real, second, unwanted transfer.

---

## 2. Decision — summary

1. **The vault is a new service, `Sangam`, with its own dedicated, non-auto-unseal key slice** (`SANGAM-VAULT`),
   sealed until first admin login — never `MTRUST` (real secrets, unlike `DWARPAL-BOOT`'s verifiers-only
   material), never Artha's `SENS` (a platform seam must not live inside one consumer's store).
2. **`act` is a second, independently-revocable grant per rail**, minted only by a new T4-gated service and
   persisted as its own row (`ActGrantRecord`) in the Sangam vault, extending — never bypassing — the existing
   `ConnectorGrant`/`ConnectorCapability` structural-absence pattern A4 already built.
3. **A new `ISangamActConnector` port, owned by Artha.Application (consumer-owned, per the ADR-0018 §3
   pattern already in code), never widening `ISangamConnector`.** Execution gets a **third** classification —
   enqueue, approval, and now execution/dispatch — run by the outbox handler immediately before the network
   call. The F6 idempotency key propagated to Plaid is `SHA-256(ProposalId)`, not a re-derived tuple.
4. **Corroboration (F9) is defined**: persistence-across-independent-sync-cycles **and** (a second independent
   source **or** an explicit member attestation) — never either alone.
5. **Ambiguous outcomes are terminal until reconciled — never auto-retried, ever.** A new `PendingReconciliation`
   proposal state exists for exactly this, and only a reconcile job (never the outbox's normal retry path) may
   close it.
6. **Sandbox-only, structurally**, until a separate, later, owner-gated wave (a later wave) flips a config gate
   that mirrors `STEPUP-DEV`'s refuse-to-boot pattern.
7. **New-payee irreversible transfers stay human-gated forever** (`artha-autonomous-finance-stakes` memory,
   verbatim) — this ADR builds a rail for *approved, human-confirmed* moves; it does not, and structurally
   cannot, build autonomy over an irreversible new-payee send.

---

## 3. The OAuth vault — custody tier and service boundary

**Decision: `Sangam` is a new sibling service (`src/Sangam/{Sangam.Domain,Sangam.Application,Sangam.Infrastructure}`)
owning consent records, the token vault, and connector lifecycle. Tokens live in a dedicated `SANGAM-VAULT` key
slice, sealed until first admin login — not auto-unsealed, not folded into `SENS`.**

- **Why not `MTRUST` (auto-unseal).** `MTRUST`'s auto-unseal set exists because mesh/CA material must be live
  *before any human is present* (ADR 0002 §2, boot-circularity). Nothing about ingestion has that constraint —
  Artha's own `SENS` DEK already waits for first admin login (`TierLockedException` pre-login, ADR 0018 §12
  test #16), and a Plaid `access_token` is a live bearer credential over real money, not a boot-circularity
  problem. Auto-unsealing it on a stolen powered box would hand an attacker a working transfer credential
  outright — categorically worse than `DWARPAL-BOOT`'s accepted exposure of password *hashes* and passkey
  *public* keys (verifiers, not secrets, ADR 0002 `:90`). **Rejected.**
- **Why not `SENS` (Artha's own tier).** ADR 0018 §3 already states the credentials-never-in-Artha's-store
  line: *"OAuth tokens … live in the Sangam vault … never materialised into any MCP tool argument, prompt,
  trace, or `MemoryFact`"* (`0018:113-117`). Folding the vault into Artha's `SENS` DEK ties its unlock timing
  to Artha's own login flow and conflates a **platform** seam (email, notifications, Plaid all pass through
  Sangam) with one consumer's encrypted store — exactly the mis-scoping ADR 0018 §3 already ruled out for
  putting Sangam's *design* inside the finance ADR. **Rejected.**
- **Chosen: a dedicated `SANGAM-VAULT` slice**, precedented by `DWARPAL-BOOT` — ADR 0002 kept that slice
  separate from `MTRUST` "so its exposure is named and bounded, not mixed with mesh/CA material" (`0002:90-91`).
  `SANGAM-VAULT` takes the same *dedicated* posture but the **opposite** unlock trigger: sealed until first
  admin login (like `SENS`/`GEN`), because unlike Dwarpal's credential store there is no cold-boot reader that
  needs it before a human authenticates — ingestion can legitimately wait. A pre-login ingestion attempt raises
  the same `TierLockedException` shape ADR 0018 test #16 already asserts for Artha's own `SENS`.
- **Consent records** are `WalledEntity`s in Sangam's own store (`{ownerId, source, scope, grantedAt,
  grantedBy}`, ADR 0018 §3 clause 1), audited via Saakshi like every other finance write (ADR 0005).
- **Ports stay consumer-owned — no shared `Sangam.Contracts` assembly.** ADR 0018 §3 already built this pattern
  into live code: `ISangamConnector` is defined in `Artha.Application.Ingestion`, not a shared library, and A4's
  own doc calls it "the seam Artha builds to" (`0018:126`). `Sangam.Infrastructure` implements it
  (`PlaidSangamConnector : Artha.Application.Ingestion.ISangamConnector`) with a one-directional project
  reference (`Sangam.Infrastructure → Artha.Application`, never the reverse), wired at Artha's composition root
  (`AddArthaRuntime`). A future email/notification consumer gets its own analogous port in its own Application
  layer. **Alternative considered and rejected:** hoist `ConnectorHandle`/the port shape into a shared
  `Sangam.Contracts` project so every consumer references one type. Rejected because it inverts code already
  shipped in A4 for zero benefit — the handle carries no secret (it is an opaque reference), so there is nothing
  a shared assembly protects that per-consumer ownership does not; the churn would be a rename with no new
  guarantee.

---

## 4. The `act` grant model — per-rail, independently revocable, T4 to enable (F4)

**Decision: `act` is minted only by a new `EnableActGrantService` that itself demands T4 via `ToolRiskGate`, and
is persisted as one `ActGrantRecord` row per `(connector, rail)` pair — never a boolean flag on the connector.**

- **What a grant row is.** `ActGrantRecord { ConnectorHandle, Rail (PlaidTransfer | PlaidMethod), EnabledBy,
  EnabledAtUtc, RevokedAtUtc? }`, stored in the Sangam vault (§3) alongside the token it authorises spending
  through — the same store that holds the secret is the store that can say whether the secret may currently be
  used, so revocation and credential custody cannot drift apart into two systems that disagree.
- **Extends `ConnectorGrant`, does not bypass it.** A4 already declared `ConnectorCapability.Act` with **no**
  public constructor path (`ConnectorGrant.cs:47-58`'s doc: *"a grant carrying `Act` is unconstructable from
  outside this type"*, and predicted exactly this: *"the A9 `enable_act` minter (F4) will widen this surface
  deliberately, tripping the golden member-list test that pins it"*). This ADR is that minter:
  `ConnectorGrant.WithAct(ConnectorHandle, TransactionSource, ActGrantRecord)` becomes the **second** public
  factory, callable only from `EnableActGrantService` after `ToolRiskGate.Classify` returns T4 and the action-bound
  factor redeems (ADR 0018 F4, §6 tier table row `artha.enable_act`). The golden member-list test trips on
  purpose, as A4's own doc said it would.
- **Enforcement is defense-in-depth, at two points.** (a) Artha's application layer checks
  `grant.Allows(ConnectorCapability.Act)` before it will even construct an execution intent (structural absence,
  §12 acceptance #6's pattern, extended). (b) **Sangam re-checks independently at the point it is about to use
  the token** — `Sangam.Infrastructure`'s adapter refuses to present the credential to Plaid unless its OWN copy
  of the `ActGrantRecord` is present and unrevoked, read fresh from the vault at the moment of use, not cached
  from Artha's earlier check. Two independent checks because the two checks answer different questions ("is
  Artha authorised to *ask*" vs "is Sangam authorised to *spend*"), and a single check is exactly the kind of
  "principal check only, no code behind it" gap named for Kavach's abort-with-code (a tracked debt) — a check
  that *looks* like the guarantee the ADR ratified but is strictly weaker against the threat named. Here the
  threat is a compromised Artha process forging or replaying a stale intent; only Sangam's independent re-read
  closes it.
- **Alternatives rejected.** (a) A single boolean `ActEnabled` on the connector — fails "per-rail" outright (F4,
  §6: *"per-rail, and independently revocable"*), a single Plaid Item with both Transfer and Method enabled
  could never disable one without the other. (b) A claim checked only by convention at call sites, no persisted
  row — rejected as the exact "policy string somebody could forget to run" shape ADR 0018 §12 acceptance #6
  already ruled out for read/act.

---

## 5. The execution contract — `ISangamActConnector`, the third classification, idempotency propagation

**Decision: a new port, never a widened `ISangamConnector`; a third, execution-time classification run by the
outbox handler; the F6 idempotency key propagated to Plaid is derived from the `MoneyMoveProposalId`, not
re-derived from the payee/amount/currency tuple.**

- **`ISangamActConnector` is a separate interface.** `ISangamConnector`'s own doc states its guarantee is
  *"the shape of the type, not a policy check somebody could forget to run"* and that its golden member-list
  test **must** turn red if a transfer-shaped method is ever added (`ISangamConnector.cs:8-13`). Widening it
  would destroy the one guarantee A4 built. `ISangamActConnector.ExecuteAsync(ActGrant grant,
  MoneyMoveExecutionIntent intent, string idempotencyKey, CancellationToken)` returns an `ActuationOutcome` —
  closed set `Succeeded(providerTransferId)`, `Refused(reason)` (rail declined before any money-moving call —
  safe, retryable), `Ambiguous(context)` (the call may have reached Plaid; outcome unknown — §6). Expected
  outcomes are returned, never thrown, mirroring `ISangamConnector`'s error model.
- **The proposal state machine is redeemed, not reinvented.** `ProposalState.Approved` becomes reachable for
  the first time: `Approve(nowUtc, tier)` (new domain method) persists `Approved` and stamps
  `ApprovedAtUtc`/`ExecutionIdempotencyKey` in one save — this is genuinely new, and it is safe *only* because
  the outbox (already built, ADR 0018 §6, exactly-once) is what guarantees `Approved` never strands, the same
  guarantee `ApproveAndRefuseExecution`'s single-save-single-call used to provide directly. **v1's method is
  kept, unchanged, as the fallback for the common case:** `MoneyMoveProposalService.ApproveAsync` checks whether
  any `act` grant covers this proposal's rail; if not, it still calls
  `ApproveAndRefuseExecutionAsync`/`NoRailConfigured` exactly as today — **zero behavioural change to a
  household with no rail enabled.** Only when a live grant exists does the new `Approve()` + outbox-execution
  path run. This directly answers the brief's question — execution *replaces* the terminal refusal only on the
  narrow path a rail is actually configured for, and never removes the honest no-rail path the rest of the
  household still exercises.
- **The third classification.** F7 already re-classifies at approval against live limits/payee state
  (ADR 0018 §6). The gap this ADR closes: the time between "approval redeemed" and "the outbox handler actually
  calls Sangam" is unbounded (crash recovery, retry backoff) — a real network hop, unlike the earlier two
  classifications which run inside one HTTP request. So the outbox execution handler runs
  `MoneyMoveClassifier.Classify` a **third** time, immediately before calling `ISangamActConnector`, against the
  live payee/limit state at that instant. A downward-favourable result (still in-limit) proceeds; **any**
  upward drift (new blocklist entry, lowered limit) refuses with `ExecutionRefused(ClassificationDriftAtExecution)`
  — never silently executes at the approval-time tier, and never asks the human to re-approve inline (money has
  not moved yet, so refusing and re-queuing for a fresh proposal is always available and always safe).
- **Idempotency propagation.** ADR 0018 F6 pinned the *proposal* dedup key to `(payee, amount, currency,
  time-window)`, deliberately a **sliding**, live-computed comparison (`MoneyMoveProposal.cs:28-37`) — not
  reproducible on a retry, because "now" moves. Plaid's idempotency key must instead be **fixed once** and
  resent identically on every retry of the *same* outbox message. **Decision:** `ExecutionIdempotencyKey =
  SHA-256(ProposalId)`, computed once at `Approve()` and persisted on the proposal. This is stronger than
  re-hashing the F6 tuple: (a) it needs no window bucketing (a `MoneyMoveProposalId` is already the dedup
  boundary — two genuinely distinct pays already cannot share one, because F6's enqueue-time check already
  refused the second as a duplicate before either was ever approved); (b) it is trivially stable across retries
  (same proposal, same ID, same key, every time); (c) it makes the two systems' idempotency windows agree by
  construction — Artha's outbox and Plaid's own dedup now key off the same value, closing the "two systems must
  agree or a partial failure double-executes" hazard named in the brief. **Alternative rejected:** re-derive the
  F6 tuple hash for Plaid too — rejected because it reintroduces exactly the boundary-straddling bucket bug F6's
  own fix (the sliding-window amendment, `MoneyMoveProposal.cs:28-37`) was written to eliminate, this time on
  the side that talks to a real bank rail.

---

## 6. The corroboration predicate (F9 / OQ-11) — a concrete deterministic rule

**Decision: an ingested row is `Corroborated` for actuation purposes when it (a) has persisted, unedited, across
≥ 2 independent `/transactions/sync` cursor advances spanning ≥ a configured minimum window (default 24h), AND
(b) either a second independent source reconciles it OR a household member has explicitly attested to it.**

- **Why persistence alone fails.** Cross-cursor persistence defeats a *one-shot* injected row (a single
  poisoned webhook payload, a MITM'd response that never repeats) — a legitimate row keeps reappearing because
  the real biller keeps billing; a one-shot fabrication does not survive a second honest sync. It does **not**
  defeat a *persistently* compromised connector that keeps returning the same fabricated row on every call —
  clause (b) exists for exactly that residual.
- **Why a fixed "two independent sources always" rule fails.** Most real-world billers in a Plaid-only
  household are single-sourced — no statement import will ever exist to cross-reconcile a Comcast bill nobody
  manually imports. A hard two-source requirement would make pre-fill permanently inert for the common case,
  reintroducing exactly the friction the product exists to remove. **Rejected as the sole rule.**
- **Why persistence + member attestation is the chosen combination.** A member attestation ("yes, this is
  really my bill" — a one-time, explicit UI confirm stamping `MemberCorroboratedAtUtc`, itself an audited
  Saakshi event) is cheap for a genuine biller and *never volunteered* for a fabricated one a human does not
  recognise — the human is the second independent source when no statement exists. Independent-source
  reconciliation (a statement import matching by payee/amount/date-window) satisfies the same clause when it
  *does* exist, without requiring a second attestation. Either satisfies (b); persistence (a) is required
  regardless, so a freshly-injected row cannot be attested away on its very first appearance by a rushed human
  clicking through a confirm dialog.
- **What this unblocks and what it still refuses.** This defines *when* a corroborated row may ever feed a
  proposal pre-fill (closing a tracked corroboration debt); it does not itself build an auto-propose
  path — none exists today (proposals are human-typed; that tracked debt is still open) — and it
  does not relax the bill-prefill debt's confirm-flag/no-one-tap requirement, which still applies on
  top: a corroborated row may pre-fill a proposal's *displayed suggestion*, but the resulting proposal is still
  confirm-flagged and barred from one-tap approval per ADR 0018 F3/§13.3, exactly as already decided.

---

## 7. Failure semantics — ambiguity is terminal until reconciled, never auto-retried

**Decision: `ActuationOutcome.Ambiguous` moves a proposal to a new terminal-ish state, `PendingReconciliation`,
which only a dedicated reconcile job — never the outbox's ordinary retry path — may resolve.**

- **`Refused` (rail declined before any money-moving call — a capability check failed, the connector 4xx'd
  before sending) is safe to retry** through the outbox's normal exactly-once contract; no money could have
  moved.
- **`Ambiguous` (timeout after send, connection drop mid-response, a 5xx that could mean "received and
  processing" as easily as "never received") is different in kind, not degree: money may already have moved.**
  The outbox message is marked `DoNotRetry` — a new, explicit annotation the outbox's normal automatic-retry
  path checks and refuses to act on, closing off even a well-intentioned generic retry mechanism from ever
  re-firing the *same* actuation call on an unresolved ambiguity. The proposal transitions to
  `PendingReconciliation` (a new `ProposalState` member) — not back to `Approved` (which would imply "safe to
  try again"), not to `ExecutionRefused` (which would imply "definitely did not happen").
- **The reconcile job is the only path out.** It queries the rail's own status-by-idempotency-key lookup
  (Plaid's transfer-status endpoint, keyed by the same `ExecutionIdempotencyKey` from §5) to establish ground
  truth. If the rail confirms the transfer happened: `Executed`, stamped with the real `providerTransferId`. If
  the rail confirms it did not: `ExecutionRefused(ReconciledNotExecuted)`, and the household may re-propose from
  scratch (a fresh `MoneyMoveProposal`, fresh F6 dedup, fresh T-tier ceremony — never an automatic resend of the
  ambiguous one). If the rail's own status check is *itself* ambiguous or unreachable, the proposal stays
  `PendingReconciliation` and escalates to a human/admin — it is never silently abandoned and never silently
  retried. **No code path in this ADR ever re-sends an `Ambiguous` actuation automatically — full stop.**

---

## 8. Sandbox-first posture

**Decision: this wave builds and proves entirely against Plaid Sandbox; Production credentials are structurally
unreachable until a separate, later, owner-gated wave flips a config gate.**

- Mirrors the `STEPUP-DEV`/`UnsafeDevStepUpFactorException` precedent already shipped: a `PlaidEnvironment`
  posture check at composition refuses to start the host if a Production-shaped Plaid config (a live
  `client_id`/`secret` pair with `env=production`) is present without an explicit, separately-gated flag —
  the same "the process dies before a socket exists" posture, not a runtime policy check. Sandbox is the only
  environment this ADR's acceptance criteria run against.
- **This does not itself decide the Production DPA/cost/legal questions** — those remain ADR 0018 §13 OQ-1/OQ-2,
  open and unresolved, exactly where they were left. This ADR only ensures nothing built under it can *reach*
  Production before that separate owner call is made.

---

## 9. What stays human-gated forever

Verbatim from the `artha-autonomous-finance-stakes` memory, restated as this ADR's own boundary: **"new-payee
irreversible transfers stay human-gated forever."** Nothing in this ADR builds, or is intended to build toward,
autonomy over a first-time payee or an irreversible external transfer. Every execution this ADR enables is the
rail underneath a proposal a human already typed, T3/T4-classified, and approved through the nonce-bound confirm
(ADR 0007 §7) — the rail moves money a human already said to move; it never decides to move money on its own.
`artha.add_payee`/`trust_payee`/`raise_payee_limit` stay T4 (F1, unchanged); Automation still cannot enqueue,
approve, or execute any money movement (ADR 0018 §6, that tracked debt unchanged by this ADR — a
future standing-rules wave may let Automation *propose*, never approve or execute).

---

## 10. Acceptance criteria / tests

1. **Vault tier isolation.** A pre-login read of any `SANGAM-VAULT` row raises `TierLockedException`, identical
   in shape to Artha's own `SENS` pre-login test (ADR 0018 test #16); a NetArchTest asserts no Sangam token type
   is reachable from `Sarathi`/agent assemblies (extends ADR 0018 test #15).
2. **`act` is structurally absent without the T4 minter.** `ConnectorGrant.WithAct` is the only path to a
   grant carrying `Act`; attempting to actuate through a grant lacking it fails at the Sangam boundary in C#
   (mirrors ADR 0018 test #6). A grant enabling Rail A never implies Rail B is enabled (per-rail test).
3. **Enabling `act` is refused below T4.** `artha.enable_act` below T4 is refused by `ToolRiskGate`; revoking one
   rail's grant does not revoke a sibling rail's grant on the same connector.
4. **Third classification catches execution-time drift.** A proposal approved in-limit whose payee is
   blocklisted before the outbox handler runs is refused `ClassificationDriftAtExecution`, never executed at
   the stale approval-time tier.
5. **No-rail behaviour is unchanged.** A household with no `act` grant still reaches
   `ExecutionRefused(NoRailConfigured)` exactly as today — a planted regression test asserts
   `ApproveAndRefuseExecutionAsync`'s existing call path is untouched when no grant resolves.
6. **Idempotency: retry collapses, distinct proposals never collide.** Two outbox redeliveries of the same
   `Approved` proposal send the identical `ExecutionIdempotencyKey`; two genuinely distinct approved proposals
   never share one (derived from distinct `ProposalId`s, mutation-killed).
7. **Ambiguity never auto-retries.** A planted `Ambiguous` outcome marks its outbox message `DoNotRetry`; a
   negative test asserts the outbox's normal retry sweep skips it (absence + mutation-kill, per CLAUDE.md
   rule #13(c)).
8. **Reconciliation is the only path off `PendingReconciliation`.** A proposal in that state cannot transition
   via any code path except the reconcile job's two terminal outcomes.
9. **Corroboration predicate, both clauses required.** A row corroborated only by persistence (no attestation,
   no second source) is refused for pre-fill; a row attested on first appearance (no persistence) is refused;
   only both together is corroborated (three-way test matrix).
10. **Sandbox-only boots.** A composed host given a Production-shaped Plaid config without the Wave-3 flag
    refuses to start (`UnsafeDevStepUpFactorException`-shaped test).

---

## 11. Open questions (owner decisions needed)

1. **`SANGAM-VAULT` slice — confirm the tier design**, or direct a different custody boundary (e.g. HSM-backed
   if the box gains one before this wave builds).
2. **Corroboration window sizing** — is 24h/2-cycles the right default, or does a real Plaid sync cadence argue
   for a different floor?
3. **Reconcile job cadence and escalation channel** — how often does it poll, and who is paged when the rail's
   own status check is itself unreachable for an extended period?
4. **Sequencing against the standing-rules/Automation-identity wave** (an accepted coordination cost) — if
   that wave lands an Automation *propose* path before this ADR is built, does `ISangamActConnector`'s execution
   handler need an Automation-shaped refusal test on day one, or is a tracked debt's closure a
   hard prerequisite first?
5. **Production DPA/cost/legal** — unchanged, still ADR 0018 §13 OQ-1/OQ-2; this ADR does not touch them.

---

## 12. Consequences

**Positive.**
- **The rail is deterministic end-to-end, same spine as everything else in Artha.** No new security scheme —
  `act` extends the existing `ConnectorGrant` structural-absence pattern; execution rides the existing
  `ToolRiskGate`/outbox/Saakshi machinery a third time, not a fourth mechanism.
- **The no-rail household is provably unaffected.** `ApproveAndRefuseExecutionAsync`'s hardened, adversarially-
  reviewed single-save shape is preserved verbatim as the default path.
- **Ambiguity has an honest terminal state.** "Money may have moved" is never resolved by guessing — it is
  resolved by asking the rail, once, deliberately, never by an automatic resend.

**Negative / trade-offs (honest).**
- **A fifth key slice adds operational surface** — one more DEK to manage, seal, and reason about in a DR
  restore (ADR 0008), beyond `LIFE`/`SENS`/`GEN`/`MTRUST`/`DWARPAL-BOOT`.
- **The corroboration predicate leans on human attestation for single-sourced billers** — a household member who
  rubber-stamps every attestation prompt without reading it degrades clause (b) to a formality. This is a real,
  named residual, not a solved one.
- **`PendingReconciliation` can, in a bad case, sit unresolved** if the rail's own status API is down for an
  extended period — the ADR chooses "stuck and honest" over "guessed and wrong," but "stuck" is a real
  household-facing cost the reconcile-job cadence (OQ-3) must bound.

---

*Promotes the [ADR 0018](0018-artha-household-finance.md) §3 minimal contract to a full design, per that ADR's
own forward cross-link. Depends on [ADR 0002](0002-key-hierarchy-and-unlock.md) (key tiers),
[ADR 0007](0007-risk-tiers-and-step-up.md) (`RiskPolicy`/step-up), [ADR 0006](0006-automation-principal-and-event-bus.md)
(outbox), [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) (consent/forget),
[ADR 0005](0005-audit-pipeline-and-tamper-evidence.md) (audit). Written under an owner ruling.
The orchestrator adds the [README](README.md) index-row status update and
SDD unit rows — this ADR does not touch those shared files beyond the index row required to record its own
existence.*

---

## Mechanism amendment — 2026-08-14, same day as ratification (unit RAIL-1's §1.1 reviews)

**The ratified DIRECTIVE stands unchanged: Sangam's tokens live under their own key material, sealed by
default, never in Artha's store, unsealed only by an explicit ceremony. The MEANS position 1 named do not
exist, and this section replaces them** (the PRECEDENCE-AMEND pattern — the owner's directive is not touched;
the named mechanism is):

- **A fifth `KeyTier` is not a thing this platform can mint.** `KeyTier` is a pinned four-value byte enum
  (`Manas.Crypto/KeyTier.cs:9-13`) with golden tests that go RED on a fifth value — including an explicit
  `0x05` plant — and ADR 0002 §2.3 reserves `0x05+` for SUB-KEYED tiers (keys derived under an existing
  tier's DEK), a deliberately different mechanism. `DWARPAL-BOOT` exists only as ADR-0002 prose (zero code
  hits) and is auto-unseal — the opposite posture — so it was never a precedent.
- **The mechanism is therefore a SUB-KEYED tier under `Sensitive`**: SANGAM-VAULT's DEK is derived under the
  Sensitive tier with the distinct sub-key id `sangam-vault` — separate key material by derivation, and the
  unseal ceremony is the EXISTING admin-login SENS unseal (`Sarathi.Api/Auth`), which is exactly the
  "explicit ceremony, never auto" property position 1 wanted. Accepted coupling, stated: one ceremony
  unseals both SENS and the vault. The alternative (extending `KeyTier`) was rejected because it contradicts
  ADR 0002 §2.3's own reservation and the golden tests exist to make that contradiction loud.
- **The composing host is `Sarathi.Api`** — the platform's one bootable production entry point
  (`ProductionGraph.cs:183`); `AddSangamRuntime` is composed there beside `AddArthaConsoleSurface`. The
  assembly boundary (no Artha↔Sangam reference) is unchanged.
- **Connection cursors are NOT duplicated into the vault**: sync cursors stay where A4 shipped them
  (`SyncCursor`, Artha SENS); the vault holds credentials and connection status only.
