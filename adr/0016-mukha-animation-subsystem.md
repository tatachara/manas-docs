# ADR 0016 — Mukha (मुख): the Animation & Mascot Subsystem

- **Status:** **ACCEPTED (owner green light, 2026-07-18 night)** — accepted with the three-lens review applied (see Review record).
- **Date:** 2026-07-18
- **Deciders:** Ananth (owner — pending), Claude (orchestrator)
- **Refines:** [ADR 0015](0015-bonnie-mascot-and-cross-surface-animation.md) — **ACCEPTED with owner riders
  (2026-07-18)**; this ADR is rider (2): the animation subsystem gets its own ADR. **ADR 0015's decisions are
  BINDING and are not re-opened here** — 0016 formalizes the SUBSYSTEM ARCHITECTURE that 0015's mascot rides on,
  growing 0015 §10's two-layer split (universal core + per-mascotSet extension) into the full design. Also refines
  [`persona.md`](../architecture/persona.md) §1/§3/§6 (the `mascotSet` field, the shared cue vocabulary, "adding a
  persona = data, zero code"), [ADR 0012](0012-skill-tutoring-capability-modules.md) §3e (closed cue vocabulary,
  box-is-its-own-renderer) and its capability-module Open/Closed discipline, [ADR 0014](0014-conversational-overreliance-teeth.md)
  §3f + unit E5 (idle-face governance — whimsy yes, attention-hook no). Cross-refs the risk register (#33/#34 from
  ADR 0015) and the SDD mascot units (ADR-0016 → M1/M2/M3 + S-0015a–f).

> **ADR 0015 decided the *mascot* — the character, the hardware-first rendering, the never-revert fix. This ADR
> decides the *subsystem* the mascot plugs into: one versioned registry (`animations.v1.json`), a persona→mascotSet
> resolution seam, and a growth path that lets a NEW animation — and eventually a NEW persona's whole animation
> set (Anik, Vani) — land as **data + art, not code** (owner directive, 2026-07-18), WITHOUT weakening the closed
> property that makes the SAFETY cues load-bearing. Mukha is presentation-and-registry only: it consumes the
> deterministic cue the ADR-0015 §5 precedence engine already decided, and NEVER decides a safety or boundary cue
> itself (persona.md §4 delivery-only, restated as a subsystem invariant).**

**What exists today (verified by reading the code, 2026-07-18):**
- `src/Apps/Manas.Console/src/lib/cues.ts` — the shared closed vocabulary is **exactly 3 cues** (`celebrating`,
  `listening`, `try-again`) — a `const` tuple + `AnimCue` type + `isAnimCue` fail-closed guard + `resolveCue`;
  `cues.test.ts` asserts the tuple deep-equals those 3. **Verified.** M1 expands it 3→14 atomically and updates
  that test in the same unit (§7, S3).
- `src/Apps/Manas.Console/src/components/avatar/FallbackAvatar.tsx` — `POSE: Record<AnimCue, FacePose>` is
  **compile-exhaustive** (a new enum member without a POSE is a TS build break). **Verified.**
