# Manas.Kernel — Foundational Spec

- **Status:** Design (sketches, not code) · **Date:** 2026-07-11 · with
  [ADR 0001](../adr/0001-manas-platform-architecture.md) §5.4/§14/§15.

> This spec was reconciled to ADRs 0002–0010 on 2026-07-11 (round-2 S11); where this doc and an ADR disagree,
> the ADR wins.

> `Manas.Kernel` is the most load-bearing library: **walling, audit, attribution, soft-delete, encryption, and
> identity all live here ONCE**, and every service inherits them identically — so a developer *cannot* forget
> the wall, the audit, or the attribution. Get this right and the platform's hardest guarantees are free for
> every service. This is the spec; implementation follows the walking-skeleton phase (TDD, 10×).

## 1. Identity & the ambient actor

Everything is scoped by **who is acting**. One ambient context flows through every operation (set at the
service edge from the verified token + service identity, never guessable):

```
ActorContext            // ambient, per-operation, immutable
  TenantId              // which household (the tenant seam — home/member-agnostic)
  UserId                // the verified acting HUMAN (propagated across services)
  ServiceId             // the acting SERVICE (its own identity — for attribution)
  Roles                 // e.g. Admin / Adult / Child
  CorrelationId         // threads the whole interaction through Saakshi
```

Strongly-typed ids (`TenantId`, `UserId`, `ServiceId`) — value objects, never bare `Guid`/`string`, so they
can't be transposed. *(corrected 2026-07-11, round 2 — see ADR 0003)* `ActorContext` is **derived**, not
carried: each callee independently validates the inbound delegation token (signature, audience, expiry,
delegation legitimacy, replay/revocation — ADR 0003 §3) and only then *constructs* its own `ActorContext` from
the verified claims. A hand-built `ActorContext` — from a request body, an unverified header, or a value passed
by the caller — is a **banned pattern**; it is never read for an authorization decision. Tokens also carry
`acr`/`amr` assurance claims (hint-grade vs verified) so a callee can distinguish a room-hint mic match from a
Face-ID-backed session; an unauthenticated/unclaimed speaker resolves to the ephemeral **Guest** principal
(T0-only by default). Sarathi (the Agent) supplies the *caller's* actor, never a super-user (ADR §5.2).

### 1.1 The correlation invariant *(normative — Ananth, 2026-07-11)*

**Exactly ONE `CorrelationId` is minted per initiated action, by whichever system initiates it; every downstream
artifact carries it, unchanged, to the end of the causal chain.** This is what makes any action traceable
end-to-end across audits, DB changes, logs, traces, tool calls, and notifications.

**Initiation points (the ONLY places a CorrelationId is minted):**
1. **User interaction** — the channel ingress adapter, stage 1 of the Sarathi turn spine (ADR 0010 §2 / agent-spec §B),
   for all three paths (console text, console voice, satellite) and GraphQL/UI requests at the edge.
2. **Scheduled/cron triggers** — each Quartz job RUN mints one (e.g. one Ritu sweep = one cid).
3. **Background service cycles** — a self-initiated background pass (cert rotation check, backup run,
   dead-man beacon fault) mints one per cycle.
4. **External events** — the inbound boundary adapter mints on arrival when nothing upstream carries one:
   HA/MQTT device events, Frigate detections, webhooks, APNs feedback, HealthKit sync batches.

**Propagation (never re-mint mid-chain):** downstream components INHERIT, never re-mint — the id travels in
`ActorContext.CorrelationId`, the delegation token's `cid` claim (ADR 0003 §2), the bus event envelope
(ADR 0006 §5), and outbox rows. An **ECA rule firing inherits its trigger event's cid** (ADR 0006 §7) — the
whole trigger → condition → action → step-up → outcome chain is one cid. A consumer that receives any carrier
bearing a cid MUST use it; minting with a carrier present is a defect (unit tests assert this).

**Capture (everything records it):** Saakshi structured logs, the interaction ledger, audit events + security
events (ADR 0005 — the cid is inside the hash-chained record AND the idempotency key), LLM traces + spend
ledger entries, actuation/idempotency records, notification deliveries, and OpenTelemetry spans (the cid rides
as a span attribute/baggage alongside the W3C trace id — the OTel trace id does not replace the cid; the cid is
the business-level key a human queries by).

## 2. The base entity (every user-scoped row derives from this)

```
WalledEntity<TId>
  Id            : TId
  TenantId      : TenantId          // tenant scope
  OwnerId       : UserId            // the user who owns this row  → the per-user wall
  CreatedAtUtc / ModifiedAtUtc
  CreatedByUser / CreatedByService  // provenance (WHICH user + WHICH service created it)
  ModifiedByUser / ModifiedByService// provenance (last change)
  IsDeleted / DeletedAtUtc / DeletedBy(User,Service)   // soft-delete tombstone (right-to-forget + audit)
  RowVersion    : byte[]            // optimistic concurrency
  DataSubjectIds: UserId[]          // *(corrected 2026-07-11, round 2 — see ADR 0010 §3)* who this row is ABOUT
                                     // (may differ from OwnerId, e.g. a guardian's turn about a Child); drives
                                     // forget fan-out alongside OwnerId
```

Rules: user-scoped data **must** derive from `WalledEntity`; platform/config/reference data that is not
user-owned uses a lighter `TenantEntity` (tenant-scoped, no OwnerId). Nothing user-owned escapes the wall.
*(corrected 2026-07-11, round 2 — see ADR 0002 §3 / ADR 0009 §2)* Fields may be encrypted under a **per-owner or
per-record sub-key** derived beneath the entity's tier DEK, not just the coarse tier key — this is what lets
right-to-forget crypto-shred one owner's/record's data without touching anyone else's.

