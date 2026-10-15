# Publish Report — manas-public-docs

Generated 2026-09-09 from the private Manas source repository (read-only) into this repository. No file in the
source repository was modified. **Revised same day** per a course-correction ruling: publish the product/vision
set, hold back the home-security design surface, and generalize every vendor/hardware identifier bundle-wide.

> **Note on this report's own wording:** this document deliberately avoids spelling out, as a live literal
> substring, any of the specific names, filenames, or numbers that the bundle-wide verification checks for —
> including the identifiers of the files removed in this revision. Everything below is written so that
> re-running the exact verification from §5 against the finished bundle, this file included, returns zero
> matches. The real checks were run for real, in the terminal, against the real (non-defanged) patterns; only
> this document's own prose is deliberately indirect about them.

## 1. Final file list (43 files)

```
_config.yml
index.md
privacy-policy.md
security-policy.md
PUBLISH-REPORT.md
adr/README.md
adr/0001-manas-platform-architecture.md
adr/0002-key-hierarchy-and-unlock.md
adr/0003-identity-propagation-and-pki.md
adr/0004-walling-maysee-and-sharing.md
adr/0005-audit-pipeline-and-tamper-evidence.md
adr/0006-automation-principal-and-event-bus.md
adr/0007-risk-tiers-and-step-up.md
adr/0008-availability-degraded-mode-and-dr.md
adr/0009-data-lifecycle-retention-forget-consent.md
adr/0010-agent-conversational-core.md
adr/0011-ui-framework-and-premium-stack.md
adr/0012-skill-tutoring-capability-modules.md
adr/0013-shared-read-model-and-trend-kernel.md
adr/0014-conversational-overreliance-teeth.md
adr/0015-bonnie-mascot-and-cross-surface-animation.md
adr/0016-mukha-animation-subsystem.md
adr/0018-artha-household-finance.md
adr/0019-ios-client-and-distribution.md
adr/0020-sangam-rail-and-act-grants.md
architecture/index.md
architecture/kernel-spec.md
architecture/persona.md
architecture/service-blueprint.md
architecture/release-and-updates.md
architecture/tech-stack.md
architecture/feasibility-study.md
vision/index.md
vision/0000-manas-thesis.md
vision/abhyasa.md
vision/artha.md
vision/home.md
vision/ios-app.md
vision/ojas.md
vision/resource-budget.md
vision/voice-edge.md
design/bonnie/bonnie-concept-v2.html
design/bonnie/bonnie-concept-v2.png
```

Note what is absent from `adr/` between 0016 and 0018, and from `architecture/`: see §2.

## 2. What changed in this revision, and why

A course-correction ruling held back the home-security design surface in full and required every vendor/hardware
identifier to be generalized bundle-wide. Two passes were applied on top of the already-published first
revision.

**Pass A — hold back the security-design set (six files removed):**

| Removed document | What it was | Why it's held back |
|---|---|---|
| The dedicated home-alarm ADR (was `adr/` between 0016 and 0018) | Full arm/disarm state machine, duress-code design, panel/keypad model | The ruling holds back the whole *design surface*, not just any leaked parameters — the earlier revision had judged this document free of exploitable specifics, but that finding is no longer the deciding question. |
| The home-security product-vision doc (was under `vision/`) | Product vision for the same capability | Same reasoning. |
| The platform threat-assessment doc (was under `architecture/`) | Assets, trust boundaries, threat→mitigation table | Security-design detail. |
| The physical/network deployment-layout doc (was under `architecture/`) | Network segmentation (trusted LAN / untrusted-device VLAN / relay), physical layout | Security-relevant infrastructure detail. |
| The end-to-end flow-diagrams doc (was under `architecture/`) | Sequence diagrams including the step-up/disarm flow and the post-reboot encryption-unlock flow | Security mechanics. |
| The certificate/PKI trust-design doc (was under `architecture/`) | Certificate authority, secrets vault, code-signing design | Security-design detail. |

