# ADR 0007 — Risk Tiers & Step-Up (unified `RiskPolicy`)

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** ADR 0001 §5.2; resolves design-review A6 (and its dependents C3 / H6 / F6); relates to ADR 0003 (Dwarpal) and ADR 0006 (Automation).

> ADR 0001 §5.2 promised "risk-based step-up" as *"a generalisation of Vatara's high-risk PIN step-up and
> Steward's Tier A/B/C"* — but never defined the generalisation. This ADR defines it: **one deterministic,
> platform-wide `RiskPolicy`** that classifies every tool call into a fixed tier, names the factor each tier
> demands, and is enforced at **every MCP tool boundary — never by the LLM**. It closes three holes: three
> services building three incompatible step-ups (A6), sensitive **reads over a bare voice channel** with no
> step-up (C3/H6 — "hear an adult's balance from a room box"), and **coverage-reducing** security edits that
> neuter the alarm without ever calling `disarm` (F6 — the soft-disarm back door).

---

## 1. Context

Every money-movement, health-read, and disarm gate on the platform depends on one shared idea — *how sensitive
is this action, and what does the caller have to prove before it runs?* — that is described but not built.
Two prior schemes exist and must be unified, not re-implemented per service:

- **Vatara's high-risk PIN step-up.** In the Kaya service, disarm/unlock required a PIN entered deterministically,
  recognised on a rule-based fast-path **before** the model could tool-call it.
- **Steward's Tier A/B/C.** Finance actions were bucketed into escalating-confirmation tiers within spend limits
  (A = trivial/within-limit, B = confirm, C = strong-confirm).

Left unreconciled, Kaya, Artha, Ojas, and Abhyasa will each grow their own gate with their own vocabulary,
their own bypasses, and their own bugs. Worse, two structural gaps already exist in the current design:

1. **Voice identity is "which room," not "who."** A satellite (the voice box) maps to a room/occupant as a
   *hint* (ADR 0001 §5.7). Nothing today stops a sensitive **read** — "what's Ananth's balance," "what were
   the child's last labs" — from being spoken to a box and answered aloud to whoever is standing there.
2. **Soft-disarm.** `disarm` is gated, but **zone bypass**, **arming-profile edits**, **siren-disable**,
   **sensor-disable**, **entry/exit-delay changes**, and **security-notification muting** each reduce coverage
   just as effectively — and may be gated as ordinary config, or not at all.

This ADR makes the step-up model **one deterministic component** shared by all services, and plugs both gaps.

---

## 2. The tier model — a single `RiskPolicy` and a fixed tier enum

There is **one** platform component, **`RiskPolicy`**, living in `Manas.Kernel` (shared contracts) and called
by **every** MCP tool boundary. It is deterministic C# — no model, no judgement — and it exposes one core
function:

```
RiskPolicy.Classify(action, context) -> RiskTier
```

`RiskTier` is a fixed, ordered enum. Higher is stricter; a tool boundary demands the factor for the tier the
policy returns.

| Tier | Name | Meaning |
|------|------|---------|
| **T0** | `Ambient` | Public/house-ambient, no personal data. Time, weather, lights on/off in a shared room, "tell me a joke." |
| **T1** | `ReadPersonal` | Reads of the **caller's own** low-sensitivity personal data. Own calendar, own reminders, own school timetable. |
| **T2** | `SensitiveRead` | Reads of **finance / health / security-config** data, or **anyone's** personal data other than the caller's own low-sensitivity set. Balances, transactions, vitals/labs, arming schedule, another member's data. |
| **T3** | `Actuate` | Writes/actuation with real-world effect but **reversible & bounded**. Set thermostat, unlock an *interior* smart-lock, record a vital, small in-limit payment (legacy Steward Tier A/B). |
| **T4** | `HighRiskIrreversible` | Money movement out / above limit, **disarm/unlock of an exterior entry**, and **any coverage-reducing security change**. Legacy Vatara PIN actions + legacy Steward Tier C. |

**Legacy mapping (the generalisation ADR 0001 §5.2 owed):**

- **Vatara high-risk PIN step-up** (disarm, exterior unlock) → **T4**. The PIN survives as *the Kavach-specific
  factor* required *in addition* to the platform strong factor (§4), not as its own scheme.
- **Steward Tier A** (trivial, well within limit) → **T3** (app confirm).
- **Steward Tier B** (larger, still within policy limit) → **T3** with re-auth, escalating to **T4** as the
  amount approaches the limit (the limit is a policy input to `Classify`, not a separate tier ladder).
