# ADR 0004 — Walling, may-see & Sharing (all five stores)

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** [ADR 0001](0001-manas-platform-architecture.md) §5.2 / §5.4, the Kernel design documentation §3.
  Resolves design-review **A3 / risk #20** (the most load-bearing security guarantee) and **B6** (Neo4j isolation).

> The per-user wall — "a member's data cannot leak to another member, and the LLM can never decide who sees
> what" — is the single most load-bearing guarantee in Manas, and it currently holds **only in Postgres**.
> This ADR extends the wall, *identically and fail-closed*, to all five stores; writes `may-see` out as an
> explicit truth table; decouples operational-Admin from guardianship; and unifies sharing into one revocable
> primitive. The worst-case leak this closes is a **child's health/journal embedding surfacing into an adult's
> LLM context** (or the reverse) because one Qdrant search forgot a filter.

---

## 1. Context

The Kernel design documentation §3 gives every `WalledEntity` an **EF Core global query
filter** that a developer cannot forget: `TenantId == actor.TenantId AND (OwnerId == actor.UserId OR
may-see(OwnerId)) AND IsDeleted == false`. That guarantee is real — **for Postgres**. But each service is
polyglot ([ADR 0001](0001-manas-platform-architecture.md) §5.4): **Neo4j** (graph), **Qdrant** (vectors —
semantic memory/RAG that *feeds the LLM's context*), **MinIO** (blobs — PDFs, voice clips, HealthKit exports),
**Redis** (cache + event bus). The design review found three defects (A3):

1. **The wall is Postgres-only.** The other four stores are walled "via helpers" — *opt-in, therefore
   forgettable*. A single missed Qdrant payload filter returns another owner's health/journal vector straight
   into the model's prompt. This is the worst leak in the system.
2. **`may-see` is undefined.** The Kernel design doc references it; no document says what it returns.
3. **"Admin" conflates operator with guardian.** Ananth and the other adult are **both** Admin. If `may-see` keys
   off the Admin role, Ananth can read the other adult's cycle data, journals, and accounts — and vice-versa. **Adults must
   be walled from each other.** The right to configure the box (operational-Admin) is a different thing from
   the right to see a Child's data (guardianship).

Identity itself is assumed trustworthy here — its *cryptographic* verification is [ADR 0003](0003-identity-propagation-and-pki.md)'s
job. This ADR takes a verified `ActorContext` as given and specifies what the data layer does with it.

**Non-negotiables carried in from [ADR 0001](0001-manas-platform-architecture.md) and the platform's threat model:**
`may-see` is **deterministic C#, never the LLM**; cross-tenant is **never**; default is **walled** (share is
opt-in); every store fails **closed**.

---

## 2. The five-store walling mechanism (deny-by-default, one shape per store)

The principle is identical everywhere: **there is exactly one walled repository per store, it is the only code
holding a raw driver/handle, and it refuses any access that does not carry an `ActorContext`-derived
tenant+owner predicate.** Raw driver access outside the walled repo is a build-time / architecture-test
failure (§7). "Forgot to filter" becomes *structurally impossible*, not *discouraged*.

### 2.1 Postgres / EF — the global query filter (exists; restated)

The `WalledEntity` global query filter (Kernel design doc §3) is the reference implementation. It is applied at
`OnModelCreating` to **every** walled type, so a new entity is walled the instant it exists. The `SaveChanges`
interceptor (Kernel design doc §4) additionally enforces `TenantId`/`OwnerId` consistency on **writes** (a write must
target the acting tenant and an allowed owner). Raw-SQL escape hatches go through the same predicate via a
repository helper — never `FromSqlRaw` directly. **No change; this ADR makes the other four match it.**

### 2.2 Neo4j — the walled Cypher wrapper (traversal-constraining)

- **One `WalledGraphSession`** wraps the driver; it is the *only* holder of a Neo4j session. It injects, into
  **every** query, a tenant + owner/may-see predicate on **every bound node**, not just the anchor:
  every node label carries `{tenantId, ownerId}` and the wrapper appends a `WHERE` clause asserting the
  predicate for each node variable in the pattern.
- **Traversals hop across owners** — the core graph risk. A path `(:JournalEntry)-[:MENTIONS]->(:Person)-[:MENTIONS]<-(:JournalEntry)`
  can walk from *my* node to *your* node in one query. Therefore the wall constrains **traversal, not just the
  root**: variable-length and multi-hop patterns are rewritten so **each** matched node must satisfy
  `may-see`, and relationships whose endpoints straddle a forbidden owner are pruned. A raw string handed to
  the driver without going through the wrapper throws — there is no ergonomic raw path.
