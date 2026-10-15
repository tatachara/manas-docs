# ADR 0003 — Identity Propagation, Delegation & Internal PKI

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** [ADR 0001](0001-manas-platform-architecture.md) §5.2 (Sarathi, MCP & identity propagation) and
  §5.4 (service identity & attribution); the internal PKI/trust design (adopted here as decided). Resolves design-review
  A2 / risk #18, **C6** (PKI/cert lifecycle), and **D8**
  (stolen-phone / relay-token hardening).

> Identity in Manas must be **cryptographically verified at every hop, never merely carried**. The acting human
> travels as a Dwarpal-issued, audience-scoped, short-lived **delegation token**; every callee re-validates it and
> **derives** roles/identity from it, never trusting request fields. This is the platform's load-bearing
> guarantee — that one prompt-injected Sarathi or one compromised service can **never** become a super-user — made
> real instead of implicit. It also folds in the internal PKI (Pramana) that authenticates the *service* side.

---

## 1. Context

ADR 0001 §5.2 states the spine: every cross-service call "carries the verified acting user," Sarathi is "a
deputy, never a super-user," and the LLM is "never the access-control decision." Design review **A2 (risk #18,
CRITICAL)** found that this is **carried but never verified downstream**. Two distinct things are conflated:

- **mTLS authenticates the *service*** (the TLS peer cert = `ServiceId`). It says nothing about *which human* the
  call is on behalf of.
- **`ActorContext`** (ambient, immutable, per-operation: `TenantId`, `UserId`, `ServiceId`, `Roles`,
  `CorrelationId`) carries the acting user — but if a callee reads `UserId`/`Roles` as **plain fields** of the
  incoming request, then **any** service that can open an mTLS connection can *assert any user and any role*.

