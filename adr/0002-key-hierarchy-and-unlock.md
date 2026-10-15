# ADR 0002 — Key Hierarchy & Unlock Model

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** ADR 0001 §15; resolves design-review A1 / risk #19; incorporates B2, C3, C4.

> This ADR replaces the single-DEK design implied by ADR 0001 §15 with a **per-sensitivity-tier DEK set**,
> so that the life-safety alarm (Kavach) can TPM-auto-unlock on cold boot **without** also unlocking finance
> and health. It fixes the cryptographic contradiction that blocks `Manas.Crypto`, and pins down key
> wrapping, recovery, `[Encrypted]` key selection, and the on-disk blob format.

---

## 1. Context

`Manas.Crypto` is the single envelope-encryption library for the whole platform (ADR 0001 §15). Its original
shape: one random **DEK** (AES-256-GCM) encrypts all data and logs; the DEK is *wrapped* by a **KEK** derived
from an admin password via a memory-hard KDF (Argon2id, PBKDF2-SHA256 as a BCL fallback behind
`IKeyDerivation`). At login, the password derives the KEK → unwraps the DEK → the DEK lives in RAM only and is
zeroed on lock. A **recovery key** is a second wrap of the same DEK, shown once and stored offline. Multi-admin
is expressed as **N wrappers** of the same DEK (Ananth's and the other adult's passwords each wrap it).

ADR 0001 §15 also committed to a **hybrid unlock model**: Kavach (the life-safety alarm — an ADT replacement)
must **TPM-auto-unlock after a reboot** for availability (the alarm cannot wait for a human to type a password
before it can arm/fire), while **finance (Artha)** and **health (Ojas)** must stay **sealed until an admin
logs in**.

**These two decisions are mutually exclusive with one DEK.** If Kavach TPM-auto-unlocks *the* DEK on cold
boot, that same key decrypts Artha and Ojas — "Kavach unlocked, finance/health locked" is cryptographically
impossible when one key protects everything. This is design-review finding **A1 / risk #19 (CRITICAL)** and it
blocks the only scaffolded library. This ADR resolves it.

