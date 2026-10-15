# Manas iOS App (`Manas.iOS`) — Vision & Architecture

- **Status:** Vision / research (informs a later ADR `0002+`)
- **Date:** 2026-07-11
- **Scope:** the iOS client/bridge app under `src/Apps/Manas.iOS` (ADR 0001 §7)
- **Reads-from:** ADR 0001 — honor local-first & private-on-the-box, deterministic-C#-first backend,
  GraphQL/MCP APIs, per-user walls, `Manas.Crypto` envelope encryption + hybrid unlock, accessibility/
  voice-completeness, Akriti design system, remote access via a private relay/Tailscale.
- **Household:** Ananth and the other adult (adult admins) + the child.

> **One-line thesis.** The iOS app is **not a domain service** — it is a **thin native bridge and remote
> client**. Its reason to exist is the handful of things that *can only* happen on an Apple device:
> reading **HealthKit** (iOS-only) and syncing it to **Ojas**, receiving **push** alerts from **Kavach**/
> **Artha**, unlocking with **Face ID**, and surfacing glanceable/actionable iOS affordances (widgets,
> Live Activities, Siri). Everything that is *not* Apple-specific stays on the box and is reached over the
> relay. Build the app **native (SwiftUI)** and keep it small; reuse the .NET stack where it lives — on the
> server — not by re-hosting it on the phone.

---

## 1. Intent / role

**Charter.** `Manas.iOS` is the family's **pocket bridge to their own house**. It does three jobs and
resists becoming a fourth:

1. **The mandatory HealthKit bridge.** Apple HealthKit is **iOS-only** and cannot be read from the home server.
   The app is the *only* path by which Apple Health data (from iPhone + Apple Watch) reaches **Ojas**. It
   reads HealthKit, and syncs new samples to the Ojas GraphQL/MCP API over the relay — including in the
   background. Without this app, Ojas has no Apple Health data at all. This is the app's non-negotiable core.
2. **The remote client & notification surface.** From anywhere, over the **private relay (Tailscale)**, the
   phone reaches the whole Manas platform (Sarathi, dashboards, approvals). It receives **push alerts**
   (Kavach security, Artha finance, reminders) and lets a verified adult **approve tiered actions**
   (an Artha payment, a Kavach disarm) with a strong on-device factor.
3. **The native-iOS integration layer.** Siri Shortcuts / App Intents, Home-screen & Lock-screen widgets,
   and Live Activities (e.g. an armed-alarm indicator) — the glanceable, hands-free, OS-level touchpoints
   that make Manas feel present without opening the app.

**What it is not.** It is **not** a place where business logic, analysis, or the LLM lives. Per ADR 0001 §4,
the deterministic engine and the model stay on the box behind services. The app **carries the caller's
verified identity** and calls governed GraphQL/MCP; it is a **deputy client**, never a super-user, never an
access-control decision point. Per-user walls (the child vs. the adults) are enforced **server-side at each
service's data layer** — the phone never sees another member's rows.