- **Writes** stamp `tenantId`/`ownerId` from the actor and reject a mismatch, mirroring the EF interceptor.

### 2.3 Qdrant — tenant + owner-scoped collections (highest stakes; LLM context)

This is the store whose leak is worst, because its output is *retrieved into the prompt*.

- **Physical partition first:** collections are named `{service}__{tenantId}` and every point's payload carries
  `ownerId` (+ `sharingGrantIds`). A search **must** pass a `Filter` with `tenant` (implied by collection) and,
  *(amended 2026-07-11, round 2 — closes S3/S15)* the **row-granular** predicate
  `owner ∈ ({actor.UserId} ∪ may-see-owners)` **OR** `payload.sharingGrantIds ∩ activeGrantIdsFor(actor) ≠ ∅`.
  This replaces the earlier owner-granular `owner ∈ … ∪ granted-owners`, which admitted **ALL** of a grantor's
  points off a single grant — a violation of §5's "never see-all-of-X" rule. Now one grant from the other adult
  exposes only the points that grant actually covers. The write path **stamps `sharingGrantIds`** on a point when a
  grant covers it; **grant revocation** either triggers a payload-update sweep clearing the id, **or** — the
  **correctness-first default we adopt** — is caught at **query time**, because `activeGrantIdsFor(actor)` is
  resolved live, so a revoked grant's id simply isn't in the set on the next search (no sweep race, no stale
  window; the trade is a small per-query grant-set lookup latency, mitigated by §8 caching). The
  `WalledVectorStore` is the only code that can issue a search, and it **refuses a search with no owner/grant
  filter** (deny-by-default) — an unfiltered `search()` is not reachable.
- **RAG context assembly** (the path that builds the LLM prompt) draws *only* from the walled store, so a
  retrieved chunk is provably owned-or-shared before it can reach the model. This is the concrete closure of
  the "child PHI → adult chat" leak.
- Right-to-forget deletes points by `ownerId` payload filter (coordinates with [ADR 0009](0009-data-lifecycle-retention-forget-consent.md)).

### 2.4 MinIO — per-owner key prefixes + short-TTL actor-scoped signed URLs

- Object keys are prefixed `{tenantId}/{ownerId}/{service}/…`. The **only** way a client (app, agent tool, box)
  fetches a blob is a **pre-signed URL minted by the walled blob repo**, which first checks the actor may-see
  the owner in the key, then signs with a **short TTL** (minutes) scoped to that one object.
- **No unsigned or unscoped `GetObject`** is exposed; the bucket policy denies anonymous and broad-prefix
  reads. A signed URL for `household/otherAdult/ojas/labs.pdf` cannot be minted for an actor who is not that
  adult or their guardian, and it expires before it can be replayed cross-owner.

### 2.5 Redis — namespaced keys + re-walling consumers

- Cache keys are namespaced `{tenantId}:{ownerId}:{service}:{key}`; the `WalledCache` composes the namespace
  from the `ActorContext`, so a `GET`/`SET` for another user's namespace is unrepresentable through the API.
- The **event bus** ([ADR 0006](0006-automation-principal-and-event-bus.md), Redis Streams) carries
  `tenantId` + `ownerId` + `correlationId` in every event envelope. A consumer is **not** trusted to inherit
  the wall — it **re-walls**: it resolves its own `ActorContext` (or the rule-owner principal) and re-checks
  may-see before acting on or persisting anything derived from the event. Payloads never carry cleartext
  sensitive values (they carry ids to re-fetch through a walled repo).

---

## 3. The `may-see` truth table (deterministic; the whole predicate)

`may-see(actorRole, actorId, ownerId, tenant, sharing)` returns **true iff a row owned by `ownerId` is visible
to `actorId`**. It is pure, total, deterministic C#. Inputs: the actor's role and id, the row's owner id, both
tenants, the `guardianOf` edges from Kutumba (§4), and any active `SharingGrant` (§5). **The Admin role is
NOT an input.**

