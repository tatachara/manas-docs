# ADR 0005 — Audit Pipeline & Tamper-Evidence

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** [ADR 0001](0001-manas-platform-architecture.md) §5.4 / §14, the Kernel design documentation §4 / §7; resolves design-review **A4**, **D1** (and F4, F8, H4).
- **Amended:** *2026-07-11, round 2 — closes S5/S6.* Adds the security-event path (§2a) and pins ledger `redacted*` payloads to the record owner's shreddable sub-key (§2, §5). See the round-2 design-review notes on S5/S6 + AM ledger item 4.

> **The promise is "every change is audited, and the audit is tamper-evident."** As written that promise is
> *unbuildable*: the Kernel interceptor claims to "write the audit record to Saakshi" **inside the data
> transaction**, but Saakshi is a separate service with its own DB — and service isolation forbids one service
> writing another's store, while a cross-service 2-phase commit on one box is both disallowed and fragile. This
> ADR makes the promise real: **audit is written to the local outbox atomically with the change, delivered
> asynchronously to Saakshi, ordered and deduplicated centrally by Saakshi, and hash-chained over a canonical
> plaintext digest that survives redaction and crypto-shredding.**

---

## 1. Context

Two ADR-0001 guarantees collide with the isolation rule (§5.4: *no service ever reads or writes another
service's store*):

1. **The Kernel design doc's `SaveChanges` interceptor step** says the interceptor, in the same transaction, *writes the
   audit record to Saakshi*. Saakshi owns its own database. A same-transaction cross-service write is
   **impossible** without either sharing Saakshi's DB (breaks isolation) or a distributed 2-phase commit
   (disallowed on one box, and a liveness hazard — if Saakshi is down, no service can commit anything).
2. **§14 "tamper-evident audit"** mandates *append-only + hash-chained*, but leaves the chain's **scope**,
   **ordering**, **canonicalization**, and **plaintext-vs-ciphertext basis** unspecified. You cannot verify a
   chain whose serialization is undefined; and if the digest were computed over the *encrypted/redacted stored
   copy*, then lawful redaction or key-rotation or crypto-shredding (ADR 0009) could be argued to "break
   integrity" — making the two features mutually exclusive.

§14 also loosely promises **"exactly-once"** for the outbox. True exactly-once delivery is not achievable over
an at-least-once transport; this ADR corrects it to the honest, buildable form.

We already have the two primitives we need, from §5.4 / §14 / the Kernel design doc's own outbox section:
- a **transactional outbox** in `Manas.Kernel` (events enqueued in the *same* DB transaction as the change), and
- the **Redis Streams event bus** (durable streams + **consumer groups**, *not* fire-and-forget pub/sub — see
  [ADR 0006](0006-automation-principal-and-event-bus.md)).

This ADR routes audit through those primitives instead of a synchronous cross-service write.

---

## 2. The write path — outbox-in-txn → async → Saakshi

The audit record is treated as **just another outbox event**, written atomically with the data change, then
delivered to Saakshi asynchronously. The emitting service never touches Saakshi's DB.

```mermaid
sequenceDiagram
    participant Svc as Service (e.g. Artha)
    participant Intc as Kernel SaveChanges interceptor
    participant DB as Service DB (own store)
    participant Disp as Outbox dispatcher (post-commit)
    participant Bus as Redis Streams (audit.stream, consumer group)
    participant Sk as Saakshi (the witness, own DB)
    Svc->>Intc: SaveChanges(WalledEntity change)
    Intc->>Intc: stamp ts + Created/ModifiedBy{User,Service}; enforce Tenant/Owner; soft-delete
    Intc->>Intc: compute canonical PLAINTEXT digest of the fact
    Intc->>DB: BEGIN txn
    Intc->>DB: write data change
    Intc->>DB: INSERT audit-event + domain events into OUTBOX (same txn)
    Intc->>DB: COMMIT  %% atomic: change + audit-event together, or neither
    Note over DB: if the outbox INSERT fails, the whole txn fails —<br/>the data change does NOT happen (audit is not optional)
    Disp->>DB: read undispatched outbox rows
    Disp->>Bus: XADD audit event (idempotency key + digest + redacted before→after)
    Disp->>DB: mark dispatched
    Bus->>Sk: consumer-group delivery (at-least-once)
    Sk->>Sk: dedup on idempotency key; assign per-tenant seq; extend hash chain; persist (encrypted, walled)
    Sk-->>Bus: XACK
```

**Key property:** the audit record's *existence* is guaranteed by the **local** commit — it lives in the
emitting service's own outbox, in the same transaction as the change it describes. Delivery to Saakshi is a
**separate, asynchronous** step. Saakshi being down can never block a service's write, and can never *lose* an
audit record — the outbox holds it until Saakshi drains the backlog.

**The audit outbox event** (what the interceptor enqueues) carries:

```
AuditEvent {
  idempotencyKey  // deterministic — see §3.2
  correlationId   // the interaction thread (ADR 0001 §5.4)
  tenantId, ownerId
  actor { userId, serviceId }        // who — propagated + acting service
  occurredAtUtc                      // when (emitter clock; ordering is NOT by this — see §3.1)
  entity { type, id, version }       // what changed
  operation                          // Insert | Update | Delete(soft)
  factDigest                         // canonical plaintext digest, hex (§4.2)
  redactedBefore, redactedAfter      // stored representation: redacted + encrypted under the record OWNER's shreddable sub-key (§2, §5) — never raw finance/health, never bare GEN
}
```

The digest is computed **before** redaction/encryption, over the canonical plaintext (§4). The stored
`redacted*` payloads are what Saakshi persists for human reading; the `factDigest` is what the chain protects.

**Payload encryption (amended 2026-07-11, round 2 — closes S6).** The `redactedBefore/After` payloads are
**encrypted under the record OWNER's shreddable sub-key** (ADR 0002 per-record sub-keys), **not** stored bare at
Saakshi's GEN tier. This is the wiring that makes crypto-shredding reach the ledger itself (§5): destroy the
owner's payload sub-key and *every* ledger copy of their before/after — live store and every backup — becomes
unreadable. The **chain fields** (`factDigest`, `linkHash`, `seq`, `idempotencyKey`) stay **GEN and are never
shredded**, so verification survives the shred. Saakshi's fan-out is registered as the Saakshi row in
[ADR 0009 §2.1 (Saakshi row)](0009-data-lifecycle-retention-forget-consent.md).

**Sarathi events — id + digest only (round-2/round-3).** Events emitted by Sarathi reference conversation
turns by **turn-id + digest only, never transcript content** — the utterance is SENS and a Saakshi payload
(GEN, plus per-owner-shreddable `redacted*`) must not downgrade it a tier. The id/digest indirection keeps a
Sarathi audit or denial event high-signal without copying conversation content into the ledger
(the companion agent-implementation spec §B).

---

## §2a — The security-event path (denials, rejections, classifications)

*(amended 2026-07-11, round 2 — closes S5.)* The §2 SaveChanges-interceptor path covers **writes only**: it
fires when a `WalledEntity` change commits. But the highest-signal events for detecting probing are things that
commit **nothing** — a read that was refused, a token that was rejected, a classification that forced step-up.
Three-plus ADRs promise these land in Saakshi (ADR 0001 §5.4, ADR 0003 §8, ADR 0007 §8 (item 4), ADR 0006 §5/§7); the
write-only interceptor cannot carry them. This section defines the parallel transport.

**What rides this path (a `SecurityEvent` outbox record kind):**

- per-user **wall denials on reads** (a may-see refusal against a specific owner's data);
- **token rejections** (ADR 0003 §8 — expired/invalid/insufficient-assurance delegation tokens);
- **RiskPolicy classifications / step-up outcomes** (ADR 0007 §8 (item 4) — tier decision, challenge issued, approved/
  denied/expired);
- **unsigned-event drops** (ADR 0006 §5 — a bus event that fails signature verification);
- **rule-firing denials** (ADR 0006 §7 — an automation rule refused by its grant/wall).

**How it rides.** The emitting service writes the `SecurityEvent` **to its own outbox in a dedicated
transaction** (not piggybacked on a data change — there is no data change to piggyback on), then it travels the
**identical** dispatcher → Redis Streams → Saakshi path as §2, with the **same idempotency-key scheme** and it
enters the **same per-tenant hash chain**. Security events are ordered, deduplicated, sequenced, and chained
exactly like write-audit events — one ledger, one chain per tenant:

```
idempotencyKey = hash(correlationId | serviceId | eventKind | subjectRef | monotonicSeq)
```

`monotonicSeq` is the emitter's per-`(serviceId, cid)` counter, so two genuinely distinct denials in the same
interaction get distinct keys while a redelivery of one denial collapses to a no-op (§3.2).

**The denial must be DETECTED to be emitted (the honest boundary).** ADR 0004's global query filter is
**silent** — it strips rows a principal may not see, and a filtered-to-empty result is **indistinguishable** from
a genuinely empty one. The filter therefore *cannot* be the denial detector. So:

- For an operation that **targets a specific owner's data** (get-this-record, act-on-this-account), the
  application service evaluates may-see **explicitly**, at the application-service layer, before the operation —
  the `Result.Refused` path. **That explicit evaluation is what emits the `SecurityEvent`** (who / what / `cid` /
  the refused target).
- **Ambient list-queries that merely filter rows emit nothing.** A "show me my dashboard" that silently drops
  another owner's rows is not a denial — it is a scoped read returning its legitimate slice. Emitting a security
  event per filtered row would bury real probing in noise. This boundary is drawn **honestly**: targeted-refusal
  is detected and audited; ambient filtering is not, by design.

**Sarathi security events — id + digest only.** When the emitter is Sarathi, a `SecurityEvent` references
the offending turn by **turn-id + digest only, never transcript content** (conversation content is SENS; Saakshi
payloads must not downgrade it — see §2 and the companion agent-implementation spec §B).

**Loss semantics — fire-and-forget relative to the user, but never silent.** A security event is
**fire-and-forget with respect to the user's operation**: a failed `SecurityEvent` insert **never blocks or
fails the read** (a user must not be denied service because the *record* of a denial could not be written).
Unlike the §2 write path — where an un-auditable change must not commit — the security event trails the
operation. **But the insert failure itself is never silent**: it raises a **Bodha alarm**, so a service that
cannot record its own denials fails loud, not degraded-quiet.

---

## 3. Ordering, idempotency & loss semantics

### 3.1 Ordering — Saakshi assigns the sequence, not the emitter

Records arrive **out of order** (multiple services, multiple partitions, redeliveries). Emitter timestamps are
untrustworthy for ordering (clock skew, and a compromised emitter could backdate). Therefore:

- **Saakshi owns a per-tenant monotonic sequence.** On ingest, Saakshi assigns `(tenantId, seq)` = the next
  integer for that tenant, **at the moment of durable append**, under a per-tenant lock/serialized writer.
- The **hash chain follows that central sequence** (§4.3), not the emitter's `occurredAtUtc`. `occurredAtUtc`
  is retained as descriptive metadata only.
- Consequence: the ledger's order is Saakshi's *ingest* order per tenant, which is well-defined and verifiable.
  Cross-service causal ordering, where it matters, is reconstructed from `correlationId` + `occurredAtUtc`, not
  from the chain position.

### 3.2 Idempotency — deterministic key + idempotent consumer

Redis Streams consumer groups give **at-least-once** delivery; a crash between processing and `XACK` causes a
redelivery. Saakshi must be an **idempotent consumer**:

- Each audit event carries a **deterministic idempotency key**:
  `idempotencyKey = hash(correlationId | serviceId | entityType | entityId | entityVersion | operation)`.
  The same logical change always produces the same key, no matter how many times it is dispatched or delivered.
- Saakshi keeps a **unique index** on `idempotencyKey`. On ingest it attempts an insert; a duplicate key is a
  **no-op** (the event is already in the ledger and chain) and is simply `XACK`ed. A duplicate delivery
  therefore **never double-appends** and never advances the chain twice.

> **Correction to ADR 0001 §14 wording.** §14 says the outbox gives *"exactly-once."* That is not achievable
> over an at-least-once bus. The honest, buildable guarantee is **at-least-once delivery + idempotent
> consumers = *effectively* exactly-once** (no loss, no observable duplication). §14 and the Kernel design doc
> should read this way; ADR 0006 specifies the bus side.

### 3.3 Loss semantics — audit is not optional

- **Saakshi down / bus down:** the audit event stays in the emitting service's outbox (already committed). The
  dispatcher retries with backoff; when Saakshi recovers, the **backlog drains** in order per emitter. No loss.
- **Emitter crash after commit, before dispatch:** the outbox row survives (it was committed); the dispatcher
  re-reads and delivers on restart. No loss.
- **The outbox itself cannot be written** (its INSERT fails inside the txn): the **entire transaction fails**,
  so the data change does **not** happen. For finance/health/security entities, *an un-auditable change must
  not occur* — auditability is a **commit precondition**, not a side effect. (A `WalledEntity` may be flagged
  `AuditOptional` for low-value/high-volume rows; the default is mandatory. See Open questions.)

---

## 4. The hash chain — scope, canonical plaintext digest, central sequencing, anchoring

### 4.1 Scope — per-tenant, owned by Saakshi

- **One chain per `tenantId`.** Cross-tenant isolation is preserved (a chain never mixes tenants), and
  verification is bounded to a tenant's ledger.
- **The chain is computed and stored *inside Saakshi*.** Emitting services **produce records**; they do **not**
  compute chain links. This is what makes central sequencing (§3.1) and anchoring (§4.4) coherent — a single
  serialized writer per tenant owns the head.

### 4.2 What is hashed — the canonical PLAINTEXT digest

The chain protects the **truth of the fact**, not its stored (redacted/encrypted) representation. The
interceptor computes, at write time, a **canonical plaintext digest**:

```
factDigest = SHA-256( canonical( auditedFact ) )
```

where `auditedFact` is the *plaintext* before → after of the audited columns plus the invariant header
(`tenantId, ownerId, actor, entity{type,id,version}, operation, correlationId`), and **`canonical(...)`** is a
**stable serialization**:

- fields emitted in a **fixed, sorted order** (by name);
- **normalized types** — UTF-8 (NFC) strings; integers/decimals in a canonical numeric form (invariant
  culture, no trailing zeros); timestamps as RFC-3339 UTC with fixed precision; booleans lowercase;
  `null` explicit and distinct from absent;
- a **length-prefixed / delimiter-safe** encoding so field boundaries are unambiguous;
- a **schema/version tag** so a column-set change is itself part of the digest.

Because the digest is over the **plaintext**, it does **not** change when the stored copy is redacted,
field-encrypted, or later crypto-shredded. Verification never needs the ciphertext or the plaintext — only the
digest and the chain (§5).

### 4.3 The link function — central sequence

For each tenant, on durable append of record *n*:

```
linkHash[n] = SHA-256( linkHash[n-1] || seq[n] || factDigest[n] || idempotencyKey[n] )
linkHash[0] = SHA-256( genesisTag(tenantId) )
```

`seq[n]` is Saakshi's per-tenant central sequence (§3.1). Each stored ledger row keeps `seq`, `factDigest`,
`idempotencyKey`, `prevHash = linkHash[n-1]`, and `linkHash[n]`. Any retroactive edit, insertion, deletion, or
reordering breaks `linkHash` from that point forward and is detectable by re-derivation (§5).

### 4.4 External anchoring — defeat the box-owner/root

A runtime `root` on the home server could, in principle, rewrite the whole ledger *and* recompute every `linkHash`
to hide it. Internal chaining alone cannot stop an attacker who owns the store. So Saakshi **periodically
publishes the chain HEAD** (`(tenantId, seq, linkHash)` — a hash only, zero sensitive content):

- to **WORM / append-only offline storage** on the box (write-once medium or an OS-enforced append-only,
  root-restricted location), and
- **optionally** off-box via the **blind relay** (ADR 0001 §2 control-plane — it sees an opaque hash, never
  data),
- on a schedule and on demand (e.g. before backups, after high-risk actions).

Because a past head is anchored where the runtime root cannot silently alter it, **any retroactive edit is
detectable**: the recomputed chain will not reproduce a previously anchored head. Anchors are themselves
signed with a key held by `Manas.Crypto` (ADR 0001 §15).

---

## 5. Crypto-shredding compatibility (ADR 0009)

Right-to-forget is implemented by **crypto-shredding**: destroy a record's field-encryption key so the stored
ciphertext becomes permanently unreadable (see [ADR 0009](0009-data-lifecycle-retention-forget-consent.md)). This composes
cleanly with tamper-evidence **precisely because the chain hashes the plaintext *digest*, not the plaintext and
not the ciphertext**:

- The ledger stores: `factDigest` (a one-way hash — reveals nothing), the chain hashes, and the
  `redactedBefore/After` payload **encrypted under the record OWNER's shreddable sub-key** (§2; ADR 0002
  per-record sub-keys), **not** bare GEN.
- **Forget destroys the owner's payload sub-keys → every ledger copy of their before/after becomes unreadable,
  in the live store and every backup, while `factDigest`/`linkHash` survive and the chain still verifies.** This
  is the wiring that closes S6: the earlier "the readable copy is gone forever" was true for a service's own
  store but unsupported *for the ledger itself* until the payloads were pinned to the owner's shreddable key.
  The forget fan-out registers Saakshi as the
  [ADR 0009 §2.1 (Saakshi row)](0009-data-lifecycle-retention-forget-consent.md) — shred the owner's ledger
  payload sub-key, leave the digest/chain intact.
- **Chain verification still passes** end-to-end after a shred, because re-derivation only needs the stored
  `factDigest` + sequence + prev link — none of which required the (now-unreadable) plaintext.
- Net effect: **the readable data is destroyed, yet the ledger remains provably un-tampered.** "We forgot X"
  and "our audit trail is intact" are simultaneously true — the two ADR-0001 promises stop being in tension.

(One-way exposure caveat: a digest is not reversible, but a *low-entropy* audited value could be brute-forced
from its digest. For crown-jewel fields we salt the digest with a per-record secret that is itself
crypto-shreddable, so forget also removes brute-force reconstructability. Tracked in Open questions.)

---

## 6. Idempotency for user actions (D1)

The same at-least-once reality applies to **user-initiated actuations**, not just audit events: a retried
"disarm", a re-submitted payment, a double-tapped device command must **not double-execute**. This reuses the
outbox/idempotent-consumer machinery on the *command* side.

- **Request-id derivation.** The **client** mints a stable **request-id** for a logical intent and **reuses it
  across retries of the *same* intent** (network retry, app relaunch, user re-tap within a debounce window).
  For voice/Sarathi turns the id is derived from `correlationId + intent-signature`, so a re-heard identical
  command maps to the same id; a genuinely new command gets a new id.
- **Consumer-side dedup.** Every actuation tool (Artha trade, Kavach arm/disarm, Kaya device command,
  notification send) records the request-id in an **idempotency table** *in the same transaction as the effect*.
  A repeat request-id returns the **stored prior result** instead of re-acting — deterministic, at the tool
  boundary, below the LLM (the model can retry freely; it cannot cause a double-effect).
- **Relation to audit:** the actuation's audit event uses the audit idempotency key (§3.2); the actuation's
  *effect* uses the user request-id. Both dedup independently, so neither the act nor its record is duplicated.

This closes design-review **D1** and completes §14's "idempotency + outbox → no double-execution on retry."

---

## 7. Acceptance criteria / tests

Following the 10× mandate (ADR 0001 §8). All are deterministic and buildable now.

1. **Atomic audit.** A committed `WalledEntity` change has exactly one corresponding outbox audit row in the
   *same* service DB, written in the *same* transaction (assert via a fault injected between change and audit
   INSERT → both roll back; the change does not persist).
2. **No cross-service DB write.** Static/integration assertion: no emitting service holds a Saakshi
   connection; audit reaches Saakshi **only** via the bus.
3. **Recovery from outage.** With Saakshi down, perform N audited changes; bring Saakshi up; the ledger
   contains all N (an audit "lost" during the outage is recovered from the outbox). No loss.
4. **Duplicate delivery does not double-append.** Deliver the same audit event twice (force a redelivery); the
   ledger appends **once**; the hash chain advances **once**.
5. **Central ordering.** Emit audit events out of order / from multiple services; Saakshi assigns a gap-free
   per-tenant `seq`; the chain follows `seq`, not emitter timestamps.
6. **Chain tamper detection.** Mutate / insert / delete / reorder a persisted ledger row → full-chain
   re-derivation fails at that row.
7. **Anchor detection.** Rewrite the ledger *and* recompute all `linkHash` (simulate a malicious root); the
   recomputed head **does not match** a previously anchored head → tamper detected.
8. **Crypto-shred compatibility (the headline).** Crypto-shred a record (destroy its key), then run full chain
   verification → **verification passes**; the record's readable payload is gone, `factDigest`/chain intact.
9. **User-action idempotency.** Submit the same disarm/payment/device command twice with the same request-id →
   the effect happens **once**; the second returns the stored result; exactly one actuation audit row.
10. **Canonicalization stability.** The same fact serialized on two runs / cultures / field orderings yields
    the identical `factDigest`; a single-field change yields a different one.
11. **Security-event on targeted denial, none on empty (S5).** A **wall-denied targeted read** produces
    **exactly one** hash-chained `SecurityEvent` in Saakshi carrying the denial's who / what / `cid`; a **merely
    empty** result of the same shape produces **none** (the ambient-filter boundary of §2a holds).
12. **Ledger payload shredded, chain intact (S6).** After **crypto-shredding an owner**, their ledger
    `redactedBefore/After` payloads are **unreadable** (owner sub-key destroyed) while **full-chain verification
    passes**. *(Supersedes/strengthens test 8, which shredded a record but never asserted the ledger's own
    before/after payloads were unreadable.)*
13. **Security-event failure never fails the read, but alarms (S5).** A **failed security-event emission** never
    fails or blocks the user's read; the emission failure raises a **Bodha alarm** (fail-loud, not
    silently-degraded).

