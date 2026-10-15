# Manas — Resource & Hardware Budget

- **Status:** Living · **Date:** 2026-07-11 · **Rev:** 2026-07-16 (model-set re-research) · reads with
  [ADR 0001](../adr/0001-manas-platform-architecture.md) + [thesis](0000-manas-thesis.md) + the risk register.

> **2026-07-16 revision.** A web-research pass re-checked every model against the mid-2026 field for best-in-class
> fit on this box (memos in the session scratchpad). The **proposed** set below revises the warm model line. The
> voice/retrieval/responder swaps are low-risk sizing changes; the **child-safety additions (8B general guard +
> a second 8B Aegis-lineage child vote + Prompt Guard) are PENDING OWNER ADOPTION** — recorded here to prove they
> *fit*, not that they are decided. All footprints remain engineering estimates until measured on the box.

> **2026-07-27 correction (measured on the box, `tools/gpu-spike/README.md`).** The "~22 GB" figure quoted
> below as the responder's **KV-cache** (see the memory-budget subtotal note at the "Verdict" line and the
> Conclusion caveat) is a **mislabelling** — it originates in the memory-budget table row below as the
> responder's **model-weights** RAM, not a KV-cache sizing. **No KV-cache budget was ever separately recorded
> in this file.** Measured reality: at the recorded 16k (`16384`) context target, the responder's actual KV
> cache is **~0.25 GiB** — roughly 90× smaller than the mislabelled figure — with total GPU-resident (VRAM +
> GTT) of **21.35 GiB** (7.94 GiB VRAM + 13.41 GiB GTT). Even at the model's full native 262,144-token context,
> total GPU-resident is only **26.51 GiB**. The two mislabelled lines below are corrected in place; the
> original ~22 GB weights figure in the table row is unchanged and was always correct in its own context.

