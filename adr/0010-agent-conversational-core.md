# ADR 0010 — Sarathi (the Agent): Conversational Core, Channels & Conversation Ownership

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** ADR 0001 §4 (Sarathi intent), §5.2, §5.6, §5.7, §7; resolves round-2 review
  findings G1–G13 (the Sarathi cluster); constrains amendments to
  ADR 0003 (assurance claims), ADR 0007 (modality/assurance split, Guest role), ADR 0009 (conversation retention),
  the persona design (streaming Governor), the home-service design (voice-edge ownership).
- **Companion spec:** the platform's agent-implementation spec — the buildable component
  spec (stage contracts, schemas, transports, tests). This ADR records the *decisions*; the spec records the *shape*.

> **Sarathi (the Agent)** is the platform's center: every family interaction flows through it, on exactly **three paths** —
> **(1) text chat in the web console** (always available), **(2) voice in the same console text box** (mic
> button), **(3) the voice box satellites**. Round 2 found the center undesigned: path 2 didn't exist, path 1
> was a folder name, and Sarathi had no conversation model, no datastore, no API, no streaming design, no loop
> contract, and a channel taxonomy that couldn't classify two of its three paths. This ADR fixes all of it with
> five decisions: **one turn spine with thin channel adapters; Sarathi is a walled service that owns
> conversation data; input modality is separated from channel assurance; streaming is sentence-buffered through a
> deterministic Governor gate; and Sarathi's loop gets an explicit, testable contract.**

---

## 1. Context

Requirements already locked by prior ADRs: Sarathi is a deputy carrying the
caller's verified identity, never a super-user (0001 §5.2, 0003 §4); T4 intents take a deterministic fast-path
*before* the model and every tool boundary re-classifies regardless (0007 §8); the input safety gate runs on the
ActorContext **role** before any command on every surface (the persona design §5); persona is resolved before the LLM and
shapes delivery only (the persona design §2/§4); all model access rides Medha with QoS (0001 §5.3); correlation ids
thread every hop (0001 §5.4); services are opt-out so the tool set is dynamic (0001 §5.4); voice identity from a
satellite is a hint, never proof (0007 §5).

What was missing is everything *between* those commitments: how a browser mic reaches STT, who owns a
conversation, how tokens stream, what the loop may do, and what "channel" even means for a logged-in user who
speaks instead of types.

---

## 2. Decision 1 — One turn spine, three thin channel adapters

Every interaction, on every path, runs the **same ordered turn pipeline** (the spine). Channels differ **only**
in their edge adapters (ingress: how an utterance arrives; egress: how a reply is delivered). No channel gets its
own cognition, safety, or authorization path.

```
ingress adapter → identity/session resolution → ActorContext + persona → input safety gate
      → deterministic fast-path classify → context assembly → Sarathi loop (LLM + MCP tools)
      → Result (immutable) → Governor (persona render + output-safety gate) → egress adapter
```

Stage contracts (inputs/outputs/failure behavior, all fail-closed) are normative in the companion agent-implementation
spec. The three adapters:

| Path | Ingress | Egress |
|---|---|---|
| 1 · Console text | GraphQL mutation `sarathi.sendTurn` | GraphQL subscription `onTurnEvents(turnId)` over graphql-ws (token/sentence deltas, tool status, step-up requests, final Result) |
| 2 · Console voice | **AudioWorklet → 16 kHz s16 mono PCM → WebSocket** → central STT (same Whisper spine as satellites) → transcript enters the spine *identically to typed text* | Same GraphQL stream (text render); **optional** server-side Piper TTS played via an `<audio>`/MediaSource element (browser-AEC-visible); **half-duplex v1** (mic gated during playback) |
| 3 · Satellite | Wyoming → HA Assist pipeline → central STT → spine | Sentence-streamed Piper TTS back over HA/Wyoming (streaming TTS is shipped upstream); `[anim:x]` mascot cue; `continue_conversation` flag |