- `src/Apps/Manas.Console/src/components/avatar/RiveAvatar.tsx` — **re-read and verified 2026-07-18** (B5): **ONE
  `.riv` per mascotSet**, **ONE state machine** (literal `RIVE_STATE_MACHINE = 'persona'`), **ONE numeric cue
  input** (literal `RIVE_CUE_INPUT = 'cue'`) driven by the **cue index** (`cueInput.value =
  CUE_VOCABULARY.indexOf(cue)`); it **runtime-no-ops** a missing input/state — it does NOT build-break (that
  guarantee is `FallbackAvatar.POSE`'s alone, §6/S12). No real `.riv` ships yet (documented slot). The §2 web
  schema is shaped to this contract.
- `src/Sarathi/Sarathi.Domain/Egress/EgressEvent.cs` — `Final(ResultKind, RenderedText, string? AnimCue)`;
  `AnimCue` is `null` in the skeleton (a later-unit concern). **Verified.**
- **The persona→mascotSet ids on disk (verified):**
  - `config/personas/bonnie.yaml:23` and `src/Platform/Kutumba/Persona/DefaultPersonas.cs:65` both carry
    **`bonnie-face-v1`** — the real value the registry keys on. (`persona.md:51` had a **stale `bonnie-face-v2`**
    example; corrected to `bonnie-face-v1` in this pass — B4. `bonnie-face-v2` never existed on disk.)
  - **Code-defined personas exist too** (S10): `src/Platform/Kutumba/Persona/Persona.cs:74` `Persona.SafeDefault`
    carries `guest-face-v1`, and `DefaultPersonas.cs` hardcodes ids in C#. Registry validation MUST cover these
    code-defined personas (SafeDefault's set must resolve), not only the YAML packs. **Verified.**
- **`config/animations.v1.json` does NOT exist yet** (verified: no file on disk). `persona.md` §1/§6 already
  *reference* the registry (startup validation: "`mascotSet` exists in the registry"), so today that validation leg
  is unsatisfiable. Creating and specifying it (under the versioned filename, S11) is this ADR's central
  deliverable (built in unit M1). **Verified absent.**
- Box firmware does not exist yet (box-side gated on S-0015a).

---

## 1. Scope — what Mukha is, and what it is NOT

**Mukha is the animation/mascot SUBSYSTEM:** the registry (`animations.v1.json` + its fail-closed loader), the
persona→mascotSet resolution, the cue→clip binding validation, and the renderer contract every surface honors.
It is the "presentation binding + registry" layer named delivery-only in ADR 0015 §4.

**Mukha is NOT:** a cue *decider*. Which cue is current is owned entirely by the ADR-0015 §5 precedence engine
(the spine/Governor/E-teeth side — distress ≻ turn-lifecycle ≻ circadian ≻ `[anim:x]` advisory ≻ ambient). Mukha
receives that already-decided cue and only *plays the right clip for it* (or fail-closes if it can't). This is
persona.md §4's delivery-only guardrail applied to the animation layer: **Mukha cannot flip, invent, or suppress
a safety cue — it has no handle on the decision, only on the pixels** (§4 below).

---

## 2. Decision 1 — the registry (`animations.v1.json`), single source of truth

`config/animations.v1.json` is the **one versioned registry** (versioned-filename convention, S11) for all
mascotSets, cues, and per-surface clip refs. It follows the repo's fail-closed loader convention (the V6a
`InputLexiconLoader`/`WakeConfigLoader` discipline persona.md §1 already applies to persona packs): **corrupt /
unknown-field / wrong-version / referenced-cue-a-renderer-lacks ⇒ the host refuses to boot at composition**
(fail-closed, never a half-registry). Hot-reload is **web/host-only** (S1) and re-runs the FULL
validation+governance gate (S5); a reload that would *remove* a core cue a live persona needs is refused.

**Schema (normative shape; illustrative values — nothing here is a hardcoded list):**

```jsonc
{
  "version": 1,
  "coreCues": [                       // the UNIVERSAL CORE — the compiled, closed, safety-load-bearing set (§4)
    {"id": "idle",             "failClose": "idle"},
    {"id": "listening",        "failClose": "idle"},
    {"id": "thinking",         "failClose": "idle"},
    {"id": "speaking",         "failClose": "idle"},
    {"id": "celebrating",      "failClose": "idle", "affectNeutral": true},
    {"id": "try-again",        "failClose": "idle"},
    {"id": "storytelling",     "failClose": "idle", "affectNeutral": true},
    {"id": "neutral-concerned","failClose": "neutral-concerned"},  // the safety fail-close TARGET; fails to self (B1)
    {"id": "comforting",       "failClose": "neutral-concerned"},  // safety/degraded — Phase-3-blocking (0015 §10)
    {"id": "need-a-grown-up",  "failClose": "neutral-concerned"},
    {"id": "cant-hear",        "failClose": "neutral-concerned"},
    {"id": "box-fault",        "failClose": "neutral-concerned"},  // SERVER-CUED fault face (≠ firmware-local death face, §5b/S7)
    {"id": "sleeping",         "failClose": "neutral-concerned"},  // circadian (E3); degraded ⇒ concerned, not happy
    {"id": "waking",           "failClose": "idle"}
  ],                                  // the id list MUST equal cues.ts CUE_VOCABULARY as an ordered set — CI (§7,S3)
  "mascotSets": {
    "bonnie-face-v1": {
      "persona": "bonnie",           // back-pointer; VALIDATED against the persona pack (nit) — informational only
      "surfaces": ["web","box"],     // this set's declared render targets; completeness checked per surface (S9)
      "boxClipSetVersion": "sha256:…",// content hash of the baked box partition this entry expects (S1)
      "extensionCues": [             // per-mascotSet whimsy — DATA-DEFINED cue ids, validated not compiled (§4)
        {"id": "reading",  "affectNeutral": true}, {"id": "dancing", "affectNeutral": true},
        {"id": "waving",   "affectNeutral": true}, {"id": "surprised","affectNeutral": true},
        {"id": "thirsty"}, {"id": "drinking-water"}, {"id": "getting-sleepy"}, {"id": "bored"},
        {"id": "poop-break"}, {"id": "yawning"}, {"id": "eating-snack"}, {"id": "humming"},
        {"id": "searching"}, {"id": "stretching"}, {"id": "queasy"}
      ],
      "riv": "bonnie/bonnie.riv",    // web: ONE .riv per mascotSet (B5 — matches RiveAvatar.tsx)
      "stateMachine": "persona",     // web: ONE state machine — default = RiveAvatar's current literal (M2 seam)
      "input": "cue",                // web: ONE numeric cue input — default = RiveAvatar's current literal
      "clips": {                     // cueId -> per-surface clip refs; EVERY core cue MUST be present here
        "comforting": {
          "web":  { "cueIndex": 8 },                                             // web: at most a declared cue index
          "box":  { "clip": "comforting", "format": "rle", "region": [80,40,160,160], "fps": 12 }
        },
        "speaking":   { "web": { "cueIndex": 3 }, "box": { "clip": "speaking", "format": "rle", "fps": 2 } }, // static/low-frame (0015 §1/H2)
        "reading":    { "web": { "cueIndex": 14 }, "box": { "clip": "reading", "format": "rle", "fps": 12 } }
        // ... one entry per core cue + per declared extension cue, on every declared surface
      },
      "llmSelectable": ["storytelling","celebrating","dancing","waving","surprised"], // = registry-derived subset (§4/B3)
      "governance": {                // per-mascotSet review metadata (ADR 0015 §7 owner art sign-off)
        "artReviewedBy": "ananth",
        "artReviewedAt": "2026-08-xx",         // ILLUSTRATIVE date; loader REJECTS a malformed date (nit)
        "rubricVersion": 1,
        "reviewedAssetChecksum": "sha256:…"    // checksum of the reviewed asset set; editing art invalidates it (S5)
      }
    },
    "guest-face-v1": { "persona": "guest", "surfaces": ["web","box"], "extensionCues": [], "clips": { /* CORE ONLY */ } }
  }
}
```

**Schema rules (load-bearing):**
- **`coreCues` is closed and machine-checked registry data.** Each entry is `{id, failClose, affectNeutral?}`. The
  id list MUST equal `cues.ts` `CUE_VOCABULARY` **as an equal ordered set of cue-id strings** (S3 — not a
  byte-for-byte file compare); CI asserts it (§7). The firmware cue enum is the M3 leg of the same check (§7/S3).
- **`failClose` is a per-cue enum: `"neutral-concerned" | "idle"`** (B1). Safety/degraded/circadian cues fail to
  `neutral-concerned`; conversation/whimsy cues fail to `idle`. **`neutral-concerned` is itself a core cue** with
  its own clip, and its fail-close is render-self — a distressed child must never see a cheerful fallback.
  **Runtime fail-close targets are read from this field — never a hardcoded list** (rule #7): the server clamp, the
  web fallback, and the box firmware all consult `failClose`.
- **Every mascotSet MUST provide a `clips` entry for EVERY core cue AND every declared extension cue, on EVERY
  surface it declares in `surfaces`** (per-layer completeness — ADR 0015 §10; S9). A mascotSet missing a clip for a
  declared surface is a boot refusal (the "a box that can't render a cue must never be sent it" rule of 0015 §6,
  made a load-time check). `surfaces` makes 0015's "same Bonnie both surfaces" a checkable property.
- **`extensionCues` are data-defined** — declared per mascotSet, not in the compiled enum. **The loader
  boot-refuses any `extensionCues` id colliding with a `coreCues` id, and any `clips`/`llmSelectable` entry that
  would reclassify a core cue as an extension** (B2) — a core cue can never be demoted to data.
- **`affectNeutral: true` is first-class, owner-gated data** (B3), set only under the §7 governance sign-off. The
  boot check is the **positive assertion `llmSelectable ⊆ {cues with affectNeutral: true}`**. The registry is the
  **SINGLE source of truth for `llmSelectable`**: the server-side precedence engine READS it from the registry (no
  second compiled copy); CI asserts the effective selectable set equals the registry's (§7). The model's only reach
  into Mukha is thus an affect-neutral variant — never a `comforting`, `sleeping`, or `box-fault`.
- **Safety/core cues are NEVER shared across mascotSets** (S6, resolves open-Q3): each set owns its own safety
  clips, and the loader asserts no two mascotSets reference the same clip for a safety-classified cue. Whimsy
  (affect-neutral extension) MAY be shared to save flash/art budget.
- **Box clip refs carry a `format` field** (S4; codec-agnostic — RLE is the leading candidate per 0015 H6,
  `animimg` the fallback) instead of a baked `.rle` extension. Authored box `fps` MUST be ≤ the S-0015b-measured
  per-state budget once measured; **speaking-family cues are clamped static/low-frame until S-0015b proves
  headroom** (loader rule; 0015 §1/H2).
- **`boxClipSetVersion` is a per-mascotSet content hash** of the baked box partition (S1). The box reports its
  baked partition version on adoption/heartbeat; the server cue authority publishes to a given box **only cues
  present in that box's reported version** (an unknown cue ⇒ the box's local safe default). Hot-reload is
  web/host-only; **the box tracks by OTA'd partition version, never by JSON reload.**
- **`governance` metadata is required per mascotSet** and has teeth (S5): a missing/mismatched `governance` block
  is a **HARD build failure for any child-facing mascotSet** (no "warning for now"). `config/animations*` and the
  art asset paths go under `CODEOWNERS = owner`. The sign-off binds a `rubricVersion` + a `reviewedAssetChecksum`
  of the reviewed asset set — **editing art invalidates the sign-off**. Child-surface (box) packs are
  boot-validated owner-signed only. Illustrative dates above are exactly that; the loader **rejects a malformed
  date**.

---

## 3. Decision 2 — per-persona resolution (persona → mascotSet, never hardcoded)

Resolution reuses the **existing** seam, unchanged: `Kutumba.PersonaFor(UserId)` → persona record → its
`mascotSet` id → the registry entry (persona.md §2/§6). **Bonnie = `bonnie-face-v1` (the cowgirl) now** (verified
on disk, §0); Anik/Vani/Guest already carry their own `mascotSet` ids in config. Mukha never maps a persona to art
in code — the mapping is the persona YAML (or a code-defined persona, below) + the registry.

- **Code-defined personas are in scope** (S10): `Persona.SafeDefault` (`Persona.cs:74`, `guest-face-v1`) and the
  `DefaultPersonas.cs` hardcoded ids are validated against the registry the same as YAML packs — **SafeDefault's
  mascotSet MUST resolve** or the host refuses to boot. A guest getting no renderable face is a safety hole, so
  this leg is not optional.
- **Guest = universal core only** (ADR 0015 §10): `guest-face-v1` declares an empty `extensionCues`, so an
  unidentified speaker gets the conversation/degraded/circadian faces and no whimsy — matching persona.md §2's
  safe-default posture (a guest gets less personality by construction).
- **Worked example — adding an "Anik minimal set" is pure data + art:**
  - **Touches:** (a) `config/personas/anik.yaml` `mascotSet: anik-face-v1` (already present); (b) a new
    `mascotSets["anik-face-v1"]` block in `animations.v1.json` (core-cue `clips` for each declared surface +
    whatever small `extensionCues` a "dignified adult" set wants — likely few or none: an adult Anik need not
    render a poop-break); (c) the art assets themselves (`.riv` for web, baked-clip partition for box); (d) the §7
    governance review + checksum-bound `governance` metadata (S5).
  - **Must NOT touch:** the compiled `coreCues` enum (`cues.ts`/firmware), the precedence engine (ADR 0015 §5), the
    `RefusalKind → cue` binding table (ADR 0015 §6.1), the loader code, or any other mascotSet. If adding a
    persona's set requires touching any of those, the design has failed the "data + art, not code" directive — that
    is the acceptance line for §7's arch tests.
  - **Vani** is identical: `vani-face-v1`, its own tone of whimsy, added the same way, later, without a rebuild.

---

## 4. Decision 3 — the growth path & the closed-vs-flexible split

**ADR 0015 §10 (finding F8) is the ACCEPTED authority** for the two-layer split (universal core + per-mascotSet
extension); it was owner-accepted with ADR 0015 on 2026-07-18 (S13). This ADR does not re-litigate it — it makes it
precise. The owner directive ("flexible to add new animations without code change") and ADR 0015 §6's load-bearing
"closed vocabulary" argument are reconciled by **two layers with two different mechanisms**:

| Layer | What it holds | Where it lives | Add mechanism | Fail-closed mechanism |
|---|---|---|---|---|
| **Universal core** | conversation + degraded/safety + circadian cues (`comforting`, `need-a-grown-up`, `neutral-concerned`, `sleeping`, …) | **compiled enum** — `cues.ts` `CUE_VOCABULARY` + box firmware enum, mirrored in `coreCues` | **code change (rare, deliberate)** — a new core cue recompiles both renderers + updates the binding table + passes governance | **data-driven from `failClose`** — safety/degraded → `neutral-concerned`, others → `idle`; the web's `FallbackAvatar.POSE` is compile-exhaustive (a missing core POSE = build break, verified §0/S12); the server clamps unknown→the cue's `failClose` |
| **Per-mascotSet extension** | that persona's whimsy (`reading`, `dancing`, `poop-break`, …) | **data** — `extensionCues` in `animations.v1.json` | **NO recompile** — art PR + registry entry + §7 governance review | **runtime, registry-validated** — renderer plays the declared clip or fail-closes per `failClose` (`idle` for whimsy); an `[anim:x]` naming an unknown/non-`llmSelectable` id is rejected (`cues.ts` fail-closed, ADR 0015 §5) |

**Why the split is safe (the justification ADR 0015 §6 demands):** the closed property is load-bearing *because the
SAFETY cues must be a fixed, exhaustively-implemented set* — a distress child must always reach a `comforting`
face, a dead box must always show a fault face, and no data edit may ever remove or shadow those. Keeping the
**core compiled** preserves that exactly: safety cues change only by a deliberate, reviewed code change; the web's
compile-exhaustive `FallbackAvatar.POSE` map turns "forgot to implement a core cue" into a build break; and
`neutral-concerned` + `box-fault` are **embedded in the box firmware locally** (§5b/§6b/B1) so a corrupt clip
partition still yields a correct safety face. The **extension layer buys flexibility only where it is safe to be
flexible** — affect-neutral whimsy, per mascotSet, that fails closed to `idle` if missing. **The closed property is
NOT weakened for the safety cues; it is *scoped* to them, and relaxed only for whimsy.**

**Adding an animation, end to end:**
1. **Extension whimsy (the common case):** bake art (per the registry's `surfaces` rules) → add the cue id to
   `extensionCues` + a `clips` entry → run the §7/E5 governance review (no dark pattern; the
   whimsy-vs-attention-hook line of ADR 0015 §7) → record `governance` metadata (checksum-bound). **No recompile.**
2. **A new core/safety cue (rare):** the above **plus** a code change to the compiled enum in `cues.ts` + box
   firmware + a `RefusalKind → cue` / degraded-cell binding-table update (ADR 0015 §6.1) + the compile-exhaustive
   `FallbackAvatar.POSE` addition + a `failClose` classification. Deliberately heavier — a safety cue is not a data
   edit.

---

## 5. Decision 4 — the cue-authority interface (Mukha never decides safety)

Mukha consumes the deterministic current cue from the **ADR-0015 §5 precedence engine** and the transport ADR 0015
§6.2/§8 defines — it does **not** re-derive precedence, and it owns only presentation binding + registry
validation.

- **Inputs Mukha subscribes to:** (a) the turn-terminal cue on `Final.animCue` (`EgressEvent.cs`, verified `string?
  AnimCue`); (b) the **ambient cue channel** ADR 0015 §6.2 adds (`AmbientCue { cue }` push on the web + the box HA
  sensor) — the standing/between-turns cue. One **server-side cue authority** (ADR 0015 §6.2/§6.4 F4) publishes to
  both transports; Mukha is a *consumer*, never a second scheduler (no client-side idle scheduler — the
  presence-blind/history-blind rule, ADR 0015 §5, stays server-side).
- **The cue authority publishes per box version** (S1): it publishes to a given box only cues present in that box's
  reported `boxClipSetVersion`; a cue the box's baked partition lacks resolves to the box-local safe default rather
  than being sent.
- **What Mukha does with a cue:** resolve the active mascotSet (§3) → look up the cue's `clips` entry → hand the
  surface renderer the clip ref. That is the whole job.
- **What Mukha must NOT do (restated invariants):** it does not choose the cue, cannot upgrade/downgrade a safety
  cue, cannot let an `[anim:x]` advisory tag override a deterministic cue. The precedence engine already clamps the
  advisory to the `llmSelectable` subset **which it reads from the registry** (B3 — no second compiled copy); Mukha
  additionally validates the tag against the registry and fail-closes. **The LLM is never a boundary** (CLAUDE.md
  rule #5): the model's only reach into Mukha is the affect-neutral `llmSelectable` variant, registry-validated and
  fail-closed.
- **Fail-closed defaults (data-driven from `failClose`, never a hardcoded list — B1):** unknown/missing core cue →
  the server clamps to that cue's `failClose`; a safety cue with a missing renderer asset falls back to
  `neutral-concerned` (never `idle`/happy); loss-of-cue-channel on the box → the **firmware-local death face**
  (§5b). Mukha implements these; it does not invent new ones.

**§5b — the box-fault reconciliation (S7).** There are TWO fault faces, deliberately distinct:
- **The firmware-local death face is persona-agnostic:** ONE fixed, firmware-embedded asset shown when the cue
  channel is dead — server-side persona resolution is unreachable *exactly when it is needed* (ADR 0015 §6.3).
  `config/satellite/liveness.v1.yaml`'s `faultFaceAssetId` (verified: `bonnie-fault-face-v1`) NAMES this
  firmware-embedded asset; a **cross-config validation** asserts the two cannot drift (§7).
- **The registry's `box-fault` clip is the SERVER-CUED fault face** (the server can still reach the box but wants a
  fault state). It is per-mascotSet data; its `failClose` is `neutral-concerned`.

---

## 6. Decision 5 — the renderer contract per surface

A **surface renderer** is any target that turns a cue into pixels. Each implements one small interface; the
registry is renderer-agnostic (it stores refs, not renderer logic).

| Surface | Renderer | Clip ref consumed | Notes |
|---|---|---|---|
| **Web (Akriti)** | Rive (`RiveAvatar`) + Motion `FallbackAvatar` | mascotSet-level `riv`/`stateMachine`/`input` + per-cue `clips[cue].web.cueIndex` | ONE `.riv`, ONE state machine, ONE numeric cue input driven by cue index — the SHIPPED RiveAvatar contract (§0/B5); cue-driven ONLY — no pointer/scroll/presence inputs (ADR 0015 §7, S-0015e) |
| **Box (the voice box device)** | Manas-owned LVGL baked-clip player | `clips[cue].box` (clip + `format` + region + fps) from the **baked partition** (§6b) | plays baked region frames from flash; Manas owns the panel (never reverts — ADR 0015 §3); `speaking` static/low-frame (ADR 0015 §1/H2); gated on S-0015a |
| **Browser emulator** | renders the box's *baked* clips at box res/fps, captures headless | `clips[cue].box` | the art-without-flash review loop (§6c); a first-class renderer, not a toy |

**B5 — the web schema seam.** `riv`/`stateMachine`/`input` live at **mascotSet level** (one `.riv`, one state
machine, one numeric cue input); per-cue web data is **at most a declared cue index**. The M2 seam: either the
literals stay pinned in `RiveAvatar.tsx` or `RiveAvatar` reads them from the registry — **decision: registry-driven,
with the current literals (`'persona'`, `'cue'`) as defaults** so today's shipped asset keeps working and the
registry can override without a code change.

**S12 — compile-exhaustiveness, scoped honestly.** The compile-exhaustive guarantee is `FallbackAvatar.POSE`'s
ALONE (a core cue without a POSE = TS build break, verified §0). `RiveAvatar` **runtime-no-ops** a missing
state/input (verified §0) — it is NOT a build-time check. The `.riv` state-completeness check (every core cue has a
state in the authored `.riv`) is therefore a **NAMED M2 deliverable**, not an assumed existing mechanism.

**§6b — the registry→box seam (B6).** **The box NEVER consumes `animations.v1.json`.** The clip partition image is
**BUILT from the mascotSet's box refs at flash/OTA time (codegen)**. A **flash-time/CI manifest gate** requires the
partition image to contain **exactly** the clips the registry names for that mascotSet (no fewer) — the build
**refuses** otherwise. Plus the B1 firmware-local safety faces: `neutral-concerned` AND `box-fault` are **compiled
into firmware** (never flash-partition-dependent), so a truncated/corrupt clip ⇒ the locally-embedded
`neutral-concerned` face, **never blank/idle**.

**§6c — the browser emulator (S2).**
- **Asset-path convention:** the emulator maps a registry box ref `{clip, format, region}` to an emulator-fetchable
  asset at a conventional path (e.g. `emulator/<mascotSet>/<clip>.<format>`), so the same refs the codegen bakes are
  the refs the emulator renders — no second manifest.
- **Clip format:** a **normative RLE-region clip format** with **ONE shared decoder definition** used by BOTH the
  emulator and the firmware (a single spec, two implementations that must agree) — the emulator is not allowed a
  private codec.
- **Fidelity boundary:** the emulator is faithful for **art / identity / frame-content** (YES) and explicitly
  **NOT** for **timing / fps / CPU-under-reply-load** (NO — that is S-0015b, real hardware only). The emulator
  de-risks the art, never the performance.

**What a NEW surface (e.g. iOS, persona.md §3) MUST implement to be Mukha-compliant:**
1. **The universal core** — a renderer for every `coreCues` member (per-layer completeness; the loader refuses to
   bind a surface a mascotSet's core `clips` don't cover for it).
2. **The fail-closed default** — unknown/missing cue → the cue's `failClose` (data-driven, B1); a missing
   **safety**-cue asset → `neutral-concerned` (never happy). No blank/frozen state (the S2 no-dead-air discipline,
   ADR 0015 §5/§10).
3. **The ambient-cue subscription** — subscribe to the ADR 0015 §6.2 ambient channel so between-turns cues render
   (the web-had-no-ambient-channel bug, F1, must not recur on a new surface).

A surface implements the cue→renderer map once and inherits every persona's mascotSet for free (persona.md §3's "a
new surface implements the map once" property, now grounded in the registry).

---

## 7. Testing strategy

Inherits ADR 0015's test posture and the E5/ADR-0014 §3f architecture tests; adds registry-specific checks.

- **Registry loader — planted-positive fail-closed tests** (the V6a loader discipline): corrupt JSON, unknown
  field, wrong `version`, a mascotSet missing a core-cue clip for a declared surface, a `clips` ref to a cue absent
  from `coreCues`/`extensionCues`, an `llmSelectable` entry not `affectNeutral`, an `llmSelectable` entry not in the
  mascotSet, a malformed governance date, a child-facing mascotSet with a missing/mismatched `governance` block,
  **an extension id that collides with a core id ⇒ refuse to boot (B2)**, and a `clips`/`llmSelectable` entry that
  would reclassify a core cue as an extension ⇒ refuse to boot — **each must refuse to boot** (planted-positive, so
  a green loader is proven, not assumed).
- **Core-enum ≡ registry-core (S3)** — a CI check asserts `coreCues`' id list equals `cues.ts` `CUE_VOCABULARY` **as
  an equal ordered set of cue-id strings** (not byte-for-byte). M1 = these two REAL artifacts (registry + `cues.ts`),
  with `cues.ts` expanded 3→14 **atomically in the same M1 unit** and `cues.test.ts` updated in that unit. The
  **firmware leg is a documented M3 activation, conditional on S-0015a green** — never a stubbed third file now.
  **The server has NO second compiled cue enum** — the server-side clamp/precedence reads the registry (B3).
- **Safety subset ≡ code-side safety handling (B1)** — a CI check asserts the safety-classified subset (every cue
  whose `failClose` is `neutral-concerned`) equals, **as an ordered set of cue-id strings**, the code-side safety
  handling; drift fails the build.
- **`llmSelectable` positive assertion (B3)** — the boot check asserts `llmSelectable ⊆ {cues with affectNeutral:
  true}`; CI asserts the **effective** server-side selectable set equals the registry's `llmSelectable` (no second
  compiled copy).
- **Per-layer completeness (S9)** — for every mascotSet × every cue (core AND extension) × every declared surface in
  `surfaces`: a `clips` entry exists. (The load-time check, additionally asserted in test.) This makes 0015's "same
  Bonnie both surfaces" checkable.
- **No shared safety clip (S6)** — the loader asserts no two mascotSets reference the same clip for a
  safety-classified cue.
- **Compile-exhaustive web core (S12)** — a new core cue without a `FallbackAvatar.POSE` is a TS build break
  (verified §0); the test locks that. The `.riv` state-completeness check is the NAMED M2 deliverable, not assumed.
- **Persona-resolution (B4/S10)** — `PersonaFor(bonnie)` → a `mascotSet` **present in the registry** (so a mismatch
  fails CI, never the child's box); guest resolves to a core-only set; **the code-defined `Persona.SafeDefault`'s
  mascotSet MUST resolve**; a persona with a `mascotSet` absent from the registry is a boot refusal (persona.md §1
  validation leg, now satisfiable once `animations.v1.json` exists).
- **Cross-config fault-face validation (S7)** — `liveness.v1.yaml`'s `faultFaceAssetId` names the firmware-embedded
  death-face asset; a test asserts the two cannot drift.
- **Flash-time/CI manifest gate (B6)** — the baked box partition contains exactly the clips the registry names for
  that mascotSet; the build refuses otherwise.
- **Architecture tests inherited (ADR 0014 §3f / E5):** the idle scheduler is presence-blind + history-blind (no
  mic/presence/arrival-time input); the animation layer holds **no reward state**; Mukha exposes no input outside
  the cue enum (the Rive-cue-driven-only assertion, S-0015e).
- **Governance (honest limit):** the load-bearing gate is **owner art sign-off** against the ADR 0015 §7 rubric — a
  text-lint over `config/animations*` cue names/descriptions is nice-to-have and honestly labeled as NOT covering
  the baked art. The `governance` block (checksum-bound, S5) records the sign-off; a child-facing mascotSet without
  it is a **hard build failure**, and child-surface packs boot-validate as owner-signed only.
- **Golden-frame** — manual owner art review per cue is the baseline (ADR 0015 §9 S-0015e, H5 demoted); an automated
  "costume tokens present" check is nice-to-have only.

---

## 8. Loader placement & Consequences

**Loader placement (S8).** `AnimationsRegistryLoader` lives in **`Sarathi.Application`** (matching the existing
loader homes — `InputLexiconLoader`, `WakeConfigLoader`, `SatelliteLivenessContractLoader`, …), reading the
versioned filename `config/animations.v1.json` (S11). Its **unknown-field strictness is a DELIBERATE STRENGTHENING**
over the repo's `IgnoreUnmatchedProperties` convention — stated explicitly because this is a safety-load-bearing
pack (a typo'd `failClose` or `affectNeutral` must fail loud, not be silently ignored). The **web obtains the
registry at build time (bundled)**; the **box obtains its clips via the §6b codegen**, never the JSON.

**Owner concept-render gate (nit).** ADR 0015's owner rider "owner sees a Bonnie concept render before build starts"
is a **gate before M2** (the web-art unit): the owner has **SEEN the v1 concept sheet**, formal approval
**pending**. M2 art build does not start until that approval lands (§10).

**Positive.** The mascot ADR 0015 designed gets a subsystem that makes its own growth cheap and safe: a new
animation is an art PR + a registry line (no recompile), a new *persona's whole set* (Anik, Vani) is data + art
against an unchanged core, and the safety cues stay in a compiled, exhaustively-implemented, reviewed closed set.
The persona→mascotSet seam already exists in config (verified), so this ADR mostly *specifies the registry the code
already assumes exists* — it closes the referenced-but-absent gap. Mukha stays delivery-only: it cannot become a
covert decision path because it never decides a cue.

**Costs (honest).** A real registry + fail-closed loader + the CI equivalence checks (registry↔`cues.ts` now,
firmware at M3) is genuine machinery, and the two-layer split means two add-mechanisms to document and test (core =
code+review; extension = data+review). Per-mascotSet completeness means every new persona set owes a clip for
*every* core cue on *every* declared surface — the discipline that prevents a missing safety face has a per-set
cost. The governance gate on baked art remains a human review (§7), not an automated proof.

**F8 blast-radius tradeoff (nit).** The single fail-closed boot gate means one malformed mascotSet refuses the WHOLE
host's boot (host-wide blast radius), rather than isolating the failure to that mascotSet. **Decision: keep the
single fail-closed boot gate for v1** — fail-closed-and-loud is the right default for a safety pack, and Bonnie is
the only child-facing set today. **Revisit trigger:** if a non-Bonnie set ever breaks a boot, add per-mascotSet
isolation (a bad Anik set degrades Anik to SafeDefault, not the whole host).

**Constrains.** `cues.ts` + the box firmware enum (both must mirror `coreCues`); `persona.md` (the
`animations.v1.json` validation leg becomes real); Akriti (`RiveAvatar`/`FallbackAvatar` implement the core per the
registry); the SDD (M1 builds the registry + loader + core-enum expansion + binding table; M2 the web art + the
`.riv` completeness check; M3 the box firmware + codegen, gated on S-0015a); the risk register (#33/#34).

**Acceptance path for this ADR itself:** (1) ✅ 3-lens adversarial hardening (child-safety · security/determinism ·
feasibility); (2) ✅ findings applied (Review record below); (3) ⏳ owner review + acceptance — OWED; (4) only then
does the SDD schedule M1 against this registry design (box-side still gated on S-0015a).

## 9. Open questions (owner may override)

- **Registry file layout** — one `config/animations.v1.json`, or a `config/animations/` directory (one file per
  mascotSet, an index) once several personas carry sets? (The schema is layout-agnostic; deferred to when the
  second real set lands. The versioned-filename convention S11 holds either way.)
- **Extension-cue governance metadata granularity** — per-mascotSet sign-off (proposed) vs per-cue sign-off for
  higher-risk whimsy (`queasy`, the night face — ADR 0015 §7 fright/dark review). Owner call.
- **`mascotSet` versioning** — how are mascotSet versions bumped/pinned against the eval/golden-frame baseline (the
  persona.md §1 `toneFragment.version` precedent)? (`bonnie-face-v1` is the current single version.)

*(Open-Q3 "cross-mascotSet clip reuse" is RESOLVED by S6: safety/core cues never shared; whimsy may share.)*

## 10. Risk-register touchpoints

This ADR does not add new rows; it is the design side of ADR 0015's rows:
- **#33 — box mascot reliability (the Jarvis scar):** Mukha's box renderer contract (§6) + the firmware-local death
  face (§5b) + the flash-time manifest gate (§6b) are the subsystem side of the S-0015a/b gates; fallback =
  static-pose Bonnie, unchanged.
- **#34 — mascot art production:** the registry's checksum-bound `governance` metadata + the emulator renderer (§6c)
  are the review surface; the owner concept-render gate (§8) precedes M2; fallback = a smaller v1 clip set (whimsy
  lands incrementally as the flash budget allows, ADR 0015 §2/§9).

---

## 11. Tensions with ADR 0015 (resolved)

1. **The "code change" ambiguity (RESOLVED, S13).** ADR 0015 §6 reads both "adding a state … adds the enum member"
   and "No code change … for a new *variant*." **ADR 0015 §10 (finding F8) is the ACCEPTED authority** for the
   two-layer split, owner-accepted with 0015 on 2026-07-18 — core = code, extension = data. This ADR formalizes that
   split; it is not a new shape.
2. **§6.4's fail-closed mechanism covers only the compiled layer (acknowledged asymmetry).** The web fail-close
   rests on `FallbackAvatar.POSE` being compile-exhaustive; that guarantee **cannot** exist for data-defined
   extension cues (they are not enum members), so a missing extension pose is a *runtime* fail-close (to `idle` via
   `failClose`), not a build break. A genuine, accepted asymmetry — acceptable because extensions are affect-neutral
   whimsy, and scoped honestly per S12 (compile-exhaustiveness is `FallbackAvatar.POSE`'s alone).
3. **`animations.v1.json` is referenced but does not exist yet.** `persona.md` §1/§6 make "`mascotSet` exists in the
   registry" a startup validation, but there is no such file today (verified). Until M1 ships the registry that
   validation leg is unsatisfiable — a pre-existing gap this ADR closes, flagged so it is not mistaken for working
   behaviour.
4. **`llmSelectable` — single source of truth (RESOLVED, B3).** The registry is the single source of the selectable
   set; the server-side precedence engine READS it from the registry (no second compiled copy); CI asserts the
   effective selectable set equals the registry's. The two lists cannot drift.

---

## 12. Review record (2026-07-18) — three-lens adversarial hardening

Three independent adversarial lenses (child-safety · security/determinism · feasibility), refute-first. Every
finding below was ACCEPTED by the owner-orchestrator and applied above — no redesign, all localized doc fixes.

**Blocking — all applied:**
- **B1** — per-cue `failClose` enum; `neutral-concerned` a first-class self-failing core cue; firmware embeds
  `neutral-concerned`+`box-fault` locally; CI safety-subset≡code check; runtime fail-close data-driven (§2/§4/§5b/§6b/§7).
- **B2** — loader boot-refuses an extension id colliding with a core id, or any reclassification of a core cue to an extension; planted-positive test (§2/§7).
- **B3** — `affectNeutral` owner-gated; boot check `llmSelectable ⊆ {affectNeutral}`; registry is the single source, server reads it, CI asserts equality (§2/§5/§7/§11.4).
- **B4** — real value is `bonnie-face-v1` (bonnie.yaml:23 + DefaultPersonas.cs:65); stale `bonnie-face-v2` in persona.md:51 corrected; registry keys on v1; four-location-agreement + `PersonaFor(bonnie)→registry-present` test (§0/§3/§7).
- **B5** — web schema matches shipped RiveAvatar: one `.riv`/state-machine/cue-input at mascotSet level, per-cue web = a cue index; registry-driven, current literals as defaults (§0/§2/§6).
- **B6** — registry→box seam: box never consumes JSON; partition codegen'd at flash/OTA; flash-time/CI manifest gate; firmware-local safety faces (§6b/§7).

**Should-fix — all applied:**
- **S1** — `boxClipSetVersion` + version-scoped publish; hot-reload web/host-only, box tracks OTA'd version (§2/§5).
- **S2** — emulator sub-section: asset-path convention, one shared RLE decoder, art-yes/timing-no fidelity boundary (§6c).
- **S3** — M1 registry≡`cues.ts` (3→14 atomic, test updated); firmware = M3 conditional on S-0015a; "equal ordered set of cue-id strings" not byte-match; no server cue enum (§7).
- **S4** — codec-agnostic `format` field replaces baked `.rle`; fps ≤ S-0015b budget; speaking clamped static/low (§2).
- **S5** — governance teeth: CODEOWNERS=owner; missing/mismatched block = hard build failure; sign-off binds `rubricVersion`+asset checksum; hot-reload re-runs full gate; box packs boot-validated owner-signed (§2/§7).
- **S6** — safety/core cues never shared, whimsy may; loader asserts it (resolves open-Q3) (§2/§7/§9).
- **S7** — persona-agnostic firmware-local death face vs server-cued registry `box-fault`; cross-config check vs `liveness.v1.yaml` `faultFaceAssetId` (§5b/§7).
- **S8** — `AnimationsRegistryLoader` in `Sarathi.Application`, versioned filename; unknown-field strictness a deliberate strengthening over `IgnoreUnmatchedProperties`; web bundled, box codegen (§8).
- **S9** — mascotSet `surfaces` + per-layer completeness across every cue × declared surface (§2/§7).
- **S10** — code-defined personas covered: `SafeDefault` (`guest-face-v1`) + `DefaultPersonas.cs` ids validated against the registry (§0/§3/§7).
- **S11** — renamed `config/animations.v1.json`; persona.md references updated (§2, persona.md).
- **S12** — compile-exhaustiveness scoped to `FallbackAvatar.POSE`; RiveAvatar runtime-no-ops; `.riv` completeness = named M2 deliverable (§0/§6/§7).
- **S13** — Tension 1 resolved by citing ADR-0015 §10 (F8) as accepted authority; hedging dropped (§4/§11.1).

**Nits — all applied:** governance dates illustrative + loader rejects malformed dates (§2/§7) · owner concept-render
a gate before M2, owner has SEEN the v1 concept sheet, approval pending (§8/§10) · mascotSet `persona` back-pointer
validated against the persona pack (§2) · F8 blast-radius tradeoff in Consequences — keep single fail-closed boot
gate for v1, per-mascotSet isolation as a revisit trigger (§8).

---

*Sibling of ADR 0015 (the mascot); composes with persona.md (delivery-only), ADR 0010 (cue on the egress),
ADR 0011/0012 §3e (closed cue vocabulary, box-is-its-own-renderer), ADR 0014 §3d/§3f + units E3/E5. Drafted
2026-07-18 as the subsystem ADR 0015's owner rider (2) requires; three-lens hardened; awaiting owner acceptance.*