The consequence is a classic **confused deputy**: one compromised service — or Sarathi following a
prompt-injected instruction — mints an `ActorContext{UserId: ananth, Roles: [Admin]}` and the callee obeys.
The "a curious child or a bad prompt fails at the data layer" guarantee (ADR 0001 driver #5) collapses, because
the data-layer wall filters on a `UserId` the caller *chose*. mTLS is necessary but **not sufficient**; it must
be paired with **verifiable user delegation**.

Separately, review **C6** noted the pervasive mTLS mesh has **no documented CA / issuance / rotation / expiry /
cold-boot** story, and **D8** noted the remote (phone → relay) path can leak a long-lived "key to reads." This
ADR owns all three because they are one question — *how does a callee know **who** (human) and **what** (service)
it is really talking to* — and they are a hard prerequisite for the Phase-0 walking skeleton (which is meant to
*retire* the identity risk and today cannot).

---

## 2. Decision — the delegation token *(amended 2026-07-11, round 2 — closes G5/S9/S3/G8)*

The acting user travels as a **delegation token**: a Dwarpal-issued, signed, **audience-scoped, short-TTL** JWT
modelled on **OAuth 2.0 Token Exchange (RFC 8693)** with `act`/`sub` delegation semantics.

**Claims (the contract):**

| Claim | Meaning | Notes |
|---|---|---|
| `iss` | Dwarpal (the one self-hosted IdP) | the **only** minter; validated against Dwarpal's JWKS |
| `sub` | the **acting human** (`UserId`) | the person on whose behalf the call runs; = `ActorContext.UserId` |
| `act` | the **acting service** (`ServiceId`) — the *delegate* | nested `act` chains allowed (Sarathi → Artha → Ojas), each hop appended, never rewritten |
| `aud` | the **specific callee** service | an Artha-audience token is **rejected** by Ojas; blocks token replay across services |
| `tid` | `TenantId` (household) | the tenant seam; enforced alongside `sub` |
| `roles` | `Admin` / `Adult` / `Child` | **the authoritative source of roles** — callees read roles from *here*, never from the request/ActorContext |
| `scope` | operation grant(s) the delegate may exercise for `sub` | e.g. `artha.read`, `kaya.actuate`; the "may `act` act for `sub` on *this* op" check |
| `acr` | the **identity-assurance class** the token achieved (`hint \| session \| strong`) | standard OIDC assurance claim (RFC 8693 tokens may carry it), reused unchanged; **stamped by Dwarpal at mint**, never asserted or raised by a service; an `acr=hint` token can never satisfy T2+ (below) |
| `amr` | the **authentication methods** that backed it (array: `passkey \| faceid \| pin \| satellite-map \| guest`) | standard OIDC claim; how the `acr` was earned; `[satellite-map]` or `[guest]` ⇒ `acr=hint` by construction |
| `subject` *(optional)* | the **human data-subject** an automation firing concerns (`§2.2`) | **delivery-targeting & audit only** — a callee **never** reads it for wall evaluation; present only on rule-owner tokens |
| `cid` | `CorrelationId` | bound into the token; ties the whole hop chain in Saakshi |
| `exp` / `iat` / `jti` | short TTL, issued-at, unique id | **minutes, not hours** (target ≤ 5 min); `jti` for replay caps / revocation |

**TTL & refresh.** Delegation tokens are **very short-lived** (single-turn / minutes). A conversational turn or
an ECA action obtains one at the edge; long operations re-exchange rather than hold a long token. Dwarpal exposes a
token-exchange endpoint: a service presents its **own mTLS identity** + the **inbound** delegation token and
receives a **downstream** token whose `aud` is the *next* callee and whose `act` chain has this service appended.
**Minting is Dwarpal-only** — no service can self-issue or self-elevate; a service without a valid inbound token
**cannot** obtain a downstream one.

**Assurance (`acr`/`amr`) — closes G5/S9.** `acr`/`amr` are the **standard OIDC assurance claims** (an RFC 8693
token may carry them), reused here unchanged — not a Manas invention. **Dwarpal stamps `acr` at mint**, from the
factor actually verified at the edge; a service can never assert it, raise it, or forge a higher class. A
**voice-turn token minted from satellite speaker-mapping alone is ALWAYS `acr=hint, amr=[satellite-map]`** — a
room hint is not proof of a person. **An `acr=hint` token can NEVER satisfy a T2-or-higher operation, regardless
of its `scope`** — the callee enforces this as a hard floor (§3, the delegation-legitimacy check): a hint-grade
`sub` carrying a broad `scope` still cannot climb, because assurance is checked *below* scope, not traded against
it. **Step-up upgrades assurance by RE-MINTING** — Dwarpal issues a fresh token with a higher `acr` only *after* it
verifies the factor (ADR 0007); a live token's `acr` is immutable once signed and is **never mutated in place**.
The per-path assurance mapping (which channel earns which `acr`) is owned by
[ADR 0010 §5](0010-agent-conversational-core.md).

### 2.1 Token lifetime mechanics *(amended 2026-07-11, round 2 — closes S9)*

**One token is minted per TURN at the edge** (§4). Within that turn Sarathi performs **token-EXCHANGE per tool
call** — each MCP call presents the inbound edge token and receives a downstream token whose `aud` = the callee
and whose `act` chain has Sarathi appended (§2, RFC 8693 semantics). Exchanges are valid **only while the edge
token is unexpired**: the edge token is the root of the chain and a downstream token never outlives it.

- **A turn that outlives the edge-token TTL re-exchanges against the still-valid edge SESSION.** On console/web,
  Dwarpal holds a live passkey **session** that is the freshness anchor; the turn obtains a *fresh* edge token from
  that session and continues — it never coasts on an expired token.
- **A satellite hint has no session to renew against** — a room mapping is not an authenticated session. When a
  satellite `acr=hint` edge token expires mid-turn the turn **ends honestly**; the *next* utterance mints a fresh
  hint token. There is no silent renewal of an unauthenticated identity.
- The **≤5-min TTL is per-token** (§2, `exp`) — a bound on any single token's blast radius, **not** the length of
  a conversation. **The session, not the token, is Dwarpal's freshness anchor:** token expiry drives re-exchange,
  session expiry drives re-auth. (This composes with ADR 0010 §5's "silent re-exchange while the edge session
  lives; expired edge session interrupts with re-auth.")

### 2.2 Automation-principal tokens *(amended 2026-07-11, round 2 — closes S3)*

For an **ECA rule firing** ([ADR 0006 §2](0006-automation-principal-and-event-bus.md)), the delegation token's
**`sub` = the RULE-OWNER PRINCIPAL id (`rule:{ruleId}`), NOT a human** — the actor is the named least-privilege
principal, never the author and never a human super-user. Where a firing concerns a specific human data-subject,
that person rides the **optional `subject` claim** (a nudge computed for the child ⇒ `subject = child`):

- **`subject` is for delivery targeting and audit ONLY.** A callee uses it to route/deliver (whose phone the
  nudge goes to) and to attribute the firing — **NEVER to evaluate the wall.**
- **The wall evaluates the PRINCIPAL's machine-grants**, not `subject`. The rule-owner principal's own scope +
  its active machine-grant covering the row decides the read (ADR 0004 **row 10**, which unit **D3** adds); the
  rule sees exactly what its enumerated grants cover and nothing more.
- This **kills the row-1 self-access laundering** (S3): the old "`sub`/`UserId` = subject" mechanism would have
  tripped the kernel's self-access disjunct and handed a rule **all** of its subject's rows. With `subject` inert
  for authorization, the subject's scope rides the **grant**, not the wall's self-access path.

### 2.3 Guest tokens *(amended 2026-07-11, round 2 — closes G8)*

An **unknown speaker** at a satellite (no resolvable identity) is never left as a bare unauthenticated context.
Dwarpal mints an **ephemeral guest token**:

- **`sub = guest:{sessionId}`** (a per-session synthetic principal, never a human `UserId`), **`acr = hint`**,
  **`amr = [guest]`**, **`tid` bound** to the household, **TTL = the turn**, **`roles` claim = `Guest`** (the
  actorRole [ADR 0010 §5](0010-agent-conversational-core.md) / ADR 0007 add).
- **T0-only by construction:** a guest token cannot be exchanged for any audience with a scope beyond T0 tools —
  `acr=hint` already bars T2+ (above), and the `Guest` role floor is child-strict (cross-ref
  [ADR 0007](0007-risk-tiers-and-step-up.md) Guest-role amendment).
- **Rationale:** enforcement never sees an unauthenticated bare context — every turn, even a stranger's, arrives
  as a signed, tenant-bound, T0-capped token that each callee validates by §3 exactly like any other. There is
  no "no token" code path at a callee.

---

## 3. What a callee **trusts** vs. what it **derives** *(amended 2026-07-11, round 2 — closes G5)*

This is the crux of A2. At a callee, **`ActorContext` is a *product of verification*, not an input.**

**On every inbound call, the callee independently validates (fail-closed):**

1. **Signature & issuer** — valid signature from Dwarpal's current JWKS; `iss` = Dwarpal. (Reject unknown/rotated-out keys.)
2. **Audience** — `aud` == *this* service. A token minted for another callee is rejected outright.
3. **Expiry / not-before** — within `iat`/`exp`; small clock skew only. Expired ⇒ reject (no grace beyond skew).
4. **Delegation legitimacy** — the `act` service is **permitted to act for `sub` on this operation** (`scope` +
   policy). Sarathi may `kaya.set_lights` for Ananth; it may **not** `artha.move_money` for Ananth without a
   token whose `scope` says so — and that scope only exists if Dwarpal's step-up (ADR 0007) granted it.
   **Assurance floor (G5):** the token's `acr` must meet the operation's minimum assurance class. **An `acr=hint`
   token is rejected for any T2-or-higher operation regardless of `scope`** — assurance is checked *below* scope,
   never traded for it, so a broad-scope hint token still cannot climb (the enforcement behind §2's mint-time
   rule and acceptance test §8.12). For a rule-owner token (§2.2), the wall reads the **principal's grants**; the
   `subject` claim is inert here — it is delivery/audit metadata, never an authorization input.
5. **Replay / revocation** — `jti` not seen beyond a bounded window; token not on the revocation list
   ("sign out all devices," §6).

**Only after all pass** does the callee **construct** its `ActorContext` **from the verified claims**:
`UserId ← sub`, `Roles ← roles`, `TenantId ← tid`, `ServiceId ← ` the caller's **mTLS cert identity** (Pramana,
§5), `CorrelationId ← cid`. The per-user data-layer wall (Kernel global query filter, ADR 0001 §5.4) then filters
on **this derived `UserId`** — which the caller *could not forge*, because forging it would require an
Dwarpal-signed token for that user with a matching audience.

> **Rule:** roles and acting-user come from the **validated token**. A `UserId`/`Roles` value that appears only
> in the request body, a header the callee didn't verify, or a hand-built `ActorContext` is **inert** — it is
> never read for an authorization decision. `ActorContext` at a callee is *derived state*, not *trusted input*.

The two identities stay distinct and are **both** recorded (ADR 0001 §5.4 provenance): `sub` → `CreatedBy.User`,
mTLS-cert `ServiceId` → `CreatedBy.Service`. mTLS answers "*which service is on the wire*"; the delegation token
answers "*for which human, with what right*." Neither substitutes for the other.

---

## 4. Gateway (composed GraphQL schema) & MCP forwarding *(amended 2026-07-11, round 2 — closes G5)*

**The GraphQL gateway (one composed HotChocolate schema — Fusion rejected per tech-stack D3; forwarding semantics unchanged).** The gateway is a **forwarder, not a re-minter**. It receives the caller's
delegation token at the edge and **forwards it unchanged** to each subgraph; **each subgraph re-validates
independently** (§3) against its own audience. The gateway does **not** strip the token and speak to subgraphs
"as the gateway" — that would *launder* identity (every subgraph would see the gateway as the user, the exact
confused-deputy we are killing). Where a subgraph fan-out needs a narrower audience, the gateway performs an
**Dwarpal token-exchange** to obtain a per-subgraph-audience token (still `sub`-preserving, `act` = gateway
appended) — it **never** hand-crafts claims.

**MCP tool calls (Sarathi).** Sarathi is the MCP client and orchestrates across services — but it **holds no
super-token**. Each tool invocation carries the **caller's** delegation token (obtained for the current speaker's
turn), audience-scoped to the target service, `act` = Sarathi. The target service validates it exactly as any
other callee. Therefore a **prompt-injected Sarathi** can only ever act **within the current speaker's own rights**:
it cannot read the other adult's finances during the child's turn, because it has no token whose `sub` is that adult.
The LLM is structurally incapable of widening access — the boundary is the token, not the model's good behaviour.