**iOS push-to-talk (G13)** is *not* a fourth path: it reuses path 2's PCM-WebSocket ingress verbatim (replacing
the undesigned Vatara `/api/voice` carry-over). ADR 0001 §6's channel list gains one entry: the **PCM
voice-ingress WebSocket** (browser + iOS), an edge transport like Wyoming — it carries audio + session binding,
never GraphQL.

**Rationale for PCM/WebSocket (from the feasibility study):** raw 16 kHz PCM ≈ 32 kB/s (trivial on LAN/Tailscale),
every frame independently decodable (MediaRecorder chunks are not), zero media-server machinery (WebRTC rejected
as scale plumbing we don't need), and it is byte-identical to the Wyoming audio format — the browser becomes just
another satellite to the same brain. Capture stays entirely in JS (no per-frame .NET interop); `getUserMedia`
requires HTTPS → a Tailscale-issued certificate for remote.

## 3. Decision 2 — Sarathi is a walled service that OWNS conversation data

ADR 0001 §5.2's "Sarathi never hoards" is **re-scoped**: Sarathi never hoards **domain data** (it re-fetches
via governed tools; tool results are not a shadow database). But **conversation data is Sarathi's own domain**,
and pretending otherwise left the platform's most sensitive new data class — cross-domain chat history that can
echo balances and lab values — with no owner, tier, wall, or retention (G3).

- Sarathi becomes a **first-class service** (`src/Sarathi` gains the standard 4 layers + its own stores,
  per the service blueprint): **Postgres** (`Conversations`, `Turns`, `Summaries`, `MemoryFacts` — all
  `WalledEntity`, `OwnerId` = the speaking member) and **one Qdrant collection** payload-walled per ADR 0004 §2.3
  (embeddings via Medha). No Neo4j, no MinIO (raw audio is never persisted — ADR 0009 §3).
- **Encryption tier: `SENS`** for turn content, summaries, and extracted memory facts (replies echo finance/
  health; SENS and GEN both unlock at first admin login, so the conservative tier costs no availability).
  Metadata (keys, timestamps, cids) is GEN. **Per-record sub-keys** (ADR 0002 §3 hook) so right-to-forget
  crypto-shreds a member's conversational footprint cleanly.
- **Retention (amends ADR 0009's matrix — new row):** *Sarathi conversation turns* — owner: **Sarathi service,
  per-member**; hot 90 days → down-sample to intent summaries + extracted `MemoryFacts` (which persist as derived
  facts with provenance turn-ids) → purge raw. Applies **uniformly to console text and voice transcripts**
  (replacing the incoherent "per-domain" transcript row for Sarathi turns). Guest turns: ephemeral, shortest-window,
  never promoted to memory (the persona design §2 upheld).
- **Memory model:** three tiers — verbatim recent window · rolling summary (threshold-triggered, not per-turn) ·
  long-term `MemoryFacts` (extracted, consolidated, per-member walled, retrieved top-k into context). Detail +
  context budget (16k working context, cache-friendly layout) in the companion spec §C; the exact split is a
  **measure-on-box gate**.
- **Subject-aware forget (round-3 R1 — the S6 pattern must not recur here).** A turn can carry **another
  member's data as its subject** (a guardian asking about the child). Turns, summaries, and MemoryFacts therefore
  carry **`dataSubjectIds[]`** alongside `OwnerId`, and a member's **forget fan-out includes every
  turn/summary/fact whose `dataSubjectIds` names them** — shredding those rows' sub-keys and Qdrant points even
  though the speaker owns them. The speaker keeps the conversation shell (metadata + the safe tombstone); the
  subject's content becomes unreadable everywhere, including backups. Acceptance test §9.10 asserts exactly this.
- **Hint-grade identity writes less (round-3 R2).** On `acr=hint` (satellite mapping), turns persist but are
  flagged `attributionAssurance=hint`, and **hint-attributed turns are never promoted to `MemoryFacts`** (a
  misattributed member's words must not train another member's memory); promotion requires a verified
  (`session`/`strong`) interaction. Guests never persist at all. **Memory extraction is itself a consented
  source** (ADR 0009 §6): a per-member "Sarathi conversation memory" consent record gates the extraction pipeline,
  which runs under the member's own identity at background QoS.
- **Cross-channel continuity (G10):** one per-member **conversation ledger** — all turns from all surfaces —
  visible in that member's console (own-data, wall-legal). **Working context is sticky per surface** (a kitchen
  turn never hijacks an open console thread); satellite conversations stay sticky to their box with the
  ESPHome continue-window; the console offers explicit "continue this."