## 3. Walling — the automatic per-user filter (can't be forgotten)

An **EF Core global query filter** is applied to **every** `WalledEntity` type at model-build:

```
tenant match:  TenantId == actor.TenantId
owner match:   OwnerId == actor.UserId  OR  actor may-see(OwnerId)     // deterministic role rule
not deleted:   IsDeleted == false                                       // tombstones hidden by default
```

- `may-see` is a **deterministic** predicate. *(corrected 2026-07-11, round 2 — see ADR 0004 §3/§4)* **The Admin
  role is NOT an input.** A guardian sees a Child's rows only via an explicit `guardianOf` edge in Kutumba
  — data, not a role bit — so two Admins (e.g. both parents) still cannot read each other; adults are walled
  from each other unless explicitly shared. Machine principals (rules/services) get no human-row access at all;
  they see a row only under an active, audited machine-grant (full 10-row truth table incl. row 10 in
  [ADR 0004](../adr/0004-walling-maysee-and-sharing.md) §3). Encoded in code, **never the LLM's judgment**
  (threat model §3, cross-user leak).
- Because it's applied to *all* `WalledEntity` types centrally, a new entity is walled the moment it exists — a
  developer can't ship an unwalled query. Raw SQL / Cypher paths get the same predicate via repository helpers.

## 4. Audit + attribution — the SaveChanges interceptor (automatic)

On every `SaveChanges`, an interceptor processes each added/modified/deleted `WalledEntity`:
1. **Stamp** timestamps + `Created/ModifiedBy{User,Service}` from the `ActorContext` (this is "which service
   updated which data point" — ADR §5.4).
2. **Enforce** `TenantId`/`OwnerId` consistency with the actor (defense in depth — a write must be for the
   acting tenant + an allowed owner; reject otherwise).
3. **Soft-delete**: convert a delete of a tombstone-policy entity into `IsDeleted = true` + `DeletedBy*`.
4. *(corrected 2026-07-11, round 2 — see ADR 0005 §2/§3.2)* **Compute the canonical plaintext digest and enqueue
   the audit event to the service's own local outbox, in the same transaction as the data change** — the
   interceptor never writes to Saakshi directly. Delivery to Saakshi is a separate, asynchronous, at-least-once
   bus hop; Saakshi is the sole owner of central sequencing and the hash chain, and dedups on a deterministic
   idempotency key. If the outbox write fails, the whole transaction fails — an un-auditable change never lands.
5. **Collect domain events** raised by the entities for post-commit dispatch (outbox → the interconnection bus).
6. *(corrected 2026-07-11, round 2 — see ADR 0005 §2a)* **Security events** — read denials, token rejections,
   step-up outcomes — are not `SaveChanges` side effects; each service writes them to its own outbox in their
   own transaction via the same security-event outbox path, so denials (not just writes) reach Saakshi's ledger.

All of this is inherited — no service writes audit/attribution/wall code itself.

## 5. Field-level encryption convention

- `[Encrypted]` marks a sensitive property (account number, health value, OAuth token).
- A central EF `ValueConverter` (backed by **Manas.Crypto**, ADR §15) encrypts/decrypts marked properties
  transparently at the DB boundary → the row is ciphertext at rest; the app sees plaintext only when the box is
  **unlocked**.
- Registered once for all services → identical crypto everywhere; key rotation via the `keyVersion` tag.
- *(corrected 2026-07-11, round 2 — see ADR 0002 §4)* **KeyTier resolution**, in order: an explicit `KeyTier` on
  the `[Encrypted]` attribute wins if present → else the property's `ITierMap` entry → else the owning service's
  registered default tier → **fail closed at startup** (an unmapped property is a boot failure, never a silent
  default key).

## 6. Result & errors

- `Result<T>` / `Result` model expected outcomes: `Ok(value)` · `Validation(errors)` · `NotFound` · `Refused`
  · `Conflict`. **No control flow by exception**; exceptions are for the genuinely exceptional (a down
  dependency), caught at the edge and degraded gracefully (F5-style).

## 7. Domain events & the outbox

- `WalledEntity` can raise **domain events**; the interceptor enqueues them to a **transactional outbox**
  (same DB transaction as the data change) → a dispatcher publishes them to the **Redis event bus**
  post-commit → the interconnection/automation engine reacts. *(corrected 2026-07-11, round 2 — see ADR 0005
  §3.2)* Redis Streams consumer groups give **at-least-once** delivery, not exactly-once; **at-least-once
  delivery + idempotent consumers = effectively exactly-once** (no loss, no observable duplication) — every
  consumer, including Saakshi, dedups on a deterministic idempotency key.
- *(corrected 2026-07-11, round 2 — see ADR 0006 §3)* Every domain event/outbox row carries a `Provenance` and a
  derived `Taint` (`Trusted` | `Tainted`) field. Taint is sticky/transitive across derivation and gates
  unattended actuation downstream (Tainted facts require human step-up, never auto-fire) — it is never dropped
  or laundered by the outbox or the bus envelope.

## 8. What Kernel deliberately does NOT contain

No domain logic, no EF provider specifics, no LLM, no transport. It is pure primitives + the cross-cutting
behaviors above, so it's tiny, dependency-light, and exhaustively unit-testable (the first thing built + the
most heavily tested — it's where the platform's guarantees live).
