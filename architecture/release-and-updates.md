# Manas — Release, Versioning & Update Delivery

- **Status:** Living · **Date:** 2026-07-11 · with [ADR 0001](../adr/0001-manas-platform-architecture.md) +
  the risk register.

> How code reaches the home server — safely, atomically, reversibly — for one box now and a **fleet of appliances**
> later. **Design principle: an update is signed, verified, A/B-applied, health-checked, and auto-rolled-back on
> failure — never a half-updated box.** Updates are a top attack surface (a malicious update = total compromise
> of a finance/health/security box), so integrity is non-negotiable.

## 1. Versioning

- **Per-service SemVer** `MAJOR.MINOR.PATCH`: **major** = breaking API/schema (needs a coordinated migration);
  **minor** = backward-compatible feature; **patch** = fix/security, no contract change.
- **Platform Release = a pinned manifest (a Bill-of-Materials)** naming the exact, mutually-compatible version
  of every service + spine + DB schema + model set. The box runs a **Release**, not a soup of drifting service
  versions. Independent service evolution *within* a Release is enabled by the **contract-versioning discipline**
  (GraphQL additive + deprecation, MCP contract versions, EF migrations — ADR §14).
- **Three update tracks**, each versioned and coordinated but shipped on its own cadence:
  1. **Core software** (the .NET services + spines, as containers) — the main track.
  2. **Satellite firmware** (the voice box, via **ESPHome OTA**).
  3. **iOS app** (App Store / TestFlight).
  Bodha (self-awareness) tracks and reports the version of all three; a Release manifest states the
  compatible-together set.

## 2. Artifacts & the build→publish pipeline

- CI (from ADR §14, on every commit): **build → 10× tests + Cobertura gate → SCA + SAST + secret-scan →
  sign → publish**.
- Output: **versioned, cryptographically-signed container images** + a **signed Release manifest** + an
  **SBOM** (+ SLSA-style provenance), pushed to a **private OCI registry**.
- **Signing is the crux:** the box verifies the publisher's signature on both images and manifest **before
  applying anything**. No signature / bad signature → refuse.

## 2a. Shipping the datastores (Postgres, Neo4j, Qdrant, Redis, MinIO)

The databases are **not** bespoke installs — they ship like everything else:
- **Pinned official container images in the signed Release BOM** (`postgres`, `neo4j`, `qdrant`, `redis`,
  `minio`), pulled + signature-verified by the update agent like any other image.
- **Data lives in persistent host volumes, separate from the replaceable engine images** — an update swaps the
  Neo4j *image* without touching the graph *data*; **DR = ZFS filesystem-consistent snapshots** of those
  volumes (the pragmatic one-box answer, and **required for Neo4j Community which has no online backup**).
- **Schema/data migrations run inside the atomic A/B step (§3):** EF for Postgres · **versioned Cypher** for
  Neo4j · a **re-embed pipeline** for Qdrant (keyed to the embedding-model version — changing the model
  invalidates all vectors) · **layout versioning** for MinIO. Expand/contract discipline; rollback = revert
  code onto the still-compatible expanded schema (never a destructive reverse-migration).
- **First run:** images pulled → volumes created → migrations seed the schema.
- **Open decision (review B6):** Neo4j Community has **no multi-database** → the per-service graph isolation is
  one shared DB with per-service labels, or N Community instances (fix the RAM budget), or Enterprise (cost).

## 3. The on-box update agent (the mechanism)

A small, supervised **update agent** on the Core:
1. **Check-in** (pull-based) to the release channel on a schedule (or on-demand / voice: "update Manas").
2. **Fetch** the target Release manifest + images from the registry over TLS; **verify signatures + SBOM**.
3. **Back up** first (encrypted snapshot — ties to ADR §14 DR).
4. **A/B (blue-green) apply**: bring up the new versions alongside the old; run **DB migrations**
   (backward-compatible + reversible; part of the atomic step).
5. **Health-check** (readiness probes + a smoke suite). Green → **promote** and retire the old; red →
   **auto-rollback** (restore old containers + reverse migration) and alert.
6. **Report** the new version + health to Bodha (and, for the product, the fleet control-plane).

Properties: **atomic** (never half-updated), **reversible**, **offline-safe** (if the registry is unreachable
the box keeps running the current Release; updates are pull + optional), and **data/config-preserving** (an
update never clobbers user data or settings; config is separate — ADR §6).

## 4. Life-safety during updates

**Kavach (the alarm) must not go dark to install a patch.** Updates run in a **maintenance window**; Kavach's
core alarm path stays up (or fails safe) across the swap, and security patches to Kavach are prioritized and
tested against a life-safety smoke suite before promote. No update proceeds if it can't guarantee the alarm's
continuity/fail-safe.

## 5. Now vs product (the mechanism scales, the operator changes)

- **Now (personal):** monorepo commit → CI builds+signs → a **private registry**; the box's update agent pulls
  the chosen Release. Simple, already the shape above.
- **Product (fleet of family appliances):** the **thin cloud control-plane** becomes the **release/update
  server** — it publishes signed Releases to **channels** (`stable` / `beta` / `canary`), and each appliance's
  update agent checks in. Adds:
  - **Staged / canary rollout** — new Releases go to canary → beta → a % of stable → all, halting on failure
    signals.
  - **Update channels** the customer picks (stable by default).
  - **Privacy-respecting fleet telemetry** — *version + health only*, opt-in, **never family data**; the
    release server is **blind** to content, exactly like the relay.
  - **Scheduled windows + deferral** — the customer controls when.
  - **Emergency security patch** path (expedited, still signed + A/B + rollback).

## 6. Security (updates are a prime attack surface — see threat model)

- **Signed images + signed manifests, verified on-box** before apply (supply-chain integrity).
- TLS to the registry; the update agent trusts only the platform's signing key(s); key rotation planned.
- **SBOM + provenance** per Release; SCA/SAST/secret-scan gate in CI.
- The release server **serves artifacts only** — it never sees or stores family data (blind, like the relay).
- A failed/tampered update **cannot leave the box in a broken or downgraded-security state** (A/B + rollback +
  signature checks + backup).

## 7. Open decisions (→ ADRs 0002+)

- Registry + signing tooling (e.g. cosign/sigstore + a private OCI registry).
- A/B orchestration mechanism (compose profiles vs a light orchestrator; **not** full k8s for one box).
- Migration framework conventions (EF migrations + a reversible-migration policy).
- Fleet rollout controller (build vs off-the-shelf) — product phase only.