**Edge minting.** The initial token is minted at the **verified edge** only: a UI/GraphQL request authenticated
by Dwarpal (passkey/session) ⇒ `acr=session`/`strong`; a voice turn after speaker-resolution + **channel-assurance**
— the previously-undefined "channel-assurance" is precisely ADR 0010 §5's **`channelAssurance` axis**
(`AuthenticatedWeb | AuthenticatedTrustedDevice | RoomSatelliteHint | None`); a satellite yields
`RoomSatelliteHint` ⇒ `acr=hint` (§2), console-voice equals console-text assurance; or an ECA rule running under
its **named least-privilege rule-owner** identity (ADR 0006, §2.2) — never "as Ananth." The initial `acr`/`amr`
follow from the edge factor per that mapping; a guest utterance mints the §2.3 guest token.

---

## 5. Internal PKI & certificate lifecycle (adopting Pramana — resolves C6)

The **service side** of identity is mTLS, and mTLS is only as trustworthy as its PKI. Manas adopts its own
internal trust design as the **decided** answer. Summary of the commitments this
ADR ratifies:

- **Hierarchy.** Hardware root = the box **TPM** (firmware TPM). An **offline root CA** (YubiKey / air-gapped)
  issues **one intermediate**, then returns to a drawer. The **on-box intermediate CA** private key lives in the
  **vault** (OpenBao/Vault), never plaintext on disk.