- **Steward Tier C** (out-of-limit / external transfer / new payee) → **T4**.

Steward's "within-limit" amount thresholds become **context inputs** to `RiskPolicy` (§3), not a parallel
enum — one ladder, parameterised, instead of two.

---

## 3. Deterministic classification `(action, context) -> tier` *(amended 2026-07-11, round 2 — closes G2, G8, S8)*

`Classify` is a pure function of the **action** (the MCP tool + parameters) and the **context**. Context carries:

- **inputModality** — `Text | Voice` — how the utterance arrived. It affects STT handling and **delivery rendering only, and NEVER changes authorization** (a mic button does not lower assurance). Normative definition in ADR 0010 §5;
- **channelAssurance** — `AuthenticatedWeb | AuthenticatedTrustedDevice | RoomSatelliteHint | None` — what the identity claim is worth; **this is the axis `RiskPolicy` consumes** (it replaces the old single `channel` input). Normative definition lives in ADR 0010 §5 — this ADR consumes it, it does not redefine it;
- **dataSensitivity** — `Ambient | Personal | Sensitive` (finance/health/security-config), from the owning service;
- **actorRole** — `Admin | Adult | Child | Automation | Guest` (the *verified* identity, ADR 0003 — never the voice hint). **`Guest`** (an unknown/unmapped speaker) is an ephemeral principal minted by Dwarpal (ADR 0010 §5; guest-token `sub` semantics in ADR 0003 §2.3) and is **T0 only, unconditionally**;
- **operationKind** — `Read | Write | Actuate | CoverageReducing`;
- **subjectIsSelf** — whether the data subject is the caller;
- **amount / limit** — for Artha, the value against the payee/daily policy limit.

The floor is **`max`**: a T3 actuation on **sensitive** data is still at least T2-gated for the read side, and
**channelAssurance** can only *raise* the required factor, never lower the tier.