| # | actorRole | actor vs owner | guardian edge? | active share? | same tenant? | **may-see** | Rationale |
|---|-----------|----------------|----------------|---------------|--------------|-------------|-----------|
| 1 | any | `actorId == ownerId` | — | — | yes | **TRUE** | You always see your own rows. |
| 2 | Adult/Admin | different adult | — | none | yes | **FALSE** | **Adults are walled from each other — even as Admin.** (Closes A3.) |
| 3 | Adult/Admin | different adult | — | **grant covers row** | yes | **TRUE** | Explicit, revocable, scoped share (§5). |
| 4 | Adult/Admin | a Child | **yes (`guardianOf`)** | — | yes | **TRUE** | A guardian sees their Child's rows — via the edge, not the role. |
| 5 | Adult/Admin | a Child | **no edge** | none | yes | **FALSE** | An adult who is not the child's guardian sees nothing. |
| 6 | Child | an Adult | — | — | yes | **FALSE** | A Child never sees an adult's rows. |
| 7 | Child | a different Child | — | none | yes | **FALSE** | Children are walled from each other. |
| 8 | any | anyone | any | any | **no** | **FALSE** | **Cross-tenant is never** — the outermost, non-overridable gate. |
| 9 | Service | (S2S, no user) | — | — | — | **FALSE** | A service principal with no acting user sees no walled rows (deny-by-default). |
| 10 | Service/Automation principal | any owner | — | **active MACHINE-GRANT covering the row** | yes | **TRUE** | A machine principal sees exactly what an explicit, audited machine-grant names — nothing else. *(amended 2026-07-11, round 2 — closes S3/S15)* |

**Reading the table:** row 8 dominates (cross-tenant short-circuits to false before anything else). Row 1 is
self-access. Guardianship (rows 4–5) is *directional* (guardian→child only; row 6 forbids child→adult) and
gated on an **explicit edge**, never on being Admin. Adult↔adult (rows 2–3) is walled unless a grant exists.
Everything not listed is **false** (deny-by-default). Grants (rows 3) are evaluated at row granularity via the
scope predicate in §5 — a grant is never "see all of X's data" unless its scope literally says so.
*(amended 2026-07-11, round 2 — closes S3/S15)* **Machine principals** (a `rule:{ruleId}` / `svc:{name}` from
[ADR 0006](0006-automation-principal-and-event-bus.md) §2) get **exactly one** true-making row: **row 10**, an
active machine-grant covering the specific row. Row 9 (no grant ⇒ FALSE) is their default. A machine principal
**never inherits the human rows (1–7)** — in particular it never gets row-1 self-access off its firing subject
(that laundering hole is killed in §5) — and **row 8 (cross-tenant) still dominates** it exactly as it dominates
a human actor.

---

## 4. Guardianship vs Admin (Kutumba edges, not roles)

The conflation in A3 is dissolved by making guardianship **data**, not a role bit:

- **Operational-Admin** (`Roles` contains `Admin`) governs **the box**: enabling/disabling services, editing
  config, and *authorising step-up* ([ADR 0007](0007-risk-tiers-and-step-up.md)). It grants **no** data
  visibility over other members. Ananth and the other adult are both Admin — and still cannot read each other.
- **Guardianship** is an explicit **`guardianOf` directed edge in Kutumba** (`guardian:UserId →
  child:UserId`, within one tenant). `may-see` row 4 reads *this edge*. Each guardian's edge to a child is its
  own edge; there is no adult-to-adult edge, so no visibility between adults. Removing the edge (a child turns
  18, a custody change) instantly and deterministically closes guardian access — no code change.