- **Chosen internal-CA model: SPIFFE/SPIRE-style workload identity, issued by the on-box intermediate.**
  *Rationale for a single co-located box:* (a) it makes **the cert's SVID identity == `ServiceId`** — one
  identity for both the TLS peer and audit attribution (ADR 0001 §5.4), no second registry to drift; (b) it is
  built for **short-lived, in-memory, auto-rotated** workload certs, which is exactly our posture; (c) it needs
  no external CA service and chains cleanly to our own offline root. *(Rejected: `step-ca` — fine, but adds a
  second identity namespace to reconcile with `ServiceId`; Dwarpal-issued short-lived certs — overloads the human
  IdP with workload PKI and couples mesh liveness to the human-facing Dwarpal service.)* SPIRE-node attestation binds to
  the TPM.
- **Issuance & rotation.** Each service gets its cert **issued at startup**, held **in memory only** (no
  long-lived cert files on disk), **short TTL (hours)**, **auto-rotated** well before expiry.
- **Expiry alerting.** A **silently-expired internal cert would wedge the whole mesh** (the failure this ADR
  refuses to allow). Bodha/Pramana **monitor every cert's TTL and re-issue ahead of expiry**; a failed rotation
  raises a **loud fault**, never a silent outage. This alerting is a Phase-0 acceptance item, not a "later."