> **2026-07-27 flag (unresolved — owner-confirmable, NOT resolved here).** This file states above and in the
> table below that the child-safety additions (8B general guard + a second 8B child-vote guard + Prompt Guard)
> are **PENDING OWNER ADOPTION**. [ADR 0001 §16.2](../adr/0001-manas-platform-architecture.md) records the
> two-guard vote (`Llama-Guard-3-8B` general guard + `Nemotron-Safety-Guard-8B-v3` child vote) as **"ADOPTED
> (owner, 2026-07-16)"**, and ADR-0001 §16 is stated elsewhere to be authoritative for these pins. **These two
> documents disagree.** This note flags the discrepancy for owner confirmation; it does **not** unilaterally
> change the "pending" language in this file, which should be read as *possibly stale* pending that
> confirmation. Separately: **Prompt Guard 2 appears ONLY as a budget line item in this file** (table row
> below) — it is never listed in ADR-0001 §16's adoption tables (§16.1 or §16.2), so its adoption status is
> genuinely unrecorded, not merely disputed. **Measured context (2026-07-27 co-residency spike):** all four
> safety/embedding models were obtained as real GGUFs and run co-resident on the box — `Llama-Guard-3-8B`
> 4.58 GiB, `shieldgemma-2b` 1.59 GiB, `Nemotron-Safety-Guard-8B-v3` 4.58 GiB, `Qwen3-Embedding-4B` 2.33 GiB
> (all Q4_K_M) — with the full stack plus the responder totalling **36.63 GiB** and leaving **30.08 GiB of GTT
> headroom**. This is evidence the set *fits* (the file's original intent); it is NOT evidence of adoption
> status, which remains the owner's call.

> **2026-08-03 note (this budget is Manas's, not the box's).** Every line item below is a **Manas** component,
> and the headroom figures are what is left after Manas — **not what is free on the box.** Since 2026-08-03 two
> sibling projects, **Mūla** and **Nakṣatra** (`~/dev/mula`), run on this same the home server and will share one
> `llama-server` with Manas; **neither appears in any row here, and nothing has been measured with them
> running (NOT-CHECKED).** No replacement number is recorded — this note names the gap rather than inventing a
> figure for it. The binding rule is CLAUDE.md **#14, "Manas always takes precedence"**: box contention resolves
> in Manas's favour, corpus harvesting and HTR work never run while Manas is serving or being measured, and the
> architectural consequence for shared inference is tracked in the living build-status document §5.

> The anti-"we need different hardware" artifact. It proves — with numbers — that the whole platform fits the
> **the home server**, and it names the ONE genuine hardware-adjacent contention point (shared GPU compute) with a
> measured spike and a fallback, **before** any code. Numbers are engineering estimates to be replaced with
> **measured** values as components are stood up (see the "measure" column).

## The box

**the home server** — a 16-core/32-thread unified-memory mini-PC: **128 GB LPDDR5X unified (~124 GB
usable; on Linux most is GPU-addressable via GTT)**, an integrated GPU (~RTX-4070-class),
~256 GB/s memory bandwidth, OCuLink (eGPU-capable), 2.5GbE, NVMe. Runs Linux (prod OS).

## Memory budget (steady-state, prod — no eval judge)

| Component | Est. RAM | Notes | Measure |
|---|---:|---|---|
| Linux + page-cache headroom | ~6 GB | OS + fs cache | `free` |
| **llama.cpp / llama-server (Vulkan) — warm model set (PROPOSED, 2026-07-16)** | **~40 GB** | responder `Qwen3.6-35B-A3B` **~22** (3 B active; was `qwen3:30b-a3b` ~21) · embedder `Qwen3-Embedding-4B` ~2.5 (was `nomic-embed-text` ~0.5) · general guard `Llama-Guard-3-8B` ~5.5 (was `llama-guard3:1b` ~1.8) · child vote `shieldgemma-2b` ~2 **+ `Nemotron-Safety-Guard-8B` ~5.5 (pending adoption)** · `Prompt Guard 2` ~0.15 · small task models (e.g. mantra pitch) ~1 | `llama-bench` + monitor KV-cache residency |
| PostgreSQL (all services' DBs) | ~2–4 GB | shared_buffers + connections; DB-per-service | pg stats |
| Neo4j | ~3–4 GB | JVM heap + page cache | neo4j metrics |
| Qdrant | ~1–2 GB | grows with vectors | qdrant metrics |
| Redis | ~0.5–1 GB | cache + pub/sub + queues | `INFO memory` |
| MinIO | ~0.5 GB | object metadata (blobs on disk) | — |
| **Frigate** (cameras + VA-API decode) | **~4–8 GB** | VA-API fixed-function decode (negligible GPU contention); detection on a dedicated accelerator / OpenVINO-CPU (off-iGPU); 4 GB realistic minimum, 8 with enrichments | frigate stats |
| **Zitadel** (OIDC IdP) | **~0.5–1 GB** | Go binary + its Postgres share (light; Keycloak-JVM would be the heavy alternative) | zitadel metrics |
| **step-ca / PKI** | **~0.05 GB** | lightweight CA for mTLS (D7) | — |
| **SOPS secrets** | **~0** (no daemon) · OpenBao **~1 GB if later adopted** | static file-based secrets to start; OpenBao only when dynamic secrets needed | — |
| **Whisper** (iGPU/NPU residency) | **+1–2 GB** | STT model residency during voice turns (can share iGPU time-slice with LLM within turn) | monitor per-turn utilization |
| Home Assistant + add-ons | ~1–2 GB | if hosted on-box | — |
| Whisper STT + Kokoro TTS | ~2 GB | `whisper large-v3-turbo` ~1.5 + `Kokoro-82M` ~0.3 (was Piper; no cloning surface) | — |
| .NET services (~10: 5 domain + Sarathi (the Agent) + foundation) | ~3–5 GB | ~0.3–0.5 GB each | dotnet-counters |
| **Subtotal** | **~63–78 GB** | (proposed set; was ~52–66 GB — the model re-research adds ~+11 GB, mostly the 8B×2 safety stage) | |
| **Headroom vs ~124 GB usable** | **~46–61 GB free** | still comfortable | |

**Verdict: RAM is not the constraint** — the model re-research does not change this. Even with the fatter safety
stage, steady-state is ~half to two-thirds of the box; **~46–61 GB spare** covers spikes, page cache, a bigger
responder (a Q4 70B ~40 GB *fits* if ever wanted), a co-resident VLM, or the **Qwen3-14B / 70B eval judge** during
eval runs (eval is not steady-state; it can co-reside here where the M4 could not). The aggregate is spread across
many *separate* model allocations, so the llama.cpp >64 GB *single-allocation* cliff (issue #14854) is not tripped
by the total — but the responder's ~22 GB **model-weights** footprint (mislabelled here previously as
KV-cache; corrected 2026-07-27 — see note above) must still be GTT-bounded (see conclusion caveat).

## GPU / compute — detection moved off-iGPU

The single **integrated GPU** now handles **LLM inference + VA-API video decode only** (decision D2). **Frigate camera detection runs on a dedicated M.2 accelerator (~$70–100, ~7 ms) or OpenVINO-CPU** (off-iGPU baseline; iGPU ONNX detection works but is not the primary path). This dissolves the earlier "one real contention point" — LLM and detection no longer share silicon.

- **Measured throughput (feasibility-study §1, community verified):** LLM generation ~85–105 tok/s on `qwen3:30b-a3b` (IQ4_XS, llama.cpp/Vulkan); VA-API decode is fixed-function (VCN) with negligible compute load.
- **Must measure (spike):** LLM tokens/sec cold and warm; Frigate FPS on the dedicated-accelerator baseline; confirm VA-API decode contention is truly negligible **under sustained voice turns + camera activity** (risk #2). **Targets: <2 s no-tool / <3 s one-tool first-audio; prompt-prefix caching load-bearing (risk #29)** (feasibility-study §4); Frigate keeps real-time FPS on all cameras.
- **NEW spike (2026-07-16, 0.5 track) — safety-stage per-turn latency.** The proposed input gate runs `Llama-Guard-3-8B` + `shieldgemma-2b` + (pending) `Nemotron-Safety-Guard-8B` + `Prompt Guard 2` per turn, vs. today's single `llama-guard3:1b`. Each classifier emits few tokens (prompt-processing ~1300 t/s dominates, so ~tens of ms each), but **two 8B classifiers is measurably more than one 1B and lands directly on the <2 s/<3 s voice budget** — measure the composite gate latency (parallel vs. sequential classifier calls, warm) before adopting the child-safety swap. This is a latency gate, NOT a capacity gate (capacity fits, above).
- **Fallbacks if contended (unlikely; recorded only):** (1) **OCuLink eGPU** — if the dedicated accelerator proves unavailable or undersized; (2) **NPU detection** — an experimental on-chip-NPU path (~16 ms, hand-patched Vitis AI), spike only; (3) cap Frigate to motion-gated detection + fewer FPS. **No architecture change** — Frigate's detector is already a swappable seam.
- **Bandwidth:** 256 GB/s unified-memory bandwidth is sufficient for LLM + decode. The **MoE `qwen3:30b-a3b` (3 B active)** is chosen precisely to fit this; a dense 70 B would *fit* but be slow (documented, not used for the live responder). Within a single voice turn, LLM and STT (Whisper iGPU-Vulkan or NPU) are time-multiplexed — contention-free unless both are peak-simultaneous (negligible risk given STT finalization overlaps with LLM round 2 prefill).

## Disk

4 TB NVMe (expandable 16 TB). Consumers: Postgres/Neo4j/Qdrant data, MinIO blobs (statement PDFs, Frigate
**clips** — the big one), model files (~30–50 GB), Saakshi logs (encrypted, bounded retention), backups.
Frigate clip retention is the dominant variable → **tiered retention + prune** policy required.

## Network

2.5GbE LAN. Cameras (RTSP) are the main internal load; the private relay (Tailscale) is thin. Fits easily.

## Conclusion

- **RAM:** fits with **~46–61 GB to spare** even with the 2026-07-16 proposed set (fatter 8B×2 safety stage) — *not* a constraint. **Caveat: the unified-memory LLM's ~22 GB model-weights residency (mislabelled here previously as KV-cache; corrected 2026-07-27 — see note above) is GPU-addressable via GTT; set GTT/ttm limits in kernel config so residency spikes cannot squeeze the OS page allocator.**
- **Disk:** fits; Frigate clip retention is the variable to manage.
- **GPU:** detection now off-iGPU (dedicated-accelerator / OpenVINO baseline) — the earlier "one real contention point" is largely dissolved. LLM + VA-API decode contention is expected negligible (fixed-function VCN blocks) but is **not yet measured on this box — confirm at the risk-#2 spike**; time-multiplexed LLM/Whisper within a voice turn is low-risk.

**This is the artifact that says: build on the home server with confidence; measure the spikes (voice-latency with cache warm/cold, Frigate dedicated-accelerator baseline, wake-word) before committing those specific designs; everything else has headroom.**

---

*(A later amendment about inference-server sharing with sibling projects on the same box is omitted from this public copy.)*