- **Kutumba is the source of truth** for both the role set and the guardian edges; `may-see` takes them as
  inputs and never infers one from the other. This matches the school-service design ("parent as
  guardian") and the finance service's per-user wall.

---

## 5. One canonical sharing model

Today there are two `SharingScope` notions: the Kernel's, and the finance service's `Personal | Shared` enum.
They are **reconciled into a single platform primitive** in
`Manas.Kernel`, and the finance service's enum becomes a thin projection over it.

- **`SharingGrant`** (a `TenantEntity` — config, not user-owned payload): `{ Id, TenantId, GrantorId (owner),
  PrincipalId, Scope, ExpiresAtUtc?, RevokedAtUtc?, CreatedBy… }`. It records **who shared what with whom, at
  what scope, until when.**
- *(amended 2026-07-11, round 2 — closes S3/S15)* **`PrincipalId` is a grantee principal, human OR machine.**
  It is either a **member** (`UserId`) or a **machine principal** — a `rule:{ruleId}` / `svc:{name}` from
  [ADR 0006](0006-automation-principal-and-event-bus.md) §2. A grant to a machine principal is a **machine-grant**;
  it is what makes truth-table row 10 true and is the *only* way a rule or service reads another member's rows.
- **`Scope`** is an explicit, deterministic predicate — never "everything": e.g. `Service=Artha,
  ObjectType=Budget, ObjectId=household-2026` (a specific shared budget), or `Service=Kaya, ObjectType=*`
  (a whole domain, chosen deliberately). `may-see` row 3 (human grantee) and **row 10 (machine grantee)** are
  true only when an *active* grant's scope covers the specific row. **The scope grammar is identical for human
  and machine grants** — `(Service, ObjectType, ObjectId|*)`, always expiring-or-revocable, always audited.
- **Machine-grants are created at rule-authoring time** via the ADR 0006 §2 consent flow — the grantor is the
  **data owner** (or the **child's guardian** where the data is a Child's), never the rule's author-by-authority;
  minting is **step-up gated per [ADR 0007](0007-risk-tiers-and-step-up.md)** at the tier the data warrants. So
  "may `rule:{r}` read the other adult's health summary?" is answered **once, explicitly, on record** as a machine-grant —
  not silently at fire time.
- *(amended 2026-07-11, round 2 — closes S3/S15)* **The rule's declared "subject" ([ADR 0003](0003-identity-propagation-and-pki.md) §2.2 /
  [ADR 0006](0006-automation-principal-and-event-bus.md) §2's `UserId`) never enters `may-see`.** Subject is
  **delivery/audit metadata only** (who the action is *on behalf of*, where the notification lands); **data
  access rides the machine-grant, not the subject**. A firing's `ActorContext.UserId = subject` must **not**
  satisfy row-1 self-access — that first-disjunct laundering hole (a rule reading ALL of its subject's rows) is
  closed here: a machine principal reads only what a machine-grant's scope literally names.
- **Default is walled.** No grant ⇒ Personal ⇒ private. "Shared" (a joint household budget) is *co-owned by an
  explicit grant*, never an implicit consequence of being household members. **A machine principal with no
  machine-grant sees nothing** (row 9).
- **Revocable and expiring.** `RevokedAtUtc`/`ExpiresAtUtc` make a grant instantly inert — the next `may-see`
  call returns false, across all five stores, with no data migration; this holds for **machine-grants**
  identically. Grants are auditable (Saakshi) and surfaced in the UI ("what have I shared / what is shared with
  me" — and which rules may read what).
- **Human grants are created only by the grantor** (or the grantor's guardian for a Child), gated at step-up tier
  appropriate to the data ([ADR 0007](0007-risk-tiers-and-step-up.md)); **machine-grants** follow the same
  grantor + step-up rule, captured at authoring time as above.

**Amendment, 2026-08-15 (unit AUTO-5-RULE-GRANT-INDEX) — a SECOND, narrower lane for a `StandingRule`'s own
scoped self-authorization; it is NOT the machine-grant mechanism described above.** The canonical model above
describes a machine-grant as created via a step-up-gated consent flow, recorded on a `SharingGrant` row with
`Scope`/`ExpiresAtUtc`/`RevokedAtUtc`. This later mechanism creates NO such row: a `StandingRule`-derived Automation
write grant for `ArthaProposalEnqueue` is DERIVED, live, from `StandingRule.Active` itself — the rule's own T4
authoring ceremony (step-up gated exactly as a machine-grant
would be) IS that consent's live record, so there is no second consent flow to record separately.
`Manas.Kernel.Walling.IAutomationGrantSource` is the extension point a future domain's own rule-backed automation
write registers against (`RuleBackedGrantIndex` fans out over every registered source) — it is deliberately narrow
(today: Artha's `MoneyMoveProposal` rows only) and MUST NOT be assumed to generalize to a domain's other rows, or
to widen into a general-purpose machine-grant mechanism, without a fresh review. It never populates or consults the
`SharingGrant` table this section describes.

**A named, accepted asymmetry, tracked internally as a NORMAL-severity debt item.** `HasActiveGrantCovering` (the
row-level write check above) and `HasActiveGrantFromOwner` (the owner-level existence check the EF global filter's
owner-set resolver uses) can now disagree for a machine principal: a covering grant can be `true` while the
owner-level check stays permanently `false`, because that owner-level check is UNCHANGED (no human sharing-grant
store exists yet). Safe today because no Automation READ path reaches that resolver — the automation WRITE path
uses a narrower, single-owner wall scope exclusively. (Operational detail omitted from the public copy.)

---

## 6. Neo4j isolation decision (B6)

**Decision: one shared Neo4j Community instance, with per-service + per-tenant + per-owner *labels/properties*
enforced by the walled Cypher wrapper (§2.2). Physical isolation is relaxed for the graph store specifically;
logical isolation is kept via labels + the wall.**

Rationale: Neo4j **Community has no multi-database** — a true "space per service" needs **Enterprise**
(licensed, conflicts with the free/self-host driver in [ADR 0001](0001-manas-platform-architecture.md)) or **N
separate Community instances** (each its own JVM ≈ 3–4 GB; five services ≈ **15–20 GB RAM**, a large bite of
the home server's 128 GB unified budget). Neither is
justified when the wall already runs *inside* every query. We therefore:

- keep **one** Community JVM (one graph, ~3–4 GB);
- give every node `{service, tenantId, ownerId}` and make the walled wrapper the **only** query path, so
  cross-service / cross-tenant / cross-owner traversal is structurally impossible even in a shared graph;
- treat this as a **deliberate, documented exception** to [ADR 0001](0001-manas-platform-architecture.md)
  §5.4's "database-per-service" (which remains the rule for Postgres/Qdrant/MinIO). It is **logical**, not
  physical, isolation for the graph tier only.

**Trip-wire:** if the graph tier ever holds cross-tenant (multi-family, post-productisation) data on one box,
revisit — at that point physical separation (N instances or Enterprise) becomes mandatory, because a wrapper
bug would be a *cross-family* leak. For the single-household box, logical isolation is the right call.

---

## 7. Acceptance criteria / property tests (the per-store negatives)

Each store ships with **FsCheck/CsCheck property tests over generated actors/owners/roles/grants** *and*
per-store **integration tests against a real container** (Testcontainers). The properties assert the
**negative** — the leak must be *impossible*, not merely *usually blocked*:

- **P1 (Postgres):** for all `(actor, owner)` with `owner ≠ actor` and no guardian edge and no grant, a query
  returns **zero** of `owner`'s rows — including when `actor` is Admin (encodes truth-table row 2).
- **P2 (adult↔adult, all stores):** **an Adult never sees another Adult's row, even as Admin** — asserted for
  EF, the Cypher wrapper, Qdrant search, a MinIO fetch attempt, and a Redis read.
- **P3 (child→adult):** **a Child never sees an Adult's row** — all five stores (truth-table row 6).
- **P4 (Qdrant):** a search issued by `actor` **never returns a point** whose payload `ownerId` is not
  may-see-able; and the walled store **refuses** a search constructed without an owner filter (deny-by-default).
- **P5 (Neo4j traversal):** a multi-hop / variable-length traversal from an owned anchor **never surfaces a
  node** owned by a non-may-see owner (traversal-constraining, not just root-constraining).
- **P6 (MinIO):** a signed URL minted for `actor` **cannot fetch** an object under another owner's prefix; an
  expired URL fails; no unsigned `GetObject` path exists.
- **P7 (Redis):** a `GET` cannot read another owner's namespace; an event consumer that skips re-walling
  fails a test that plants a cross-owner event.
- **P8 (guardian edge):** a guardian sees the Child's rows **iff** the `guardianOf` edge exists; deleting the
  edge closes access on the next call (all stores).
- **P9 (grant lifecycle):** a shared row is visible **iff** an active grant covers it; revoking / expiring the
  grant closes access immediately, all stores, no migration.
- **P10 (cross-tenant):** no actor ever sees any row from another tenant, in any store, under any role, grant,
  or guardian edge (truth-table row 8 dominates).
- **P11 (architecture test):** no code outside a `Walled*` repository references a raw driver/handle
  (`IDriver`, `QdrantClient`, `IMinioClient`, `IConnectionMultiplexer`, `FromSqlRaw`) — a NetArchTest rule in
  CI, so a future contributor cannot open an unwalled path.
- *(amended 2026-07-11, round 2 — closes S3/S15)* **P12 (machine principal, no grant — all stores):** a
  `rule:{ruleId}` / `svc:{name}` principal with **no machine-grant** returns **zero** rows in all five stores,
  under any firing `subject` — extends truth-table row 9 to machine principals (deny-by-default).
- *(amended 2026-07-11, round 2 — closes S3/S15)* **P13 (machine principal, scoped grant — all stores):** a
  machine principal holding a grant scoped to **one object** sees **exactly that object** and **nothing else** of
  the grantor's — across EF, the Cypher wrapper, Qdrant search (row-granular via `sharingGrantIds`), a MinIO
  fetch, and a Redis read (encodes truth-table row 10 at row granularity).
- *(amended 2026-07-11, round 2 — closes S3/S15)* **P14 (machine-grant revocation + no subject-laundering):**
  revoking a machine-grant **closes access on the next call**, all stores, no migration; and a firing whose
  `ActorContext.UserId = subject` but which holds **no** covering machine-grant sees **zero** rows — the
  `subject` claim **alone never opens anything** (row-1 self-access does not apply to a machine principal).

**Definition of done:** P1–P14 green in CI against real containers; a red test here blocks merge — this is the
platform's crown-jewels guarantee.

---

## 8. Open questions

- **Grant scope grammar.** How expressive is `Scope`? Start with `(Service, ObjectType, ObjectId|*)`; defer
  attribute-level (e.g. "share the budget total but not line items") until a real need — over-expressive scopes
  are their own leak surface.
- **may-see caching.** Guardian edges and grants change rarely; may we cache the resolved may-see-owner set per
  `(actor, service)` in Redis with tight invalidation on Kutumba/grant change? Weigh latency vs. a stale-grant
  window (a revoked grant must close *immediately* — favor correctness). *(amended 2026-07-11, round 2 — closes
  S3/S15)* **This question now ALSO covers machine-grants** — the same cache/invalidation applies to a machine
  principal's `activeGrantIdsFor(actor)` set, with the same immediate-revocation requirement.
- *(amended 2026-07-11, round 2 — closes S3/S15)* **Qdrant grant enforcement: stamp vs query-time join.** Do we
  stamp `sharingGrantIds` on points at write time and sweep them on revocation, or resolve
  `activeGrantIdsFor(actor)` live at query time (the correctness-first default adopted in §2.3)? A **build-phase
  measurement** — compare the per-query grant-set lookup latency against the sweep cost and its stale-window
  risk on real Qdrant containers, then confirm the default or switch.
- **Neo4j wrapper completeness.** Can we *prove* the Cypher rewrite covers every pattern (OPTIONAL MATCH,
  `apoc.path.*`, projections)? Consider restricting the allowed Cypher subset the wrapper accepts so
  unrewritable constructs are rejected rather than silently unwalled.
- **Shared-object ownership under forget.** When a co-owned (granted) object's grantor exercises right-to-forget
  ([ADR 0009](0009-data-lifecycle-retention-forget-consent.md)), what happens to grantees' view? Likely: forget
  destroys the row's key; grantees lose access with the owner. Confirm with 0009.
- **Break-glass.** Is there a guardian/Admin emergency-access path (a child in crisis) that widens may-see under
  heavy audit + step-up, or is that explicitly out of scope? Default: **out of scope** unless Ananth requests it.

---

## 9. Consequences

**Positive**

- The wall is now **one mechanism per store, fail-closed** — the "forgettable helper" class of bug is gone, and
  the worst leak (child PHI → LLM context) is structurally closed at the retrieval boundary.
- `may-see` is **written down and testable**; "who can see what" is a truth table anyone can audit, not folklore.
- **Adults are walled from each other**, and guardianship is explicit, directional Kutumba data — Admin no
  longer implies omniscience. Custody/age changes are edge edits, not code.
- **One sharing primitive** — revocable, scoped, default-off — replaces two divergent `SharingScope`s.
- The negative property tests make the guarantee **provable in CI**, per store.

**Negative / costs**

- **Every store grows a walled-repo layer** and loses its raw driver ergonomics — more up-front plumbing, and
  discipline enforced by the P11 architecture test.
- **may-see runs on every read across five stores** — a small latency tax (mitigated by §8 caching, carefully).
- **Neo4j gives up physical per-service isolation** (§6) — a wrapper bug is a same-tenant cross-owner risk;
  mitigated by P5 and the single-household trip-wire, but it is a real accepted trade-off.
- The Cypher-rewrite and Qdrant-filter layers are **genuinely hard to get exhaustively right** — hence the
  restricted-subset open question and the heavy property-test bar.

**Neutral**

- Depends on [ADR 0003](0003-identity-propagation-and-pki.md) for a *verified* actor (this ADR assumes
  it), and hands off deletion semantics to [ADR 0009](0009-data-lifecycle-retention-forget-consent.md) and step-up
  gating to [ADR 0007](0007-risk-tiers-and-step-up.md). It **refines**, and does not supersede,
  [ADR 0001](0001-manas-platform-architecture.md) §5.4 — except the explicit Neo4j-isolation carve-out in §6.
