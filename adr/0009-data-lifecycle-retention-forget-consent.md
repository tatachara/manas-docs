# ADR 0009 — Data Lifecycle: Retention, Right-to-Forget & Consent

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** [ADR 0001](0001-manas-platform-architecture.md) §14 / driver #7, the Kernel design documentation §2 (`WalledEntity` tombstone), [ADR 0002](0002-key-hierarchy-and-unlock.md) (crypto-shred hook), [ADR 0005](0005-audit-pipeline-and-tamper-evidence.md) (audit chain over plaintext digest); resolves design-review **C9**, **D10**, **D12**.

> **Three ADR-0001 promises are, as written, mutually exclusive.** Driver #7 says *"keep everything forever,
> append-only / event-sourced."* §14 says *"right-to-forget is designed into the data model."* ADRs 0005/0008
> say the audit ledger is append-only + hash-chained and backups are immutable off-site ZFS snapshots. You
> cannot hard-delete a person's data from an append-only chain, an immutable snapshot, or the lossy embedding
> that Qdrant already derived from it. This ADR resolves the contradiction with **one mechanism — crypto-
> shredding** — plus a **per-data-class retention matrix** and a **bystander/guest privacy floor**, so that
> "forget" is real, growth is bounded, and non-members are never profiled.

---

## 1. Context — the contradiction (C9)

Four locked decisions collide:

1. **Driver #7 — keep forever, append-only / event-sourced.** Historical trend analysis (health vitals,
   finance) *needs* the past. Facts are not overwritten; they accrue.
2. **§14 — consent per source + soft-delete/tombstone + export + right-to-forget in the data model.** A member
   (or the law) can demand erasure of *their* data.
3. **[ADR 0005] — the audit chain is append-only and hash-chained**, and (crucially) it hashes a **canonical
   plaintext *digest*, not the plaintext and not the ciphertext**. You must never mutate it.
4. **[ADR 0008] — backups are immutable, client-side-encrypted ZFS snapshots, off-site.** By design you cannot
   reach into a snapshot to surgically delete a row.

A naïve "DELETE the rows" forget therefore **cannot** work: it would either break the audit chain, be
impossible against immutable snapshots, or leave the data alive inside Qdrant (an embedding is a lossy but
real copy of the source). And driver #7's "keep every byte forever" is itself in tension with data-
minimization — a household does not need to retain raw microphone audio or a passer-by's face indefinitely.

We already hold the primitives that dissolve this: **per-tier DEKs with a crypto-shred hook** (ADR 0002 —
per-owner / per-record sub-keys), the **`WalledEntity` soft-delete tombstone** (the Kernel design doc §2), and the
**plaintext-digest audit chain** (ADR 0005). This ADR wires them into a lifecycle.

---

## 2. Crypto-shredding as the forget mechanism (C9)