## 4. Decision 3 — Sarathi owns the voice edge (resolves G7)

**The Wyoming conversation service ("the brain") is Sarathi's ingress adapter, not Kaya's.** The home-service
design is amended: Kaya is a *domain service* reached by tools like any other; it does not terminate voice,
resolve speakers, or edge-mint identity. Rationale: whoever terminates Wyoming resolves the speaker and requests
token minting — that is Sarathi-edge work; routing every Artha/Ojas voice turn through the Kaya service would
couple all voice to one domain service's availability and pollute its `act` chains. (The *STT/TTS engines* are
Medha's; the *pipeline position* is Sarathi's.)

## 5. Decision 4 — `inputModality` × `channelAssurance` replace the channel enum (resolves G2/G5/G8)

ADR 0007's `channel ∈ {Voice, Web, Phone}` conflated two axes and left "bare voice channel" undefined. Amended:

- **`inputModality ∈ {Text, Voice}`** — how the utterance arrived. Affects STT handling and **delivery** only.
  It never changes authorization.
- **`channelAssurance ∈ {AuthenticatedWeb, AuthenticatedTrustedDevice, RoomSatelliteHint, None}`** — what the
  identity claim is worth. **This is what `RiskPolicy` consumes.** "Bare voice channel" is now defined precisely:
  `channelAssurance = RoomSatelliteHint` (or `None`).

| Path | Modality | Assurance | Tier reach |
|---|---|---|---|
| Console text | Text | AuthenticatedWeb (passkey session) | T1 free; **T2 within the freshness window**; T3 confirm; T4 strong factor on trusted device |
| Console voice | Voice | **AuthenticatedWeb — identical to console text** | same as console text: *a mic button does not lower assurance* |
| Satellite, mapped speaker | Voice | RoomSatelliteHint | T0/T1 on the mapped identity; **any T2+ forces trusted-device step-up** (ADR 0007 §5 unchanged in effect) |
| Satellite, unknown speaker | Voice | None → **Guest** | T0 only; child-strict; no memory writes |

- **Delivery caveat (modality-aware rendering, owned by the Governor):** T2 content answered on console-voice
  renders as **text by default; speaking sensitive values aloud is a per-member opt-in** — screen privacy ≠
  room-audio privacy. On satellites the ADR 0007 §5 fork is resolved: default policy **deliver-to-phone** for
  T2 reads after step-up, per-home configurable to speak-in-room.
- **Step-up resume protocol (round-3 R3, completes G10).** When a turn hits a step-up requirement it **parks**:
  the turn's state machine is `parked → challenged → approved | denied | expired`. The originating surface gets
  an immediate honest interim ("I've sent a confirmation to your phone"); the Dwarpal challenge carries the ADR 0007
  §7 nonce+action+TTL binding, **TTL ≤ the surface's continuation window** (satellite: the ESPHome
  continue-window) so a park never outlives its conversation. On **approve**, the answer resumes **on the surface
  the delivery policy names** (satellite T2 default: the phone — the parked box speaks a completion cue, not the
  content; console: in-thread). On **deny/expire**, the originating surface states it plainly and the turn closes.
  Parks are per-turn, single-use, audited under the turn's `cid`; a parked turn holds no unexpired elevated token.