- **Cold-boot bootstrap.** The chicken-and-egg (services need mTLS → certs need the CA key → the CA key needs the
  vault → the vault needs the master key → nobody has logged in) is resolved by tiering: the **machine-trust
  tier** (intermediate CA key + core-infra unseal + Kavach) is **TPM-sealed to PCR boot state**, so the **mTLS
  mesh and Kavach come up on a cold boot before any password**; the **data tier** (finance/health DEKs, vendor
  secrets) stays **sealed until the admin logs in**. Net: a trusted internal mesh + a live alarm at boot;
  sensitive data is ciphertext until unlock.

**Interaction with §3:** mTLS establishes the **channel** and the caller's `ServiceId`; the delegation token
rides **inside** that channel and establishes the **human**. A callee requires **both** — a valid peer cert
*and* a valid, audience-matching delegation token — before it constructs `ActorContext`.

---

## 6. Remote / phone token hardening (resolves D8)

The iOS app reaches the box only through the **blind cloud relay** (a managed tunnel technology; APNs is an opaque
"dumb doorbell"). A **stolen unlocked phone or an extracted Keychain token must not be a durable key to reads.**

- **Short-lived relay tokens.** The phone never holds a long-lived bearer credential. It authenticates to Dwarpal
  (passkey/biometric) and receives **short-TTL** access tokens; the refresh credential is **bound to the device**
  (below), so exfiltrating an access token yields minutes, not months.
- **Device attestation & binding.** Tokens are **cryptographically bound to the device** — the refresh/session
  key is generated in and **non-exportable from the Secure Enclave**, registered as a **passkey/WebAuthn**
  credential. Dwarpal verifies **device attestation** (App Attest / Secure Enclave) at enrollment and on high-value
  operations; a token presented from a different device key is rejected (no bearer replay off the phone).
- **Re-auth for sensitive *reads*, not just writes.** ADR 0007's step-up already gates sensitive *actions*; D8
  adds that **sensitive reads** (opening finance/health detail, viewing camera history) require a **fresh strong
  factor** — a stolen, already-unlocked phone still faces a biometric wall on the crown jewels.
- **Server-side "sign out all devices."** Dwarpal maintains a **revocation list** (by `jti` / device / session);
  the owner can **invalidate every outstanding token** for a user from any trusted device. Because delegation
  tokens are short-TTL and every callee checks revocation (§3.5), a "sign out all" propagates in **minutes**.
- **Relay carries no authority.** The relay is **blind**: it forwards ciphertext and cannot mint, read, or
  elevate a token. Compromising the relay yields traffic timing, never identity.

*(This section's device-attestation posture and its enrollment-ceremony details are refined by a later ADR; the
commitments above are the decisions this ADR made and stand as recorded.)*

---

## 7. IdP recommendation

The one self-hosted IdP (ADR 0001 §5 "Dwarpal") must do: **passkeys/WebAuthn** (first-class), **token-exchange with
`act`/`sub`** (RFC 8693) for delegation, **risk-based step-up**, and clean **.NET / OIDC** integration, on a
**single low-footprint household box**.

| | **Authentik** | **Keycloak** | **Zitadel** |
|---|---|---|---|
| Footprint (one box) | light (Python/Go, modest RAM) | heavy (JVM, ~1 GB+) | light (Go, Postgres) |
| Passkeys / WebAuthn | strong, easy | strong, mature | strong, passkey-first |
| Token-exchange (`act`/`sub`, RFC 8693) | partial / evolving | **yes** (mature, standards-complete) | **yes** (first-class) |
| Step-up / policy flows | good (flow builder) | strong (authn flows) | good |
| .NET / OIDC | standard OIDC | standard OIDC | standard OIDC + good docs |
| Fit | great UX, delegation less proven | most complete, heaviest | modern, light, delegation-strong |

