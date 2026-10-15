---
title: Architecture Decision Records
---

# Manas — Architecture Decision Records

One decision per file. **ADR 0001 is the root** — read it first; no later ADR contradicts it without explicitly
superseding it. A later ADR refines one of two things: a section of ADR 0001, or another ADR / vision doc.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-manas-platform-architecture.md) | **Manas Platform Architecture** (root) | Accepted |
| [0002](0002-key-hierarchy-and-unlock.md) | Key Hierarchy & Unlock Model | Accepted |
| [0003](0003-identity-propagation-and-pki.md) | Identity Propagation, Delegation & Internal PKI | Accepted |
| [0004](0004-walling-maysee-and-sharing.md) | Walling, may-see & Sharing (all five stores) | Accepted |
| [0005](0005-audit-pipeline-and-tamper-evidence.md) | Audit Pipeline & Tamper-Evidence | Accepted |
| [0006](0006-automation-principal-and-event-bus.md) | Automation Principal & Event Bus | Accepted |
| [0007](0007-risk-tiers-and-step-up.md) | Risk Tiers & Step-Up (unified risk policy) | Accepted |
| [0008](0008-availability-degraded-mode-and-dr.md) | Availability, Degraded-Mode, DR & Incident Response | Accepted |
| [0009](0009-data-lifecycle-retention-forget-consent.md) | Data Lifecycle: Retention, Right-to-Forget & Consent | Accepted |
| [0010](0010-agent-conversational-core.md) | **Sarathi (the Agent): Conversational Core, Channels & Conversation Ownership** | Accepted |
| [0011](0011-ui-framework-and-premium-stack.md) | **UI Framework & Premium Front-End Stack (Akriti)** | Accepted |
| [0012](0012-skill-tutoring-capability-modules.md) | **Skill-Tutoring Capability Modules** (grader × content × model × UI × persona) | Accepted |
| [0013](0013-shared-read-model-and-trend-kernel.md) | **Shared Read-Model & Trend Kernel** (walled LLM-consumable aggregation/trends) | Accepted |
| [0014](0014-conversational-overreliance-teeth.md) | **Conversational Over-Reliance & Engagement-Integrity Teeth** | Accepted |
| [0015](0015-bonnie-mascot-and-cross-surface-animation.md) | **Bonnie the Mascot & Cross-Surface Animation** (web + satellite device) | Accepted |
| [0016](0016-mukha-animation-subsystem.md) | **Mukha (मुख): the Animation & Mascot Subsystem** | Accepted |
| [0018](0018-artha-household-finance.md) | **Artha (अर्थ): Household Finance** | Accepted |
| [0019](0019-ios-client-and-distribution.md) | **iOS / Phone Client & Distribution** | Accepted |
| [0020](0020-sangam-rail-and-act-grants.md) | **Sangam (सङ्गम): OAuth Vault, Grants & the Rail Execution Contract** | Accepted |

> A handful of later ADRs (numbered 0021–0023) are process- and infrastructure-focused working documents —
> build-agent contracts, device-enrollment security review detail, and cross-project inference-server sharing
> notes — and are not included in this public copy; nothing in the table above depends on their content to be
> understood.

## The critical cluster (why 0002–0007 exist)

An early adversarial design review found the real defects clustered in the **internal security/data spine** —
the "cheap now, brutal to retrofit" mechanisms a walking skeleton is meant to *retire*. As written, ADR 0001
couldn't decide them from the text alone, so each became its own ADR **before any code**: keys (0002), verified
identity (0003), fail-closed walls (0004), a buildable audit chain (0005), a non-super-user automation principal
(0006), and one unified step-up model (0007). Availability/DR (0008) and data-lifecycle/forget (0009) cover the
operational and compliance findings the design under-served.

## Why 0010 exists

A later review found **Sarathi — the platform's center — undesigned**: of its three interaction paths (console
text, console voice, satellites), one didn't exist yet and another was a folder name; there was no conversation
ownership, no streaming, no loop contract. ADR 0010 resolves that cluster.

## Conventions

- Header block: **Status · Date · Deciders · Refines**.
- A leading blockquote states the doc's purpose; each ADR ends with **Acceptance criteria / property tests**,
  **Open questions**, and **Consequences**.

## Related architecture docs (not ADRs, but load-bearing)

[Kernel spec](../architecture/kernel-spec.md) ·
[Service blueprint](../architecture/service-blueprint.md) · [Persona layer](../architecture/persona.md) ·
[Release & updates](../architecture/release-and-updates.md) ·
[Tech-stack review](../architecture/tech-stack.md) · [Feasibility study](../architecture/feasibility-study.md).