**Principle: forget by destroying keys, not by mutating append-only or immutable stores.** Every owner has a
**per-owner sub-key** derived under the relevant **tier DEK** (ADR 0002); where finer granularity is needed
(a single health record, one journal entry, one source's import), a **per-record / per-record-class sub-key**
is derived under the owner sub-key. All sensitive fields and blobs are encrypted under the leaf sub-key.

> **"Forget X" = destroy X's sub-key.** Every ciphertext encrypted under it — in Postgres, MinIO, and *every
> backup snapshot that ever captured it* — becomes permanently unreadable, **without writing to any append-only
> store or reaching into any immutable snapshot.** The bytes remain; they are now indistinguishable from noise.

### 2.1 Per-store propagation

Forget fans out to each store by that store's nature. Derived data (embeddings, graph) is **deleted**;
key-protected data (fields, blobs) is **shredded**; the audit chain is **left intact by design**.

| Store | What lives there | Forget action | Why |
|---|---|---|---|
| **Postgres** (SoR) | Field-encrypted columns on `WalledEntity` rows | **Shred** the owner/record sub-key → columns unreadable; **set the tombstone** (`IsDeleted/DeletedAt/DeletedBy`) so queries skip the row. Row is *not* hard-deleted (would break event-sourced history / FK integrity). | Ciphertext under a destroyed key is unrecoverable; the tombstone hides it from live reads; append-only history is preserved. |
| **Qdrant** (embeddings) | Vectors derived from source text — a **lossy copy**, and a vector is *not* field-encrypted the way a column is | **Hard-DELETE** the vectors (point-delete by `ownerId` / `sourceId` payload filter). Vectors are **owner-tenant-scoped** in payload so the delete is exact. | An embedding can leak the source via similarity/inversion; key-shredding a raw float vector protects nothing. It is *derived*, so deletion loses no SoR truth — it is rebuildable from surviving SoR facts if needed. **Chosen over "encrypt-then-shred vectors"** because ANN search needs plaintext vectors to be useful; encrypted vectors defeat the index. Delete is the honest option. |
| **MinIO** (blobs — PDFs, camera clips, HealthKit exports) | Objects encrypted under the leaf sub-key (SSE-C / envelope) | **Shred** the sub-key (object becomes unreadable) **and** issue the object delete for space reclaim. Shred is the *guarantee*; delete is housekeeping. | Even if a delete is delayed or a snapshot retains the object, the shredded key makes it unreadable. |
| **Neo4j** (graph) | Nodes/edges + a few plaintext structural properties | **Shred** encrypted properties; **DELETE** the node/edges that are personal to the owner (structure is not append-only here). | Graph is a projection, not the SoR; deleting the owner's subgraph is safe and removes relationship inference. |
| **Backups / ZFS snapshots** (ADR 0008) | Immutable, client-side-encrypted copies of all of the above | **Nothing to do in the backup** — and that is the point. **The key never lived in the backup; it lived in the box.** Shredding the sub-key on the box renders every snapshot copy of that ciphertext unreadable too. | You cannot (and must not) mutate an immutable snapshot. Key-custody-in-the-box is exactly what makes offline/off-site copies forgettable. |
| **Redis** (cache) | Transient projections | TTL-expire + explicit `DEL` on the owner's keyspace. | Ephemeral; nothing durable is lost. |
| **Saakshi ledger** *(amended 2026-07-11, round 2 — closes S6)* | `redactedBefore/After` payloads (owner-sub-key encrypted — [ADR 0005 §2](0005-audit-pipeline-and-tamper-evidence.md) (owner-sub-key payload encryption)) | **Shred** the owner's payload sub-keys; digests/links/seq untouched | The chain survives (hashes only); the readable copies die everywhere including backups. |

### 2.2 The audit chain survives — the key reconciliation

Because [ADR 0005] hashes a **canonical plaintext *digest*** (a one-way hash of the fact), **not** the
plaintext and **not** the ciphertext, crypto-shredding a record removes the *readable* data while leaving the
digest and every `linkHash` untouched. Full-chain re-derivation still passes end-to-end after a shred. So
**"we forgot X" and "our audit trail is provably un-tampered" are simultaneously true** — the two ADR-0001
promises stop contradicting. (Low-entropy digest brute-force is mitigated by the shreddable per-record salt;
ADR 0005 §5.)

### 2.3 Consent + forget are themselves audited

Every consent grant, consent revocation, and forget execution is a `WalledEntity` change and therefore emits
its own audit event (ADR 0005): **who forgot / revoked what, for which owner and source, when, and which
stores were fanned out to** (with per-store completion status). Forget is not a silent side-channel; it is a
first-class, tamper-evident act. The forget record is retention-exempt (like the chain).

---

## 3. Retention matrix (D10)

Driver #7 is refined to: **keep the SoR *facts* for historical analysis; prune transient, raw, and derived
data aggressively.** Historical value ≠ retaining every raw byte forever. Tiers: **hot** (live queryable in
Postgres/Redis), **warm** (compacted / down-sampled / cold object storage), **cold** (ZFS snapshot only, not
live), **purge** (crypto-shred + delete).

| Data class | Owner service | Hot | Warm | Purge policy |
|---|---|---|---|---|
| **Raw voice audio** (mic satellites) | Kavach / voice | **≈ 0** — held only in-RAM for the STT turn | none | **Dropped immediately after transcription.** Never written to disk. Keep redacted **text** only. |
| **Voice transcripts** (post wake+intent) — *superseded for Sarathi turns; see the Sarathi conversation turns row below* | per-domain | 90 days | down-sample to intent summaries | Purge raw transcript after warm; keep derived structured facts. Retained only for any non-Sarathi voice paths. |
| **Sarathi conversation turns (all surfaces: console text, console voice, satellite)** *(amended 2026-07-11, round 2 — closes G3)* | Sarathi service (per-member walled, SENS) | 90 days hot | down-sample to intent summaries + extracted MemoryFacts (derived, provenance-linked) | purge raw; subject-aware: rows naming a forgotten member in `dataSubjectIds` shred with THAT member ([ADR 0010 §3](0010-agent-conversational-core.md)). |
| **Camera clips** (Frigate) | Kavach | 7–30 days (config) | flagged/event clips → 90 days | Unflagged clips **auto-purge** on the rolling window; only security-relevant events promote. |
| **Security events** (arm/disarm, alarms, entries) | Kavach | 1 year | 7 years (compacted) | Long — safety + dispute history. SoR facts retained. |
| **Finance transactions** (Plaid) | Artha | 2 years | 7 years | **7 years** (tax/audit statute), then purge. Balances/trends kept as aggregates. |
| **Health vitals** (Apple Health / HealthKit) | Ojas | 2 years | lifetime aggregates | Long — trend analysis is the point. Raw high-frequency samples down-sampled to daily/weekly after warm. |
| **Child PHI + wellness journals** | Ojas | live | encrypted, per-record sub-key | Long **but** strictest access + first to shred on forget; journal entries are per-record shreddable. |
| **School records** (Abhyasa) | Abhyasa | school year | graduation + N years | Purge on policy after retention window. |
| **LLM traces / logs** (Bodha) | Bodha | 30 days | 90 days aggregates | **Bounded** — prune raw traces at 30d; keep cost/quality aggregates. Never keep forever. |
| **Embeddings** (Qdrant) | per-domain | live | — | **Tied to source lifecycle.** When the source row is purged/forgotten, its vectors are deleted in the same job. |
| **Audit chain (digests + links)** | Saakshi | live | live | **Never pruned** (verification would break). Payloads *around* the digest may be shredded; the digest/link cannot. |
| **Consent + forget records** | Kernel/Saakshi | live | live | Retention-exempt. |
| **Guest / bystander data** | see §5 | shortest | — | Shortest window of its class; **never** promoted to a profile. |

### 3.1 The scheduled retention/purge job

A **Quartz.NET** scheduled job (already in the stack; Hangfire is the alternative) — call it the **Ritu**
("season") sweeper — runs nightly per service:

1. Selects rows/objects/vectors past their class's warm/purge boundary (policy is **data, not code** — a
   `RetentionPolicy` table keyed by data class, editable without redeploy).