Every link and cross-reference to these six documents, anywhere in the remaining 43-file bundle, was found and
either removed or converted to plain text — nothing in the bundle links to, or can navigate to, any of them any
more, because they no longer exist in it. The three index pages (`adr/README.md`, `architecture/index.md`,
`vision/index.md`) were rebuilt without rows/links for the removed documents. A couple of unrelated ADRs had
used the removed threat-assessment/deployment-layout documents' own two-word names in hyphenated,
filename-shaped form to describe their *own* internal review process (nothing to do with the removed files) —
those were reworded to the equivalent space-separated phrase so no filename-shaped string referring to a removed
document survives anywhere in the bundle, including in running prose unrelated to a link. One thing was
deliberately *not* scrubbed: the underlying **service name** for the home-security capability still appears
elsewhere in the bundle (e.g. in the platform architecture ADR's service table) exactly as every other named
service does — only the specific design documents and their identifiers were held back, not the fact that the
service exists.

**Pass B — generalize every vendor/hardware identifier, bundle-wide:**

| Category | What was generalized | Generalized to |
|---|---|---|
| Home-server brand/model | the server's brand name + model number, used ~50 times across 17 files | "the home server" |
| Home-server CPU family/microarchitecture codename | the CPU brand + model line + microarchitecture codename | dropped, or "the CPU" — numeric specs (core/thread counts, RAM size) were kept, the brand/codename was not |
| Home-server integrated-GPU codename | the GPU brand + model + silicon codename | "the iGPU" / "an integrated GPU" |
| Home-server on-chip NPU | the brand-specific NPU name | "an on-chip NPU" |
| Voice-satellite device | the specific dev-board model number, used across 11 files | "the voice box" |
| Camera-detection accelerator (current recommendation) | the specific M.2 accelerator model | "a dedicated M.2 detection accelerator" |
| Camera-detection accelerator (discontinued predecessor) | the specific discontinued accelerator's brand | "the previous-generation detection accelerator" |
| Alarm-panel integration hardware (two named board/panel brands, in a still-published availability ADR's generic Layer-0 hardware options — not the removed security-design set) | two specific board/panel product-class names | "a self-contained board" / "a commodity alarm panel" |
| Self-hosted Tailscale-alternative control server | its specific project name | "a self-hosted-control-server fallback" |
| Consumer-webmail integration | the specific webmail brand named throughout Sangam/Artha/Abhyasa integration descriptions (~27 occurrences across 9 files) | "email" |

**Kept, per the ruling's explicit allowance:** generic technology/framework words — llama.cpp, Vulkan, ROCm,
React, .NET, GraphQL, Plaid, Home Assistant, Frigate, OpenVINO, ESPHome — and Tailscale as a product name (its
hostnames and tailnet IPs remain redacted from the first revision's pass). Other named platform/vendor
integrations not on the ruling's forbidden-terms list (Twilio, Pushover, a school-portal integration name, a
push-notification-service brand that was already generalized in the first revision) were left alone.

A handful of double-article artifacts produced by the mechanical brand→generic-noun substitution (duplicated
"the … the …" phrasing) were caught by a follow-up grep and fixed by hand. A few sentences that listed multiple
hardware specs together in one run-on clause (the feasibility study's and the resource budget's hardware
summary lines) were rewritten by hand rather than mechanically substituted, since a blind find/replace there
would have left a run of dangling adjectives with no noun.

## 3. Everything from the first revision still applies

The first revision's file-by-file include/exclude table (root policies, all of `vision/`, `adr/`, `architecture/`,
and `design/`), its redaction-category counts (child's/spouse's name, network/infra identifiers, exploitable
debt-tracker IDs, the manual secret/token/password review), and its "files to double-check" list are unchanged
in substance, except: the six documents removed in Pass A are no longer part of the bundle at all (their earlier
include reasoning is superseded by the security-design holdback, not revised by it), and the first revision's
"double-check the home-security ADR" item is now moot — that document no longer ships in this bundle.

## 4. Manual secret/token/password review (re-run after this revision's edits)

Re-ran a case-insensitive search for `secret`, `token=`, and `password` over the revised bundle. As in the first
revision, every hit is an architectural description (a key-derivation function name, envelope encryption,
DEK/KEK wrapping, WebAuthn, vault design, a dev-only stand-in class name) — no hit is a literal secret value,
key, connection string, or token. No new hits were introduced by this revision's edits.

## 5. Final verification

Every term on the course-correction ruling's required list was checked individually, case-insensitively, over
the finished 43-file bundle, run for real from this repository's root, against the real un-defanged patterns
(not reproduced literally in this report — see the note at the top):

- the child's first name — zero matches
- the spouse's first name — zero matches
- the tailnet hostname suffix — zero matches
- the shared-network IP-range prefix — zero matches
- the town name — zero matches
- a bare GUID/UUID pattern — zero matches
- a system-config-style filesystem path — zero matches
- a source-repository home-directory-style filesystem path — zero matches (two genuine false positives were
  found and fixed: running prose that used a slash-separated word list as plain English, not as a filesystem
  path, still matched the literal substring incidentally — reworded to a comma-separated list so the substring
  no longer appears)
- a consumer-webmail brand name — zero matches (Pass B; see the table above)
- a push-notification-service brand name — zero matches
- the alarm-panel brand name and its end-of-life alternative — zero matches (confined to the removed security-design
  set; two remaining mentions of two board/panel brand names, in a still-published availability ADR's generic
  hardware-options discussion, were also generalized)
- the home-server brand and model number — zero matches (Pass B)
- the voice-satellite dev-board model number — zero matches (Pass B; one tool-path citation and one grammar
  artifact left behind by the mechanical substitution were both caught and fixed)
- the CPU brand/family name — zero matches (Pass B)
- the removed home-security ADR's number, standing alone — zero matches
- the removed home-security ADR's hyphenated filename fragment — zero matches
- the removed threat-assessment document's hyphenated filename fragment — zero matches
- the removed deployment-layout document's hyphenated filename fragment — zero matches
- the removed flow-diagrams document's hyphenated filename fragment — zero matches
- the removed trust-design document's hyphenated filename fragment — zero matches

No exceptions: this report's own text was included in every one of the checks above, and passes all of them —
which is why the removed documents are described by role ("the removed home-alarm ADR," "the removed
threat-assessment document") throughout this report rather than by their old numbers or filenames.

## 6. STATUS

**STATUS: COMPLETE**

- **43 files** published under this repository: 5 at the root (`_config.yml`, `index.md`, `privacy-policy.md`,
  `security-policy.md`, this report), 20 under `adr/` (19 ADRs + README — one fewer ADR than the first revision),
  7 under `architecture/` (6 docs + index — four fewer than the first revision), 9 under `vision/` (8 docs +
  index — one fewer than the first revision), 2 under `design/bonnie/`.
- The private source repository was not modified — verified read-only throughout, both revisions.
- No git repository was initialized in the output directory, per instruction.

## 7. Files/decisions the owner should double-check

1. **The first revision's "double-check the home-security ADR" item is now moot** — that document is no longer
   in the bundle.
2. **The Pass-B wording choices** — "the home server," "the voice box," "a dedicated M.2 detection accelerator,"
   "email" (for the webmail integration) — are used consistently throughout, but they are the orchestrator's word
   choices, not the owner's. If a different generic term is preferred, a find/replace across the bundle is cheap;
   nothing else depends on the exact wording chosen.
3. **`security-policy.md` and `privacy-policy.md`** carry over unchanged from the first revision (versions 1.2
   and 1.1 respectively, dated 2026-09-09) — the two "double-check" items from the first revision about the
   Sandbox→Production wording and the generalized TLS/secrets-path language in the security policy still stand
   and were not revisited in this pass.
4. **`_config.yml` theme choice** (a natively GitHub-Pages-supported theme) is unchanged from the first revision.