- **Guest becomes a real principal (G8):** `actorRole` gains `Guest`; Dwarpal mints an **ephemeral guest token**
  (`sub = guest:{sessionId}`, `tid` bound, TTL = the turn) so enforcement never sees an unauthenticated bare
  context. Guests are T0-only, child-strict, no persistence.
- **Token mechanics (G5/S9, constrains the ADR 0003 amendment):** tokens carry **`acr`/`amr`-class assurance
  claims** (`acr: hint | session | strong`, `amr: [passkey|faceid|pin|satellite-map]`); voice-hint tokens are
  minted `acr=hint` and can never satisfy T2+ regardless of `scope`. Minting is **per turn** at the edge; the
  Sarathi re-exchanges per tool call (`aud` = callee, `act` += sarathi). Mid-conversation expiry → silent re-exchange
  while the edge session lives; an expired *edge* session interrupts with re-auth, never silently continues.

## 6. Decision 5 — Streaming: tone-at-generation + a sentence-buffered Governor gate (resolves G12/G4)

The persona design's render-after-complete-Result model cannot stream, and the latency budget (feasibility study)
requires streaming on every path. Amendment, preserving the invariants that matter:

- The **persona tone fragment is injected at generation time** (persona is resolved *before* the LLM — already
  required — so this is legal); there is **no second LLM rephrase pass**.