**Design constraints inherited from ADR 0001.** Voice-completeness and WCAG 2.2 AA (here delivered via
native iOS accessibility — VoiceOver, Dynamic Type, Siri); privacy (no data to third parties; payloads
encrypted/opaque even through Apple's push relay); step-up gating for money/health/security actions.

---

## 2. Feature list

Kind legend: **[native-ios]** = a native Apple capability the app implements; **[bridge]** = moves data
between an Apple API and a Manas service; **[integration:name]** = talks to a named Manas service over the
relay. Feasibility: **now** = doable with just the $99/yr Apple Developer Program; **needs-apple-dev** =
requires a specific entitlement/capability request (usually granted, but a gate); **needs-3rd-party** =
depends on a non-Apple piece (Tailscale, a self-hosted push signer).

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| **HealthKit read + background sync → Ojas** | `HKObserverQuery` + `enableBackgroundDelivery`; on wake, `HKAnchoredObjectQuery` pulls deltas and posts to Ojas over the relay | [bridge] + [integration:Ojas] | needs-apple-dev (HealthKit + background-delivery entitlement) | **P0** |
| **HealthKit permission onboarding** | Per-type read authorization prompts (BP, glucose, weight, HR, HRV, sleep, workouts, SpO2, ECG, respiratory) with clear rationale strings | [native-ios] | needs-apple-dev | **P0** |
| **Relay connectivity (Tailscale)** | Reach the home server's GraphQL/MCP from anywhere; on-demand VPN; no port-forwarding; box stays un-exposed to the public internet | [integration:relay] | needs-3rd-party (Tailscale app + tailnet) | **P0** |
| **Face ID / passcode unlock** | `LocalAuthentication` gates the app session and the local credential in Keychain/Secure Enclave; ties to the ADR §15 failsafe (see §4/§7) | [native-ios] | now | **P0** |
| **Encrypted push receipt (Kavach/Artha/reminders)** | Register APNs token; receive alerts; **decrypt opaque payload on-device** in a `UNNotificationServiceExtension` before display | [native-ios] + [integration:Kavach/Artha] | needs-apple-dev (push capability) + needs-3rd-party (home-side APNs signer) | **P0** |
| **Manual vital entry → Ojas** | Voice/typed BP, blood sugar, weight, etc., posted to Ojas (redundant with the box UI but native/offline-friendly) | [bridge] + [integration:Ojas] | now | **P1** |
| **Approve tiered actions** | Confirm an Artha payment / Kavach disarm from a push or in-app, gated by Face ID + server-side step-up | [native-ios] + [integration:Artha/Kavach] | now | **P1** |
| **Armed-alarm Live Activity** | Lock-screen + Dynamic Island indicator while Kavach is armed; remote state via ActivityKit push token | [native-ios] + [integration:Kavach] | needs-apple-dev (push) | **P1** |
| **Home-screen & Lock-screen widgets** | Glanceable house/health/finance status via WidgetKit; refreshed by timeline + push | [native-ios] | now | **P1** |
| **Siri Shortcuts / App Intents** | "Hey Siri, arm the house", "log my blood pressure", "ask Manas…" → routes to Sarathi/service tools | [native-ios] + [integration:Sarathi] | now | **P1** |
| **Embedded platform console (WKWebView)** | Host the **Akriti React (Vite) PWA** (ADR 0011) for the full dashboard/Sarathi chat, reached over the relay — reuse, not rebuild | [integration:Console] | now | **P1** |
| **Offline queue & replay** | Buffer HealthKit deltas + manual entries locally when the box is unreachable; replay idempotently on reconnect | [bridge] | now | **P1** |
| **Push-to-talk to Sarathi** | Native mic capture → Sarathi voice endpoint (carry over Vatara's `/api/voice` pattern) over the relay | [native-ios] + [integration:Sarathi] | now | **P2** |
| **Apple Watch companion** | Watch app for quick vitals + arm/disarm + alarm glance (extends HealthKit reach) | [native-ios] | needs-apple-dev | **P2** |
| **HealthKit write-back** | Write Ojas-recorded vitals back into Apple Health (share-permission per type) | [bridge] | needs-apple-dev | **P2** |
| **Sign in with Apple** | Optional; **not required** for a private app (see §4) unless another social login is offered | [native-ios] | now | **P2** |
| **Location/geofence presence → Kaya** | Contribute arrive/leave presence to Kaya/Kavach automations | [native-ios] + [integration:Kaya] | needs-apple-dev (location entitlement) | **P2** |

---

## 3. Architecture recommendation — SwiftUI vs .NET MAUI vs Blazor Hybrid

### The real question

The app's **reason to exist is precisely the set of deep-iOS integrations** that are hardest to reach from
.NET: HealthKit **background delivery**, the **`UNNotificationServiceExtension`** that decrypts push
payloads on-device, **ActivityKit** (Live Activities), **WidgetKit**, and **App Intents/Siri**. The pull the
other way is reuse — the .NET stack and the **Akriti** design system already exist. The key realization that
breaks the tie: **the app's UI surface is small** (a bridge, a few status/approval screens, notifications),
and **the big shared UI is a React (Vite) PWA** (ADR 0011) that can be *embedded* rather than *re-hosted* — and a React PWA renders *lighter* in a WKWebView than Blazor WASM (no multi-MB mono-wasm over the relay). So the
reuse argument mostly evaporates — you get Akriti reuse "for free" via a WebView, without paying the native-
interop tax on the parts that matter.

### Comparison

| Dimension | **Native SwiftUI** | **.NET MAUI (native UI)** | **Blazor Hybrid (MAUI + WebView)** |
|---|---|---|---|
| HealthKit + background delivery | First-class, Apple-documented, zero friction | Works but via a community plugin / hand-written **binding** to `HKObserverQuery`/background delivery; you own the binding | Same binding tax as MAUI; UI-in-WebView adds nothing here |
| `UNNotificationServiceExtension` (E2EE decrypt) | Native target, straightforward | App-extension targets in MAUI are **awkward**; extension must be lightweight — .NET runtime in an extension is heavy/fragile | Same problem as MAUI; the extension is not Blazor at all |
| ActivityKit / WidgetKit / App Intents | First-class; these are **SwiftUI-only** APIs by design | Must bridge; widgets/Live Activities are effectively **written in Swift anyway** | Widgets/Live Activities still Swift; WebView can't render them |
| Reuse Akriti design system | **No** direct reuse (Akriti is Razor/Blazor) | No direct reuse (MAUI native controls ≠ Razor) | **Yes** — but best delivered by embedding the existing **PWA**, which any host can do |
| Reuse .NET domain code | Low (app is a thin client; little domain code belongs here) | High in principle, low in practice (little logic lives in the app) | High in principle, low in practice |
| iOS-native feel / a11y (VoiceOver, Dynamic Type) | **Best** — native by construction | Good, native controls | WebView UI is a step below native for feel + some a11y nuances |
| App-extension + entitlement ergonomics | **Best** — this is the whole toolchain's home turf | Rough edges around extensions, entitlements, Info.plist | Rough edges + WebView constraints |
| Team fit (this project is .NET) | New language (Swift) — a real cost for a .NET-centric solo build | Familiar C#/.NET | Familiar C#/.NET **and** Blazor (already used for the console) |
| Long-term maintenance of the hard parts | Lowest risk — no self-owned bindings to break on iOS updates | You maintain bindings across iOS releases | You maintain bindings **and** a WebView bridge |
| Startup/perf | Native | Native UI, good | WebView adds a small startup/perf tax |

### Recommendation: **Native SwiftUI**, thin, with the Akriti PWA embedded for the "full console" surface

For a **solo/small build**, pick **native SwiftUI**. The counter-intuitive but correct reasoning:

- **Every feature that justifies the app's existence is Swift-first.** HealthKit background delivery, the
  push-decrypt extension, Live Activities, widgets, and App Intents are all APIs where MAUI/Blazor make you
  **write or maintain a Swift binding anyway** — you'd be doing the hard part in Swift *and* paying an
  interop tax. Native removes a whole failure mode (bindings breaking on iOS updates) on exactly the
  fragile, high-value surfaces.
- **The reuse win from MAUI/Blazor is smaller than it looks.** The app holds almost **no domain logic** (by
  ADR 0001 design, logic lives in services on the box). So "reuse the .NET stack" mostly means "reuse UI" —
  and the shared UI (**Akriti**) is a **React (Vite) PWA** (ADR 0011) and can be **embedded in a
  `WKWebView`** and reached over Tailscale. You get the design-system reuse without hosting Blazor natively.
- **The app is small.** A bridge + notifications + a few approval/status screens + widgets is a modest
  SwiftUI surface. The Swift-learning cost is real but bounded, and it buys the smoothest path through
  Apple's most finicky subsystems.
- **Honesty about the trade-off:** the cost of SwiftUI is (a) a **second language/toolchain** for a .NET
  shop, and (b) **no code sharing** with the platform except over the network (GraphQL/MCP DTOs — which is
  the *correct* boundary anyway). If the household ever needs **Android**, SwiftUI buys nothing there and
  MAUI/Blazor would — but **HealthKit is iOS-only**, so the health-bridge value is inherently Apple-bound,
  and Android would be a separate, later decision (Google Fit/Health Connect bridge), not a reason to
  compromise the iOS app now.

**Rejected alternatives, briefly.** *.NET MAUI (native UI):* reuses C# but forces you to own bindings for
every hard API and fights the extension/entitlement toolchain — you'd write Swift for the widgets/Live-
Activities/extension regardless. *Blazor Hybrid:* adds a WebView tax on top of the same binding tax, and its
one advantage (Akriti reuse) is better achieved by embedding the already-built PWA inside a thin native
shell. Both are defensible if a solo dev flatly refuses to learn Swift — in that case **MAUI + hand-written
HealthKit/push bindings** is the fallback — but they optimize for the wrong thing (UI reuse) at the expense
of the app's actual hard core (native OS integration).

**Net:** *native SwiftUI shell (small) + embedded Akriti PWA (for the heavy dashboards) + the GraphQL/MCP
boundary as the only "code sharing."* Native where native matters; the web app where reuse matters.

---

## 4. HealthKit bridge design

**Why the app is mandatory.** HealthKit runs only on Apple devices and exposes no server API; the home server
cannot read it. So the iPhone app is the **sole ingestion path** for Apple Health → **Ojas**. This is the
single most important thing the app does.

**What data is available** (per-type, user-granted **read** authorization): heart rate, HRV, resting HR,
steps & distance, active/basal energy, workouts, sleep analysis, blood pressure (systolic/diastolic), blood
glucose, body mass/weight, body temperature, respiratory rate, blood oxygen (SpO₂), ECG, and more. **Privacy
nuance to design around:** iOS **does not tell you whether read permission was denied** — a denied type
simply returns **empty results** (so the user can't infer what you tried to read). The bridge must therefore
treat "no data" as ambiguous and never assume a gap means a healthy zero.

**Background delivery mechanics (the crux).**

- Pair an **`HKObserverQuery`** (per type, or a set of types) with
  **`enableBackgroundDelivery(for:frequency:)`** and the **`com.apple.developer.healthkit.background-delivery`
  entitlement**. Without background delivery, the observer only fires while the app is foregrounded.
- **The observer only says "something changed for type X" — not what changed.** On each wake, run an
  **`HKAnchoredObjectQuery`** with the saved anchor to fetch just the **new/deleted samples** since last
  sync, then advance and persist the anchor.
- **Frequency is a floor, not a promise.** `.immediate` / `.hourly` / `.daily`; iOS coalesces and rate-limits
  — many types realistically deliver **at most hourly**, and delivery is best-effort (subject to system
  budget, Low Power Mode, etc.). Sensible mapping: fast-changing/low-volume clinical types (BP, glucose,
  weight, ECG) → `.immediate`; high-volume streams (steps, HR, energy) → `.hourly` to avoid thrash. **Do not
  promise real-time health sync** — it's near-real-time at best, and that's fine for a trend/coaching product
  like Ojas.
- **watchOS caveat** (if a Watch companion is added later): background updates share a limited budget
  (~4/hour, and only with an active complication) — another reason not to over-promise latency.

**Sync pipeline (deterministic, idempotent, offline-safe).**

1. On background wake, anchored-query each observed type → a batch of new/deleted samples.
2. Map to Manas health DTOs (`Manas.Contracts`); attach the **acting user** (whose phone/health this is) and
   a **correlation id**.
3. **Idempotency key** per sample = HealthKit's stable `UUID` (+ type + source). Ojas's ingest is
   **effectively exactly-once** via the outbox/idempotency pattern (ADR 0005 §3.2) — replays and overlapping wakes never
   double-record.
4. POST to **Ojas over the relay** (GraphQL mutation / `health.record_vital`-style MCP tool). Ojas enforces
   the **per-user wall** at its data layer — the phone can only write to its owner's record.
5. On failure/offline, **enqueue locally** (a small on-device store) and replay on next wake/reconnect;
   persist the HealthKit **anchor only after** the box acknowledges, so nothing is lost if the box is down.
6. HealthKit data at rest on the device sits in the iOS **encrypted, biometric-gated** health store; in
   transit it rides **TLS over the Tailscale tunnel**; the app caches only what it must and clears on lock.

**Consent & lifecycle** (ADR 0001 §14): record **consent per source** ("Apple Health, this member, these
types, granted on <date>"); support re-prompt, revoke, and **right-to-forget** flowing back to Ojas. The child's
health data is **strictly walled**; child-health ingestion follows the same wall + adult-visibility policy
Ojas defines.

**Failure honesty.** If the family disables Background App Refresh, force-quits the app, or the phone is off,
**no health data flows** — the bridge is only as live as iOS lets a background app be. Ojas must treat gaps as
"unknown," and the app should surface a plain "last synced N hours ago" so the family isn't misled.

---

## 5. Push notifications in a local-first world

**The honest tension, stated plainly.** Manas is local-first: data and inference live on the box. But **there
is no way to wake or notify an iPhone that does not route through Apple's cloud.** APNs is the *only* push
path on iOS. So **every** Kavach/Artha/reminder alert must transit **Apple's servers** — a genuine,
unavoidable dependency on a third party for a system whose whole point is that data stays home. We don't hide
this; we **contain** it.

**How APNs actually works here.**

- The app registers with APNs and gets a **device token**; it hands that token to the box (over the relay).
- To send, the **home server signs a request** (with an APNs **auth key `.p8`**, key id, team id) and POSTs
  over **HTTP/2 to `api.push.apple.com`**. So the **the home server needs outbound HTTPS to Apple** — a small, blind,
  **control-plane** egress (exactly the "thin cloud relay" ADR 0001 §2 permits), *not* a data path.
- Apple's gateway terminates TLS: **Apple can see the payload plaintext** at that hop. (This is the same
  channel that surfaced the 2023 push-surveillance concerns — governments can subpoena push metadata.) TLS
  protects the wire, **not** Apple's own visibility.

**Keeping payloads private despite Apple** — the E2EE pattern:

1. The box sends a **minimal, opaque payload**: either a generic visible stub ("Manas: security alert") **or**
   a **ciphertext blob** in a custom field, with `"mutable-content": 1`.
2. A **`UNNotificationServiceExtension`** on the device intercepts the delivered push (a few-second budget),
   **decrypts** the real content **on-device** using a key held in the **Keychain/Secure Enclave** (shared
   out-of-band with the box, not via APNs), and rewrites the notification body before the user sees it.
3. Apple therefore only ever learns **"wake device X, sometime"** — never "the alarm tripped at the back
   door" or "a $4,200 charge posted." The **plaintext alert never exists on Apple's servers.**
4. For richer content, the stub can carry only an **opaque reference id**; the extension (or the app on open)
   **fetches the real alert from the box over the relay** and renders it locally — Apple sees nothing but a
   wake-up.

**Live Activities** (armed-alarm indicator) use the **same APNs channel**: the app obtains an **ActivityKit
push token**, the box pushes state with the **`liveactivity`** push type; content state should likewise be
**minimal/non-sensitive** (e.g. "Armed" / "Alarm" — deliberately low-information on the lock screen). iOS
17.2+ can even *start* a Live Activity via push. Same containment rule: don't put sensitive detail in the
lock-screen state.

**Security-critical caveat (ADR 0001 §4/Kavach).** A push is a **notification, not an authorization.**
Tapping "Disarm" from a Kavach alert must **not** disarm anything by itself — it opens the app, which
performs a **Face-ID-gated, server-side step-up'd** disarm through the deterministic Kavach path. The LLM and
the notification layer are **never** the security decision. Approving an Artha payment is identical: push →
open → Face ID → server verifies step-up → deterministic execution, all audited in Saakshi.

**Alternatives considered & rejected.** *No push (poll over the relay):* impossible in the background on iOS —
a suspended app can't poll, so life-safety alerts would be missed; unacceptable for Kavach. *Self-hosted push
without Apple:* not possible on stock iOS — there is no non-Apple push on the platform. So APNs-as-a-blind-
relay is the **only** viable design; the mitigation is **opaque/encrypted payloads + on-device decrypt**,
which reduces Apple's role to a dumb doorbell.

---

## 6. Distribution — a private family app on iOS

**The uncomfortable truth:** iOS has **no clean "install my private app on my family's phones forever"**
path. Every option costs the **$99/yr Apple Developer Program** and carries friction. The realistic choices:

| Path | How it works | Fit for a 3-person family | Honest cost |
|---|---|---|---|
| **TestFlight — internal testers** | Up to 100 internal testers, **instant, no review**; but testers must be **members of your App Store Connect team** (they get some account access) | Works, but adding the other adult as a "team member" over-grants; **builds expire after 90 days** and must be re-uploaded | 90-day re-upload cadence; team-role over-grant |
| **TestFlight — external testers** | Email invite or **public link**, up to 10,000; testers need only an Apple ID | **Cleanest for family**: send the other adult/relatives a link, no team membership | **Light Beta App Review** on first build; **90-day expiry** re-uploads; Apple's rule: **don't use TestFlight to permanently avoid the App Store** |
| **Ad Hoc distribution** | Register each device **UDID** (100/yr/type), sign with an Ad Hoc profile, install via download/MDM | Great for a **fixed, known set** of family devices; **no review, no 90-day churn** | **Provisioning profile expires yearly** → annual re-sign & re-install; must collect UDIDs; clumsy install |
| **App Store (public)** | Full review, public listing | **Poor fit** — a private household app isn't "useful to the general public"; likely **rejected** or forces contrived generality | Review friction; wrong distribution model for a private tool |
| **Apple Developer Enterprise Program** | In-house distribution, no App Store | **Not eligible** — requires 100+ employees; explicitly not for families | N/A |

**Recommendation.** For the current 3-person household, **TestFlight external testers via a public link** is
the least-friction path to get the app onto everyone's phone — accept the **90-day re-upload** as a
recurring chore (scriptable in CI). If the 90-day churn becomes annoying and the device set is stable,
switch to **Ad Hoc** (annual re-sign instead of quarterly). Keep an eye on **App Store** only if Manas is
ever genuinely productized for other families (ADR 0001 §11 phase 5) — at which point a real listing +
review is the right move, not a workaround.

**Honest bottom line:** the private-family-app story on iOS is **workable but never frictionless** — budget
for the $99/yr, a periodic re-signing/re-upload ritual, and the fact that Apple actively discourages using
beta channels as a permanent private-distribution mechanism.

---

## 7. Face ID, Sign in with Apple, and the encryption failsafe

**Face ID unlock — what it actually protects.** The phone is a **client, not the box.** It does **not** hold
the platform's master DEK (ADR 0001 §15 — the DEK lives in the home server's memory, unwrapped by an admin
password/recovery key). On the phone, **Face ID (`LocalAuthentication`) gates a device-local credential** —
a passkey / session token / cached relay auth stored in the **Keychain / Secure Enclave** — used to
authenticate to Manas's **Dwarpal** service over the relay and to authorize step-up actions. So "Face ID to
unlock" means *unlock the phone's Manas session and its stored credential*, which in turn lets the phone
request box actions; it is **not** the box's master-key unlock. This keeps the failsafe intact: a stolen
phone yields **no platform data at rest** (the data is on the box, encrypted, and the phone only cached
authz-scoped DTOs it can clear on lock).

**Sign in with Apple — optional, not required.** SIWA is only **mandated by App Store review when an app
offers other third-party/social logins.** Manas offers **its own self-hosted Dwarpal** (passkeys + biometrics,
ADR 0001 §4). A private family app with **no social login has no obligation to add SIWA.** The stronger,
more on-brand choice is **passkeys via Associated Domains** (WebAuthn against Manas's own Dwarpal), unlocked by
**Face ID**, with a passcode fallback — self-hosted identity, no dependence on Apple's account graph. SIWA
stays a **P2 "nice, not needed"**; it'd only matter if productization later adds social sign-in.

---

## 8. Out of scope (and why)

- **Any domain logic, analysis, or LLM inference.** Belongs on the box (ADR 0001 §4: deterministic-C# bulk,
  thin-LLM-skin). The app calls governed GraphQL/MCP; it never becomes a second brain.
- **Being a security/authorization boundary.** Walls and step-up are **server-side and deterministic**
  (ADR 0001 §5.2). A push tap or an app screen only *requests*; the box decides. The phone is a deputy.
- **Storing the platform's data locally.** Only an ephemeral **offline queue** (outbound HealthKit deltas /
  manual entries) + authz-scoped view DTOs, cleared on lock. No mirror of Ojas/Artha/Kavach on the phone —
  that would multiply the breach surface and contradict local-first.
- **Android (now).** HealthKit is iOS-only, so the health-bridge value is inherently Apple-bound; SwiftUI is
  the right iOS call. A future Android client (with a **Health Connect** bridge) is a **separate later
  decision**, not a reason to compromise the iOS app's architecture today.
- **Re-hosting the Akriti design system natively.** The full dashboard is the **Akriti React (Vite) PWA** (ADR 0011),
  embedded via WebView and reached over the relay — reuse without re-implementation.
- **Exposing the box to the public internet.** All remote access is via the **private relay/Tailscale**; the
  only outbound cloud touchpoints are the **blind control-plane** ones iOS forces (APNs signer egress to
  Apple), never a data path.
- **Cloud health/finance backends, analytics SDKs, crash reporters that phone home.** Contradicts the
  privacy charter; any telemetry is local (Saakshi on the box).

---

## 9. Summary of the honest trade-offs

1. **Push forces an Apple-cloud dependency** even for a local-first system — unavoidable on iOS. Mitigation:
   **opaque/encrypted payloads + on-device `UNNotificationServiceExtension` decrypt**, so Apple is a dumb
   doorbell that learns only "wake device X," never the alert's content.
2. **HealthKit background sync is near-real-time at best**, best-effort, and silently empty when denied or
   backgrounded off. Ojas must treat gaps as "unknown"; the app must show "last synced N ago."
3. **Native SwiftUI means no .NET/UI code sharing** and a second toolchain — accepted, because the app's hard
   core is Swift-first native integration and its shared UI is delivered by embedding the existing PWA.
4. **iOS private distribution is workable but never frictionless** — $99/yr + a 90-day TestFlight re-upload
   ritual (or annual Ad Hoc re-sign); Apple discourages beta channels as permanent private distribution.
5. **The phone is a deputy, not the box** — Face ID unlocks the phone's session/credential, not the master
   key; security decisions stay deterministic and server-side.

---

## Sources

- [HealthKit `enableBackgroundDelivery` — Apple Developer](https://developer.apple.com/documentation/HealthKit/HKHealthStore/enableBackgroundDelivery(for:frequency:withCompletion:))
- [HealthKit background-delivery entitlement — Apple Developer](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.healthkit.background-delivery)
- [Getting Apple Health data into a backend — Open Wearables](https://openwearables.io/blog/getting-apple-health-data-into-your-backend)
- [End-to-End Encrypted Push Notifications Explained — Push0](https://push0.com/blog/end-to-end-encrypted-push-notifications-explained/)
- [Communicating with APNs — Apple Developer (archive)](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/CommunicatingwithAPNs.html)
- [Apple Push Notification service — Wikipedia (incl. 2023 privacy concerns)](https://en.wikipedia.org/wiki/Apple_Push_Notification_service)
- [Displaying live data with Live Activities — Apple Developer](https://developer.apple.com/documentation/activitykit/displaying-live-data-with-live-activities)
- [iOS Live Activities guide — Newly](https://newly.app/guides/ios-live-activities)
- [TestFlight overview — App Store Connect Help](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview/)
- [iOS App Distribution: TestFlight, Ad Hoc, Enterprise & App Store — Appcircle](https://appcircle.io/guides/ios/ios-app-distribution)
- [Why does my TestFlight build say 'Expired'? — PTKD Journal](https://ptkd.com/journal/testflight-build-expired-fix)
- [Plugin.Maui.Health (Apple Health for MAUI) — GitHub](https://github.com/0xc3u/Plugin.Maui.Health)
- [CommunityToolkit Maui.NativeLibraryInterop — GitHub](https://github.com/CommunityToolkit/Maui.NativeLibraryInterop)
- [Why MAUI Blazor Hybrid — Rockford Lhotka](https://blog.lhotka.net/2025/04/14/Why-MAUI-Blazor-Hybrid)
- [Passkeys Overview — Apple Developer](https://developer.apple.com/passkeys/)
- [Sign in with Apple requirement (2025) — WorkOS](https://workos.com/blog/apple-app-store-authentication-sign-in-with-apple-2025)
- [Using VPN On Demand for iOS — Tailscale Docs](https://tailscale.com/docs/features/client/ios-vpn-on-demand)