2. **Warm**: down-samples / compacts / moves blobs to cold object storage.
3. **Purge**: crypto-shreds the leaf sub-key **and** issues the store deletes (Postgres tombstone, Qdrant
   point-delete, MinIO object-delete, Neo4j node-delete) — the *same* fan-out as §2.1.
4. Emits an audit event per purge batch (what class, how many, cutoff date).
5. **Budget the DB growth.** The sweeper reports per-class row/byte counts to Bodha; unbounded growth in any
   *append-forever* class (security, finance, health) is a budgeted, monitored line item, not a surprise.

Purge is **idempotent** (re-running over an already-shredded record is a no-op) and respects the tombstone.

---

## 4. Reconciling "keep forever" with minimization

Driver #7 stands **for SoR facts** — the events that carry historical/analytical value (a transaction, a vital
reading, a security event). It does **not** license hoarding **raw/derived/transient** data:

- **Raw** (microphone audio, full-res camera frames): near-zero retention — transcribe/detect, then drop.
- **Derived** (embeddings, graph projections, caches): rebuildable, tied to source lifecycle — never a
  reason to *retain* beyond the source.
- **Transient** (LLM traces, debug logs): bounded windows; keep aggregates, prune raw.
- **SoR facts**: kept per the matrix (years-to-long), down-sampled when the analytical value is in the trend,
  not the raw sample rate.

So "keep forever" and "minimize" coexist: **we keep the *meaning* long, and the *bytes* only as long as they
carry meaning.**

---

## 5. Bystander / guest privacy (D12)

Always-on mics and cameras capture **non-members** who never consented. The floor:

- **Mics — process only after wake + intent.** Audio is held in RAM for the turn, transcribed, and the **raw
  audio is dropped immediately** (§3). Speech before/without a wake event is never persisted. A **non-member
  speaker is never enrolled or profiled** — speaker-ID that fails to match an enrolled member yields
  `guest`, and a `guest` turn writes **no** profile, embedding, or long-term memory.
