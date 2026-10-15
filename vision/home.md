# Kaya — Vision & Feature Charter

> Service codename: **Kaya** (decided 2026-07-12, ADR 0001 §3; was working name `Home`; was the Vatara device
> side). Part of the **Manas** platform.
> Read [`docs/adr/0001-manas-platform-architecture.md`](../adr/0001-manas-platform-architecture.md) first — this
> document refines the Kaya service's charter within those platform decisions and must not contradict them.
>
> - **Status:** Vision (pre-build). Supersedes/absorbs the Vatara device work.
> - **Date:** 2026-07-11
> - **Layer map:** `Kaya.Domain` / `Kaya.Application` / `Kaya.Infrastructure` / `Kaya.Api` (Clean Architecture,
>   dependencies inward). All device I/O lives behind one port in Infrastructure (`IKayaProvider` →
>   `HomeAssistantProvider`). The LLM lives only behind a port; the actuation boundary is deterministic C#.

---

## 1. Intent

**Kaya is the household's senses and hands over the physical world.** It reads the live state of the home —
every light, thermostat, media player, blind, lock, door, plug, sensor, and who is present in which room — and
acts on it through **Home Assistant as the single, guarded path to every device**, so the home stays safe,
comfortable, responsive, and cheap to run. It is **deterministic-first**: the heavy lifting — the device
registry, state projection, the event→condition→action rule engine, scene composition, energy aggregation,
occupancy fusion, and the validated actuation pipeline — is pure, testable C#; the local LLM is a **thin skin**
that only understands a spoken/typed intent, disambiguates, and phrases a reply, and is **never** the thing that
decides to move a physical device. **Kaya never actuates on a guess:** every command is parsed to a typed,
validated intent, checked against the live device registry and safety/permission policy, **step-up-gated for
high-risk actions** (unlocking a door, disarming, opening the garage — a strong factor, never the model's
judgment), executed **idempotently** through the one provider, and **audited end-to-end** with a correlation id.
It is local-first and private — all control runs on the family's own hardware, no device touches a vendor cloud
where a local path exists, and untrusted-by-origin gear is VLAN-isolated and egress-blocked. Kaya is one mind's
body: it hands life-safety concerns to **Kavach**, energy-cost reasoning to **Artha**, presence and context to
**Sarathi, Ojas, and Abhyasa**, and in return it makes the whole platform *able to feel and change the room
you are standing in.*

---

## 2. Capability areas

**Kind legend:** `[deterministic]` = pure C#, no model · `[LLM]` = the local LLM genuinely helps (intent,
phrasing, disambiguation) · `[integration:name]` = depends on an external system/adapter.
**Feasibility:** `now` = buildable on today's stack · `needs-hardware` = requires a device/sensor purchase or
install · `needs-3rd-party` = a specific integration/library must exist & be wired.
**Priority:** P0 = foundational / first slice · P1 = core value · P2 = later polish or specialised.

### 2.1 Sensing — the home's state (the senses)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Live state projection | Subscribe to HA's WebSocket event stream; maintain an in-memory + persisted projection of every entity's current state & attributes. | `[integration:HomeAssistant]` | now | P0 |
| Device/entity registry sync | Pull HA's entity/device/area registries; derive the canonical Kaya device model (rooms → devices → entities → capabilities). Auto-drop `unavailable` devices; honour an `ExcludeEntities` config. | `[deterministic]` `[integration:HomeAssistant]` | now | P0 |
| Capability model | Classify each entity into typed capabilities (Light, Climate, Cover, Lock, MediaPlayer, Switch, Sensor, BinarySensor, …) with the actions/attributes each supports — the schema the actuation boundary validates against. | `[deterministic]` | now | P0 |
| Historical state store | Append-only history of state changes (event-sourced where it matters) for trend/analytics and "what happened at 9pm?" queries. | `[deterministic]` | now | P1 |
| Environmental sensors | Ingest temperature, humidity, illuminance, air quality (CO₂/PM2.5/VOC), water-leak, smoke/CO where present. | `[integration:HomeAssistant]` | needs-hardware | P1 |
| Door/window/contact state | Read open/closed contacts and expose to rules & Sarathi (Kavach owns the *security* interpretation). | `[integration:HomeAssistant]` | needs-hardware | P1 |
| Natural-language state query | "Is the garage open?", "How warm is the child's room?" — resolve entity + phrase the answer from the deterministic projection (no fabrication; unknown → "I don't have that"). | `[LLM]` `[deterministic]` | now | P0 |

