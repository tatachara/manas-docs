---
title: Manas
---

# Manas

**मनस् — "the mind."** Manas is a private, local-first software platform for running a household. It is built
and run by one person — me — on hardware I own, in my own home. There is no company, no cloud service, and no
public sign-up: this site exists because part of building it honestly means writing the architecture down where
it can be checked, and because a banking integration (Plaid) requires a public privacy policy.

Manas runs across every domain of family life:

- **Family finance** (Artha) — accounts, transactions, budgets, and a teaching-sandbox allowance ledger for a
  child, read via Plaid, never used for advertising or sold.
- **A child-safe conversational assistant** (Sarathi, with a per-person persona) — one reasoning core that wears
  a different face for each household member; the child's assistant is built to be loyal, patient, and safe
  above all else.
- **Home safety and security** (Kavach) — deterministic alarm, panic, and life-safety logic that does not depend
  on a cloud service or a language model to work.
- **Health** (Ojas) and **learning** (Abhyasa) — tracked locally, walled per family member, never shared outside
  the household.

## The core engineering principle

Almost everything here follows from one rule: **the language model is never a security or safety boundary.**
Access control, money movement, arm/disarm, and per-person data walls are deterministic code — proven with
tests, not with a prompt. The model is a thin conversational skin on top of a system that would still be safe
if the model were removed entirely. The [platform architecture decision](adr/0001-manas-platform-architecture.md) starts from this.

## Read more

- **[Privacy Policy](privacy-policy.md)** — what data Manas collects, how it's used, and your rights.
- **[Security Policy](security-policy.md)** — how data is encrypted, stored, and protected.
- **[Vision](vision/0000-manas-thesis.md)** — why Manas exists and what it's for, service by service.
- **[Architecture decisions](adr/README.md)** — the numbered ADRs that record how the platform is actually built.
- **[Architecture reference](architecture/kernel-spec.md)** — the deterministic kernel, threat model, deployment
  topology, and key end-to-end flows.

---

Author: Ananth Tatachar
