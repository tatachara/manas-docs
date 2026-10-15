# Manas — the Followup Thesis (synthesis of the service visions)

- **Status:** Living synthesis · **Date:** 2026-07-11
- **Reads on top of:** [`../adr/0001-manas-platform-architecture.md`](../adr/0001-manas-platform-architecture.md)
- **Synthesises the seven vision docs** (the detail lives in each; this is the connective tissue):
  [home](home.md) · [artha](artha.md) · [ojas](ojas.md) · [abhyasa](abhyasa.md) ·
  [ios-app](ios-app.md) · [voice-edge](voice-edge.md)

> Planning artifact. No code. This thesis exists to make the *whole* legible at once — the one-mind story, how
> the services feed each other, the honest MVP, and the phasing — so that when we build, we build the right
> thing in the right order and are never surprised by something that was knowable now.

---

## 1. The thesis in one paragraph

**Manas is the shared mind of a household.** Four domain services — the **Kaya** (senses & hands), **Kavach**
(the shield), **Artha** (the money), **Ojas** (body & mind), **Abhyasa** (the children's learning) — each own
their data and expertise, and one conversational **Sarathi** reasons and acts *across* them, wearing a different
**persona** for each person (dry *Anik* for Ananth; loving **Bonnie** for the child). It runs entirely on the
family's own hardware, data never leaving home, with **deterministic C# doing the real work** and the local LLM
a **thin skin** for conversation and framing. The product is not four apps; it is one mind that happens to have
four kinds of expertise — and the value compounds where they meet.

## 2. The seven pillars (intents, one line each)

| Pillar | Intent (one line) | Maturity |
|---|---|---|
| **Kaya** | The home's senses & hands — deterministic, guarded device control via Home Assistant; never actuate on a guess. | new (HA exists) |
| **Kavach** | A private, no-subscription ADT replacement — arm/disarm/panic deterministic + step-up, fail-safe, self-checking. | new |
| **Artha** | Watches the family's money and acts in tiers within set limits, per-user walled. | **most mature** (already deep) |
| **Ojas** | The family's private health record & body-and-mind coach — wellness, never diagnosis. | new |
| **Abhyasa** | The children's learning steward — record grades, surface where a child thrives and where a nudge helps. | new |
| **iOS app** | The thin native bridge — the only place HealthKit, push, Face ID, widgets can happen. | new |
| **Voice & edge** | The voice box satellites — the home's ears/mouth/face; cognition stays central. | partly (Vatara) |

## 3. The one mind — the interconnection graph

This is *why* it's a platform and not a bundle. Every arrow is a **governed cross-service tool call carrying the
caller's identity** — never a shared database. A few load-bearing flows:

- **Ojas → Artha → Kaya:** a glucose trend suggests a diet change → Ojas asks Artha for the grocery budget
  and Sangam for the shopping list → the nutrition plan respects real money.
- **Kavach ↔ Kaya:** Kaya exposes *raw device capability*; **Kavach owns the security meaning** (arm/disarm,
  intrusion, sirens). A single, sharp life-safety boundary — no ambiguity about who decides the alarm.
- **Kaya → Artha:** Kaya meters **kWh**; Artha turns it into **dollars** and flags waste.
- **Abhyasa → Ojas:** exam-stress signals (late nights, low mood) can gently surface a wellness nudge.
- **Kavach → iOS:** an armed-away intrusion → an encrypted APNs "dumb doorbell" → a Live Activity + one-tap on
  the phone.
- **Everything → Saakshi/Bodha:** every hop is logged (encrypted, redacted) and the platform stays self-aware.

The graph is realised by three **foundation spines that make interconnection safe**: **Kutumba** (who's
who), **identity propagation + per-user walls** (a curious child can't reach an adult's money), and
**Sarathi + MCP** (the only cross-service path).

## 4. Cross-cutting themes that repeated across all seven docs

1. **Deterministic core, LLM as skin** — every doc puts the math/rules/tracking in C# and confines the model to
   explaining, framing, and (child-safe) encouragement. Ojas even proved it (chanting graded by **DSP**, not
   ASR); Steward already does it (statistical detection, no LLM).
2. **The LLM is never a boundary** — not the alarm decision (Kavach), not access (walls), not a health/finance
   number. Structural, deterministic guardrails throughout.
3. **Safety/guardrail layers built first** — Ojas builds a wellness-framing + crisis-detector layer *before*
   features; Abhyasa fences the tutor from doing homework; Kavach fails safe.
4. **Honest feasibility, flagged not hidden** — school-portal imports are limited (Canvas token / email digest
   best; PowerSchool has no parent API); HealthKit sync is best-effort; APNs routes through Apple; custom
   wake-word is UNPROVEN. These are **risks with fallbacks**, not assumptions (see the risk register).
5. **Pluggable capability modules** — the mantra-chanting module in Ojas is the canonical example: a feature is
   a module against an extension point, optionally with its own small model.

## 5. The honest MVP — what "P0 across services" actually is

Do **not** build all five services at once. The MVP is a **walking skeleton + one vertical slice**, then grow:

**Phase 0 — Walking skeleton (foundation, proves every hard seam):**
`Manas.Kernel` (base entity + wall filter + audit/attribution interceptor) + **`Manas.Crypto`** (envelope
encryption, hybrid unlock) + **Saakshi** (encrypted interaction log) + **Medha** (one warm model) + **Dwarpal**
(one admin login that unlocks the box) + **Sarathi** calling **one trivial tool** through to **one service's
own DB** — walled, attributed, audited, and *tested* (10× from the start). This single skeleton exercises
crypto, walls, audit, identity propagation, MCP, and the Sarathi loop before anything is expensive to change.

**Phase 1 — First vertical slice (pick ONE, end-to-end on the skeleton):**
- Recommended: **Abhyasa "record a grade → trend → improvement suggestion"** *or* **Ojas "record a BP →
  trend → wellness note"**. Both are almost-pure deterministic (low LLM risk), exercise write/read/recommend +
  walls + the child-safe/wellness guardrail, and give Ananth something *real and useful* fast.
- Steward is the tempting shortcut (it's already deep) — but re-homing it onto the platform is Phase 2, after
  the seams are proven on a *fresh* small service.

**Phase 2+:** re-home Steward; add Kaya (device control) + Kavach (security) once the box + satellites are in;
the iOS app when Ojas needs HealthKit; voice satellites + persona once the wake-word spike resolves.

## 6. The complete foundation (spines) — recap

Kernel · **Crypto** · Contracts · **Medha** (LLM gateway) · **Akriti** (UI) · **Sangam** (connectors) ·
**Saakshi** (logging/audit) · **Bodha** (self-awareness) · **Dwarpal** · **Kutumba** · **Vesha** (§5.6) ·
**Voice/edge** (§5.7). *(Canonical spine list lives in ADR §5.3.)* Three spines this synthesis surfaced as *missing* and recommends adding:
- **Notifications** — email/SMS/push/Slack as one shared spine (Steward already has it; promote it).
- **Interconnection/automation engine** — a shared event bus + cross-domain ECA rules (Vatara's rule engine
  generalises); this is what makes §3's flows *proactive* rather than only on-request.
- **Onboarding/setup** — first-run: create admin, generate keys + **recovery key**, connect integrations,
  define the household. Nowhere yet; big for a product.

## 7. Design-artifact roadmap (what's recorded vs next)

- ✅ ADR 0001 (root architecture) · ✅ 7 service vision docs · ✅ **this thesis** ·
  ✅ [resource budget](resource-budget.md) · ✅ the risk register.
- ⏭ ADRs 0002+ per open decision (unlock model detail, IdP choice, Medha gateway, wake-word spike result,
  Notifications + interconnection spines) · per-service data-model & GraphQL/MCP contract specs · key-flow
  sequence diagrams · threat model · deployment topology (box + satellites + iOS).

## 8. Bottom line

The vision is coherent, the services genuinely compound, and — critically — the honest limits are **named and
carry fallbacks**, not buried. The two things that decide feasibility (does it fit the hardware; does the
wake-word work) are addressed in the [resource budget](resource-budget.md) and the risk register
respectively. Nothing here is blocked on "different hardware" — the home server has the headroom; the
hardware-adjacent unknown *inside Manas* (GPU compute shared by LLM + Frigate) has a measured spike and an
OCuLink-eGPU fallback. **Corrected 2026-08-03 — it is no longer the only one:** two sibling projects
(Mūla / Nakṣatra) share this box and will share one `llama-server`, and **Manas takes precedence
over both** (the consequence and its gate are tracked in the living build-status document). **We can build this — deterministically, in slices, on the box we have — and we know where the sharp
edges are before we start.**