### 2.2 Actuation — the guarded boundary (the hands)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Typed intent → validated command | Parse an utterance to a typed `DeviceIntent`, resolve it against the live registry, validate target/params, refuse if ambiguous or impossible — **never actuate on a guess**. | `[deterministic]` | now | P0 |
| Single provider path | Every actuation goes through `IKayaProvider`/`HomeAssistantProvider` (HA `call_service`). No other code path may command a device. | `[deterministic]` `[integration:HomeAssistant]` | now | P0 |
| Idempotent execution | Every command carries an idempotency key + outbox pattern → effectively exactly-once (at-least-once + idempotent consumers, ADR 0005 §3.2), no double-actuation on retry. | `[deterministic]` | now | P0 |
| High-risk step-up gate | Unlock, disarm, open garage/gate, and other high-blast-radius actions require a strong factor (PIN/passkey), **deterministic — never the model's decision**; a fast-path recognises these and routes to the shared step-up. | `[deterministic]` | now | P0 |
| Permission / per-user walls | Actions are authorised against the caller's verified identity + role (child cannot unlock the front door; guest scope is narrow). Enforced deterministically, not by the LLM. | `[deterministic]` | now | P0 |
| Output/response governor | The spoken/written confirmation passes a deterministic governor before TTS/UI (defense in depth; child-safe persona for a child speaker). | `[deterministic]` | now | P1 |
| Tool-calling actuation | Sarathi's MCP `kaya.*` tools wrap the same application services (`kaya.set_lights`, `kaya.set_climate`, …); the LLM either answers or calls a guarded tool. | `[LLM]` `[deterministic]` | now | P0 |
| Ambiguity clarification | When a target is under-specified ("turn on the light" with 3 lit rooms), ask by name rather than guess. | `[LLM]` | now | P1 |
| Optimistic + reconciled state | Reflect the intended state immediately, reconcile against the real HA event to catch failures/timeouts and report honestly. | `[deterministic]` | now | P1 |

### 2.3 Lighting

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| On/off/toggle/dim by room or name | Core light control resolved against the registry. | `[deterministic]` `[integration:HomeAssistant]` | now | P0 |
| Brightness / color-temp / RGB | Set brightness %, warm/cool, and color where the light supports it (capability-checked). | `[deterministic]` | now | P1 |
| Grouped & whole-home | "Turn off all the lights", "downstairs lights" — resolve area/group targets. | `[deterministic]` | now | P0 |
| Adaptive / circadian lighting | Time-of-day color-temp & brightness curves (deterministic schedule; opt-in per room). | `[deterministic]` | now | P1 |
| Presence-driven lighting | Auto-on/off from occupancy with a grace timeout (rule-engine + occupancy fusion). | `[deterministic]` | needs-hardware | P1 |
| "Movie/dinner/reading" light moods | Named scenes composing multiple lights; see Scenes. | `[deterministic]` | now | P1 |

### 2.4 Climate

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Set temperature / mode / fan | Thermostat control (heat/cool/auto/off, setpoint, fan) capability-checked per device. | `[deterministic]` `[integration:HomeAssistant]` | now | P0 |
| Schedules & setback | Time/occupancy-based setpoint schedules with comfort + eco setbacks. | `[deterministic]` | now | P1 |
| Presence-aware comfort | Pre-condition a room before an occupant arrives (from occupancy + calendar hints via Sarathi). | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P2 |
| Multi-zone / per-room | Coordinate several thermostats/TRVs; per-room targets. | `[deterministic]` | needs-hardware | P2 |
| Cost-aware climate hints | Surface "cooling to 68 costs ~$X/day" via Artha energy pricing — advice, deterministic actuation. | `[integration:Artha]` `[deterministic]` | needs-3rd-party | P2 |