---

## 8. Open questions

- **`AuditOptional` policy.** Which (if any) high-volume, low-value entities may downgrade audit from a commit
  precondition to best-effort? Default is **mandatory**; finance/health/security are always mandatory.
- **Digest salting for low-entropy fields — resolved (round 2).** The per-record shreddable salt (§5 caveat)
  **is the same owner sub-key mechanism** now pinned in §2/§5: the salt is derived under the record owner's
  shreddable sub-key, so destroying that key on forget removes brute-force reconstructability of a low-entropy
  digested value at the same instant it makes the `redacted*` payload unreadable — one key, one shred, both
  properties. (The remaining tuning question — salt *every* digested field vs. only `[Encrypted]` crown-jewel
  columns — stays a `RetentionPolicy` knob, not a mechanism gap.) **The `AuditOptional` question above is
  unaffected** by this resolution.
- **Anchor cadence & medium.** How often to anchor, and the exact WORM medium on the home server (append-only ZFS
  dataset vs. external write-once) + whether to also anchor via the blind relay by default.
- **Backfill/genesis for existing rows.** Vatara-era rows predate the chain; define a one-time genesis anchor.
- **Retention vs. chain.** Bounded retention (§5.4) prunes *payloads*; the **chain of digests must never be
  pruned** or verification breaks. Confirm digests + links are retention-exempt (like the spend ledger).