**Recommendation: Zitadel** — it is **light** (Go + Postgres, fits the home server budget beside our own Postgres),
**passkey-first**, and has **first-class token-exchange / delegation** which is the *specific* mechanism this ADR
depends on. **Keycloak** is the fallback if a delegation/step-up edge case needs its maturity (we accept its JVM
footprint if so). **Authentik** is deprioritized *only* because RFC 8693 delegation is less proven there, despite
its excellent UX. **Open question OQ-1** confirms this after a token-exchange spike.

*(A later ADR revisits this recommendation in light of a decision to build enrollment natively rather than
against an off-the-shelf IdP immediately; that reconciliation is recorded there, not here.)*

---

## 8. Acceptance criteria / property tests (concrete negatives) *(amended 2026-07-11, round 2 — closes G5/S3/G8)*

The walking skeleton is **not** accepted until these pass — most are stated as things that **must be rejected**:

1. **Forged actor.** A service opens a valid mTLS connection to Artha and sends `ActorContext{UserId: otherAdult,
   Roles:[Admin]}` in the body/headers **with no matching delegation token** (or a token whose `sub` ≠ otherAdult).
   → **Rejected.** *Property:* "a service **cannot** assert a `UserId`/`Role` it does not hold a valid,
   audience-matching, unexpired delegation token for."
2. **Wrong audience.** A token minted with `aud = artha` is presented to **Ojas**. → **Rejected** (audience mismatch).
3. **Expired.** A token past `exp` (beyond skew) → **Rejected**; a turn mid-flight must re-exchange, not coast.
4. **Roles from token, not field.** A token with `roles:[Child]` accompanied by a request field `roles:[Admin]`
   → the callee authorizes as **Child**; the field is **inert**.
5. **Delegation scope.** Sarathi presents a token for the child with `scope` lacking `artha.move_money` and calls
   `artha.move_money`. → **Rejected** at the callee (delegation-not-permitted), independent of any LLM output.
6. **Prompt-injection containment.** During the **child's** turn, Sarathi is induced to read the other adult's
   finances. It holds only a `sub: child` token → the `artha` wall returns the **child's** empty/own scope;
   **no** other-adult data leaks.
7. **Self-mint refusal.** A service attempts token-exchange **without** a valid inbound token (or with its own
   cert only) → Dwarpal **refuses**; no downstream token is issued.
8. **Gateway forwarding fidelity.** For a fan-out through the composed-schema gateway, each module's derived `ActorContext.UserId`
   equals the original edge `sub` (identity **not** laundered to "gateway").
9. **Revocation latency.** After "sign out all devices," a previously-valid token is **rejected within one TTL /
   revocation-window** at every callee.
10. **Cert-expiry alarm.** A service cert allowed to approach expiry raises a Bodha/Pramana **fault** (and
    auto-rotates); the mesh **never** wedges silently. Cold boot brings up the mTLS mesh + Kavach **before** any
    password.
11. **Device binding.** A relay access token replayed from a **different** device key → **Rejected**.
12. **Assurance floor (hint can't climb).** An `acr=hint` token presented for **any T2-or-higher** operation is
    **Rejected at every callee, regardless of `scope`**. *Property:* "assurance is a hard floor beneath scope — a
    hint-grade token with a broad `scope` still cannot perform a T2+ op; only a re-minted `session`/`strong`
    token can."
13. **Subject does not widen the wall.** A rule-owner token carrying a `subject` claim naming a member does
    **not** widen wall evaluation — the wall sees only the **rule-owner principal's machine-grants** (ADR 0004
    row 10); `subject` reaches delivery/audit only, **never** a data read. *(Kills the S3 row-1 self-access
    laundering.)*
14. **Guest is T0-capped.** A guest token (`sub = guest:{…}`, `acr=hint`, `amr=[guest]`, role `Guest`) **cannot
    be exchanged for any audience with a scope beyond T0 tools**; any T1+ attempt is **Rejected** and audited.

Every rejection above is **logged as a security event** in Saakshi (ADR 0001 §5.4) with `sub`, `act`,
attempted-vs-allowed, and `cid`.

---

## 9. Threats defeated

- **Confused deputy.** A callee never acts on caller-supplied identity; it acts only on a Dwarpal-signed,
  audience-scoped delegation. A service cannot "speak for" a user it lacks a token for.
