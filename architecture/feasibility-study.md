# Manas — Feasibility Study (quantified, 2026-07-11)

- **Status:** Living — every number here is **community/benchmark evidence, not yet measured on the actual box**;
  each verdict names its on-box gate. Re-stamp after box bring-up. · **Date:** 2026-07-11
- **Method:** research agent with web evidence current to 2026-07; the decision-driving claims independently
  re-verified by a second fact-checker (fresh sources). Cross-checked against
  [`../vision/resource-budget.md`](../vision/resource-budget.md) and the risk register.
- **Box:** the home server — 16-core/32-thread CPU, 128 GB LPDDR5X unified memory (~256 GB/s),
  an integrated GPU, an on-chip NPU, Linux.

> Bottom line, honestly: **nothing says "different hardware."** Throughput on this exact chip is community-proven.
> Three things the earlier docs got wrong or left open are now decided: the LLM runner (llama.cpp/Vulkan, not
> Ollama), the Frigate fallback (a dedicated detection accelerator; the previous one is dead), and the voice-latency target (**<2 s is only honest for
> no-tool turns; one-tool turns budget ~2.8 s and structurally require prompt caching + streaming**).

## Verdict summary

| # | Question | Verdict | The number that matters | On-box gate |
|---|---|---|---|---|
| 1 | qwen3:30b-a3b inference | **GREEN** (llama.cpp/Vulkan) | ~85–105 tok/s gen; ~1,100–1,400 tok/s prefill; warm TTFT ~0.1–0.4 s | llama-bench + verify prefix-cache hits (risk #3) |
| 2 | Whisper STT | **YELLOW** | 0.78–0.93 s measured @7.6 s clip *on this silicon* (iGPU-Vulkan / NPU) → ~0.5–0.8 s for 3–6 s utterances (extrapolated) | 30-min spike: turbo on Vulkan vs NPU vs CPU, real clips |
| 3 | Piper TTS | **GREEN** | ~100–250 ms to first audio, CPU; sentence-streaming is upstream default | confirm piper1-gpl + persona voice rules |
| 4 | End-to-end voice turn | **YELLOW** | **≈2.2–3.5 s warm, one tool** (mid ~2.8 s); no-tool ≈1.6–2.5 s; +~1 s per extra tool round; cold cache +1.4–3.6 s | full-pipeline latency spike; targets restated below |
| 5 | Frigate + GPU contention | **RED** for iGPU detection · **GREEN** for VA-API decode | a dedicated detection accelerator does ≈7 ms off-GPU detection; one iGPU-vs-iGPU contention datapoint: +69% (unconfirmed) | risk #2 rewritten: dedicated-accelerator/OpenVINO baseline; iGPU = LLM + decode only |
| 6 | Memory budget | **GREEN** | revised steady state **~52–66 GB** → ~58–72 GB headroom of ~124 GB | update budget table (below) |
| 7 | Browser voice path | **GREEN** (transport) · YELLOW (browser AEC) | 32 kB/s raw PCM — trivial | AEC device-matrix test (Chrome/Safari/iOS) |
| 8 | Solo-ops load | **YELLOW → GREEN with simplifications** | ~25 containers; 3 named recurring burdens | decisions D7/D8/D9 in [`tech-stack.md`](tech-stack.md) |

## 1. LLM inference (risk #3, #6)

Measured community numbers on this chip class (llm-tracker, community backend benchmarks — double-verified):
Vulkan/RADV beats ROCm on both prefill and generation for this MoE model; current builds at 4-bit quants reach
**~1,438 tok/s prefill / ~103 tok/s generation** (IQ4_XS), older builds ~85 t/s. ROCm on this iGPU remains
officially *Preview* with open crash issues. **Ollama specifically** carries a vendored-llama.cpp lag (~56%
reported gap) and needs ROCm env workarounds → **decision D1 (tech-stack.md): Medha fronts llama.cpp/llama-server
on Vulkan behind `IChatClient`.**

**The load-bearing detail: prompt-prefix caching.** Cold prefill of a 2–4k prompt costs 1.4–3.6 s — the <2 s
budget dies without `cache_prompt` keeping the static prefix (system + persona + tool schemas) warm. Consequences
for Sarathi's design (the Agent, ADR 0010): cache-friendly prompt layout (static prefix first, append-only history),
per-persona server slots, and an honest +1.5–3.5 s on cold/evicted turns.

## 2–4. The voice-turn budget (drives ADR 0010 latency targets)

Component table (warm, one tool call): wake ~0.1 · endpointing 0.3–0.5 · transport ~0.05 · STT 0.5–0.8 ·
**input safety gate (Llama-Guard/ShieldGemma pass, warm + QoS-reserved) ~0.1–0.5 — a budgeted line item, added
round 3 (R6); overlap it with STT finalization where possible** · LLM round 1 (cached prefill + tool-call tokens)
0.75–1.1 · tool exec 0.05–0.2 · LLM round 2 (result prefill + first sentence) 0.3–0.6 · TTS first chunk
0.1–0.25 · return ~0.05 → **≈2.3–4.0 s un-overlapped; ≈2.2–3.5 s with gate/STT overlap** (measure both at the
risk-#29 spike).

Dominant terms: STT + the two LLM rounds; endpointing and the safety gate are the sneaky third and fourth. Structural requirements this imposes
(not optimizations): (a) prefix caching, (b) STT on iGPU-Vulkan or NPU (the NPU path — measured 0.78 s on this
silicon via Lemonade — is attractive because it leaves the iGPU to the LLM), (c) sentence-streamed TTS (upstream
default now), (d) a deterministic fast-path for common intents that skips LLM round 2 entirely, (e) spoken
acknowledgment ("let me check…") on multi-tool turns.

**Targets (restated, supersede "voice turn < ~2 s" in resource-budget.md):**
| Path | Metric | v1 target | Goal |
|---|---|---|---|
| Console text | time-to-first-token | ≤ 1 s | ≤ 0.5 s |
| Console voice | end-of-speech → first text/audio | ≤ 2 s | ≤ 1 s |
| Satellite | end-of-speech → first audio | **≤ 2 s no-tool · ≤ 3 s one-tool** | ~0.8 s no-tool |

## 5. Frigate & the iGPU (risk #2 rewritten)

Evidence (double-verified): Frigate removed the ROCm detector (AMD now via ONNX in the `-rocm` image); this
iGPU class is not in the auto-override table; a maintainer explicitly recommends against this CPU/GPU class for Frigate
enrichments; nearest supported iGPU hangs periodically. **But:** VA-API hardware decode works on this chip class as
of Frigate 0.17 (fixed-function VCN — negligible compute contention), one experimental on-chip **NPU**
detection success exists (~16 ms, hand-patched Vitis AI), and the maintainer reports AMD iGPU ONNX detection
"works well with YOLOv9" in quick tests. The previous-generation detection accelerator is dead (driver archived).

**Risk #2 rewrite:** baseline = **a dedicated M.2 detection accelerator (~$70–100, ~7 ms) or OpenVINO-CPU** for detection; iGPU does LLM +
VA-API decode only; the NPU detection path is an optional spike, not a plan. This *dissolves* most of the
"one real contention point" in resource-budget.md — LLM and detection no longer share silicon.

## 6. Memory (resource-budget corrections)

Revised rows: qwen3:30b-a3b loaded **~21 GB** (was 18); warm model set ~27–28 GB; Frigate 4–8 GB (was 2–6);
**add missing rows:** Zitadel 0.5–1 GB, CA/step-ca ~0.05 GB, SOPS ~0, (OpenBao ~1 GB if/when adopted), Whisper
GPU/NPU residency +1–2 GB. **Revised steady state ~52–66 GB → ~58–72 GB free.** Still half the box; eval judge
still co-resides for eval runs. One new caveat: unified memory means the LLM's ~21 GB is GPU-addressable GTT —
set GTT/ttm limits deliberately so a KV-cache spike can't squeeze the OS.

## 7. Browser voice (path 2) — transport verdict

**AudioWorklet → 16 kHz s16 mono PCM → WebSocket** (≈32 kB/s), terminating in the same central STT/turn spine as
the satellites (raw PCM is exactly the Wyoming wire format). Rejected: MediaRecorder (chunks not independently
decodable — only the first blob carries the container header), WebRTC (its benefits are moot over Tailscale; a
media server is scale machinery), SignalR for audio bytes (protocol overhead; fine for control-plane). Blazor
WASM: keep the capture loop entirely in JS; don't marshal per-frame audio through .NET interop. `getUserMedia`
needs HTTPS — `tailscale cert` on the private relay domain for remote. **Echo cancellation:** browser AEC only cancels
audio the browser itself plays — TTS playback must go through an `<audio>`/MediaSource element, and v1 ships
**half-duplex** (mic gated during TTS); browser barge-in only after an AEC device-matrix test (Chrome/Safari/iOS).
Partial/streaming STT is P2 polish (VAD-gated whole utterances first). Full design: ADR 0010 §5.2.

## 8. Solo-ops honesty

Feasible for one disciplined person **only with the simplification levers**: step-ca over SPIRE (D7), SOPS over
OpenBao to start (D8), one shared Postgres (D9), HA upgrades batched quarterly (HA's monthly breaking changes are
the #1 reported self-hoster burden), ZFS via sanoid/syncoid (genuinely low-touch). As originally specced
(SPIRE + OpenBao + N Postgres + monthly HA tracking) the design trends unsustainable for solo ops — that would
have been a "discovered later" failure; it is now a set of recorded decisions instead.

---

*Provenance: Pass-1 feasibility agent (14 sources) + Pass-2 fact-checker corroboration. Every number is
community evidence until the box bring-up spike reproduces it; the spike list lives in the risk register
(#2, #3, #6 updated this round). No file outside `docs/` was touched.*
