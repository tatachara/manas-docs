# Manas — Tech Stack Review (verified 2026-07-11)

- **Status:** Living (re-verify at each phase start) · **Date:** 2026-07-11
- **Method:** every pin in `Directory.Packages.props` / `global.json` / ADR 0001 §6 was web-verified against
  mid-2026 reality by a research agent; the 12 decision-driving claims were then **independently re-verified by a
  second fact-checking agent with fresh sources** (≥2 sources each). Verification labels: ✅ = double-verified ·
  ✔ = single-pass web-verified · ❓ = unverified (stated honestly).
- **Reads with:** an internal design review (round 2) §ST ·
  [`feasibility-study.md`](feasibility-study.md) · the risk register.

> Bottom line: the stack is sound in shape but **several pins were stale or pointed at dying projects**, and two
> round-1 assumptions (an earlier detection-accelerator fallback, Ollama runner) are wrong on current evidence. Everything below is a
> pre-code decision — nothing requires an architecture change; every risky dependency already sits behind a port.

---

## 1. Decisions this review forces (pre-code)

| # | Decision | Was | Now (verified) | Rationale |
|---|---|---|---|---|
| D1 | ✅ **LLM runner: llama.cpp / llama-server on Vulkan (RADV)** behind Medha's `IChatClient`; Ollama demoted to dev-convenience alternative | Ollama | llama.cpp direct | Ollama's vendored llama.cpp lags upstream Vulkan (~56% t/s gap reported, [ollama#15601]); ROCm path needs `HSA_OVERRIDE` + SDMA workarounds ([ollama#14855]); standalone Vulkan hits ~85–105 tok/s gen on qwen3:30b-a3b on this exact silicon. Same port, one-line swap. |
| D2 | ✅ **Frigate detector fallback: a dedicated M.2 accelerator (~$70–100, ~7 ms) or OpenVINO-CPU** — the previous-generation accelerator is REMOVED from the design | previous accelerator | dedicated accelerator / OpenVINO | Previous accelerator's driver archived 2026-04, unbuildable on kernels ≥6.4 unpatched; Frigate no longer recommends it. iGPU reserved for LLM + VA-API decode (decode works as of Frigate 0.17). |
| D3 | ✅ **GraphQL: HotChocolate 16, ONE composed schema (modular monolith), no Fusion** for the single-box deployment | HC 15.0.0 + Fusion 15 | HC 16.4.x, no gateway | HC 16 is the current major (core rewrite); Fusion v16 is a complete re-implementation with a staged migration from v1 — adopting Fusion now buys churn for zero network-boundary benefit on one box. Service isolation is preserved by module boundaries + the Kernel wall, not a gateway. Revisit only if services de-colocate. |
| D4 | ✅ **Tests: xunit.v3 + Microsoft.Testing.Platform from commit #1** | xunit 2.9.2 | xunit.v3 (3.2+/4.x line) | v2 is maintenance-mode per the maintainers; a 10× test estate on the dead line is a self-inflicted migration. |
| D5 | ✅ **KDF: NSec.Cryptography (libsodium) Argon2id** — amends ADR 0002 §7 | Konscious 1.3.1 | NSec 26.4.0 | Konscious last released 2024-06 (2 years stale) on a KEK-load-bearing path; NSec actively maintained, audited native core. `IKeyDerivation` port unchanged; PBKDF2-BCL fallback stands. |
| D6 | ✅ **Blob store: decision REOPENED** — MinIO community server is in maintenance mode (Dec 2025) and the repo was archived 2026-04; no official community binaries | MinIO server | evaluate **Garage / SeaweedFS / pinned MinIO build** | Code strictly to the S3 API behind the existing repository port; the choice becomes a small ADR in the storage phase. The .NET `Minio` SDK (7.x) still works against any S3 endpoint. |
| D7 | ✔ **PKI: step-ca-class lightweight CA instead of SPIRE** (Pramana ADR to ratify) | SPIFFE/SPIRE | step-ca (or minimal built-in CA) | SPIRE is fleet-scale machinery with platform-team ops cost and zero homelab adoption evidence; one host needs short-lived mTLS certs from one small CA. `ServiceId`≡cert-identity is preserved. |
| D8 | ✔ **Secrets: SOPS+age to start; OpenBao only when dynamic secrets demand it** (Pramana ADR to ratify) | OpenBao/Vault | SOPS+age → OpenBao later | OpenBao (2.5.0, MPL-2.0) is healthy but requires unseal-on-every-reboot wiring; at household scale SOPS is the honest fit. Auto-unseal design (TPM) recorded for the upgrade path. |
| D9 | ✔ **One shared Postgres instance, logical DB per service** (ratifies what ADR 0001 already implied) | ambiguous | one instance, N databases | Ops load: Postgres major upgrades are the recurring pain; N instances multiplies it for no isolation gain (the wall is Kernel + roles + separate DBs). |
| D10 | ✔ **Embedder: decide nomic-embed-text v1.5 vs v2-moe before the first vector write** | nomic-embed-text | choose at Phase 0 | Changing embedders invalidates every Qdrant vector (re-embed pipeline); v2-moe adds multilingual (relevant: Indic content). |

## 2. Pin corrections (mechanical, at scaffold time)

| Package | Pinned | Current (2026-07) | Action |
|---|---|---|---|
| .NET SDK | 10.0.301 | 10.0.8xx band (runtime 10.0.8) ✔ | `rollForward: latestFeature` already absorbs; install current at scaffold. LTS→Nov 2028. |
| EF Core / Sqlite | 10.0.0 | 10.0.9 ✔ | bump |
| Npgsql.EFCore.PostgreSQL | 10.0.0 | 10.0.3 ✔ | bump |
| Neo4j.Driver | 5.26.0 | **6.2.1** ✅ (needs .NET 8+; GQL error model) | pin 6.x before code |
| Qdrant.Client | 1.12.0 | 1.18.1 ✔ | bump; match server version |
| StackExchange.Redis | 2.8.0 | **3.0.17** ✅ (new major) | pin 3.x; review breaking changes (❓ list not yet enumerated — do at scaffold) |
| Minio SDK | 6.0.3 | 7.0.0 ✔ | bump (works against any S3 endpoint; see D6) |
| HotChocolate.* | 15.0.0 | **16.4.0** ✅ | per D3 |
| Microsoft.Extensions.AI | 10.7.0 | 10.7.0 ✔ current | keep |
| OllamaSharp | 5.4.25 | 5.4.25 ✔ current | keep only if Ollama kept as dev alternative (D1) |
| Serilog.AspNetCore | 9.0.0 | **10.0.0** ✅ | bump |
| OpenTelemetry.Ext.Hosting | 1.10.0 | 1.16.0 ✔ | bump |
| xunit / Test.Sdk / coverlet | 2.9.2 / 17.11.1 / 6.0.2 | v3 line / 18.5.1 / **10.0.1** ✅ | per D4; coverlet 10 has MTP support — verify Cobertura output in CI gate |
| Konscious Argon2 | 1.3.1 | (stale) ✅ | remove per D5; add NSec.Cryptography 26.4.x |
| — add | | **ArchUnitNET** ✔ (NetArchTest unmaintained since 2023) | for the P11 architecture tests |
| — add | | **CsCheck 4.7** ✔ (or FsCheck 3.3) | property tests; CsCheck is the C#-idiomatic pick |
| — add | | **Quartz 3.18** ✔ | scheduler (MIT, storage-agnostic; over Hangfire) |

## 2a. Model pins — AMENDED 2026-07-17 (owner-approved 2026-07-16; ADR 0001 §16 is the authority)

Owner approved the low-risk swaps and adopted the child-safety restructure 2026-07-16 (evening); every pin
below was **web spot-checked 2026-07-17** (links + evidence in ADR 0001 §16). Old pins are struck, not
deleted — **each old pin is the recorded fallback**, and the old set remains the **live dev-Mac seam** until
the box-gated wiring (0.5 track) lands.

| Role | Pin | Verdict (2026-07-17) | Notes |
|---|---|---|---|
| Responder | ~~`qwen3:30b-a3b`~~ → **Qwen3.6-35B-A3B** | ✔ verified-exists (rel. 2026-04-15, Apache-2.0) | MoE 35B/3B-active, 262k native ctx; GGUF via llama.cpp confirmed. Ollama Q4_K_M = **24 GB** (budget said ~22 — use an IQ4-class quant or absorb in headroom; measure on box). |
| Embedder | ~~`nomic-embed-text`~~ → **Qwen3-Embedding-4B** | ✔ verified-exists (Apache-2.0, first-party GGUF) | Q6_K 3.31 GB / Q8_0 4.28 GB; 2560-dim Matryoshka. D10 stands: must land before the first production vector write (none exist yet). |
| Eval judge | ~~`qwen2.5:7b` (dev seam)~~ → **Qwen3-14B** (thinking) | ✔ verified-exists (Apache-2.0, first-party GGUF) | Eval-time only; judge seam swaps by config (`EVAL_JUDGE_URL`). `qwen2.5:7b` stays the dev-Mac judge until the box. |
| TTS | ~~Piper (`piper1-gpl` v1.4.2)~~ → **Kokoro-82M** | ✔ verified-exists (Apache-2.0 weights) | 82M, 54 fixed voices, 24 kHz, no cloning surface. **Runtime: ONNX/PyTorch separate process behind the TTS port — NOT llama.cpp.** |
| Safety — general guard | ~~`llama-guard3:1b`~~ → **Llama-Guard-3-8B** | ✔ verified-exists (Llama 3.1 Community License) | GGUF Q4_K_M 4.92 GB / Q5_K_M 5.73 GB. License re-review owed at productisation. |
| Safety — child vote | `shieldgemma-2b` (kept) **+ `nvidia/Llama-3.1-Nemotron-Safety-Guard-8B-v3`** (NEW) | ✔ verified-exists, **name precision recorded** — the approval shorthand "Nemotron-Safety-Guard-8B" resolves to exactly this HF id (rel. 2025-10-28) | S7 "Sexual (minor)" category; NVIDIA Open Model License + Llama 3.1 CLA; **community GGUF only** — pin quant + checksum at Medha phase. OR-to-block vote, deterministic. |
| Safety — grooming | **Thorn = SPIKE, not a pin** | ⚠ **commercial-only** (Safer Predict / Hive CSE API; Thorn-hosted cloud or enterprise self-hosted; no open weights, no household tier advertised) | A cloud grooming API would ship a child's text off-box — conflicts with ADR 0001 driver #1. Spike gates on self-hosted attainability; recorded fallback = Nemotron S7 + the child-recall eval's grooming corpus (ADR 0001 §16.3–16.4). |
| STT / VAD / wake | `whisper.cpp` · Silero · openWakeWord built-in | unchanged | Silero tech-stack pin still owed before V3; wake spike #1 RED → built-in is the plan of record. |

**Gates before wiring (0.5 track):** 8B×2 composite-gate latency vs the <2 s/<3 s voice budget (parallel vs
sequential, warm) · Thorn availability/licensing spike · child-recall eval equal-or-better bar (ADR 0001 §16.4).

## 3. Verified-healthy (no action)

- **Zitadel** ✅ — RFC 8693 token exchange incl. `act`/delegation shipped in the self-hosted product (the specific mechanism ADR 0003 depends on). Current release number ❓ — confirm at the Dwarpal phase. Keycloak 26.7 is the heavier fallback.
- **OpenZFS 2.4** ✔ — DR design assumption sound; pin kernel ≤6.18 compatibility in the runbook.
- **Tailscale** ✔ — free Personal plan covers the household; a self-hosted-control-server fallback is recorded for sovereignty.
- **HA voice stack** ✅ — Wyoming integration current; **streaming TTS shipped (HA 2025.7/2025.8, ~10× first-audio improvement)**; ESPHome continue-conversation since 2025.5; the voice box still supported (HA Voice PE noted as better-eared alternative for far-field rooms).
- **Piper TTS** ✅ — rhasspy/piper archived 2025-10; active successor **OHF-Voice/piper1-gpl v1.4.2** (GPL-3.0, separate process over Wyoming so license is contained). Thin-maintainer risk: keep behind the TTS port; verify the Bonnie voice choice against persona.md's pitch rules during that phase. *(AMENDED 2026-07-17: TTS pin is now **Kokoro-82M** — §2a; Piper/piper1-gpl is the recorded fallback.)*
- **whisper.cpp** ✔ — runs on this box (Vulkan image available); faster-whisper has no ROCm backend — don't design around it.
- **qwen3:30b-a3b** ✅ — 32k native context (YaRN to 131k; the -2507 refresh is 262k native — **pin which snapshot** at Medha phase); consistently rated a reliable local tool-caller. llama-guard3:1b + shieldgemma:2b available; both 2024-era — the eval harness owns the safety bar, not model reputation. *(AMENDED 2026-07-17: responder pin is now **Qwen3.6-35B-A3B** and the guard tier restructured — §2a / ADR 0001 §16; this set is the recorded fallback + the live dev-Mac seam until the box gates pass.)*

## 3a. Web-stack spike findings (unit 0.16 — first console, 2026-07-12)

The GraphQL/graphql-ws/Blazor-WASM path was exercised live (real Kestrel; frames observed) — it WORKS, with two
footguns recorded so no one re-learns them:

1. **HC16 subscriptions must use `[Subscribe(With=...)]` + `[EventMessage]`** — a plain resolver returning
   `IAsyncEnumerable<T>` becomes a `[T!]!` LIST field, not a stream. (Cost an hour to diagnose.)
2. **R12 subscription authz must throw SYNCHRONOUSLY** (in the subscribe body, before returning the iterator).
   A `GraphQLException` thrown *inside* the async-iterator is masked by HC16 to "Unexpected Execution Error" with
   an empty code — the typed `TURN_FORBIDDEN`/`TURN_NOT_FOUND` is lost. Load-bearing for the turnId-not-a-capability rule.
3. **graphql-ws worked cleanly; SSE fallback was NOT needed.** StrawberryShake deferred (unpinned, heavy codegen) —
   the skeleton console uses a hand-rolled `ClientWebSocket` graphql-ws client (token in `connection_init` payload,
   since WASM can't set WS upgrade headers). Revisit StrawberryShake at the Akriti (premium UI) phase.
4. **UI FRAMEWORK — RESOLVED 2026-07-12 (ADR 0011): Akriti = React 19 + TS (Vite SPA), NOT Blazor.** The 4×-blah
   root cause was an ecosystem mismatch (premium animation/charting lives in React, thin in Blazor). Stack:
   shadcn/ui + Tailwind v4 · Motion v12 · Recharts/Tremor + ECharts · **urql + graphql-ws** (StrawberryShake
   dropped) · Rive persona avatar. The Blazor `Manas.Console` skeleton is retired; served as static assets by
   Kestrel (one origin); the two HC16 footguns above are backend/contract and survive the swap. `getUserMedia`/
   voice (path 2) is Phase 2. Blazor WASM pins become React/Node toolchain pins at the Akriti-foundation unit.

## 4. Standing stack risks (watch list)

1. **ChilliCream/HotChocolate churn** — single-vendor, major rewrites; budget upgrade windows even on the no-Fusion path (D3).
2. **ROCm on this iGPU class is "Preview"** — Vulkan-first design absorbs it; re-check each ROCm release (risk #3).
3. **Piper successor maintenance** — port-isolated; Kokoro-class alternatives exist. *(2026-07-17: Kokoro-82M is now the pin — this risk inverts to "Kokoro maintenance", same port isolation contains it; Piper stays the fallback.)*
4. **Blob-store successor choice** (D6) — small ADR owed before the storage phase.
5. **HA monthly breaking changes** — the #1 recurring ops burden; quarterly batched upgrades + HA-as-appliance posture (risk #25) contain it.
6. **StackExchange.Redis 3.x breaking-change review** still owed (❓) — do at scaffold, before the event-bus unit.

---

*Provenance: Pass-1 stack agent (46 web lookups) + Pass-2 independent fact-checker (fresh searches, ≥2 sources on decision-driving claims). Items marked ❓ are honestly unverified; nothing ❓ is load-bearing for a D1–D10 decision.*