- **Sarathi prompt-injection.** Sarathi holds no super-token; each tool call is bounded to the current speaker's
  own delegation. The worst a hijacked prompt achieves is an action the *speaker themselves* was already entitled
  to — and high-risk ones still hit ADR 0007 step-up.
- **Compromised service.** An owned service is confined to the tokens it legitimately receives; it cannot mint,
  cannot self-elevate roles, and cannot forge another user — blast radius = its own audience + granted scopes.
- **Stolen phone / extracted token.** Short-TTL, device-bound, attested tokens + re-auth for sensitive reads +
  server-side global sign-out reduce a stolen unlocked phone to a minutes-long, biometric-walled, revocable window.
- **Wedged mesh from silent cert expiry.** Proactive rotation + loud expiry alerting + TPM-sealed cold-boot
  bring-up remove the silent-outage failure mode.

---

## 10. Open questions *(amended 2026-07-11, round 2 — OQ-2 partially resolved by §2.1)*

- **OQ-1 (IdP).** Confirm **Zitadel** vs. **Keycloak** after a token-exchange (`act`/`sub`) + step-up spike on
  the home server — validate delegation-chain depth, revocation propagation, and footprint under our load.
- **OQ-2 (TTL tuning).** The exact delegation-token TTL and the `jti` replay-window vs. re-exchange chatter
  trade-off (target ≤ 5 min; measure Sarathi-turn latency impact). *(Partially resolved 2026-07-11 round 2: §2.1
  fixes the **mechanism** — one token per turn, per-tool-call exchange, session-anchored renewal, satellite hint
  ends honestly with no session to renew against. What remains open is only the TTL **number** and the
  replay-window tuning against measured Sarathi-turn latency.)*
- **OQ-3 (nested `act` depth).** Cap and audit-shape for delegation chains (Sarathi → Artha → Ojas → Sangam);
  how deep before we require re-consent.
- **OQ-4 (offline/degraded Dwarpal).** If Dwarpal is briefly down, can core mesh operations proceed on unexpired
  cached tokens + JWKS, and for how long, without opening a bypass? (Fail-closed default; define any grace.)
- **OQ-5 (SPIRE operational weight).** Confirm SPIRE's footprint is justified vs. a thinner step-ca + a small
  `ServiceId`-reconciliation shim, on a single box.

---

## 11. Consequences

**Positive.**
- The load-bearing guarantee (no prompt / no child / no owned service becomes a super-user) is **structural and
  testable**, not aspirational — retiring design-review risk #18 (A2) and unblocking the Phase-0 skeleton.
- One coherent identity story: **mTLS = which service**, **delegation token = which human + what right**, **cert
  identity = `ServiceId` = audit attribution** — no parallel, driftable identity registries.
- C6 (PKI lifecycle) and D8 (remote hardening) are **owned and specified**, not deferred.

**Costs / trade-offs.**
- **Every hop validates a token** (signature/audience/expiry/delegation) — real per-call latency; mitigated by
  cached JWKS, cheap symmetric-verify where possible, and the short-chain topology. Measured under OQ-2.
- **Dwarpal becomes critical-path** for token-exchange; its availability + the cold-boot ordering (Dwarpal vs. machine
  mesh) must be designed (OQ-4). The machine-trust tier deliberately comes up **without** Dwarpal so the mesh/Kavach
  survive a Dwarpal outage.
- **Operational surface**: SPIRE + vault + IdP are three moving parts to run and monitor on one box (Bodha/Pramana
  cover expiry/health).
- Developers **must** use the Kernel helper that derives `ActorContext` from the validated token; constructing an
  `ActorContext` by hand in service code is a **banned pattern** (lint/analyzer + code review) — the seam only
  works if no path fabricates identity.

**Follow-on ADRs.** [`0004`](0004-walling-maysee-and-sharing.md) (per-user walling / may-see / sharing) consumes the
derived `UserId` at the data layer; [`0007`](0007-risk-tiers-and-step-up.md) (risk tiers / step-up) supplies the
`scope` grants this token carries; [ADR 0005](0005-audit-pipeline-and-tamper-evidence.md) records the validated `sub`/`act`/`cid`;
[ADR 0006](0006-automation-principal-and-event-bus.md) defines the named rule-owner identity that edge-mints ECA
tokens.