Two dependent review items are folded in here because they are inseparable from the key design: the **Argon2id
package choice** (review B2 — no BCL Argon2) and the **recovery-key crackability** hardening (review C4 / risk
#24). The **running-box-DEK-in-RAM** residual (review C3 / risk #21) is stated honestly and cross-referenced to
the availability ADR (0008).

---

## 2. Decision — a per-sensitivity-tier DEK set

`Manas.Crypto` manages **four DEKs**, one per **sensitivity tier**, not one DEK for everything. Each DEK is an
independent AES-256-GCM key with its own key id; a blob encrypted under one tier's DEK is **undecryptable** by
any other tier's DEK. The tiers differ only in **how their DEK is wrapped and when it is unsealed** — the
cipher and envelope machinery are identical.

| Tier | Key id | Services / data it encrypts | TPM-sealed? | Password-derived (admin wrap)? | Unlock timing |
|------|--------|------------------------------|-------------|-------------------------------|---------------|
| **Life-safety** | `LIFE` | Kavach alarm state, zone config, arm/disarm history, siren/entry-delay policy | **Yes** — sealed to boot-state PCRs | **Also yes** (admin can decrypt at rest for audit/config) | **Cold boot, automatic** (TPM unseal) so the alarm resumes unattended |
| **Sensitive** | `SENS` | Artha (finance), Ojas (health) | **No** | **Yes** | **First admin login** — stays sealed until a password unwraps it |
| **General / operational** | `GEN` | Kaya (devices/voice), Abhyasa (school), platform config, Saakshi audit logs | **No** (default; see §2.1) | **Yes** | **First admin login** |
| **Machine-trust** | `MTRUST` | Mesh/session keys, internal CA private key, device-identity material (owned by **Pramana**) | **Yes** — sealed to boot-state PCRs | **Yes** (admin recovery/rotation) | **Cold boot, automatic** so the mesh + CA come up before any human is present |

The **core safety property** (asserted as a test in §9):

> **P1 — Tier isolation.** The TPM-sealed life-safety DEK (`LIFE`) **cannot** decrypt an Ojas or Artha
> (`SENS`) blob. They are different AES keys; the blob header carries the tier/key id, and AES-GCM
> authentication *fails closed* when the wrong key is presented. TPM auto-unseal on boot therefore exposes
> **only** `LIFE` (and `MTRUST`); `SENS` and `GEN` remain ciphertext until an admin authenticates.

This is what makes "Kavach live, finance/health locked" real rather than aspirational: it is enforced by key
separation, not by application-layer access checks that a compromised process could skip.

### 2.1 Why `GEN` defaults to the password tier

`GEN` holds Kaya, Abhyasa, config, and the Saakshi audit log. A tempting optimization is to TPM-auto-unlock
`GEN` too, so the house (lights/voice) works before login. We **reject** that as the default:

- Saakshi audit logs and school records are **privacy-sensitive**; auto-unlocking them widens the cold-boot
  attack surface (a stolen, powered box would decrypt them without a password).
- The availability need for `GEN` is *soft* — it is acceptable for lights/voice to say "log in to continue"
  after a reboot; it is **not** acceptable for the alarm.
- Keeping only `LIFE` + `MTRUST` in the auto-unseal set keeps the TPM-exposed blast radius minimal.

`GEN` is therefore **password-tier by default**. The tier model supports splitting a narrow `GEN-BOOT` sub-tier
(TPM auto-unseal) if a concrete need appears — and one now has (S2): **§2.3** frames that decision, still OPEN.

### 2.2 Platform-store tier assignments — Dwarpal & Kutumba *(new 2026-07-11, round 2 — closes S1, S13)*

The §2 table covers the five **domain** services but omitted two **platform** stores that every request depends
on: **Dwarpal (Zitadel)** — its credential store and JWT signing keys — and **Kutumba** (roles + `guardianOf`
edges). Both are load-bearing for the wall and for boot; leaving them untiered let the credential store and
signing key silently drift into (or out of) the cold-boot auto-unseal blast radius (S1), and left Kutumba
with no unlock timing (S13). Assigned explicitly:

| Store | Tier / slice | Sealed? | Unlock timing | Rationale & honest trade-off |
|-------|--------------|---------|---------------|------------------------------|
| **Dwarpal JWT signing keys** | `MTRUST` | **Yes** — PCR-sealed | **Cold boot, automatic** | Token *validation* is a machine-trust function: automation/mesh tokens for `LIFE`/`MTRUST` services are minted and verified before any human is present, so the signing key must be live at boot — it joins `MTRUST`. **Forgery risk (documented):** a stolen, powered box holds a live signing key ⇒ an attacker who extracts it can mint valid platform JWTs. **Mitigations:** PCR sealing (tampering breaks unseal), tamper-detection → **zeroize** per **ADR 0008 §4**, short token lifetimes + rotation. Residual is the same powered-box window as `LIFE`/`MTRUST` (§10). |
| **Dwarpal credential-store DB** | dedicated **`DWARPAL-BOOT`** slice (MTRUST-adjacent) | **Yes** — TPM-auto-unsealed, shares the boot slice | **Cold boot, automatic** | It **must** be readable *before* first admin login, or boot is **circular**: unlock needs login, login needs to verify a credential, verification needs the credential store — which would be sealed until login. We break the cycle by auto-unsealing it. **Honest cost:** a stolen powered box exposes password/passkey **verifier** material — Argon2/bcrypt password *hashes* and passkey *public* keys — **not secrets**. Verifiers don't authenticate the holder anywhere (hashes are offline-crackable only, behind Zitadel's own hardened hashing; passkey public keys are public by design). We judge that acceptable against the alternative — an **unbootable** security appliance that cannot recover unattended from a reboot, a non-starter for a life-safety box. Kept a **dedicated slice** (not folded into `MTRUST`) so its exposure is *named and bounded*, not mixed with mesh/CA material. |
| **Kutumba** (roles + `guardianOf`) | `GEN` | **No** | **First admin login** | Kutumba must be readable whenever **any walled read** can occur — every may-see call consults it. But the first walled read *is* the admin login; **no walled read happens pre-login** (state it plainly: nothing legitimately reads a member's walled data before a member authenticates). So `GEN` (password-tier) suffices and keeps Kutumba out of the auto-unseal set. **Constraint (tripwire):** *if* a future ADR 0008 boot step ever introduces a **pre-login walled read**, Kutumba must move into whatever boot slice Dwarpal lands in (`DWARPAL-BOOT`/`MTRUST`) — it cannot be sealed while a wall it feeds is being evaluated. Not a current need. |

**Blast-radius note.** Assigning the signing key to `MTRUST` and the credential DB to `DWARPAL-BOOT` **enlarges the
cold-boot auto-unseal set** from `{LIFE, MTRUST}` to `{LIFE, MTRUST, DWARPAL-BOOT}` (plus the signing key inside
`MTRUST`). This **extends — it does not contradict — §10's auto-unseal trade-off**: the added exposure is
*verifier* material (still not finance/health), narrowed by the same PCR-seal + tamper-zeroize host mitigations.
The boot chain that consumes these assignments is **ADR 0008 §8**, corrected by SDD unit **D7**.

### 2.3 The `GEN-BOOT` decision — **★ DECIDED: Option B (Ananth, 2026-07-11)** *(new 2026-07-11, round 2 — records S2)*

S2 surfaced a real, undecided fork: boot steps 4–5 want Kaya's HA connection alive so **Kavach Layer 1**
(sensor-based intrusion sensing) and device availability work after an unattended reboot — but Kaya's config is
`GEN`, sealed until login. Either a narrow slice auto-unseals, or Layer 1 is honestly down until an admin logs
in. This is a **security-vs-availability trade the owner must make**, not one to let drift. Both options, equal
rigor:

**Option A — a narrow `GEN-BOOT` sub-tier (recommended, but OPEN).** A new TPM-auto-unsealed slice (blob key id
`0x05`, from §5's reserved range) holding **only** `{Kaya's HA connection config, the HA long-lived token, the
HA entity-registry cache}` — nothing else from `GEN` (logs, school records stay password-`GEN`). Gains:
**pre-login Layer-1 Kavach sensing + device availability** — the house senses intrusion and actuates the moment
power returns, no human required. **Blast radius:** a stolen, **powered** box holds a **live HA credential** to
every lock and sensor. **Mitigations:** HA lockdown (scoped token, no admin API, allowlisted entities) per
**ADR 0008 §6**; long-lived-token **rotation**; **network binding** of the credential to the box's mesh
identity. **Residual (honest):** within the powered-box window, an attacker who extracts the token via DMA/RAM
(§10) controls locks/sensors until rotation — bounded by the §10 host mitigations plus HA-side scoping, but
real.

**Option B — no `GEN-BOOT`.** Nothing from `GEN` auto-unseals. After every reboot, **Kavach Layer 1, cameras,
notifications, and voice are honestly DOWN until admin login**; only **Layer 0** (the standalone panel/keypad,
HA-independent per **ADR 0008 §2**) protects the house. No enlarged blast radius. This is only honest if **ADR
0008 §8's "what works while LOCKED" boot table is corrected** to say Layer 1 is down pre-login — cross-ref: SDD
unit **D7** performs that correction.

**Recommendation was Option A. ★ DECISION: OPTION B — Ananth, 2026-07-11 ("Option B, green light").**
Nothing from `GEN` auto-unseals; the auto-unseal set stays `{LIFE, MTRUST, DWARPAL-BOOT}` and no live HA
credential exists on a stolen powered box. **Accepted cost, stated plainly (the never-surprised rule):** after
every unattended reboot, Kavach Layer 1, cameras, notifications, and voice are **down until an admin logs in**
— only Layer 0 (panel/keypad/siren) protects the house in that window. **Tripwire:** if post-reboot blindness
proves painful in practice, reopen this section — the Option A design above is retained verbatim as the
ready-made alternative (blob key id `0x05` stays reserved). ADR 0008 §8/§9 resolve to **Branch B**; ADR 0010
§8's pre-login satellite behavior resolves to the generic on-device offline state.

---

## 3. Key wrapping & recovery (per DEK)

Every DEK is wrapped **independently and multiply**. For each DEK there is a set of **wrappers**; any one
wrapper can unwrap that DEK. Wrappers come in three kinds:

1. **Admin-password wrapper** — `KEK = IKeyDerivation.Derive(password, salt, params)`; `wrap = AES-GCM(KEK,
   DEK)`. One per admin **per DEK**.
2. **TPM wrapper** (only for `LIFE` and `MTRUST`) — the DEK is sealed by the TPM to a set of boot-state PCRs
   (and released only when those PCRs match). Materially, the wrap material lives in **Pramana's vault**
   (OpenBao/Vault); the **machine-trust subset** of that vault is TPM-auto-unsealed on boot, which is exactly
   what releases `LIFE` and `MTRUST`. See Pramana's internal trust design documentation.
3. **Recovery wrapper** — one per DEK, derived from a high-entropy recovery code with a **dedicated hardened
   KDF** (§8), shown once at provisioning and stored offline.

**Multi-admin, per DEK.** Because wrapping is N-of-set, each DEK carries one admin-password wrapper for
**Ananth** and one for **the other adult**. Adding/removing an admin, or a password change, **re-wraps only the DEK(s)
that admin can reach** — it never touches the DEK plaintext or other admins' wrappers. A password change is a
**re-wrap**, not a re-encrypt: derive the new KEK, unwrap the DEK with the old KEK (in memory), re-wrap with
the new KEK, replace that one wrapper.

**Recovery, per DEK.** Each DEK has its **own** recovery wrapper, so a single recovery code (or a small booklet
of per-tier codes) can restore access even if all admin passwords are lost. Recovery of `LIFE`/`MTRUST` via the
recovery code is possible even when the TPM is wiped/replaced (the TPM wrapper is an *availability* path, not
the *only* path). Recommended presentation: **one recovery code per tier** shown once, so a leaked single tier
code (e.g. `GEN`) does not expose finance/health.

**DEK rotation** re-encrypts **lazily** via `keyVersion` (see §7): a rotated DEK gets a new `keyVersion`; new
writes use it; old blobs are re-encrypted opportunistically on next write or by a background sweeper. Rotation
is **per tier** — rotating `SENS` never rewrites `LIFE` blobs.

**Crypto-shredding hook.** The tier DEK is the *coarse* key. The model deliberately **does not preclude
per-owner / per-record sub-keys** derived under a tier DEK (e.g. a child-specific sub-key under `GEN`, or a
per-account sub-key under `SENS`). Destroying such a sub-key renders just that owner's/record's data
unrecoverable — the mechanism behind right-to-forget. Full detail is deferred to the data-lifecycle ADR
(**0009**); this ADR only guarantees the tier design leaves room for it (sub-keys wrap under the tier DEK, and
the blob header's key id can name a sub-key).

---

## 4. `[Encrypted]` key selection mechanism

`Manas.Kernel` exposes the `[Encrypted]` attribute; a central EF Core `ValueConverter` (backed by
`Manas.Crypto`) transparently encrypts/decrypts marked properties at the DB boundary (ADR 0001; see the Kernel
design documentation). With multiple DEKs the converter must
know **which tier** each property belongs to. Two layered mechanisms:

1. **Explicit `KeyTier` on the attribute (wins if present):**
   `[Encrypted(KeyTier.Sensitive)] public string AccountNumber { get; set; }`.
   `KeyTier` is an enum `{ LifeSafety, Sensitive, General, MachineTrust }`. This is the override for a property
   whose sensitivity differs from its owning service's default.

2. **Per-service default tier mapping (fallback):** each service registers its **default tier** with the
   Kernel at startup via an `ITierMap` (e.g. `Artha → Sensitive`, `Ojas → Sensitive`, `Kavach → LifeSafety`,
   `Kaya/Abhyasa/config/Saakshi → General`, Pramana mesh/CA → `MachineTrust`). The resolver keys off the
   entity's owning **DbContext / assembly**, so a bare `[Encrypted]` inside Artha resolves to `Sensitive`
   with no per-property annotation.

**Resolution order at the converter:** explicit `KeyTier` on the attribute → per-entity override in
`ITierMap` → per-service default → **fail closed** (no default tier ⇒ throw at startup, never silently pick a
key). The resolved tier selects the DEK; on **write** the converter stamps the tier's key id + version into the
blob header (§7); on **read** it reads the header's key id and **requires** the matching, currently-unsealed
DEK — if that tier is still sealed (e.g. `SENS` before login), the read raises a typed `TierLockedException`
rather than returning garbage or a wrong-key result.

Startup **validation**: the Kernel enumerates every `[Encrypted]` property and asserts each resolves to exactly
one tier; an unmapped property is a build/boot failure. This makes tier assignment auditable and prevents a new
entity from silently landing in the wrong (or a TPM-auto-unlocked) tier.

### 4.1 Sarathi (the Agent) — the canonical mixed-tier mapping *(new 2026-07-11, round 2 — closes R15 / §AM item 1)*

ADR 0010 §3 makes **Sarathi** a first-class walled service that owns conversation data — the platform's most
sensitive *new* data class (cross-domain chat can echo balances and lab values). It is the first service whose
own data **splits across two tiers**, so it is the reference case for the §4 mechanism:

| Sarathi data | Tier | How selected |
|------------|------|--------------|
| Conversation **content** — turn text, rolling summaries, extracted `MemoryFacts` | **`Sensitive`** | Sarathi's **default** tier in `ITierMap` (`Sarathi → Sensitive`), so a bare `[Encrypted]` on content is conservative by default |
| Conversation **metadata** — encrypted keys, timestamps, correlation ids / cids | **`General`** | explicit **`[Encrypted(KeyTier.General)]`** overrides on those properties (the §4 "sensitivity differs from the service default" case) |

`SENS` and `GEN` both unlock at first admin login, so the conservative content tier costs **no** availability
(ADR 0010 §3). **Per-record sub-keys under the `OwnerId`** (the speaking member) hang off the tier DEK via §3's
crypto-shredding hook, so right-to-forget crypto-shreds a member's conversational footprint — **including the
subject fan-out** (a turn whose `dataSubjectIds` names another member is shredded on *that* member's forget).
Full wiring lives in **ADR 0010 §3** (owner/subject model, retention) and **ADR 0009** (data lifecycle &
crypto-shredding); raw audio is never persisted (ADR 0009 §3), so only these rows and the Qdrant points carry
the content.

---

## 5. Blob format

The at-rest blob layout gains a **key-tier / key-id** field so every ciphertext is self-describing and the
right DEK is selected on read (and the wrong one provably rejected):

```
keyVersion(1) | keyTier(1) | nonce(12) | ciphertext(N) | tag(16)
```

- `keyVersion` — `byte`; the DEK generation within a tier (for lazy re-encryption on rotation).
- `keyTier` — `byte`; the tier/key-id: `0x01 LIFE`, `0x02 SENS`, `0x03 GEN`, `0x04 MTRUST` (reserved high
  values for future sub-keyed tiers). This is the field that makes **P1** enforceable and observable.
- `nonce` — random **96-bit** value, fresh **per operation** (never reused under a given DEK).
- `ciphertext` / `tag` — `System.Security.Cryptography.AesGcm` output; the tier byte + keyVersion are fed as
  **AES-GCM associated data (AAD)**, so a tampered/mismatched tier byte fails authentication (it cannot be
  swapped to trick the reader into using a different key).

On read: parse `keyTier` → look up the unsealed DEK for that tier (or `TierLockedException` if sealed) → verify
`keyVersion` → AES-GCM decrypt with the tier byte + version as AAD. Presenting a `LIFE` DEK against a `SENS`
blob mismatches both the selected key **and** the AAD and fails closed — **P1** holds at the byte level.

---

## 6. (reserved)

*Section intentionally omitted to keep numbering aligned with the review checklist; wrapping is §3, selection
§4, format §5.*

---

## 7. Argon2 package decision (review B2)

*(amended 2026-07-11, round 2 — decision FLIPS per ST5)*

.NET's BCL has **no Argon2**. `IKeyDerivation` already abstracts the KDF (with PBKDF2-SHA256 as the pure-BCL
fallback), so the only question is which Argon2id implementation backs the primary adapter. Options weighed:

- **Managed** — `Konscious.Security.Cryptography` or Isopoh. Pure-.NET, no native dependency, trivially
  portable, easy to audit. **Slower**; and — decisively — `Konscious` **last released 2024-06** (two years
  stale, single maintainer) on a **KEK-load-bearing** path.
- **Native** — libsodium via **NSec.Cryptography** (26.4.x). Actively maintained, audited native core
  (libsodium), constant-time, and **faster** (which also helps size the Kavach PIN profile). Adds a **native
  dependency** (per-arch binaries; must ship the linux-x64 libsodium for the home server) and a small supply-chain +
  packaging cost.

**Decision (flipped): use `NSec.Cryptography` (libsodium) Argon2id as the primary `IKeyDerivation` adapter.**
Round 1 chose managed `Konscious` for audit-everything, minimal-native-surface purity. **ST5 (verified) reverses
that:** the managed-purity argument **loses to a two-year-stale dependency on the key-derivation path the whole
platform's confidentiality hangs on.** An unmaintained KDF on the KEK path is a worse risk than one vendored,
audited native library. NSec is actively maintained (26.4.x), its core is audited (libsodium), and it is
faster — which also helps keep the Kavach PIN profile under the entry-delay budget. Concretely: `NSec.Cryptography
26.4.x` in, `Konscious 1.3.1` out (tech-stack **D5**).

- **`IKeyDerivation` is unchanged.** The port already isolates the KDF, so this is a **one-line adapter swap**
  with no call-site changes — the exact property that made round 1 comfortable deferring it.
- **PBKDF2-SHA256 stays as the pure-BCL fallback** behind the port, for environments where the native library
  can't ship or as a break-glass path.
- **Native-dep cost, stated honestly:** we now ship and pin a per-arch libsodium (linux-x64 for the home server),
  adding a supply-chain surface and a packaging step. Accepted as the **lesser** risk versus a stagnant KDF.
- **The three KDF profiles are unchanged** (see §8): **`Interactive`** (login/unlock), **`Pin`** (Kavach
  disarm — low-entropy and online rate-limited/lockout-protected, so a **lighter** profile sized to keep disarm
  well under the entry-delay budget), **`Recovery`** (hardened, offline-grade). Tier `LIFE` unseal on boot is via
  the **TPM**, not the PIN, so the PIN KDF is never on the cold-boot availability path.

---

## 8. Recovery-KDF hardening (review C4 / risk #24)

The recovery wrapper is the **weakest wrapper offline**: an exfiltrated off-site backup can be attacked offline
against whichever wrapper has the lowest work factor, with **no online lockout to slow the attacker**. If the
recovery code were derived at *interactive-login* cost, a GPU farm would grind it.

**Decision:**

- The recovery code is a **high-entropy, system-generated** secret (≥ 128 bits; grouped Crockford-base32,
  shown once, offline-stored). It is **never** a human-chosen phrase.
- Its wrapper uses a **dedicated, deliberately expensive KDF profile** — a distinct `IKeyDerivation`
  configuration tuned **against offline GPU/ASIC cracking**, materially higher memory + iteration cost than the
  interactive-login profile (targeting seconds of derivation on the box, accepting that recovery is a rare,
  patience-tolerant event).
- Because the code carries full entropy, the KDF is defense-in-depth, not the primary barrier — but with no
  online lockout offline, **a high work factor is the only defense** if the code's entropy is ever partially
  known or the backup medium is stolen intact.
- Recovery-code entry is **online-rate-limited too** (so an online guess against the running box is
  lockout-bounded), but the design assumes the *offline* threat and sizes the KDF for it.

Three named KDF profiles therefore exist behind `IKeyDerivation`: **`Interactive`** (login/unlock),
**`Pin`** (Kavach disarm, lighter), **`Recovery`** (hardened, offline-grade).

---

## 9. Acceptance criteria / property tests

Concrete, buildable assertions `Manas.Crypto` must satisfy (each maps to a unit/property test):

1. **P1 tier isolation (the headline).** Encrypt a blob under `SENS` (an Ojas record); attempt decryption with
   the `LIFE` DEK ⇒ the operation **fails** (AES-GCM auth failure / typed rejection). *A unit test proves
   decrypting an Ojas blob with the life-safety key fails.*
2. **Selective boot exposure.** Simulate cold-boot TPM auto-unseal ⇒ `LIFE` and `MTRUST` DEKs are available;
   `SENS` and `GEN` reads raise `TierLockedException` until an admin login unwraps them.
3. **Multi-admin per DEK.** A DEK wrapped for both Ananth and the other adult unwraps with **either** password;
   removing Ananth's wrapper leaves the other adult's intact and vice-versa.
4. **Password change is re-wrap, not re-encrypt.** After a password change, existing ciphertext is byte-for-byte
   unchanged (same `keyVersion`), yet unwraps under the new password and no longer under the old.
5. **Per-tier recovery.** Each tier's recovery code unwraps **only** its own DEK; a `GEN` recovery code cannot
   unwrap `SENS`.
6. **Blob self-description + AAD binding.** Flipping the `keyTier` byte of a stored blob causes decryption to
   fail (tier byte is authenticated as AAD).
7. **Nonce freshness.** Two encryptions of identical plaintext under the same DEK produce different nonces (and
   ciphertext); no nonce reuse across a large sample.
8. **Fail-closed tier resolution.** A `[Encrypted]` property with no explicit `KeyTier` and no service default
   causes a **startup** failure, never a silent default key.
9. **Lazy rotation.** After a `SENS` DEK rotation, old-`keyVersion` `SENS` blobs still decrypt; new writes carry
   the new `keyVersion`; `LIFE`/`GEN` blobs are untouched.
10. **Zeroization.** On lock, each unsealed DEK's in-memory buffer is zeroed (assert via a test double / memory
    inspector).

---

## 10. Residual risks

- **DEKs in RAM on a running box (review C3 / risk #21).** A running, **unlocked** box necessarily holds the
  unsealed DEKs (`LIFE`, `MTRUST`, and — after login — `SENS`/`GEN`) in RAM. They are recoverable by a
  **cold-boot RAM-remanence attack** or **DMA** over OCuLink / USB4 / Thunderbolt on the home server. **We state this
  honestly: no at-rest key scheme defends a powered, unlocked machine.** Mitigations are *availability/host*
  concerns and belong to the **availability ADR (0008)**: IOMMU on + disable Thunderbolt/OCuLink DMA (or
  allowlisted only), TPM-seal `LIFE`/`MTRUST` to **PCR boot state** so tampering breaks unseal, physical
  tamper-detection → **auto-lock / zeroize**, and minimizing the window `SENS`/`GEN` stay unsealed (idle
  re-lock). This ADR guarantees the key *structure* that those host mitigations protect; it does not itself
  solve the powered-box case. Cross-reference **ADR 0008**.
- **`LIFE`/`MTRUST` auto-unseal is a deliberate availability↔confidentiality trade.** A stolen, powered box
  exposes alarm + mesh/CA material (not finance/health). Accepted because the alarm and mesh **must** survive an
  unattended reboot; blast radius is bounded to the two tiers whose confidentiality we consciously trade for
  availability, and PCR-sealing + tamper-zeroize (0008) narrow it.
- **Recovery backup theft.** Mitigated by §8 (full-entropy code + offline-grade KDF); residual is the medium's
  physical security, which is the owner's responsibility (documented in the runbook).

---

## 11. Open questions

1. **`GEN-BOOT` sub-tier — OPEN, Ananth to pick.** *(updated 2026-07-11, round 2 — S2)* The concrete need has
   now appeared (S2: pre-login Kavach Layer-1 sensing + device availability). The decision is fully framed in
   **§2.3** — **Option A** (a narrow TPM-auto-unsealed slice of `{HA config, HA token, entity-registry cache}`,
   recommended) vs **Option B** (Layer 1 honestly down until login). No longer "deferred until a need appears";
   it is a **live decision awaiting the owner's call**, and the ADR 0008 §8 boot table (SDD unit D7) branches on
   the answer.
2. **Recovery-code presentation** — one combined booklet vs strictly one code per tier. Leaning per-tier for
   blast-radius; confirm with Ananth for usability.
3. **Sub-key granularity** (per-owner vs per-record) and its performance cost — owned by **ADR 0009**; this ADR
   only reserves header space and the wrap-under-tier-DEK path.
4. **PIN KDF exact profile** — Argon2id-light vs PBKDF2-moderate for Kavach disarm; settle by measuring disarm
   latency against the entry-delay budget on the home server.
5. **TPM PCR selection** — which PCRs `LIFE`/`MTRUST` seal to (firmware/bootloader/kernel measure set); jointly
   owned with **ADR 0008** and Pramana.

---

## 12. Consequences

- **Unblocks `Manas.Crypto`.** The A1/#19 contradiction is resolved; the library can be implemented directly
  from §3–§5 and validated against §9.
- **Hybrid unlock is now real, not aspirational** — "Kavach live after reboot, finance/health sealed until
  login" is enforced by key separation (P1), not by trusting application code.
- **Slightly more machinery**: four DEKs, an `ITierMap`, a `KeyTier` enum, one extra blob byte, and three KDF
  profiles. This is a modest, testable cost for a hard security guarantee, and it is invisible to service code
  (a bare `[Encrypted]` in Artha still "just works" via the per-service default).
- **Forward-compatible** with crypto-shredding sub-keys (ADR 0009) — the tier DEK is the coarse key and the
  header can name a sub-key.
- **Constrains ADR 0008** (availability): it must deliver the host-level DMA/tamper/PCR mitigations this ADR's
  residual-risk section depends on, and **Pramana** (the trust/credential layer)
  owns the vault path that releases `LIFE`/`MTRUST` on boot.

---

*Refines [`0001-manas-platform-architecture.md`](0001-manas-platform-architecture.md) §15. Cross-references
the Pramana trust design and the Kernel design documentation, and the forthcoming ADR 0008 (availability
& host hardening) and ADR 0009 (data lifecycle & crypto-shredding).*