- **Cameras — bounded clip retention + consent-law setting.** Unflagged clips auto-purge on a days-to-weeks
  rolling window (§3). Recording-consent law is **jurisdiction-dependent** (one-party vs. all/two-party
  consent varies by state/country). Manas therefore exposes a **documented `RecordingConsentMode` setting**
  (default conservative) plus **guest-notice guidance** (a visible/audible "recording in progress" affordance
  in shared spaces). **For a household deployment this is a config default; for a PRODUCT it needs legal
  review — flagged in Open questions.**
- **Non-member data gets the shortest retention of its class and is never used to build a profile.** A guest's
  transcribed utterance or a passer-by's camera frame lives only as long as the shortest window of that data
  class and is excluded from all learning/personalization paths.
- **Tie to consent (§6):** members' capture is consented per source; bystanders cannot consent, so the design
  *minimizes their footprint by default* rather than relying on a consent that can't be obtained.

---

## 6. Consent per source, revocation → forget (folds in ADR 0001 §14)

- **Recorded consent per source.** Each external source — **Apple Health (HealthKit), Plaid (finance),
  email (Sangam), cameras, mics** — has an explicit, timestamped consent record (owner, source, scope,
  grantedAt, grantedBy), stored as an audited `WalledEntity`. No source is ingested without a live consent.
- **Sarathi conversation memory** *(amended 2026-07-11, round 2 — closes R2 consent)* — a per-member consent
  record gating the `MemoryFact` extraction pipeline ([ADR 0010 §3](0010-agent-conversational-core.md),
  the companion agent-implementation spec §C). Turns and summaries persist under the retention row
  above regardless; this consent governs only whether raw turns are **promoted** into long-term `MemoryFacts`.
  Revoking it **stops future extraction** and **enqueues the crypto-shred fan-out (§2.1) for that member's
  existing `MemoryFacts`** — the same revocation→forget mechanism as any other source, scoped to the
  memory-extraction sub-key.
- **Revocable.** Consent can be withdrawn at any time; revocation is audited (§2.3).
- **Revocation triggers the forget path.** Withdrawing consent for a source **stops future ingestion** and,
  where required (regulatory or user-elected), **enqueues the crypto-shred fan-out (§2.1) for that source's
  data** — scoped by the per-source / per-owner sub-key so only that source's footprint is forgotten, not the
  owner's whole record. Partial forget (one source) and full forget (an owner leaving the household) are the
  same mechanism at different key granularities.