- The **Governor becomes a deterministic, sentence-buffered release gate** *(amended 2026-07-29 — see
  §6.1 below; the release rule changed, the gate did not)*: **every** generated sentence passes the output-safety
  scan + persona rules, and **only then** do the sentences release to the egress adapter (console delta / TTS
  chunk — composing exactly with wyoming-piper's sentence streaming). The Governor keeps **stream-kill
  authority**: a failing sentence **anywhere in the answer** kills the whole answer — **no sentence is released
  at all** — and the reply is replaced with the safe rendering.
- **Decisions still precede rendering:** refusals, wall denials, and step-up requirements are *formed as typed
  Results before any token streams* — what streams is only the wording of an already-decided outcome. The
  persona-invariance eval suite (persona design §7) is unchanged and remains the lock.
- **Refusals are rendered from templates, never by the model (round-3 R4).** Blocked/denied Results (safety-gate
  blocks, wall denials, tier refusals) render from **versioned per-persona refusal templates** — config data
  beside the tone fragments (`config/personas/refusals/{persona}.vN.yaml`), pinned by the eval corpus. The LLM
  words only *allowed* outcomes; unsafe input is never re-fed to the model to phrase its own refusal.
- **Stream-kill is atomic with the store (round-3 R8):** on a kill, the persisted `Turn` row is rewritten to the
  safe rendering in the same transaction — replay-from-store can never re-deliver killed sentences; the only
  leak surface is what already left the socket, and the event is audited. *(amended 2026-07-29 — §6.1: with
  nothing released before the scan completes, there is no killed wording in any persistable artefact and no
  socket leak surface at all; this bullet's guarantee is subsumed, not weakened.)*

### 6.1 Amendment 2026-07-29 — scan-before-release (owner-authorised)

> **ADR discipline:** this subsection *amends* §6's release rule and §9 acceptance #5; nothing else above is
> rewritten. Owner record: the child-safety fix and this ADR amendment were **authorised by Ananth on
> 2026-07-29** (unit SCAN-BEFORE-RELEASE). The superseded wording is recorded here, not deleted.

**Superseded normative rule (§6, as written 2026-07-11).** The Governor released each sentence as it passed the
scan; a kill therefore stopped the stream at the offending sentence, and "the only leak surface is what already
left the socket" — the sentences *before* the offending one had already been delivered, and only the offending
sentence onward was withheld.

**New normative rule.** **No sentence is released until every sentence of the answer has been scanned.** If any
sentence fails, the **whole answer is killed** — nothing is released — and the refusal path runs: the reply is
the safe (template) rendering, with the deterministic comfort/redirect append where the turn carries one.

**Reason.** Harm does not respect sentence boundaries. Under the superseded rule an answer whose opening
sentence was innocuous on its own and whose next sentence carried a harmful payload delivered the setup to a
child and withheld only the payload. **A child who has received the setup of a harm whose payload was withheld
has not been partially protected** — the delivered half is the instructional half, and delivery to a child
cannot be retracted (least of all on the voice surface, where it has already been spoken into the room).
Splitting an unsafe answer across sentences was, under the old rule, a way to get its setup past the output
floor.

**Cost: none.** §6's streaming language governs the LLM → Governor stage; the Governor's own render never
streamed to the edge. `IGovernor.Render` returns the complete `GovernorRendering` synchronously and the egress
stage emits from it afterwards, so no sentence had left the process before the scan loop finished even under the old
rule. Scanning every sentence before releasing any therefore adds no round trip, no model call, and no
first-token latency — **the §6 latency targets are unchanged**, and the scan still short-circuits at the first
failing sentence.

**Unchanged by this amendment.** The decision kind still passes through untouched (a kill is an output-delivery
action, never a decision flip); refusal wording still comes from versioned per-persona templates, never the
model; the comfort/redirect append is still added to a killed answer, so a killed turn still
carries its deterministic redirect; and the **input** safety gate and the guardian-alert path — both of which
run *before* the Governor — are untouched by this amendment.

**Latency targets (normative, from the feasibility study):** console text ≤1 s to first token (goal 0.5 s);
console voice ≤2 s end-of-speech → first text/audio; satellite **≤2 s no-tool / ≤3 s one-tool** to first audio
(goal ~0.8 s no-tool). **The input safety gate (Llama-Guard/ShieldGemma pass) is a budgeted line item
(~0.1–0.5 s, measured at the spike stage), not a free stage** (round-3 R6) — targets hold only if the gate
runs on a warm, QoS-reserved model and overlaps STT finalization where possible. Structural enablers (required,
not optional): llama.cpp prompt-prefix caching with a cache-friendly prompt layout (static prefix first,
append-only history), STT on iGPU-Vulkan or NPU, sentence streaming end-to-end, and the deterministic fast-path
below.

## 7. Sarathi's loop contract (resolves G6/G11)

- **Fast-path first (pre-model, deterministic):** a config-driven **intent grammar** (lexicons + patterns as
  data — the platform's sanctioned pattern, per ADR 0001 §6; per-language packs) recognizes (a) **T4-class
  intents** — routed Classify → step-up → owning service, never as a model tool-call (0007 §8 upheld); (b) a
  small set of **T0 instant intents** (time, timers, lights on/off, stop/cancel, volume) answered with **no LLM
  round at all** — the latency floor and the Medha-down floor. A fast-path **miss** falls through to the model;
  the tool boundary remains the enforcing gate (0007 acceptance #4 reworded accordingly — S17).
- **Loop budget:** max **6 tool invocations** or **3 sequential rounds** per turn (config); exceeding → the
  Sarathi reports honestly ("that needs more steps than I can take at once") rather than looping. Cross-service
  chains are bounded by the `act`-chain depth cap (ADR 0003 OQ-3 → set to **3** pending measurement).
- **Tool-call validation:** every model-emitted call is schema-validated Sarathi-side before dispatch; **one**
  repair attempt (re-prompt with the validation error), then honest failure. Malformed rate is an eval metric.
- **`Result` surfacing:** tool `Refused/Validation/NotFound` outcomes surface to the user honestly in persona
  voice; the model may *not* retry a `Refused` with altered arguments (denials are not puzzles).
- **Multi-tool turns** (the ADR 0001 §2.3 flagship): permitted within the loop budget; each hop is a fresh
  audience-scoped token exchange; the turn's plan is auditable (every call + outcome under one `cid`). On
  satellite paths, tool rounds >1 trigger a spoken acknowledgment ("let me check…") to hold the conversational
  contract while the budget runs.
- **Concurrency (G9):** conversations are per-surface (no interleaving); **actuations serialize per member** via
  the D1 idempotency table plus a short cross-surface duplicate-intent window. **Dedup scope (round-3 R7):** the
  window applies only to **absolute-state actuations** (idempotent targets: "unlock the door", "set thermostat to
  70", "arm away") keyed on `hash(tool + normalized-args)`; **relative/incremental intents ("turn it up", "dim
  more", "again") are exempt** — a deliberate repeat must repeat. Interactive turns from different members are
  concurrent by design; Medha QoS ranks interactive > background, FIFO within a class.

## 8. Degraded modes (normative table in the companion spec §F)

Medha contended → QoS priority, honest slowness. **Responder down** → fast-path T0 grammar + T4 path still work
(they never needed the model); everything else refuses honestly with Bodha status; mascot shows degraded state.
**Safety models down → children fail closed** (Bonnie: "I need a nap — ask Dad"); adults proceed on the lexical
floor with a logged, alarmed degrade (the floor's spec is a separate build unit). **HA down** → satellites dead
(honest box state), console fully alive, Kaya/Kavach tools refuse honestly. **Box locked (pre-login)** → the
Sarathi does not converse (history/persona/config are sealed — GEN/SENS by design); console shows the unlock
screen. **Satellites pre-login (round-3 R11) — RESOLVED by the S2 decision (Option B, Ananth 2026-07-11; ADR 0002
§2.3):** the voice ingress does **not** come up pre-login; the commitment is the box's **generic on-device
offline state** ("can't reach home" — voice-edge §5). There is no served "locked" line before admin login.
Kavach Layer 0 alarm paths are independent of Sarathi either way. **Dwarpal down** → no new identified turns (fail closed);
in-flight turns finish on unexpired tokens.

## 9. Acceptance criteria (property/integration tests; full suite in the companion spec §G)

1. **Spine identity:** the same utterance through all three adapters produces identical tool calls, identical
   Results, identical audit records (modulo channel metadata) — proven by a parameterized contract test.
2. **Console-voice assurance:** `artha.get_balance` spoken via console mic with a fresh passkey session serves
   **without** phone step-up (T2, AuthenticatedWeb); the identical utterance on a satellite **requires**
   trusted-device step-up. The **spoken-aloud** rendering of the console answer requires the per-member opt-in.
3. **Hint tokens can't climb:** a token minted `acr=hint` is refused for any T2+ operation at every callee,
   regardless of scope claims.
4. **Guest floor:** an unknown speaker's turn yields `role=Guest`, T0-only; no row, embedding, or memory fact is
   written; a T1+ attempt is refused and audited.
5. **Stream-kill:** *(amended 2026-07-29 — scan-before-release, §6.1)* an output-safety violation in **any**
   sentence stops delivery of the **whole answer** on every path: **no sentence of the answer is released —
   including the sentences that passed the scan before the offending one**; the persisted turn holds the safe
   rendering only, and replay-from-store after reconnect delivers only the safe rendering (never any model
   wording); the event is audited. *(Superseded wording: "a mid-stream output-safety violation stops delivery on
   every path; **the persisted turn is atomically rewritten to the safe rendering, and replay-from-store after
   reconnect delivers only the safe rendering** (never the killed sentences); the event is audited." The
   atomic-rewrite guarantee is subsumed: with nothing released, no killed wording ever enters a persistable
   artefact.)*
6. **Refusal precedes rendering:** for a walled read, the typed `Refused` Result exists before any token streams;
   Bonnie's kind wording and Anik's dry wording carry the same decision (persona-invariance suite).
7. **Loop budget:** a turn requiring >6 tool calls terminates with the honest over-budget reply; no unbounded
   loops; malformed tool-call repair happens at most once.
8. **Fast-path floor:** with the responder model stopped, "set a timer for 10 minutes" and "disarm the alarm"
   both still work (grammar + step-up path); "summarize my spending" refuses honestly.
9. **Duplicate-intent window:** the same **absolute-state** actuation submitted from console and satellite
   within the window executes once (one idempotency record, second returns first's result); **and the negative:**
   a deliberately repeated **relative** intent ("turn it up" twice) executes **twice**.
10. **Forget (subject-aware):** crypto-shredding a member's conversation sub-keys renders their turns/summaries/
    memories unreadable (including in backups) and deletes their Qdrant points — **including turns owned by
    another speaker whose `dataSubjectIds` names the forgotten member** (the guardian-asked-about-the-child case);
    the audit chain still verifies.
11. **Retention:** the Ritu sweep down-samples >90-day turns to summaries+facts and purges raw content, on all
    surfaces' turns equally.
12. **Pre-login:** before admin unlock, `sarathi.sendTurn` returns the locked state; no conversation row is
    readable or writable; satellites show their generic on-device offline state (S2 decided: Option B — no
    pre-login voice ingress).
13. **Hint identity writes less:** a satellite turn with `acr=hint` persists flagged, is **never promoted to a
    `MemoryFact`**, and the safety gate registers on the *mapped* role conservatively; a guest turn persists
    nothing. Memory extraction refuses to run for a member without an active "conversation memory" consent.
14. **Park/resume:** a satellite T2 read parks the turn, speaks the interim, and on phone approval delivers per
    policy (phone by default — the box speaks only a completion cue); on timeout/deny the box states it and the
    park is dead (single-use, TTL-bounded, audited under one `cid`).

## 10. Open questions (defaults chosen; Ananth may override)

1. **Speak-sensitive-aloud opt-in** default OFF per member — confirm. 2. **T4 fast-path grammar scope** —
ships with disarm/unlock/money verbs + per-language packs; extend per miss-rate evals. 3. **Context budget
16k** and the qwen3 snapshot (32k vs 262k native) — pinned by a box-side measurement gate. 4. **Browser barge-in**
— off in v1 (half-duplex); revisit after the AEC device-matrix test. 5. **Duplicate-intent window
length** — default 10 s. 6. **`act`-chain depth cap 3** — re-measure at Phase 1. 7. **Guest voice depth** —
default: answers only ambient questions with no persona memory; deepen deliberately if desired.

## 11. Consequences

**Positive.** The platform's center is now designed to the same standard as its security spine: all three paths
share one testable spine; conversation data has an owner, a tier, a wall, and a forget path; `RiskPolicy` can
express every real situation including a logged-in speaker; latency targets are honest and traceable to measured
evidence; the loop is bounded and evaluable; iOS voice stops being an ungoverned ingress.

**Costs.** Sarathi grows real infrastructure (stores, two new transports, the grammar, the streaming Governor)
— all of it was implicitly required and unbudgeted before; now it is explicit and testable. "Never hoards" needed
an honest re-scoping. Two upstream design docs (persona, home-service) and three ADRs (0003/0007/0009) take amendments —
tracked in the round-2 amendment ledger.

**Constrains.** ADR 0003 (assurance claims, **guest-token `sub` semantics**, per-turn minting), ADR 0007
(modality/assurance split, Guest role, S8's self-recording carve-out interacts with the §5 assurance table),
ADR 0009 (conversation retention row + the "Sarathi conversation memory" consent source), **ADR 0002 (an `ITierMap`
row for Sarathi's mixed-tier entities — SENS content / GEN metadata, per-property `KeyTier`)**, Medha's design
(prefix-cache-aware serving, QoS classes, warm safety models), Akriti (the console chat surface — streaming UI,
premium interaction bar per the owner's UI mandate), and the platform's phased build-out.

---

*Refines ADR 0001; resolves round-2 review findings G1–G13; companion spec: the platform's agent-implementation spec.*