---

## 9. Consequences

**Positive.**
- The core guarantee becomes **buildable**: audit is atomic with the change (never lost) yet respects service
  isolation (no cross-service DB write, no distributed 2PC).
- **Tamper-evidence is now precisely specified** — per-tenant scope, canonical plaintext digest, central
  sequencing, external anchoring — so it can be *implemented and verified*, not just asserted.
- **Redaction, encryption, key-rotation, and crypto-shredding all coexist with integrity**, because the chain
  hashes the plaintext *digest*. "Right-to-forget" and "provably un-altered audit" stop contradicting.
- The honest **at-least-once + idempotent-consumer** model is robust and reuses ADR 0006's bus; the same
  machinery closes D1 for user actions.

**Negative / trade-offs.**
- Audit is now **eventually** in Saakshi, not instantly — a just-committed change may lag by the dispatch
  interval before it is queryable in the ledger (the *record* is never at risk, only its visibility latency).
- Saakshi is a **serialized per-tenant writer** (central sequence + chain), a throughput ceiling per tenant —
  acceptable for a household; revisit only at product scale.
- An audit-mandatory commit precondition means **a broken outbox blocks writes** to audited entities. This is
  intentional for finance/health, but must be monitored (Bodha/Saakshi self-health) so it fails loud, not
  silently-degraded.
- More moving parts (dispatcher, consumer dedup, anchoring job) — but all reuse existing Kernel + ADR 0006
  primitives; nothing here is a new bespoke transport.

---

**See also:** [ADR 0001](0001-manas-platform-architecture.md) §5.4 / §14 / §15 · the Kernel design documentation §4 / §7 · [ADR 0006 — Event bus & interconnection](0006-automation-principal-and-event-bus.md) · [ADR 0009 — Consent & right-to-forget](0009-data-lifecycle-retention-forget-consent.md).