- **Export before forget.** §14's export right is honored first: a machine-readable export of the owner's (or
  source's) SoR facts is offered prior to shred, so "forget" is not silent data loss to the user.

---

## 7. Acceptance criteria / tests

Following the 10× mandate (ADR 0001 §8). All deterministic and buildable now.

1. **Forget makes Postgres unreadable + tombstoned.** After forget, the owner's encrypted columns fail to
   decrypt (key gone) and the row is tombstoned (absent from live queries); the row still exists for chain/FK.
2. **Qdrant cannot surface a forgotten vector.** After forget, a semantic search that previously returned the
   member's content returns **zero** of their points (hard-deleted by owner-scoped payload filter).
3. **MinIO object unreadable after shred.** A blob written under the leaf sub-key cannot be decrypted after
   the key is destroyed, **even from a snapshot copy** restored to a scratch store.
4. **Neo4j subgraph gone.** The owner's personal nodes/edges are deleted; relationship-inference queries no
   longer traverse them.
5. **Backup copy is unreadable without touching the backup.** Restore a pre-forget snapshot; the shredded
   record's ciphertext is present but undecryptable (the key was on the box, now destroyed).
6. **The audit chain still verifies after a crypto-shred.** Shred a record, run full-chain re-derivation →
   **passes**; `factDigest`/`linkHash` intact (cross-check ADR 0005 test #8).
7. **Raw voice audio is not retained past transcription.** After a voice turn, no raw-audio artifact exists on
   disk anywhere; only the redacted transcript/derived fact remains.
8. **Guest is never profiled.** A `guest`-classified turn writes no `UserProfile`, no embedding, no long-term
   memory; retention equals the shortest window of its class.
9. **Retention sweep purges on policy.** Seed rows across classes past their cutoffs; run the Ritu sweeper →
   exactly the past-cutoff rows are shredded/deleted, chain digests untouched, a purge audit event emitted.
10. **Revocation triggers scoped forget.** Revoke Plaid consent → future ingestion stops **and** the forget
    fan-out runs for Plaid-sourced data only; Health/email data for the same owner is unaffected.
11. **Consent + forget are audited.** Every grant/revoke/forget produces a tamper-evident audit event naming
    who/what/when + per-store completion status.
12. **Export precedes forget.** A full-forget request first yields a machine-readable export of the owner's
    SoR facts.
13. *(amended 2026-07-11, round 2 — closes G3)* **Subject-aware Sarathi forget.** Forgetting member X shreds
    turns/summaries/facts owned by OTHER speakers where X ∈ `dataSubjectIds`; the speaker's own unrelated
    conversation content is unaffected.
14. *(amended 2026-07-11, round 2 — closes R2 consent)* **Memory extraction requires live consent.** The
    `MemoryFact` extraction pipeline refuses to run for a member without an active sarathi-conversation-memory
    consent record; revoking the consent stops extraction and enqueues the shred of that member's existing
    `MemoryFacts`.

---

## 8. Open questions

- **Legal review for productization (flagged).** Recording-consent law (one/two-party), minors' PHI handling
  (COPPA/HIPAA-adjacent for the child's journals), and retention statutes vary by jurisdiction. The household
  default is conservative; **a product requires counsel** to set `RecordingConsentMode`, camera-notice, and
  finance/health retention windows per market.
- **Vector rebuild vs. permanent loss.** After forgetting an owner and later restoring their consent, is any
  embedding rebuild expected, or is derived-data loss permanent-by-policy? (Lean: permanent; rebuild only from
  surviving SoR facts on explicit re-consent.)
- **Key granularity ceiling.** Per-owner is cheap; per-record sub-keys multiply key count. Which classes
  warrant per-record shred (journals: yes; routine vitals: per-owner)? Define the boundary in `RetentionPolicy`.
- **Snapshot key-custody rotation.** Confirm no historical wrapping key ever leaves the box into a backup such
  that a shredded leaf could be re-derived (must not — ADR 0002/0008 assumption to pin).
- **Retention policy authorship.** Who may edit `RetentionPolicy` (owner-only? admin-quorum?) — it governs how
  long the household's most sensitive data lives.
- **Down-sampling fidelity.** Exact warm-tier down-sample rates for health/finance that preserve trend value
  while shedding raw volume.

---

## 9. Consequences

**Positive.**
- **Right-to-forget becomes buildable** without breaking append-only stores, immutable backups, or the audit
  chain — one mechanism (crypto-shred) with a precise per-store fan-out; the C9 contradiction is resolved.
- **Backups and off-site snapshots are forgettable** *because* the key lives in the box, not the backup —
  turning ADR 0002/0008's design into a privacy feature.
- **Growth is bounded** by a data-class retention matrix + a scheduled, audited, idempotent purge job, with DB
  growth budgeted and monitored — driver #7 reconciled with minimization (keep meaning, not every byte).
- **Bystanders and guests are protected by default** — raw audio dropped, non-members never profiled, camera
  clips time-boxed — independent of a consent they cannot give.
- **Consent is a first-class, revocable, audited object**, and revocation *does something* (scoped forget).

**Negative / trade-offs.**
- **Forget is not instantaneous everywhere.** Key-shred is immediate; the store-delete housekeeping (Qdrant
  points, MinIO objects, Neo4j nodes) runs in the fan-out job — a window exists where a shredded-but-
  undecryptable row still occupies space. The *guarantee* (unreadability) is immediate; reclaim lags.
- **Derived-data loss on forget is permanent** (embeddings/graph are deleted, not archived) — rebuild only
  from surviving SoR facts. Acceptable, but it means a forgotten member's semantic recall is gone for good.
- **Key sprawl.** Per-record sub-keys raise key-management load on `Manas.Crypto`; must be bounded by policy.
- **The legal surface is real.** As a product, recording-consent and retention-window law demand review — this
  ADR flags it but does not settle it.
- **More moving parts** (Ritu sweeper, per-store fan-out, consent objects) — but all reuse existing Kernel,
  ADR 0002 keys, and ADR 0005 audit primitives; nothing here is a new bespoke subsystem.

---

**See also:** [ADR 0001](0001-manas-platform-architecture.md) §14 / driver #7 · the Kernel design documentation §2 (`WalledEntity` tombstone) · [ADR 0002 — Key hierarchy & unlock](0002-key-hierarchy-and-unlock.md) (crypto-shred hook) · [ADR 0005 — Audit pipeline & tamper-evidence](0005-audit-pipeline-and-tamper-evidence.md) (plaintext-digest chain) · [ADR 0008 — Backups & disaster recovery](0008-availability-degraded-mode-and-dr.md) (immutable off-site snapshots).
