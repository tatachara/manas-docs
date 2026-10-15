# ADR 0019 — iOS / Phone Client & Distribution

- **Status:** **Accepted (2026-07-19)** (owner: Ananth) — accepted at the scope §0 records: the **server-side
  slice + the web PWA** build this phase; every native-Swift row is **DEFERRED** behind OQ-1 (no Apple Developer
  account this phase). No threat model HIGHs were raised against this ADR.
- **Date:** 2026-07-19
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines / promotes:** [`../vision/ios-app.md`](../vision/ios-app.md) (Status: *Vision/research*) — this ADR
  promotes that vision doc to a design-of-record and splits it into a **this-phase buildable slice** and a
  **recorded-for-later native slice**. Consumes [`0003`](0003-identity-propagation-and-pki.md) §6 (remote/phone
  token hardening — the this-phase slice is already ratified there), [`0007`](0007-risk-tiers-and-step-up.md) §4/§5
  (T4 trusted-device factor; push ≠ authorization), [`0008`](0008-availability-degraded-mode-and-dr.md) §2/§3/§11
  (the phone is the dead-man's-switch **notification** surface, never the alarm), [`0011`](0011-ui-framework-and-premium-stack.md)
  §4/§5 (React/Vite PWA in WKWebView over Tailscale; the standing render-pass gate), [`0012`](0012-skill-tutoring-capability-modules.md)
  §3d/§4.7 (the box ↔ native-shell bridge version gate).
- **Evidence:** the substance of §§3–5 is the decision-grade research pass captured in `ios-app.md` (web-verified,
  mid-2026). A grep of `src/` on 2026-07-19 confirms **no iOS/Swift code exists** — this ADR describes a greenfield
  surface, nothing is being retrofitted.

> **The phone is a thin deputy client, never the box and never a boundary.** Its only irreducible jobs are the
> handful that can *only* happen on an Apple device — read HealthKit, receive an APNs wake, unlock with Face ID —
> and everything else is the box's Akriti PWA reached over Tailscale. This ADR records the full recommended native
> architecture so it is ready the day the owner opens an Apple account, but the **owner has decided NOT to open an
> Apple Developer account this phase** (§0), so every native-Swift unit is **explicitly DEFERRED** and the
> this-phase scope is the **server-side slice + the web PWA** — all of it testable without Apple.

---

## 0. Scope split — what this phase builds vs. what is recorded-for-later

**Owner decision (given, 2026-07-19): NOT opening an Apple Developer account ($99/yr) this phase.** That single
gate cleanly partitions the work. This ADR is honest about the line: *decided and buildable now* on one side,
*designed, recorded, and deferred* on the other. Nothing native is started until the owner opts in — but the
architecture is fully specified now so opting in is a build, not a redesign (CLAUDE.md rule 6: name the sharp edge
up front, never discover it late).

| Slice | This phase? | Why the line falls here |
|---|---|---|
| Dwarpal **device-enrollment** endpoint (passkey/WebAuthn + App-Attest shape) | **YES — build** | Server-side; ADR 0003 §6 already ratifies it; testable with a fake attestation (real-device attestation deferred) |
| **Short-TTL relay-token** issuance + **sign-out-all** revocation (jti/device) + sensitive-read re-auth | **YES — build** | Pure Dwarpal/C#; ADR 0003 §6/§8; no Apple dependency |
| **E2EE push-payload envelope — server half** (box encrypts alert → opaque blob for a device SE key; `IPushSender` behind an interface + fake) | **YES — build** | The *encryption* and *interface* are server-side and deterministic; only the wire sender + on-device decrypt are Apple-gated |
| **Web-passkey factor verifier** (`IStepUpFactorVerifier` real FIDO2/WebAuthn impl, server-side) | **YES — build** | The step-up seam already exists and is green (§4); a server-side WebAuthn assertion verifier needs **no** Apple account — usable from the Akriti console over Tailscale |
| Web PWA (**Akriti**) reachable **over Tailscale** | **YES — measure** | ADR 0011 already chose the PWA; the WKWebView-over-Tailscale render pass is the standing owner gate (§2) |
| Real APNs `.p8` HTTP/2 sender (`IPushSender` concrete) | **DEFERRED** | Requires an Apple push key + team id — Apple-account-gated |
| Native **SwiftUI thin shell** + embedded WKWebView-Akriti | **DEFERRED** | Requires Xcode signing identity, a provisioning profile — Apple-account-gated |
| **HealthKit → Ojas** bridge (+ background delivery entitlement) | **DEFERRED** | Needs the HealthKit + background-delivery entitlements — Apple-account-gated |
| On-device **`UNNotificationServiceExtension`** (push decrypt) | **DEFERRED** | Native app-extension target — Apple-account-gated |
| **Face-ID** as the on-device WebAuthn factor (native `LocalAuthentication`) | **DEFERRED** | The *server* WebAuthn verifier is in this phase's scope; the *native* Face-ID ceremony is Apple-gated |
| **TestFlight / Ad-Hoc** distribution | **DEFERRED** | Both require the $99/yr program |

**The master gate is OQ-1 (the Apple Developer account decision).** Until it flips, the DEFERRED rows do not
appear in the SDD as buildable units; the YES rows do.

---

## 1. This-phase deliverables (buildable now — no Apple dependency)

These are server-side and testable end-to-end **without** an Apple account by seaming Apple out behind an interface
with a fake. The passkey enrollment, device-bound relay tokens, and sensitive-read re-auth are already ratified by
ADR 0003 §6; **App-Attest device attestation is introduced here** (behind `IDeviceAttestor`, §1a) as an extension
that sits alongside the 0003 §6 passkey binding — not claimed from 0003 §6. Note the step-up **seam itself already exists and is green**
(`StepUpChallengeAuthority` + `ToolRiskGate` + the `IStepUpFactorVerifier` port, §4) — this phase adds the real
**web-passkey factor verifier** (§1d) to it, not a new gate.

### 1a. Dwarpal device-enrollment endpoint (passkey/WebAuthn + App-Attest shape)

A phone enrolls into Manas's PKI as a **WebAuthn credential** whose key is generated in and **non-exportable from**
the device's Secure Enclave (ADR 0003 §6). The endpoint:

- Registers the SE-backed public key as a passkey against Dwarpal (WebAuthn registration ceremony), bound to a
  `UserId` + `TenantId` + a per-device `deviceId`.
- Records an **App-Attest device-attestation** assertion at enrollment. **Real hardware attestation verification is
  DEFERRED** (it needs a real device + Apple's App-Attest root); this phase builds the endpoint, the attestation
  *envelope* parsing, and a **fake attestor** for tests. The seam is `IDeviceAttestor` with a fake that returns a
  known-good/known-bad verdict — the enrollment logic (accept genuine, reject forged) is fully exercised without Apple.
- This is **not** an mTLS mesh cert (§4). The phone is a *human-side* WebAuthn credential in Dwarpal, not a Pramana
  workload identity (ADR 0003 §5). The two identity planes stay distinct.

> **Device- and passkey-enrollment mechanics (which sense of "Dwarpal" owns the ceremony, the exact relying-party
> identifier, the rule that a child principal is never a party to an enrollment or recovery event, and who writes
> the credential pack) are refined in a later, unpublished decision.** Nothing in §1a is superseded, contradicted,
> or deleted by that refinement — it extends this section from the phone case to the console case, and the
> `IDeviceAttestor` seam named here is engaged, not re-derived. That refinement is designed but not yet built.

### 1b. Short-TTL relay tokens + sign-out-all revocation + sensitive-read re-auth

Straight from ADR 0003 §6:

- The phone never holds a long-lived bearer. It authenticates to Dwarpal (passkey/biometric) and receives
  **short-TTL** access tokens; the refresh/session key is **device-bound** to the SE credential (1a), so an
  exfiltrated access token is worth minutes, not months.
- **Sign-out-all**: Dwarpal's revocation list is keyed by `jti` / device / session; the owner invalidates every
  outstanding token for a user from any trusted device, and because every callee checks revocation (ADR 0003 §3.5)
  it propagates **within one TTL / revocation window**.
- **Sensitive reads re-auth**: opening finance/health detail or camera history from the phone demands a fresh strong
  factor (ADR 0003 §6) — a stolen, already-unlocked phone still hits a biometric wall on the crown jewels.

### 1c. E2EE push-payload envelope — server half

The box, when it wants to alert a phone, **encrypts the alert content into an opaque blob** for that device's
Secure-Enclave key and hands it to a sender. This phase builds:

- `IPushSender` **behind an interface**, with a **fake** (records the envelope, asserts it carries *no plaintext*).
- The **envelope crypto**: the box seals the real content to the device's enrolled public key; APNs only ever
  carries `{ "mutable-content": 1, ciphertext-blob }` or a generic stub (ios-app.md §5). Apple learns "wake device X,"
  never the alert text.
- **DEFERRED (Apple-gated):** the concrete `IPushSender` that signs an APNs `.p8` request and POSTs HTTP/2 to
  `api.push.apple.com`, and the on-device `UNNotificationServiceExtension` that decrypts the blob in the
  Secure Enclave before display. The interface + envelope are proven now; only the wire and the on-device half wait.

### 1d. Web-passkey step-up factor verifier (the keystone — no Apple account)

The step-up **seam already exists and is green** — `StepUpChallengeAuthority` (ADR 0007 §7 nonce/action-hash/TTL/
single-use), `Manas.Kernel.Risk.ToolRiskGate` (enforces below both the GraphQL and MCP ingresses), and the
`IStepUpFactorVerifier` **port** whose default is the fail-closed `RejectingFactorVerifier`. This phase builds the
**real WebAuthn/passkey factor verifier** — a concrete `IStepUpFactorVerifier` doing server-side FIDO2/WebAuthn
**assertion verification** against Dwarpal (design-complete via ADR 0003 §6 + ADR 0007 §6/§7). It is **pure C#, no
Apple account**, and is usable **this phase from the Akriti console over Tailscale** (a platform authenticator or
roaming key). It replaces `RejectingFactorVerifier` for the passkey factor class. **Native Face-ID (§3d) is the
Apple-gated variant of this same port** — same verifier, device-bound `amr=[faceid]` class, deferred with the Apple
account (§4).

---

## 2. Access model this phase — the web PWA over Tailscale

**This phase's phone (and any browser) reaches Manas as the Akriti React (Vite) PWA over Tailscale.** ADR 0011 §4
already chose the PWA and noted a Vite/React PWA renders *lighter* in a WKWebView than Blazor WASM (no multi-MB
mono-wasm over the relay). No native shell is required to *use* Manas from a phone this phase — a phone on the
tailnet opens the PWA in mobile Safari or any browser.

**Feasibility measurement = the standing owner gate.** ADR 0011 §6's prove-it spike includes a
**"WKWebView-over-Tailscale render pass"**; that same render pass is the feasibility gate for the phone access model
here — it measures that the premium Akriti surface (motion, charts, the streaming Sarathi panel) clears the bar
*embedded on a phone, over the relay*. This ADR **references** that gate; it does not add a second one. Green there
⇒ the phone-access story is proven for both the this-phase PWA-in-browser and the deferred WKWebView-in-shell case.

---

## 3. Recorded-for-later (deferred, native) — the recommended architecture

Recorded now so the opt-in is a build. All rows here are **$99/yr-Apple-account-gated** (§0).

### 3a. Native shell: SwiftUI thin shell + embedded WKWebView-Akriti (the choice, argued)

Three options were weighed in `ios-app.md` §3; the decision stands:

- **PWA-only (no native app).** Rejected as the *permanent* answer: a suspended PWA **cannot receive background
  push** and **cannot read HealthKit** — the two things that are the app's whole reason to exist. Fine as the
  this-phase access model (§2), insufficient as the end state.
- **Full-native SwiftUI (rebuild Akriti in SwiftUI).** Rejected: it throws away the entire premium Akriti design
  system (ADR 0011) and re-creates the "four redos stayed blah" risk on a third surface. The app holds almost no
  domain logic (ADR 0001 design), so there is little to gain and a large UI to lose.
- **CHOSEN — a thin SwiftUI shell with an embedded WKWebView hosting the *same* Akriti PWA.** Native only where
  native is unavoidable (HealthKit background delivery, the push-decrypt extension, ActivityKit/WidgetKit,
  App Intents, `LocalAuthentication`); the heavy dashboards are the existing PWA over Tailscale (ADR 0011 §4).
  Every feature that justifies the app is Swift-first anyway, so MAUI/Blazor-Hybrid would make you *own a Swift
  binding and* pay an interop tax — native removes a whole failure mode on the fragile surfaces. The GraphQL/MCP
  boundary is the only "code sharing," which is the correct seam regardless.
- **Fallback** (only if a solo dev flatly refuses Swift): MAUI + hand-written HealthKit/push bindings, at a
  documented lower ceiling. Not co-equal.

### 3b. Distribution — TestFlight external (Ad-Hoc fallback; App Store rejected)

Per `ios-app.md` §6, for a private 3-person household:

- **Chosen: TestFlight — external testers via a public/email link.** Cleanest for family (no App-Store-Connect team
  over-grant); light Beta App Review on first build; **90-day re-upload** cadence, scriptable in CI.
- **Fallback: Ad-Hoc** (register device UDIDs; annual re-sign instead of 90-day churn) if the stable device set makes
  the quarterly re-upload annoying.
- **Rejected: App Store (public).** A private household app is not "useful to the general public" — likely rejected
  or forces contrived generality. Revisit only if Manas is ever productized (ADR 0001 phase 5).
- **Ineligible: Apple Developer Enterprise** (requires 100+ employees).

### 3c. HealthKit → Ojas bridge

The **sole ingestion path** for Apple Health → Ojas (HealthKit is iOS-only; the home server cannot read it). Design is
fixed in `ios-app.md` §4: `HKObserverQuery` + `enableBackgroundDelivery` + the background-delivery entitlement;
anchored delta queries; idempotent posts to Ojas keyed on HealthKit `UUID` (exactly-once via ADR 0005 §3.2 outbox);
per-user wall enforced server-side; "last synced N ago" honesty because background delivery is best-effort. Gated on
the HealthKit + background-delivery **entitlements** (OQ-3).

### 3d. Native Face-ID ceremony (the Apple-gated variant of the §4 port)

The **server** WebAuthn factor verifier (the real `IStepUpFactorVerifier`, §4) and the enrolled SE credential
(§1a) are built this phase and are exercised by the **web-passkey** class from the console. What is DEFERRED here is
only the **native `LocalAuthentication` ceremony** that gates the SE key with Face ID on device — i.e. Face ID
*unlocks the phone's Manas WebAuthn credential*, which then satisfies the *same* Dwarpal step-up seam. It is not a
new verifier, just the device-bound `amr=[faceid]` class of the port §4 already builds. Face ID never unlocks the
box's master DEK (that lives in the home server RAM, ADR 0001 §15) — a stolen phone yields no platform data at rest.

---

## 4. Face-ID / web-passkey step-up = factor classes of the EXISTING keystone seam

This is the load-bearing link to the identity spine — and the crucial framing correction: **the step-up
challenge + gate infrastructure already exists and is green.** This ADR adds no net-new step-up machinery. What
already exists (verified in code):

- **`StepUpChallengeAuthority`** — the challenge issuer (ADR 0007 §7: nonce / action-hash / TTL / single-use).
- **`Manas.Kernel.Risk.ToolRiskGate`** — enforces step-up **below both ingresses** (GraphQL + MCP), deterministically.
- **`IStepUpFactorVerifier`** — the **factor PORT**. Today it has a dev shared-secret impl; the **default is
  `RejectingFactorVerifier` (fail-closed)** — no real factor passes until a real verifier is registered.

**Face-ID and web-passkey are two FACTOR CLASSES of this one existing port — not new step-up plumbing.** The
phone becomes a **T-tier strong factor on a trusted device** (ADR 0007 §4, T4 = "passkey/WebAuthn or Face ID on a
TRUSTED DEVICE — the enrolled iOS app") purely by being one class the shared `IStepUpFactorVerifier` accepts:

- **The keystone (a planned this-phase build unit, not yet started) = the real WebAuthn/passkey factor verifier** (a
  concrete `IStepUpFactorVerifier`), design-complete via ADR 0003 §6 + ADR 0007 §6/§7. It replaces
  `RejectingFactorVerifier` for the passkey class.
- **Web-passkey factor is buildable THIS phase WITHOUT an Apple account.** Server-side FIDO2/WebAuthn **assertion
  verification** is pure C# against Dwarpal — usable **right now from the Akriti console over Tailscale** (a platform
  authenticator / roaming key). This is a §1 this-phase deliverable, not deferred.
- **Native Face-ID is the Apple-account-gated VARIANT of the same port.** iOS Face ID gates the phone's
  Secure-Enclave WebAuthn credential via native `LocalAuthentication`; the *server* verifier is identical (§1a/§3d),
  only the on-device ceremony is Apple-gated. So Face-ID adds a `amr=[faceid]` device-bound class to a seam that
  already verifies `amr=[passkey]` from the web this phase — **no second step-up engine.**
- **Enrollment = a WebAuthn registration** against Dwarpal (§1a). The credential is a **passkey/WebAuthn** identity
  in the human-identity plane (ADR 0003 §6) — **explicitly NOT an mTLS mesh cert** (Pramana workload identity,
  ADR 0003 §5, is service-side and never issued to a phone). The phone authenticates a *human*; mTLS authenticates a
  *service*; neither substitutes for the other (ADR 0003 §5).
- **Step-up = `ToolRiskGate` demands → `StepUpChallengeAuthority` issues a nonce/action-hash challenge →
  `IStepUpFactorVerifier` verifies the passkey/Face-ID assertion bound to that exact action → Dwarpal re-mints** a
  token at the higher `acr` (ADR 0003 §2 — assurance is never raised in place). The verified assertion is the
  `amr=[passkey|faceid]` factor behind that `acr`.
- **No-phone fallback (ADR 0007 §4).** When no enrolled phone is reachable, a **console passkey (platform
  authenticator) + the Kavach PIN** satisfies T4 — and because the web-passkey verifier lands this phase, that
  console factor is *itself real this phase*. A dead or absent phone never hard-blocks money moves or coverage
  changes; the phone is an *additional* trusted-device factor class, not a single point of failure.

---

## 5. The phone is a NOTIFICATION surface, never the alarm/authorizer (rule #9)

This is the child-safety / boundary invariant restated for the phone (CLAUDE.md rule #9; ADR 0007 §5: **push ≠
authorization**):

- **Dead-man's-switch (ADR 0008 §2/§3).** The phone's *only* Phase-4 exit-gate obligation is to **receive** the
  external dead-man's-switch APNs alert ("Manas Core is not responding") when the box stops beaconing for the 5-min
  grace window. The phone is the **notification** surface for box-death; it is **never** the alarm. The alarm is the
  standalone Layer-0 siren + local keypad that survives box death (ADR 0008 §2) — the phone cannot arm, disarm, or
  substitute for it. The delivery of that alert is the touchpoint asserted in ADR 0008 §11 #2. Crucially, the box is
  **dead** when this fires, so this is **NOT** the §1c box-sealed E2EE envelope: the **blind relay** (ADR 0008 §3 —
  it holds no house keys and learns only "box-id alive at T") emits a **generic, contentless** "Core not responding"
  push on its **own** path. This ADR builds that relay-originated stub, kept distinct from the box-alive §1c envelope.
- **"Disarm" on a push disarms nothing (ADR 0007 §5, ADR 0008).** Tapping *Disarm* on a Kavach push **opens the
  app → Face ID → server-side step-up → deterministic Kavach path** (T4 + Kavach PIN, ADR 0007 §4). The push tap and
  the notification layer are **never** the security decision — the box decides, deterministically, server-side. An
  Artha payment approval is identical: push → open → Face ID → server verifies step-up → deterministic execution,
  audited in Saakshi.
- **Lost/stolen phone.** Revocable (sign-out-all, §1b), short-TTL, biometric-walled (sensitive-read re-auth, §1b),
  and device-bound (§1a). The two acceptance negatives that pin this are ADR 0003 §8 **#11 (device binding — a token
  replayed from a different device key is rejected)** and **#9 (revocation within one TTL)**. A stolen unlocked phone
  is a minutes-long, biometric-walled, revocable window — not a durable key to the house.

---

## 6. Acceptance criteria (this-phase, server-side — all testable without Apple)

The this-phase slice (§1) is **not** accepted until these pass. Most are stated as things that **must be rejected**
(ADR 0003 §8 discipline). All run **without an Apple account** — the web-passkey path (#6) needs no Apple at all;
the push/attestation paths seam Apple out behind a fake.

1. **Enrollment device-binding — replay rejected (ADR 0003 §8 #11).** A relay access token minted for device key A,
   replayed from a **different** device key B → **Rejected**. *Property:* "a token is worthless off the device whose
   Secure-Enclave key it is bound to."
2. **Sign-out-all — revocation within one TTL (ADR 0003 §8 #9).** After "sign out all devices," a previously-valid
   phone token is **rejected within one TTL / revocation window** at every callee.
3. **Push envelope carries no plaintext.** The E2EE push server half (§1c) produces a payload that the fake
   `IPushSender` asserts contains **only** an opaque ciphertext blob (and/or a generic stub) — **no** alert text,
   amount, zone name, or member name in cleartext anywhere in the APNs payload.
4. **Sensitive-read re-auth.** Opening a finance/health-detail or camera-history read from a phone session **without**
   a fresh strong factor → **Rejected** (step-up demanded), even when the session is already unlocked (ADR 0003 §6).
5. **Enrollment rejects a forged attestation.** With the fake `IDeviceAttestor` returning a known-bad verdict, the
   enrollment endpoint **refuses** to register the credential. (Real-device attestation is DEFERRED; the accept/reject
   *logic* is proven now.)
6. **Web-passkey step-up passes; `RejectingFactorVerifier` refuses (§1d/§4).** With the real web-passkey
   `IStepUpFactorVerifier` registered, a valid FIDO2/WebAuthn assertion **bound to the challenge's action-hash**
   (`StepUpChallengeAuthority`, ADR 0007 §7) satisfies `ToolRiskGate` for a T3/T4 op and Dwarpal re-mints at the
   higher `acr`. *Negatives:* a replayed or already-used nonce → **Rejected** (single-use); an assertion whose
   action-hash ≠ the demanded action → **Rejected**; and with the **default `RejectingFactorVerifier`** the same op
   → **Rejected** (fail-closed). Proves the seam gates on the real factor without any Apple dependency.
7. **Dead-man's-switch delivery touchpoint (ADR 0008 §11 #2, server+relay half).** When the box stops beaconing past
   the 5-min grace, the **blind relay** (ADR 0008 §3 — no house keys; the box is dead and can seal nothing) emits a
   **generic, contentless** "Manas Core is not responding" push on its **own** path — explicitly **NOT** the box-sealed
   E2EE envelope of §1c. A fake relay-sender records exactly one such generic alert (and the re-page on back-off). This
   is the server/relay half of ADR 0008 §11 #2 — the on-device receipt is DEFERRED with the native shell.

Every rejection above is logged as a security event in Saakshi (ADR 0001 §5.4) with `sub`, `act`, device, and `cid`
(ADR 0003 §8).

---

## 7. Open questions

- **OQ-1 (MASTER GATE) — the Apple Developer account decision.** The owner has decided **not** to open a $99/yr
  Apple Developer account this phase. Every DEFERRED row in §0/§3 is blocked on this one flip. Until it flips, no
  native-Swift unit is scheduled; when it flips, §3 is a build, not a redesign. **This is the gate that governs
  every other native question below.**
- **OQ-2 (APNs Apple-cloud dependency containment).** APNs is the *only* iOS push path — an unavoidable Apple-cloud
  touchpoint for a local-first system (ios-app.md §5). The containment is the E2EE envelope (§1c): confirm Apple ever
  learns only "wake device X," never content, and that the box's outbound HTTPS to `api.push.apple.com` stays a blind
  control-plane egress (ADR 0001 §2), never a data path. Re-validate the on-device decrypt budget when the extension
  is built.
- **OQ-3 (HealthKit entitlements).** The HealthKit read + `com.apple.developer.healthkit.background-delivery`
  entitlements are a request/approval gate (usually granted, but not guaranteed and not this phase). Confirm both are
  obtainable for a private/TestFlight app before committing the §3c bridge.
- **OQ-4 (re-enrollment ceremony for a lost trusted phone — ADR 0007 OQ-5).** The **availability** half of ADR 0007
  OQ-5 is resolved (no-phone T4 fallback = console passkey + Kavach PIN, §4). What remains open is the **enrollment-loss
  recovery** ceremony: how a new phone is enrolled and the lost device's credential revoked, without the lost phone —
  and how that ceremony is itself step-up-gated so a thief cannot self-re-enroll. This ADR inherits ADR 0007 OQ-5 for
  the phone case; it is Apple-gated (needs a real device) and therefore deferred with §3.

  > **Part of this open question was later decided in a follow-on, unpublished decision; this OQ's text is
  > unchanged and stays governing for the half it still owns.** The recovery-ceremony half below remains open.
  > The half since decided: the console-passkey fallback (ADR 0007 §4) only exists if that authenticator was
  > enrolled in advance, so "have a console fallback" and "enroll a second authenticator" turned out to be one
  > policy at one time — settled as a **two-authenticator floor** (no adult durably holds only one authenticator;
  > at least one is not the phone; a reduced boot verdict until it holds) plus who may authorise a re-enrollment
  > (an already-enrolled adult stepped up on an authenticator that is NOT the one being replaced — so a thief
  > holding the lost phone cannot self-re-enroll, which is what this OQ asks for — or physical presence at the box).
  > **Left deferred here**, unchanged and still Apple-gated behind OQ-1: the *phone-replacement* ceremony itself,
  > which needs a real device and App-Attest to build or prove.
- **OQ-5 (native-shell bridge version gate — ADR 0012 §3d/§4.7).** When the native shell ships, the **box ↔
  native-shell bridge contract** is the real skew surface (TestFlight's ~90-day cadence), not the near-zero-skew web
  seam. Adopt ADR 0012 §3d's fail-closed three-state availability ("available / update your iOS app / honestly
  console-only") for any bridged native capability. Deferred with the shell.

---

## 8. Consequences

**Positive.**
- The phone's identity/security half is **buildable and testable now** with zero Apple dependency — device-bound
  short-TTL tokens, sign-out-all, sensitive-read re-auth, and the E2EE push envelope are all server-side and land
  behind interfaces with fakes. The owner gets the security-critical slice without spending $99 or waiting on Apple.
- The **real web-passkey step-up factor** (§1d) drops into the *existing green* step-up seam
  (`StepUpChallengeAuthority` + `ToolRiskGate` + `IStepUpFactorVerifier`) this phase — turning the fail-closed
  `RejectingFactorVerifier` default into a real factor usable from the Akriti console today. Native Face-ID is later
  just another class of the *same* port, so no step-up rework is deferred.
- The full native architecture is **recorded and argued** (rule #6), so the Apple opt-in is a build against a fixed
  design, not a fresh decision under time pressure.
- The boundary invariant is preserved on a new surface: the phone is a **notification surface and deputy**, never the
  alarm and never an authorizer (rule #9, ADR 0007 §5, ADR 0008 §2/§3).

**Costs / trade-offs (honest).**
- Until OQ-1 flips, phones use the **PWA-over-Tailscale in a browser** (§2) — **no background push and no HealthKit**
  this phase (a suspended PWA can do neither). Ojas simply has no Apple-Health data and no phone alerts until the
  native shell ships; that is stated plainly, not hidden.
- The E2EE push *server half* is built and tested against a **fake** sender; the real APNs wire + on-device decrypt
  remain unproven end-to-end until Apple opt-in (mitigated by the envelope + interface being the load-bearing,
  Apple-independent parts).
- A second toolchain (Swift/Xcode) and a TestFlight re-upload ritual arrive **only** when OQ-1 flips — deferred cost,
  not this-phase cost.

**What would change the decision.** OQ-1 flipping (owner opens the Apple account) promotes all §3 rows to buildable
units. Android becoming P0 does not change the iOS design (HealthKit is iOS-only; a Health-Connect bridge is a
separate later decision, ios-app.md §8).

---

*Promotes [`../vision/ios-app.md`](../vision/ios-app.md) to a design-of-record; consumes
[`0003`](0003-identity-propagation-and-pki.md) §6/§8, [`0007`](0007-risk-tiers-and-step-up.md) §4/§5,
[`0008`](0008-availability-degraded-mode-and-dr.md) §2/§3/§11, [`0011`](0011-ui-framework-and-premium-stack.md)
§4/§5/§6, [`0012`](0012-skill-tutoring-capability-modules.md) §3d/§4.7. The orchestrator adds the README index row
and the SDD Phase rows (this-phase §1 units; deferred §3 units gated on OQ-1).*