**Self-recording carve-out (restores Ojas's P0 charter).** A member logging
their *own* append-only entry is a low-risk, high-frequency act the platform must not tax with a phone tap per
reading. Where `subjectIsSelf ∧ operationKind = Write ∧ append-only ∧ non-destructive` (recording one's own
vital / mood / food / reading entry), the action classifies **T1** — **including via a satellite mapped to that
member**: `channelAssurance = RoomSatelliteHint` (i.e. `acr = hint`) **suffices for a T1 self-write**, an
explicit exception to the §5 voice-raises-the-bar rule for this predicate only. **Corrections, edits, and
deletes are destructive and stay T3**; recording *another* member's data is never a self-write. Concrete table:

| Service | Action (MCP tool) | Context | → Tier |
|---------|-------------------|---------|--------|
| Kaya | `kaya.set_lights` (shared room) | any | **T0** |
| Kaya | `kaya.set_thermostat` | any | **T3** |
| Kaya | `kaya.unlock` (interior door) | any | **T3** |
| Kaya | `kaya.unlock` (exterior/front door) | any | **T4** |
| Abhyasa | `abhyasa.get_timetable` (own) | AuthenticatedWeb/TrustedDevice | **T1** |
| Ojas | `ojas.get_trend` / `get_labs` | **any** | **T2** |
| Ojas | `ojas.record_vital` (own, append-only) | satellite, mapped self | **T1** (self-write carve-out) |
| Ojas | `ojas.record_vital` (correction/delete, or another's) | any | **T3** |
| Artha | `artha.get_budget` / `get_balance` | **any** | **T2** |
| Artha | `artha.pay` (in-limit, known payee) | AuthenticatedWeb/TrustedDevice | **T3** |
| Artha | `artha.pay` (out-of-limit / new payee) | any | **T4** |
| Artha | `artha.transfer` (external) | any | **T4** |
| Kavach | `kavach.arm` (coverage-**increasing**) | any | **T3** — exempt from the §5 step-up (arming is the safe direction; §6 untouched) |
| Kavach | `kavach.disarm` (exterior) | any | **T4** |
| Kavach | `kavach.bypass_zone` | any | **T4** (§6) |
| Kavach | `kavach.set_arming_profile` / `set_entry_delay` | any | **T4** (§6) |
| Kavach | `kavach.disable_siren` / `disable_sensor` | any | **T4** (§6) |
| Kavach | `kavach.mute_security_notifications` | any | **T4** (§6) |
| *any* | *any T2+ action* | **channelAssurance = RoomSatelliteHint\|None** | **step-up forced** (§5) |
| *any* | **all T4; any actuation with a Tainted trigger chain; money movement & disarm/unlock always** | **actorRole = Automation** | **refused** (§8, ADR 0006 §3–4) — Trusted-trigger T3 (e.g. `kavach.arm`) is **permitted** under machine-grants |

Two rules are non-negotiable and encoded in the table's last rows: **a hint-grade channel raises the bar for
T2+** (§5) and **automation cannot self-satisfy T4 or a tainted actuation** (§8).

---

## 4. Required factor per tier *(amended 2026-07-11, round 2 — closes S7)*

Each tier names what must be presented, verified **deterministically by Dwarpal (ADR 0003)** at the tool boundary.
Dwarpal issues a short-lived **delegation token** carrying the achieved assurance; `RiskPolicy` checks the token's
assurance against the tier.

| Tier | Factor required |
|------|-----------------|
| **T0** | None. Anonymous/ambient is fine. |
| **T1** | An **authenticated session** (the caller is a verified identity, ADR 0003). |
| **T2** | **Recent strong-auth** — a fresh authenticated session on a **known device**, or a re-auth if the session is stale (bounded freshness window). A **bare voice channel never satisfies T2** (§5). |
| **T3** | An explicit **PIN or one-tap app confirm** naming the action (§7). Reversible actuation, so a device-bound confirm suffices. |
| **T4** | A **strong factor — passkey/WebAuthn or Face ID — on a TRUSTED DEVICE** (the enrolled iOS app), bound to the exact action (§7). **For Kavach specifically**, the strong factor **plus the Kavach PIN** (defence in depth — the legacy Vatara PIN, retained). |

"Recent strong-auth" and "trusted device" are Dwarpal concepts; `RiskPolicy` only asserts the **required** level and
lets Dwarpal prove it. The freshness window and trusted-device enrolment are Dwarpal's (ADR 0003) to size.

**Honest exceptions (availability trade-offs, named — not hidden).**

- **The Layer 0 keypad disarms with a panel-verified PIN alone** ([ADR 0008](0008-availability-degraded-mode-and-dr.md) §2).
  When the box — and therefore the whole .NET/Dwarpal stack — is dead, the standalone alarm must still be operable,
  so the local entry-door keypad clears the alarm on a **PIN verified by the panel itself**, with **no** platform
  strong factor. This is an **accepted availability trade** (the alarm must be operable with the box dead), and is
  documented here as the **floor of effective disarm assurance** for the house: whatever T4 demands on the smart
  path, the box-dead path is panel-PIN. Mitigations (owned with Kavach/Dwarpal): **panel-side lockout /
  anti-bruteforce**, a keypad PIN kept **distinct from the step-up factors**, and **tamper alerting** on the panel.
  While no dedicated Layer-0 hardware panel is fitted, the console keypad currently serves as this floor; the
  keypad PIN is kept structurally distinct from the step-up factors (it lives only in Kavach's own user-code
  store, never compared against them at issuance time).
- **No-phone T4 fallback.** When **no enrolled phone is reachable**, a **console passkey (platform authenticator)
  + the Kavach PIN** satisfies the T4 factor — so a dead or absent phone does not block money moves or
  coverage-changes outright. This resolves the **availability half** of OQ-5; enrolment-loss recovery (the
  *permanently lost* trusted phone) remains a Dwarpal-owned open item (§10).

  *(A later ADR refines this fallback's enrollment-time policy in detail — including a two-authenticator floor
  and a child-specific ceiling that keeps a minor's passkey from satisfying a T4 factor on its own — building on
  the commitments recorded here.)*

---

## 5. Voice is a HINT, not identity *(amended 2026-07-11, round 2 — closes G2, C3/H6)*

The satellite→occupant mapping is a **weak signal about which room**, not proof of **who** is speaking (and
voiceprint, when it lands, is a *convenience* signal, still not a strong factor). Authorization keys off
**`channelAssurance`, never `inputModality`** — *speaking rather than typing changes delivery, not what you may
do*; modality-aware rendering is the Governor's job, owned per ADR 0010 §5. Therefore:

- **"Bare voice channel" is now defined precisely** as `channelAssurance ∈ {RoomSatelliteHint, None}` — a
  satellite room-hint or an unauthenticated context. (The previously-undefined term is closed; the normative
  split lives in ADR 0010 §5, which this ADR consumes.)
- An **authenticated console session carries its session assurance** (`AuthenticatedWeb` /
  `AuthenticatedTrustedDevice`) **whether the member types or speaks** — a mic button in a logged-in console is
  **not** a bare voice channel, and a T2 read there is served within the freshness window with no phone tap.
- A **T0/T1** turn may be served over a bare voice channel on the mapped identity's behalf (ambient, or the
  mapped occupant's own low-sensitivity data — and even T1 can be tightened per-home). The **self-recording
  carve-out** (§3) additionally admits a T1 *own append-only write* on `acr = hint`.
- **Any T2+ request where `channelAssurance ∈ {RoomSatelliteHint, None}` REQUIRES step-up on a trusted
  device.** "What's my balance," "read my labs," "what's tonight's arming schedule" spoken to a box → the box
  does **not** speak the answer; it triggers a **phone confirm** (Face ID one-tap) and delivers per the ADR 0010
  §5 delivery policy (to the phone by default), not to the room. This closes C3/H6: **you cannot hear an adult's
  balance or health simply by talking to a room box.**
- **Coverage-increasing security ops are exempt from this row.** `kavach.arm` (and any coverage-*increasing*
  op) is the *safe direction* — arming by voice or schedule needs **no** step-up. Coverage *reduction* stays
  **T4** (§6), untouched.
- A **spoken passphrase is explicitly NOT sufficient for T3+** (it is replayable audio in a shared room). T3/T4
  demand the device-bound factor of §4, never "say the magic word."

This rule is enforced by the `channelAssurance = RoomSatelliteHint|None` row in §3: a bare voice channel can
*request* anything, but the factor bar for T2+ is raised to a trusted-device confirm regardless of the tier's
baseline.

---

## 6. Coverage reduction is disarm-class (T4)

**Any operation that reduces the alarm's protective coverage is classified exactly like `disarm` (T4).** There
is **no** lower-gated "config" path that can neuter Kavach. Explicitly T4:

- **zone bypass** (excluding a door/window from the armed set),
- **arming-profile edits** (which zones arm in Home/Away/Night),
- **siren-disable** and **sensor-disable**,
- **entry/exit-delay changes** (a long entry delay is a silent disarm window),
- **muting security notifications** (an un-alerted breach is an un-alarmed one).

Rationale: an attacker (or a curious child, or a mis-routed model) who cannot `disarm` must not be handed an
equivalent outcome through a differently-named tool. These all carry the same **T4 factor** — strong factor on a
trusted device **plus** the Kavach PIN — and the same anti-replay binding (§7). Kavach registers every such tool
as `operationKind = CoverageReducing`, and `RiskPolicy` maps that kind to **T4** unconditionally.

---

## 7. Anti-replay binding (nonce + exact action + TTL)

A step-up approval is **not** a reusable "user is elevated" flag. Each approval is a **single-use assertion
bound to**:

- a **nonce** (server-minted per request; the approval is void once redeemed);
- the **exact action and its parameters** (a hash of `tool + normalised args` — "disarm the **front-door** zone,"
  "pay **$4,000** to **payee X**"), so an approval for one action **cannot** be replayed against another;
- a **short TTL** (seconds — long enough for a human tap, too short to stockpile), after which it is dead.

The confirmation the human sees names the **concrete action** — *"Disarm the front-door zone now?"*, *"Send
$4,000 to Acme now?"* — never a generic *"Approve?"*. Dwarpal (ADR 0003) mints the nonce, binds the approval to the
action hash + TTL, and the owning service **redeems it once** at execution; a redeemed or expired or
wrong-action assertion is refused. This makes a captured confirmation useless against a *different* or *later*
action.

---

## 8. Enforcement point — every tool boundary, never the LLM *(amended 2026-07-11, round 2 — closes S4, S10)*

`RiskPolicy` is called **at each MCP tool boundary**, by the owning service, on the **verified identity**
(identity propagation, ADR 0001 §5.2) — the same place the per-user data wall runs. Concretely, **`RiskPolicy`
is invoked in the owning service's APPLICATION layer — below both the MCP tool boundary and the GraphQL
resolvers — so both ingresses share one gate; a per-service parity contract test asserts every tool/query pair
classifies identically** (closes the GraphQL-ingress-parity gap, S10). The sequence:

1. A request arrives (voice/web/phone) → Sarathi turn begins.
2. **`RiskPolicy.Classify` runs FIRST**, deterministically, before the model would choose a tool. A **T4** op is
   recognised on the **rule-based fast-path**: Sarathi does **not** route it as a free
   model tool-call — it goes Classify → Dwarpal step-up → the **owning service executes deterministically**.
3. For T1–T3, the model may select the tool, but the tool boundary **still** calls `RiskPolicy` + Dwarpal before
   acting — the model's choice is never the gate. The service is the deterministic backstop even if Sarathi's
   loop is wrong or compromised.
4. Every classification + step-up outcome is a **Saakshi security event** (who, action, tier, factor, allow/deny),
   ADR 0001 §5.4.

**Automation refusal is scoped, not blanket (ADR 0006 §3–4).** An unattended ECA rule runs as a least-privilege
principal; `RiskPolicy` classifies its actions with `actorRole = Automation` and refuses **(a)** all **T4**,
**(b)** any **actuation whose trigger chain carries `Taint`** (ADR 0006 §3 — a crafted email or OCR'd bill can
inform and propose, never actuate), and **(c)** **money movement and disarm/unlock always** (ADR 0006 §4, even
from a fully-Trusted trigger). **Trusted-trigger T3 actuation is PERMITTED** — a coverage-*increasing*
`kavach.arm` (scheduled / geofenced auto-arm), a `kaya.set_thermostat` schedule — running unattended **under the
rule-owner principal's explicit, enumerated machine-grants** (ADR 0006 §2), never as a human super-user. A rule
may always *request* a refused action — queue a payment or a disarm for a human to approve via the same
nonce-bound confirm (§7) — but it can **never** complete one itself. (This narrows the earlier blanket "refuse
any T3/T4," which had wrongly killed Kavach scheduled/geofenced auto-arm and Kaya thermostat schedules — S4.)

**Amendment, 2026-08-15 (unit AUTO-5-RULE-GRANT-INDEX) — Automation write authority is a LIVE, use-time grant
derived from the standing-rule table itself.** A money-move-proposal enqueue write (enqueue only, never itself a
money movement) reaches the Kernel wall's own defense-in-depth
gate as every other Automation write does; this
unit closes a platform gap that had made that gate refuse EVERY such write
unconditionally (no grant-index implementation existed that could ever grant a machine principal). The fix is a
real grant index whose Automation grants derive from active
`StandingRule` rows, via a fan-out over per-domain automation grant sources (today: a narrow one scoped ONLY to
Artha's money-move-proposal rows). The grant check
is re-run LIVE, at the exact instant of the write, scoped to the SPECIFIC firing rule — never a token claim baked
in at mint time (the explicitly rejected alternative: threading a rule-id claim through `ActorContext`/Dwarpal
tokens, which would reintroduce a grant-time-vs-use-time drift this section's own discipline exists to close
at the money-classification layer). The entity being written carries its own firing rule's id, read live off the
row, never a pre-minted claim; deactivating a
`StandingRule` refuses its writes on the very next check, even while the SAME owner holds another
simultaneously-active rule whose writes remain live. See ADR 0004 §5's own amendment for the sharing-model side of
this mechanism.

---

## 9. Acceptance criteria / tests *(amended 2026-07-11, round 2 — closes S17, S4)*

Each maps to a deterministic test in `Manas.Kernel` / the owning service:

1. **Sensitive read over a bare voice channel demands step-up.** `artha.get_balance` with
   `channelAssurance = RoomSatelliteHint` (spoken to a satellite) classifies **T2** and is refused until a
   trusted-device confirm; the box does not speak the balance. The *same* read on an authenticated console mic
   (`AuthenticatedWeb`) is served within the freshness window. (C3/H6)
2. **Zone-bypass == disarm.** `kavach.bypass_zone`, `set_entry_delay`, `disable_siren`, and
   `mute_security_notifications` each classify **T4** and demand the *same* factor as `kavach.disarm`. (F6)
3. **Replayed approval is refused.** A step-up assertion redeemed once, or presented against a *different*
   action hash, or after TTL expiry, is rejected by the owning service.
4. **No T4 executes without the boundary gate, regardless of routing.** No T4 action executes without a boundary
   `Classify` + step-up, *regardless of routing*; the pre-model fast-path (ADR 0010 §7) is a UX/latency layer
   whose **misses MUST still be caught at the tool boundary** — asserted by a **paraphrase corpus** that evades
   the lexical fast-path yet still classifies + step-up-gates at the boundary. (S17)
5. **Automation refusal is scoped, not blanket.** An ECA rule attempting a **T4**, a **tainted actuation**, or
   **money movement / disarm** is refused with `actorRole = Automation` and may only enqueue a human-approval
   request; **and the positive case:** a **Trusted**-trigger scheduled `kavach.arm` under an explicit
   machine-grant **executes unattended** — no step-up (coverage-increasing, the safe direction). (ADR 0006 §3–4)
6. **Legacy mapping holds.** A Steward Tier-C transfer classifies **T4**; a Vatara-era PIN disarm classifies
   **T4** and additionally requires the Kavach PIN.
7. **Fail-closed classification.** An MCP tool with no registered sensitivity/operationKind causes a **startup**
   failure (like the Kernel's `[Encrypted]` tier check, ADR 0002 §4) — never a silent T0 default.
8. **Voice passphrase insufficient.** A spoken passphrase never satisfies T3+; only the device-bound factor does.
9. **Self vs other.** `ojas.get_labs` for the caller's own child (guardian) is served; for another adult it is
   walled at the data layer **and** classified T2 — both gates hold independently.

---

## 10. Open questions *(amended 2026-07-11, round 2 — OQ-5 partially resolved, S7)*

1. **Freshness window for T2** — how stale a session may be before a sensitive read forces re-auth (minutes?);
   owned jointly with Dwarpal (ADR 0003), sized against real usage.
2. **Child T1 scope** — exactly which of a child's own reads are T1 (served on the box) vs T2 (parent
   confirm); a per-home policy knob, defaulted conservative.
3. **Voiceprint as a factor tier** — if/when speaker-id lands, does it lift a voice turn from "hint" to a *weak*
   factor (enough for T1-on-a-child, never for T3+)? Deferred; the enum already supports it as context, not proof.
4. **Artha limit source** — where the per-payee/daily limits that feed `amount/limit` context live (Artha
   policy vs a platform policy store), and how they are themselves step-up-gated to edit (a limit raise is T4).
5. **Trusted-device enrolment/loss** *(partially resolved, round 2)* — the **no-phone T4 fallback factor is now
   defined** (console passkey + Kavach PIN, §4), closing the availability half. What **remains** open is
   enrolment-loss recovery — the path when the one trusted phone is permanently *lost* (re-enrolment) — owned
   with Dwarpal (ADR 0003).

   *(A later ADR narrows this further: who may authorise a re-enrolment — an already-enrolled adult stepped up
   on an authenticator that is not the one being replaced, or, absent such an authenticator, physical presence
   at the box under argued-equivalent assurance. The phone-specific ceremony (platform-gated) stays open.)*

---

## 11. Consequences

- **One gate, not three.** Kaya, Kavach, Artha, Ojas, Abhyasa all call the same `RiskPolicy` + Dwarpal at their
  tool boundaries; A6 is resolved — there is a single, testable definition of "step-up," and the two legacy
  schemes are folded into it (Vatara PIN → T4's Kavach factor; Steward A/B/C → tiers + amount context).
- **The two structural holes are closed by construction.** Voice can't leak a sensitive read (§5); nothing can
  soft-disarm the alarm through a side door (§6). Both are enforced by the classification table, not by hoping
  each service remembers.
- **Anti-replay is real.** A captured "approve" can't be re-used against another action or later (§7) — the
  confirmation is action-specific and single-use.
- **The LLM is never the gate.** T4 is deterministic fast-path; T1–T3 still re-check at the tool boundary. A bad
  prompt, a spoofed voice, or a compromised Sarathi loop fails at deterministic code (ADR 0001 §5.2).
- **Modest, shared machinery.** A `RiskTier` enum, a `Classify` function, a per-tool sensitivity/operationKind
  registration (fail-closed at startup), and Dwarpal's nonce-bound assertion. Invisible to routine service code —
  a normal read "just works" at its registered tier.
- **Constrains ADR 0003 (Dwarpal)** to deliver the assurance levels, trusted-device enrolment, freshness window,
  and nonce/TTL-bound action assertions this ADR requires; and **ADR 0006 (Automation)** to run rules as a
  principal `RiskPolicy` can recognise and refuse for T3/T4.

---

*Refines [`0001-manas-platform-architecture.md`](0001-manas-platform-architecture.md) §5.2. Cross-references
[`0002-key-hierarchy-and-unlock.md`](0002-key-hierarchy-and-unlock.md)
§4 (fail-closed tier resolution), and sibling ADRs [`0003-identity-propagation-and-pki.md`](0003-identity-propagation-and-pki.md)
(Identity) and [`0006-automation-principal-and-event-bus.md`](0006-automation-principal-and-event-bus.md) (Automation). Resolves design-review A6
(C3 / H6 / F6).*