### 2.5 Media

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Play/pause/stop/skip/volume | Core media_player transport control by room/device. | `[deterministic]` `[integration:HomeAssistant]` | now | P1 |
| Source & speaker-group control | Switch source/input; group/ungroup multi-room audio where supported. | `[deterministic]` | now | P2 |
| Play named content | "Play [artist/playlist] in the kitchen" — resolve service + provider (local library / cast). | `[LLM]` `[integration:HomeAssistant]` | needs-3rd-party | P2 |
| TTS announcements | Speak a message to a room/all speakers (Piper via HA media_player or the box). | `[deterministic]` `[integration:HomeAssistant]` | now | P1 |
| Story / narration to a device | Chapters synthesised ahead and streamed to a media_player; voice "stop" (carried from Vatara's BoxNarrator). | `[deterministic]` | now | P2 |
| TV control | LG/webOS control is **deferred to prod HAOS** (aiowebostv bug on the dev Mac box; Hue works). Seam only for now. | `[integration:HomeAssistant]` | needs-3rd-party | P2 |

### 2.6 Covers — blinds, shades, garage, gates

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Open/close/stop/position | Cover control by name/room; set % position where supported (Matter-over-Thread motors, Motionblinds, etc.). | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P1 |
| Sun/temperature-driven shades | Close shades on high sun/heat (illuminance + climate + rule engine) for comfort/energy. | `[deterministic]` | needs-hardware | P2 |
| Garage/gate as high-risk | Opening a garage/gate is treated as **high-risk actuation** (step-up gated) and coordinated with Kavach. | `[deterministic]` | needs-hardware | P1 |

### 2.7 Locks & physical access (boundary with Kavach)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Lock / unlock | Deterministic; **unlock is always high-risk step-up-gated**, never the LLM's call. | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P0 |
| Lock-state readout | "Is the front door locked?" from the projection. | `[deterministic]` | needs-hardware | P1 |
| Auto-lock rules | Lock at night / when everyone leaves (occupancy) — rule-engine, coordinated with Kavach arming. | `[deterministic]` | needs-hardware | P1 |
| Temporary guest access codes | Issue/expire a keypad code for a guest window (see Guest Access); **admin-gated + audited**. | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P2 |
| Security interpretation | Intrusion/tamper/panic/alarm semantics live in **Kavach**, not Kaya — Kaya exposes the raw lock/door capability; Kavach owns life-safety. | `[integration:Kavach]` | now | P0 |

### 2.8 Plugs, switches & appliances

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Smart plug / switch on-off | Control plugs & in-wall switches by name/room. | `[deterministic]` `[integration:HomeAssistant]` | now | P1 |
| Per-plug energy read | Read power/energy where the plug meters it (feeds Energy area). | `[integration:HomeAssistant]` | needs-hardware | P1 |
| Appliance state persistence | Persist logical appliance toggles across restart (carried from Vatara's SqliteKnowledgeGraph behaviour → now Neo4j graph). | `[deterministic]` | now | P2 |
| Runtime/standby detection | Flag always-on standby draws (deterministic threshold analysis) → hand cost to Artha. | `[deterministic]` `[integration:Artha]` | needs-hardware | P2 |

### 2.9 Presence & occupancy

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Home/away per person | Fuse device-tracker signals (phone, router, BLE) into a per-member home/away state. | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P1 |
| Room-level presence | Fuse mmWave (LD2410 / Everything Presence) + BLE (Bermuda/ESPresence via the voice box proxies) into per-room occupancy with confidence. | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P1 |
| Occupancy fusion engine | Deterministic multi-sensor fusion (motion + mmWave + BLE + door events) with hysteresis/grace to kill false vacancy — the input to presence-driven lighting/climate. | `[deterministic]` | needs-hardware | P2 |
| Presence as platform context | Publish "who is where" to Sarathi, Ojas, Abhyasa, Kavach (arming), Artha — a shared context signal. | `[deterministic]` | needs-hardware | P1 |
| Vacation / nobody-home mode | Whole-home state that drives eco setback, auto-lock, and Kavach arming. | `[deterministic]` | now | P1 |

### 2.10 Energy & efficiency (boundary with Artha)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Energy metering ingest | Ingest whole-home + per-circuit/per-plug power/energy (HA Energy dashboard sources, shelly/emporia/etc.). | `[integration:HomeAssistant]` | needs-hardware | P1 |
| Consumption aggregation | Deterministic roll-up by device/room/time; historical store for trends. | `[deterministic]` | needs-hardware | P1 |
| Efficiency detection | Flag lights-left-on, HVAC-with-window-open, standby vampires (deterministic detectors). | `[deterministic]` | needs-hardware | P2 |
| Cost translation | **Kaya meters kWh; Artha owns the $** — Kaya hands usage to Artha, which applies tariff/pricing and coaches. | `[integration:Artha]` | needs-3rd-party | P2 |
| Solar / battery / EV (if present) | Read production/SoC/charging; shift loads to cheap/solar windows (rule-engine). | `[integration:HomeAssistant]` | needs-hardware | P2 |

### 2.11 Scenes

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Named scenes | Compose a set of device target-states ("Movie night", "Good morning") — data-driven, not code. | `[deterministic]` | now | P1 |
| Scene capture | Snapshot the current room state into a new scene. | `[deterministic]` | now | P2 |
| Voice/tool activation | "Activate movie night" via voice or `kaya.activate_scene` tool. | `[LLM]` `[deterministic]` | now | P1 |
| Scene from description | "Make a cozy reading scene in the den" → propose a scene (LLM suggests, human confirms before save). | `[LLM]` | now | P2 |

### 2.12 Routines & automations (Event → Condition → Action)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| ECA rule engine | Deterministic event→condition→action engine (carried from Vatara) with dedup/conflict/validation. | `[deterministic]` | now | P0 |
| Triggers | State, time/sun, presence, sensor-threshold, device-event triggers. | `[deterministic]` | now | P1 |
| Conditions | Time window, occupancy, mode, entity-state, and cross-service context conditions. | `[deterministic]` | now | P1 |
| Actions | Any validated device command, scene, notification, or Sarathi hand-off — all through the guarded boundary. | `[deterministic]` | now | P1 |
| Conflict/dup resolution | Detect contradictory/duplicate rules at author time; contention → ask-by-name. | `[deterministic]` | now | P1 |
| NL rule authoring | "When the child gets home after dark, turn on the hall light" → propose a structured rule; **human confirms before it's saved** (LLM drafts, deterministic validation + persist). | `[LLM]` `[deterministic]` | now | P1 |
| Simulate / dry-run | Explain what a rule *would* do without executing (Bodha-style introspection). | `[deterministic]` | now | P2 |
| Schedule-based routines | Durable scheduler (Quartz/Hangfire) for time-based routines + retention. | `[deterministic]` | now | P1 |

### 2.13 Voice control *(amended 2026-07-11, round 2 — closes G7)*

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Voice actuation end-to-end | STT → typed intent → validated command → actuation → TTS confirm, all through the guarded boundary. **This is Kaya's row**: the device-actuation path for voice-triggered commands, reached by `kaya.*` tools. | `[LLM]` `[deterministic]` `[integration:HomeAssistant]` | now | P0 |
| Wyoming/Assist pipeline ownership *(amended 2026-07-11, round 2 — closes G7)* | **Sarathi** owns the HA conversation integration and the voice-pipeline position — it terminates Wyoming, resolves the speaker, and mints identity ([ADR 0010](../adr/0010-agent-conversational-core.md) §4). **Kaya is a domain service reached by `kaya.*` tools like any other**; Kaya does not terminate Wyoming, resolve speakers, or mint identity. | `[integration:HomeAssistant]` | now | P1 |
| Per-occupant persona/voice | Anik (adult) vs Bonnie (child) persona + Piper voice per member; child-safe persona for a child speaker. | `[deterministic]` `[LLM]` | now | P1 |
| Speaker identification | Resolve *who is speaking* (satellite mapping now; voiceprint ECAPA-TDNN later) to carry verified identity + walls. | `[deterministic]` | needs-hardware | P2 |
| Wake word per person | Per-occupant wake word → member (e.g. the child's "Hey Bonnie Blue"); on-device. | `[integration:HomeAssistant]` | needs-hardware | P2 |
| Mascot/animation cues | LLM `[anim:x]` tag / keyword fallback drives a device screen animation (carried from Vatara mascot engine). | `[LLM]` `[deterministic]` | now | P2 |
| Input safety gate | Child-safety classifier on inbound utterances (lexical floor → Llama Guard → ShieldGemma child policy) before any command runs. | `[LLM]` `[deterministic]` | now | P1 |

### 2.14 Guest & temporary access

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Guest scope | A narrow, time-boxed capability set for a guest identity (e.g. living-room lights + media, no locks/security). | `[deterministic]` | now | P2 |
| Temporary lock codes | Issue/auto-expire keypad codes for a stay window; **admin-gated + audited**; coordinated with Kavach. | `[deterministic]` `[integration:HomeAssistant]` | needs-hardware | P2 |
| Guest voice mode | A guest can control the narrow scope by voice without a member identity. | `[LLM]` `[deterministic]` | now | P2 |
| Auto-revoke | Expire guest grants on window end / departure; audit the full lifecycle. | `[deterministic]` | now | P2 |

### 2.15 Accessibility (blind-first, voice-complete — platform mandate)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Voice-complete parity | Every Kaya capability reachable by voice, not just the UI (platform requirement). | `[LLM]` `[deterministic]` | now | P0 |
| Screen-reader-first UI | Kaya's Akriti surfaces are WCAG 2.2 AA by construction; state announced, not only shown. | `[deterministic]` `[integration:Akriti]` | now | P1 |
| Spoken state summaries | "Everything downstairs is off, front door locked, 71 degrees" — deterministic summary on request. | `[deterministic]` `[LLM]` | now | P1 |
| Confirmations & undo | Clear spoken confirmation of every actuation + "undo that" for reversible actions. | `[deterministic]` | now | P1 |
| Audible alerts routing | Route important state changes to speakers/TTS for a low-vision user. | `[deterministic]` | now | P2 |

### 2.16 Self-awareness, health & notifications

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Device/integration health | Track offline/unavailable/low-battery/stale devices; surface to Bodha + Sarathi. | `[deterministic]` | now | P1 |
| "What can you control?" | Kaya answers its own capability/topology via a `self`-style introspection (feeds Bodha). | `[deterministic]` | now | P1 |
| Proactive notifications | Push meaningful state (garage left open, freezer door ajar, battery dying) via the platform notification path. | `[deterministic]` | now | P1 |
| Battery/maintenance reminders | Deterministic low-battery / filter / maintenance nudges from sensor state + schedule. | `[deterministic]` | needs-hardware | P2 |
| Explain-a-decision | "Why did the hall light turn on?" → trace the triggering rule/event (Saakshi + rule engine). | `[deterministic]` | now | P2 |

---

## 3. Data model sketch (key entities)

Structured system-of-record in **Postgres** (per-user walls where scoped); the **home topology graph** in
**Neo4j** (`IKnowledgeGraph` — House→Room→Device relationships, the cross-domain interconnections); semantic
memory/RAG in **Qdrant**; blobs (scene thumbnails, narration WAVs) in **MinIO**. All behind repository ports.

- **Home / Site** — the premises (tenant-scoped): `HomeId`, `TenantId`, name (`homeName`), timezone, address ref
  (from **Kutumba**). One per install; multi-home is a Kutumba seam.
- **Room / Area** — `RoomId`, name, floor, adjacency; the primary spatial grouping (mirrors HA areas). *Graph node.*
- **Device** — a physical thing: `DeviceId`, `RoomId`, manufacturer/model, HA device id, online/health,
  battery, trust-origin (for VLAN policy). *Graph node, edged to its Room.*
- **Entity** — a controllable/observable endpoint on a device: `EntityId`, HA entity id, `Capability`
  (Light/Climate/Cover/Lock/MediaPlayer/Switch/Sensor/BinarySensor…), supported actions & attribute schema.
- **StateSnapshot / StateEvent** — append-only history: `EntityId`, state, attributes (JSONB), timestamp,
  source, correlation id. Event-sourced projection → current state.
- **DeviceIntent** — the typed, validated parse of a request: verb, target(s), params, risk tier, requested-by
  (verified user), status (parsed/validated/refused/executed), idempotency key.
- **Command / Actuation** — an executed action: `CommandId`, resolved service call, idempotency key, outcome,
  reconciliation result, actor (user + service), correlation id → Saakshi.
- **Scene** — named target-state set: `SceneId`, list of (`EntityId`, desired state/attrs), owner/scope.
- **Rule (ECA)** — `RuleId`, trigger(s), condition(s), action(s), enabled, priority, conflict-group, author,
  provenance. Data-driven, versioned.
- **OccupancySignal / OccupancyState** — per-room fused occupancy: room, present-members, confidence, source
  sensors, timestamp. And **PresenceState** — per-member home/away.
- **EnergyReading** — `EntityId`/circuit, power (W), energy (kWh), interval, timestamp → aggregates (device/room/
  time). (Cost is Artha's.)
- **GuestGrant** — `GrantId`, guest identity, capability scope, window (start/end), lock-code ref, status,
  full audit trail.
- **Member / Persona / Voice ref** — read from **Kutumba**; Kaya holds only the assistant persona/voice binding
  per member for voice replies (Anik/Bonnie), not PII.
- **NotificationEvent** — `EventId`, kind, severity, target member(s), delivered-via, timestamp.

All entities carry `TenantId` + `OwnerId`/scope + provenance (`CreatedBy{User,Service}` / `ModifiedBy…`) via the
`Manas.Kernel` base entity; the audit interceptor records every change; the per-user wall filter applies
automatically.

---

## 4. Integrations needed

- **Home Assistant** (`[integration:HomeAssistant]`) — the one path to every device. WebSocket API for the live
  event stream + entity/device/area registries; `call_service` for actuation; long-lived access token; local
  network only. Carried forward as `HomeAssistantProvider` behind `IKayaProvider`. **Non-negotiable dependency.**
- **Matter / Thread** — Home Assistant as Matter controller + Thread Border Router for local, cloud-free device
  connectivity (lights, climate, covers, locks, plugs, contact/motion/illuminance sensors). Preferred transport
  for new devices (local by design). *needs-hardware.*
- **Zigbee / Z-Wave (via HA — ZHA/Z2M)** — for the large installed base of non-Matter sensors, plugs, contacts.
  *needs-hardware.*
- **Presence hardware** — mmWave (LD2410 / Everything Presence Lite / Apollo MTR-1 via ESPHome) + BLE room
  presence (the voice box Bluetooth proxies + Bermuda/ESPresence). *needs-hardware.*
- **Energy metering** — whole-home/circuit (Emporia Vue / Shelly EM) + per-plug metering, surfaced via HA.
  *needs-hardware.*
- **Voice stack (local)** — Wyoming protocol to HA Assist; whisper.cpp (STT), Piper (TTS), openWakeWord/local
  wake; the on-device the voice box satellite/box. All local, $0. Carried from Vatara.
- **Medha (LLM gateway)** — the only path to the local models (`qwen3:30b-a3b` responder + safety models +
  `nomic-embed-text`), with QoS so the always-listening voice path outranks batch. Kaya never calls Ollama
  directly.
- **Akriti** — shared accessible UI components for Kaya's dashboards/controls.
- **Dwarpal** — verified identity + risk-based step-up for high-risk actuation (unlock/disarm/garage). The
  deterministic security boundary.
- **Kutumba** — household, home, address, plus members/roles; Kaya reads, never owns, the who's-who.
- **Saakshi** — correlation-id'd interaction ledger + audit + traces for every command and device event.
- **Frigate / cameras** — local object detection; **camera streams and security semantics belong to Kavach** —
  Kaya only references, does not own.

---

## 5. Interconnections with other Manas services

- **Kavach (security) — the closest neighbour, sharp boundary.** Kaya exposes raw device *capability* (lock,
  door/window contact, garage cover, siren-as-device, motion sensor). **Kavach owns the life-safety
  interpretation**: arm/disarm/panic/intrusion/tamper are **deterministic, step-up-gated, and never routed
  through Kaya's LLM**. Kaya's auto-lock and vacation mode coordinate with Kavach arming; garage/gate opens are
  jointly high-risk. Presence flows Kaya→Kavach for auto-arm-when-empty. Rule of thumb: *Kaya makes it
  comfortable; Kavach keeps it safe.*
- **Artha (finance / energy $).** Kaya meters **kWh and device runtime**; Artha applies **tariff/pricing and
  coaches on cost**. Kaya hands raw energy usage + efficiency findings (vampire draws, HVAC-with-window-open);
  Artha turns them into dollars and savings advice. Kaya never computes money.
- **Ojas (health).** Presence/occupancy + environmental context (room temp, air quality, sleep-room activity)
  feed wellness context; conversely Ojas context (e.g. someone resting) can bias Kaya comfort. Kaya actuates,
  Ojas reasons about wellbeing — Kaya never gives health advice.
- **Abhyasa (school).** Presence ("the child is home", "in her room") and a "study/focus scene" (lights + quiet media)
  support study routines; Abhyasa provides the schedule/context, Kaya provides the room.
- **Sarathi (the Agent, orchestrator).** Sarathi is the MCP client; Kaya publishes governed `kaya.*` tools wrapping its
  application services. Sarathi resolves *who is speaking*, carries verified identity into every Kaya call, and
  Kaya enforces walls/step-up at its own boundary — Sarathi is a deputy, **never a super-user**, and its LLM is
  **never** the access or actuation decision.
- **Medha / Saakshi / Bodha / Akriti / Dwarpal / Kutumba (foundation).** Kaya reaches models only via Medha;
  every command is traced/audited via Saakshi; Kaya's topology & health feed Bodha's self-model; UI is Akriti;
  identity/step-up is Dwarpal; people, home, and rooms source-of-truth is Kutumba (Kaya projects rooms/devices into the
  graph but reads members from Kutumba).

---

## 6. Explicitly out of scope (and why)

- **Security / alarm logic (arm/disarm/intrusion/panic/monitoring).** Belongs to **Kavach** — life-safety is a
  separate, deterministic, step-up-gated service with its own fail-safe posture and self-health. Kaya only
  exposes the raw device capability. Mixing them would blur the safety boundary.
- **Camera video, recording, and object detection semantics.** **Kavach + Frigate** own camera streams and the
  security interpretation; cameras are egress-blocked/local-only. Kaya does not process video.
- **Money / cost calculation / tariffs.** **Artha** owns dollars. Kaya stops at kWh and runtime. Keeps the
  finance boundary clean and the walls physical.
- **Health advice / diagnosis.** **Ojas.** Kaya actuates the physical room; it never interprets vitals or gives
  wellness guidance.
- **Cloud-dependent device control.** Anything that *requires* a vendor cloud round-trip is out — local-first is
  a hard requirement. Cloud-only gadgets are either not adopted or bridged locally; untrusted-origin gear is
  VLAN-isolated + egress-blocked and reached only through HA.
- **The LLM as the actuation or access decision.** Structurally forbidden: the model understands intent and
  phrases replies; **deterministic C# decides and gates**. High-risk actions never depend on model judgment.
- **General-purpose "smart home AI" that guesses.** No probabilistic auto-actuation on a hunch. Automations are
  explicit, authored, validated ECA rules a human confirmed — not emergent model behaviour.
- **Building the physical network/VLAN/firewall hardening itself.** That's platform/ops (`[SEC-*]`); Kaya assumes
  and depends on it but doesn't implement the network layer.
- **Multi-tenant billing / productisation.** The tenant seam is present (`TenantId` from Kutumba) but
  multi-home/billing is deferred per the root ADR.
- **Replacing Home Assistant's device drivers.** Kaya does **not** re-implement integrations/protocols; it stands
  on HA's driver ecosystem and adds the guarded, deterministic, audited, per-user-walled brain on top.
- **Speculative hardware we can't get data from.** No features invented on sensors/telemetry the home doesn't
  actually have — capability is derived from the live HA registry, not assumed.

---

*Sources for the ecosystem grounding: Home Assistant WebSocket/REST APIs (developers.home-assistant.io),
HA Matter & Thread integration + 2026.1 Matter/Thread improvements (home-assistant.io, matteralpha.com),
room-level presence via mmWave + Bermuda/ESPresence (smarthomescene.com, derekseaman.com). All tailored to the
Manas local-first / deterministic / privacy constraints in ADR 0001.*
