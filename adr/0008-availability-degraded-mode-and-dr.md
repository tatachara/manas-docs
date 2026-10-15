# ADR 0008 — Availability, Degraded-Mode, DR & Incident Response

- **Status:** Accepted (Ananth, 2026-07-12) — green-lit for build; see CLAUDE.md rule 11
- **Date:** 2026-07-11
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** [`0001`](0001-manas-platform-architecture.md) §14, the platform's deployment topology and threat model documentation; resolves design-review **C1, C2, C3, C5, C7, C8, B4, D5, D11, E15** / risks **#21, #22, #23, #25, #4**.

> ADR 0001 answered *durability* (RAID/ZFS/UPS/backup keep the **data** alive). It never answered
> *availability*: what protects the **family** during the days the one box is dead, who tells them it died,
> and what a life-safety alarm built on GC'd .NET inside Docker behind Home Assistant can actually promise.
> This ADR draws the honest availability envelope for the single-box Core, adds a **standalone fail-safe
> security core** and an **external dead-man's-switch** so that box death is loud and never disarms the house,
> hardens the running box against DMA/RAM extraction, designs DR across the polyglot stores, locks down the
> Home Assistant bypass path, pins trusted time to monotonic clocks, orders the encrypted cold boot so a locked
> box is still debuggable, and writes the incident runbook.

---

## 1. Context

Manas runs on **one** machine — the home server (128 GB unified memory, firmware TPM, OCuLink/USB4).
Everything of consequence lives there: **Kavach** (the self-monitored security system that *replaces ADT* —
arm/disarm/panic, sirens, Frigate cameras), **Artha** (finance), **Ojas** (health), **Kaya** (devices/voice
via Home Assistant), **Saakshi** (observability/audit), **Bodha** (self-awareness), **Pramana** (trust/vault/CA),
**Sangam** (per-user OAuth). Cameras, sensors and sirens reach the box **only** through Home Assistant. The only
cloud is a **thin BLIND relay** (a managed tunnel technology that carries an encrypted stream it cannot read) plus
opaque APNs push.

ADR 0001 §14 and the platform's deployment topology documentation covered **durability** —
RAID/ZFS + UPS + encrypted off-site backup + tested restore. That is necessary and not sufficient. The
design-review flagged a whole **availability/ops theme the design under-served**:

- **C1 / #22 — box DEATH = life-safety GONE.** Backup restores *data*, over **days**. For those days Kavach —
  the thing that replaced ADT — does not exist. That is a **physical-safety** gap, not a data gap.
- **C2 / #23 — the monitor runs on the machine it monitors.** Saakshi/Bodha/alerting die *with* the box, so the
  failure is **silent** — nobody is told the house is unprotected (the Vatara "stuck, stops working" failure).
- **C3 / #21 — a running seized box has the DEK in RAM.** Cold-boot remanence / DMA over OCuLink/USB4 defeats
  "stolen box = ciphertext." (Key model is [`0002`](0002-key-hierarchy-and-unlock.md); the host mitigations
  belong here.)
- **B4 / #4 — DR across 5 heterogeneous stores isn't designed**, and **Neo4j Community has no online backup**.
- **C5 / #25 — Home Assistant is a parallel actuation path** with a long-lived token that **bypasses** Manas
  step-up; a compromised HA could disarm/unlock or forge an "all-clear" to Kavach.
- **C7 — no trusted-time / monotonic-clock story**, which Kavach entry/exit delays, TOTP, cert validity and
  audit ordering all depend on.
- **C8 — cold-boot dependency ordering is unspecified beyond Kavach's TPM path, and an encrypted box is
  undebuggable** (Saakshi logs are ciphertext before unlock).
- **D5 — composability is stated as absolute, but there are HARD deps** (Kavach is useless without Kaya's device
  path).
- **D11 — no incident-response runbook.**
- **E15 — Kavach isn't alarm-panel-grade** (GC pauses, Docker, HA in the path); its latency/availability
  envelope is unstated and the siren isn't fail-safe-independent of .NET.

The unifying principle of this ADR: **a home security system may not have its physical-safety guarantee gated on
a single general-purpose computer being healthy.** We do not fix this by buying a second server (that just moves
the single point) — we add a **dumb, independent, commodity fail-safe** below the smart layer, and we make box
death **loud**.

---

## 2. Standalone fail-safe security core (box-death survival)

**Decision.** Kavach is a **two-layer** system, not one:

- **Layer 0 — the standalone core (dumb, independent, always-on).** Commodity alarm hardware that can **arm,
  disarm, and ALARM without the box, without Docker, without .NET, and without Home Assistant**:
  - A **dedicated alarm panel / supervised siren relay** (a self-contained board or a hardwired
    siren driven by a supervised relay) with door/window contacts and at least one motion sensor wired to it
    **directly**, plus a **hardwired/battery-backed siren** on its **own power** (its own small battery, not the
    box's UPS).
  - A **local physical keypad** at the entry door for arm/disarm using a **PIN verified on the panel itself**.
  - The siren-fire path is **hardware-latching**: a tripped zone in the armed state fires the siren via the
    panel's own logic. **The .NET process is not in this loop.** If Kavach (or the whole box) is dead, an
    intrusion still sounds the local siren.
- **Layer 1 — the smart Kavach (.NET, on the box).** Everything that makes it *good*: camera corroboration
  (Frigate), voice arm/disarm, phone notifications, geofencing, schedules, entry/exit UX, audit, the
  self-monitoring. It **observes and enriches** Layer 0 and can arm/disarm it over a **local, wired** control
  line — but it is **not required** for Layer 0 to protect the house.

**Warm-spare for state (optional, recommended).** Kavach's arm-state machine is small; its authoritative
arm/disarm state is mirrored (a) into Layer 0 (the panel holds armed/disarmed itself) and (b) to a tiny
persisted `kavach.state` file on ZFS. On box restart Kavach **reconciles** to the panel's actual state rather
than assuming disarmed. A full second-box hot spare is **explicitly rejected** — it re-introduces the SPOF one
level up and costs more than the commodity panel that already closes the gap.

**Honest RTO / what survives box death.**

| Box state | Alarm can fire? | Arm/disarm? | Cameras/notify/voice? | Finance/health? |
|-----------|-----------------|-------------|-----------------------|-----------------|
| Healthy | Yes (both layers) | Voice / app / keypad | Yes | After admin login |
| Kavach .NET crashed, box up | **Yes — Layer 0** | **Local keypad** | Degraded (no camera corroboration/notify from Kavach) | Unaffected |
| **Whole box dead** | **Yes — Layer 0 siren** | **Local keypad only** | **No** (relay dead-man's-switch alerts phones — §3) | No (data safe at rest, unreachable until restore) |
| Restore in progress | Yes — Layer 0 | Local keypad | Comes back with the box | After restore + login |

- **Box-death RTO (smart features):** **hours to days** (bare-metal reprovision from the pinned Release image +
  restore from backup — §5). Stated plainly to the family: *if the computer dies, the alarm still rings and the
  keypad still works, but there are no phone alerts, no cameras, and no voice until it's rebuilt.*
- **Security guarantee that survives box death:** **local intrusion detection + local siren + local keypad
  arm/disarm.** What is lost: remote notification, camera verification, panic-from-phone, and any Manas step-up.

This is the direct answer to **C1** and **E15**: the life-safety promise is delivered by dumb commodity hardware
on its own power; the .NET layer is a smart enhancement that is *allowed* to fail.

---

## 3. External dead-man's-switch (the only monitor that survives box death)

**Problem (C2/#23):** Saakshi and Bodha run **on** the box. If the box dies they die silently — the exact
"stops working and nobody notices" failure. On-box monitoring **cannot** report its own host's death.

**Decision — an opaque liveness heartbeat to the blind relay.**

- The box emits a periodic **"alive" beacon** to the thin cloud relay: an **opaque, authenticated token**
  (`box-id`, monotonic counter, HMAC) and **nothing else** — no house data, no state, no telemetry. The relay
  learns exactly one bit: *box-id was alive at time T.* This fits the **blind-relay** model perfectly (it
  already can't read the tunnel; the heartbeat is deliberately contentless).
- **Cadence:** beacon every **60 s**.
- **Alert threshold:** the relay's dead-man timer fires when **no beacon arrives for a grace window of 5 min**
  (≈ 5 missed beacons), then it pushes an **APNs alert to the family's phones**: *"Manas Core is not
  responding."* The relay re-notifies on a back-off (e.g. at 5 min, 15 min, 1 h) until the beacon resumes or a
  human acks.
- **False-positive story (network blips):** the 5-min grace absorbs Wi-Fi/ISP hiccups and the box's own reboots.
  Two independent egress paths are used when available (home broadband **and** an LTE/failover path if present)
  so a single-ISP outage doesn't masquerade as box death; the alert copy names the ambiguity (*"...this may be a
  network outage"*). The family can ack ("I know, it's the internet") to silence the back-off.
- **Layering:** the on-box Bodha self-checks (service liveness, clock health §7, disk, model warmth) remain and
  feed Saakshi for *rich* diagnostics **while the box lives**; the dead-man's-switch is the **only** signal that
  survives the box's death, and it is intentionally minimal. Bodha also **feeds** the beacon: if Bodha detects a
  critical internal fault it can **stop emitting** the beacon (or flip a health bit) to trip the same alert
  path, so "box alive but Kavach wedged" also pages the family.

This is the single outside observer, and it costs the blind relay nothing it doesn't already do.

---

## 4. DMA / RAM / tamper mitigations (residual honesty)

**Problem (C3/#21):** a **running, unlocked** box holds the tier DEKs (`LIFE`, `MTRUST`, and post-login
`SENS`/`GEN` — see [`0002`](0002-key-hierarchy-and-unlock.md) §10) in RAM. Physical access defeats
"stolen box = ciphertext": **cold-boot RAM remanence** and **DMA** over OCuLink/USB4/Thunderbolt.

**Host mitigations (all on the home server, no special hardware):**

1. **IOMMU on + kernel DMA protection.** Boot with the AMD IOMMU enabled and Linux **kernel DMA protection**
   engaged so external DMA-capable ports cannot freely read system RAM.
2. **Disable / authorize-only the DMA ports.** OCuLink and USB4/Thunderbolt DMA set to **disabled or
   authorized-only** (allowlist known-good internal devices; reject hot-plugged DMA masters). The home server's
   OCuLink is for our own NVMe/eGPU use — pin it to the known device, refuse others.
3. **Memory encryption if available — VERIFY.** The home server's CPU is expected to support AMD
   **SME/TSME** (transparent system-memory encryption). If present, **enable TSME** so RAM contents are
   encrypted against a physical bus/cold-boot read. **This must be verified on the actual silicon/firmware**
   (open question §11) — do not assume; if unsupported, the residual below stands undiminished.
4. **TPM-seal machine-trust to PCR boot state.** `LIFE`/`MTRUST` unseal (0002 §3) is bound to **boot-state
   PCRs**; an altered bootloader/kernel/firmware **won't unseal**. Tamper or an evil-maid boot therefore fails
   to bring up the auto-unlock tiers rather than silently handing keys to a modified OS.
5. **Physical tamper detection → auto-lock / zeroize.** A chassis-intrusion / case-open / motion signal (wired
   into Layer 0 or a GPIO) triggers Manas to **re-lock `SENS`/`GEN` and zeroize their in-RAM DEKs** immediately
   (0002 §9 zeroization), shrinking the window an opened box exposes finance/health.
6. **Idle re-lock.** `SENS`/`GEN` DEKs are **zeroized on idle timeout**, minimizing how long the sensitive tiers
   sit unsealed in RAM (Kavach's `LIFE` stays up by design — that's the availability trade in 0002).

**Residual — stated honestly.** A **powered, unlocked** box is inherently attackable by an adversary with
**physical presence** and time; no at-rest key scheme defends live RAM completely. TSME + IOMMU + port-DMA-off
raise the bar to specialised hardware attacks; PCR-sealing defeats altered-boot; tamper-zeroize shrinks the
sensitive window. **We accept the residual for a home appliance** — the threat model is burglary/theft, not a
funded lab with the box powered and cabled — and we document it (carried from [`0002`](0002-key-hierarchy-and-unlock.md)
§10 and the threat model documentation §5). Note the deliberate trade: `LIFE`/`MTRUST`
auto-unseal means a stolen *powered* box exposes alarm + mesh/CA material (not finance/health) — accepted so the
alarm survives an unattended reboot.

---

## 5. DR across the polyglot stores (B4/#4) *(amended 2026-07-11, round 2 — closes S12)*

**Problem.** Manas spans heterogeneous stores — the **LOCKED** stack per [`0001`](0001-manas-platform-architecture.md)
§6: **PostgreSQL** (structured system-of-record), **Neo4j** (graph), **Qdrant** (vectors), the **S3-blob store**
(unstructured files — the MinIO-successor decision), **Redis** (cache/streams), the **Pramana/OpenBao vault**, plus
**Frigate** media. **SQLite is dev/test only**
(ADR 0001 §6) — it holds no production data and is out of scope for this DR design; earlier drafts of this ADR
said "SQLite (operational)/`operational.db`," which was wrong and is corrected here. A per-engine dump strategy
is six fragile pipelines that can drift out of mutual consistency, and **Neo4j Community has NO online/hot
backup** (that's an Enterprise feature). A logical dump of a live Neo4j Community DB is not point-in-time
consistent.

**Decision — filesystem-consistent ZFS snapshots as the single DR primitive.**

- All data dirs live on **ZFS**. DR takes an **atomic `zfs snapshot`** across the dataset holding every store,
  giving a **single, filesystem-consistent, point-in-time** image of *all* engines at once — no per-engine
  coordination, and it is the **only** correct way to back up Neo4j Community (snapshot the files while briefly
  quiescing writes / using each engine's checkpoint hint). Kavach `LIFE` state and the Pramana vault are in the
  same snapshot, so a restore is mutually consistent across security + keys + data.
- **Why a ZFS snapshot of a live PostgreSQL is correct (not just convenient).** A ZFS snapshot is a **crash-consistent**
  image — it captures the on-disk bytes at one instant, the same as if the box had lost power at that instant.
  PostgreSQL is designed for exactly this: on restore, PostgreSQL replays its **write-ahead log (WAL)** from the
  last checkpoint, the identical recovery path it runs after any power-cut, and comes up transactionally
  consistent. This equivalence holds **only if the data directory and its WAL live in the same atomic snapshot**
  — i.e. `PGDATA` (including `pg_wal`) must sit inside the **one** ZFS dataset (or a set of datasets snapshotted
  **together**, atomically) that this DR primitive snapshots; a design that splits WAL onto a separately-scheduled
  dataset breaks the guarantee and must not be built. Under a shared single Postgres instance with a logical DB
  per service, this one argument covers **every** Postgres-backed
  service (Artha, Ojas, Saakshi's relational tables, Dwarpal's store, Kutumba) with no per-service special-casing.
  **Neo4j Community remains the reason ZFS snapshots are mandatory** rather than one option among several (§5
  below, unchanged) — Postgres's WAL-replay argument is the *additional* correctness case this amendment closes,
  not a replacement for the Neo4j argument.
- **Redis is best-effort in-snapshot, and that is acceptable.** Redis is cache + event-bus (Streams), not a
  system-of-record. Its RDB/AOF files, if present, are captured in the same atomic snapshot as everything else,
  but Redis is **not** given special quiescing treatment — a restore may land Redis with a stale or empty cache
  and a Streams backlog that doesn't perfectly match Postgres's restored point. This is fine because caches
  are rebuildable and stream **consumers are idempotent** (ADR 0005 §3.2's idempotency-key + at-least-once
  design already tolerates redelivery and gaps); document this loss semantics rather than engineering around
  it.
- Snapshots are `zfs send` → **client-side encrypted** (keys stay home, 0002) → **off-site**. Restore =
  reprovision the box from the pinned Release image, `zfs receive`, unlock, reconcile Kavach to Layer 0.
- **Backup ↔ Release-version pinning.** Every snapshot is **tagged with the Release BoM** it was taken under
  (see the platform's release-and-updates documentation). A restore **must**
  land on a **compatible Release** — the restore tool refuses to `receive` a snapshot whose schema/BoM is
  incompatible with the target image, and the off-site keeps enough Release images to restore any retained
  snapshot. This prevents "restored data, incompatible code."
- **Restore-drill cadence.** Backups that are never restored are theatre. A **quarterly restore drill** brings
  the latest snapshot up on a scratch target (or a VM), verifies Kavach arms, `SENS`/`GEN` unlock, Neo4j graph
  loads, and Qdrant answers — **test restores, not just backups.** A drill failure is a Sev-2 incident.

**RPO / RTO per data class:**

| Data class (tier) | Store(s) | RPO (max data loss) | RTO (time to service) | Backup mechanism |
|-------------------|----------|---------------------|-----------------------|------------------|
| **Security state** (`LIFE`) | Kavach state, panel/zone config (Postgres) | **near-zero** (arm/disarm mirrored to Layer 0 panel instantly; ZFS snapshot on every state change) | Layer 0 = **0** (never lost); smart layer = hours | Layer 0 hardware + continuous snapshot |
| **Finance / health** (`SENS`) | Artha, Ojas (Postgres, logical DB per service) | **minutes** (frequent snapshots; WAL-replay consistent, see above) | Hours (needs admin unlock post-restore) | ZFS snapshot |
| **Operational / audit** (`GEN`) | Postgres (config, Saakshi audit relational tables), Neo4j, Qdrant | **minutes–1 h** | Hours | ZFS snapshot |
| **Cache / streams** | Redis (RDB/AOF, best-effort) | **best-effort — may lose the cache and an in-flight Streams window** (acceptable: rebuildable cache, idempotent consumers, ADR 0005 §3.2) | Minutes (cache warms, streams resume) | ZFS snapshot (no special quiescing) |
| **Media** (Frigate clips) | Frigate storage | **hours** (large, least critical; retention-bounded) | Best-effort | ZFS snapshot (lower cadence) |
| **Keys / trust** (`MTRUST`) | Pramana vault | near-zero | Hours (unseal chain, §8) | ZFS snapshot + offline recovery codes (0002 §3) |

Snapshot cadence is **class-weighted**: security/finance/health frequent; media coarse. The Neo4j-Community
constraint is the reason ZFS snapshots are **mandatory** rather than one option among several; the Postgres
WAL-replay argument above is why that same primitive is also **correct**, not just convenient, for the
system-of-record.

---

## 6. Home Assistant lockdown (C5/#25)

**Problem.** HA is the **only** device path (good — one guarded boundary), but HA has **its own** UI, automations
and long-lived token. Anyone with that token, or HA's dashboard, can **disarm/unlock directly**, **bypassing
Manas step-up**; a compromised HA could also **forge sensor state** ("all-clear") to Kavach.

**Decision — treat HA as a locked-down appliance, not a trusted peer.**

1. **Manas's service account is the ONLY actor.** HA is provisioned with a **single** Manas service user; **no
   other HA users, dashboards, or `automation:`/`script:` entities touch high-risk domains** (`alarm_control_panel`,
   `lock`). Those domains are controlled **exclusively** by Kavach via the service account. HA's own UI for
   locks/alarm is **removed/disabled**; HA is a **transport + sensor bus**, not a control surface.
2. **Token in the vault + rotation + network restriction.** The HA long-lived token lives in the **Pramana
   vault** (the internal trust design documentation), never in a file/env; it is
   **rotated** on a schedule and on any suspicion; HA is reachable **only** from the Manas Core over the
   segmented VLAN (carried from Vatara) — **egress-blocked**, no inbound from anywhere but the box.
3. **Sign/validate critical sensor state.** Life-safety-critical sensor readings that Kavach acts on are
   **not trusted from a single HA JSON field.** Where a sensor supports it, its state is **corroborated**; the
   Kavach↔HA channel is **authenticated** (mTLS from the Pramana CA, ADR 0003), so a rogue on-VLAN device can't
   inject state. A "disarm"/"all-clear" is **never** accepted purely on HA's word.
4. **Kavach must not trust a single HA heartbeat.** For any life-safety decision that turns the alarm **off** or
   declares safety, Kavach **corroborates across independent paths** — the Layer 0 panel's own zone state, a
   second sensor, and (where relevant) camera confirmation — before trusting it. A forged HA "all-clear" that
   contradicts the Layer 0 panel is **rejected** and raises a tamper alert. Disarm remains **deterministic and
   stepped-up in Manas** (PIN/step-up), never a bare HA call.

This closes the parallel-actuation bypass: HA can *sense* and *transport*, but it cannot *decide* to disarm or
forge safety.

---

## 7. Trusted time & monotonic clocks (C7)

**Problem.** Nothing pins time. Audit ordering, TOTP step-up windows, cert validity, and — most dangerously —
**Kavach entry/exit delays** all depend on it. A **wall-clock jump** (NTP correction, manual set, drift) must
**never skip a security countdown.**

**Decision.**

- **Authenticated time — NTS.** The box syncs via **NTS (Network Time Security, authenticated NTP)** so an
  attacker on the network can't shift wall-clock to expire certs or open TOTP windows. Multiple NTS sources;
  disagreement is a Bodha alarm.
- **Monotonic clocks for ALL countdown/delay logic — non-negotiable.** Every duration — Kavach **entry/exit
  delays**, siren auto-cutoff, TOTP validity, cert-validity *countdowns*, lockout timers, idle re-lock (§4) —
  is measured against a **monotonic clock** (`Stopwatch` / `CLOCK_MONOTONIC`), **never** wall-clock subtraction.
  A monotonic clock cannot jump or run backward, so **a clock correction can never shorten or skip an exit
  delay**. Wall-clock is used **only** for *displaying* and *timestamping* (audit records carry wall-clock **and**
  a monotonic/sequence stamp so audit order survives a wall-clock adjustment).
- **Clock health is a self-check.** Bodha/Kavach continuously check: monotonic-vs-wall drift, NTS reachability,
  and **step detection** (a sudden wall-clock jump). A failed clock check degrades gracefully (keep the last
  good offset, alarm) and **never** causes a security delay to be skipped. On boot, if time is not yet
  trustworthy, security **countdowns still run on the monotonic clock** (they don't need wall-clock at all);
  only wall-clock-dependent features (schedules, cert *expiry* checks) wait for NTS.

The load-bearing rule, asserted as a test (§10): **advancing wall-clock time mid-countdown does not shorten a
Kavach exit delay.**

---

## 8. Cold-boot ordering & debuggability (C8) *(amended 2026-07-11, round 2 — closes S1/S13)*

**Problem.** Only Kavach's TPM path was specified, omitting **Dwarpal** and **Kutumba** entirely (round-2 finding
**S1**: Dwarpal has no key tier and no boot slot, risking either an unbootable circular deadlock — unlock needs
login needs Dwarpal needs an unlocked store — or Dwarpal's credential store/signing keys silently landing in the
auto-unseal blast radius undecided; **S13**: Kutumba, the source of `guardianOf`/roles for **every** may-see
call, likewise had no key tier, boot slot, or dependency-manifest row). And because Saakshi logs are
**encrypted**, a boot fault is **undebuggable** — you can't read the logs that tell you why boot failed.

**Decision — an explicit, health-gated startup order:**

```
1.  Base OS + systemd + Docker              (unencrypted; brings up the runtime)
2.  Pramana machine-trust UNSEAL            (TPM auto-unseal LIFE + MTRUST + Dwarpal signing
                                              keys + Dwarpal store slice,
                                              per the ADR 0002 amendments — 0002 §2/§3)
3.  Core datastores up                      (Postgres, Redis ENGINES start; the sealed
                                              SENS/GEN tier DATA inside them stays ciphertext
                                              until step 9 — an up engine is not the same as
                                              readable data; see §5's crash-consistency note)
4.  mTLS mesh + internal CA up (Pramana)    (services can authenticate each other — ADR 0003)
5.  Dwarpal (Zitadel) up                    (its signing keys + store slice were unsealed at
                                              step 2, Postgres is up at step 3 — LOGIN BECOMES
                                              POSSIBLE here; nobody has logged in yet)
6.  Kavach + Layer 0 reconcile              (alarm live ASAP; reconcile to the panel's real
                                              state — Layer 0 depends on neither Dwarpal nor Kaya)
7.  Kaya / HA connection                    (device path, sensor bus — GATED, see below)
8.  Admin login (human)                     ── gate ──
9.  Crypto UNLOCK sensitive tiers           (password unwraps SENS/GEN — 0002)
10. Finance/health/audit DBs readable       (Artha, Ojas, Saakshi decrypt)
```

**Step 7 branches on the S2 decision (owned by ADR 0002, §12 open questions — not decided by this ADR).** Kaya's
connection config and the HA long-lived token are `GEN`-tier data (password-sealed) unless ADR 0002 carves out a
narrow `GEN-BOOT` sub-tier for them — **★ DECIDED: Option B / Branch B (Ananth, 2026-07-11; ADR 0002 §2.3).
Branch B below is the operative design; Branch A is retained as the documented alternative behind 0002's
tripwire.** *(The following pre-decision text is kept for the record:)* **Both branches must be kept
live in this doc until Ananth decides:**

- **Branch A — a `GEN-BOOT` sub-tier exists** (TPM-auto-unsealed at step 2, alongside `LIFE`/`MTRUST`/Dwarpal/
  Kutumba). Step 7 runs **here**, pre-login, between Kavach-Layer-0-reconcile (6) and admin login (8): Kaya/HA
  comes up, so Kavach Layer 1's HARD dependency on Kaya (§9) is satisfiable **before** anyone logs in — cameras,
  phone notify, and voice arm/disarm work pre-login. **Trade-off, stated honestly:** a stolen *powered* box now
  also exposes the HA token and device/entity registry, not only alarm and mesh material — an enlarged theft
  blast radius that ADR 0002 must accept and document explicitly if it picks this branch.
- **Branch B — no `GEN-BOOT` sub-tier; Kaya stays fully `GEN`-sealed.** Step 7 **moves to after step 8**, running
  alongside steps 9–10 once the admin password unwraps `GEN`. **Pre-login, Kavach Layer 1 is honestly down** —
  no cameras, no notify, no voice, no HA-mediated sensors — and **only Layer 0** (§2) protects the house between
  step 6 and step 8. This is not a new regression; it is naming out loud what "Kaya is HARD-gated on login"
  costs, which the pre-amendment §8/§9 text asserted was fine while its own dependency table (§9) made it
  self-contradictory.

Both branches leave the **Layer-0 guarantee untouched and absolute**: Layer 0 alarms and the local keypad
arm/disarm from step 6 onward regardless of which branch ADR 0002 picks, because Layer 0 depends on neither
Dwarpal nor Kaya (§9). This ADR does not choose between A and B.

- **Each step is health-gated:** a step that fails to become healthy **halts the dependent chain with a typed
  error** rather than half-starting (feeds §9's startup validation and the dead-man's-switch).
- **What works while LOCKED (pre-step-8), both S2 branches:**

  | Capability | Branch A (`GEN-BOOT` exists) | Branch B (no `GEN-BOOT`) |
  |---|---|---|
  | Alarm (Layer 0) + local keypad arm/disarm | Yes — from step 6 | Yes — from step 6 |
  | Login | Possible from step 5; performed at step 8 (gate) | Possible from step 5; performed at step 8 (gate) |
  | Kavach Layer 1 (cameras/notify/voice arm-disarm) | Yes — from step 7 | **No** — waits for step 8+ |
  | Kaya devices / sensor bus | Yes — from step 7 | **No** — waits for step 8+ |
  | Finance/health/audit | **No** — sealed until step 9/10 | **No** — sealed until step 9/10 |

  This is the intended hybrid posture from [`0002`](0002-key-hierarchy-and-unlock.md): the house is protected
  before anyone logs in in **both** branches, private data waits in **both** branches, and the only thing the S2
  decision changes is whether the *smart* Kavach/Kaya slice is available pre-login.

**Debuggability of an encrypted/locked box:**

- **A minimal UNENCRYPTED boot/system log** records **service/health/boot state only** — which step reached
  healthy, which failed, timestamps, error codes — and carries **no sensitive payload** (no user data, no
  secrets, no request content). This lets you diagnose *why boot stalled* on a box whose Saakshi log is still
  ciphertext. It is the one deliberately-plaintext log, scoped to boot/ops telemetry.
- **An admin-gated log-decryption tool.** Once an admin authenticates, a CLI/console tool decrypts the Saakshi
  audit log for a time window for inspection — the encrypted logs are readable **after** unlock, never before,
  and only by an admin. This is the bridge between "logs are encrypted for privacy" and "we still need to read
  them to debug."

---

## 9. Dependency manifest (D5) *(amended 2026-07-11, round 2 — closes S2/S13)*

**Problem.** Composability is stated as **absolute** ("any service toggleable"), but there are **hard** deps:
**Kavach's Layer 1 without Kaya's device path can't reach a siren/sensor** — it's not "degraded," it's
non-functional and dangerous to pretend otherwise. The pre-amendment table also stated Kavach's HARD dep on Kaya
without qualifying it to Layer 1 only, which **contradicted §8's own "what works while LOCKED" claim** that
Layer 0 alarms pre-login (round-2 finding **S2**) — Layer 0 does not and must never depend on Kaya. Separately,
**Kutumba** — the source of `guardianOf` and role data every may-see check reads — and **Dwarpal** had no row at
all (finding **S13** and part of **S1**), even though a Kutumba or Dwarpal outage is fail-closed platform-wide.

**Decision — every service declares a dependency manifest; startup validation refuses invalid combos.**

- Each service declares **HARD** vs **OPTIONAL** dependencies (extends the `Services:<Name>:Enabled` model from
  [`0001`](0001-manas-platform-architecture.md)):

  | Service | HARD deps | OPTIONAL deps (degrade, don't fail) |
  |---------|-----------|-------------------------------------|
  | **Kavach** | **Layer 0 core + Pramana** (both layers, always); **Kaya device path — HARD for Layer 1 only** (Layer 0 never depends on Kaya; Layer 1's pre-login availability follows the ADR 0002 `GEN-BOOT` decision, §8) | Frigate/cameras, Sangam (notify), Bodha |
  | **Artha** | Manas.Crypto (`SENS`), Dwarpal | Sangam (bank connectors), Notifications |
  | **Ojas** | Manas.Crypto (`SENS`), Dwarpal | HealthKit import |
  | **Kaya** | Home Assistant | voice, Frigate |
  | **Saakshi** | operational store (Postgres) | Qdrant, Neo4j |
  | **Sangam** | Pramana (token vault) | — |

  Two rows are infrastructure, not domain services — their dependency is expressed as "who is HARD-broken
  without them," not "what they need":

  | Component | HARD dep **of** | Boot step | Notes |
  |-----------|------------------|-----------|-------|
  | **Kutumba** | Every walled service — `guardianOf`/role data feeds **every** may-see check platform-wide; Kutumba-down is **fail-closed** everywhere, not a degrade | **Step 9 (GEN unlock)** — Kutumba is `GEN`-tier and stays **out of the auto-unseal set** per [ADR 0002 §2.2](0002-key-hierarchy-and-unlock.md): **no pre-login walled read exists**, so nothing needs it before login; 0002's tripwire moves it into the boot slice only if that ever changes *(reconciled 2026-07-11, round-2 D13/C1 — this row previously contradicted 0002)* | Not optional for anyone; there is no "run without Kutumba" mode because the walls themselves read it. |
  | **Dwarpal** | Login, step-up, token exchange (ADR 0003) | Step 5 | **The alarm survives Dwarpal-down.** Kavach (both layers, once up) and the mTLS mesh authenticate via the **machine-trust tier**, not Dwarpal — cross-ref [ADR 0003 §5](0003-identity-propagation-and-pki.md) (internal PKI is Pramana-issued, independent of Dwarpal). If Dwarpal is down after boot: no new logins, no step-up, no token exchange; already-unsealed tiers and already-issued mesh certs keep working. |

- **The platform degrades only over OPTIONAL deps.** Losing Frigate ⇒ Kavach runs without camera corroboration
  (documented degrade). Losing **Kaya's device path** with Kavach enabled ⇒ **NOT** a degrade for Layer 1 —
  Kavach's Layer 1 smart features are down (**Layer 0 still alarms, unaffected — it has no Kaya dependency**),
  and Manas **refuses to pretend Kavach Layer 1 is fully up**. Losing **Kutumba** is never a degrade for
  anyone — every may-see check fails closed. Losing **Dwarpal** post-boot degrades only login/step-up/token-exchange
  (see table); it does not take down the mesh or an already-armed Kavach.
- **Startup validation REJECTS invalid combinations** with a **clear error**, never a silent half-run: e.g.
  *"Kavach is enabled but Kaya's device path is unavailable — Kavach Layer 1 cannot start. Layer 0 alarm remains
  active. Enable Kaya or disable Kavach Layer 1."* A missing **HARD** dep is a **boot failure for that service**;
  a missing OPTIONAL dep is a logged degrade.

---

## 10. Incident-response runbook (D11)

A short, actionable checklist. Kept here so it's versioned with the design; the on-box console links to it.

**Priority order is deliberate — Sangam/email first** (email is the account-recovery factor for *everything*).

1. **Compromise suspected → ISOLATE.** Pull the box's WAN uplink (kill the tunnel/relay egress) but **leave Layer
   0 armed** — the alarm must not go down during an incident. Note the time.
2. **Revoke Sangam OAuth tokens FIRST.** From another trusted device, revoke **all Sangam grants** — email
   especially (**email = account-recovery for every other account**, highest blast radius). Then bank/health
   connectors. Change the Google/email password out-of-band.
3. **Rotate the affected DEK(s).** Per [`0002`](0002-key-hierarchy-and-unlock.md) §3 (lazy re-encrypt via
   `keyVersion`), rotate the tier(s) touched (`SENS` for finance/health, `GEN` for ops). Force idle re-lock so
   old in-RAM DEKs are zeroized.
4. **Rotate the HA long-lived token** (§6) and any Sangam/service credentials in the Pramana vault.
5. **Re-issue certs.** Have Pramana's CA **re-issue the mTLS mesh certs** and revoke the old ones (ADR 0003) —
   assume mesh identity material may be tainted.
6. **Rotate admin credentials + step-up secrets** (passwords, TOTP seeds).
7. **Notify the family.** Out-of-band (not via the possibly-compromised box): what happened, what to watch
   (phishing from the exfiltrated email), that the alarm still works via the keypad.
8. **Restore from a KNOWN-GOOD snapshot** (§5) predating the compromise, onto a **compatible Release** image;
   run the restore drill checks before trusting it; reconcile Kavach to Layer 0.
9. **Post-incident.** Decrypt the relevant Saakshi window (§8 tool), write up the timeline, file the gap.

**Box-dead (non-malicious) mini-runbook:** dead-man's-switch fires → confirm it's the box, not the ISP (§3) →
family knows **Layer 0 keypad still arms/disarms and the siren still fires** → reprovision from the pinned
Release image → `zfs receive` latest snapshot → unlock → reconcile → verify with the drill checklist.

**Dwarpal-down (non-malicious) mini-runbook:** *(added 2026-07-11, round 2 — closes S13, backs §11 test 12)* Dwarpal
(Zitadel) health check fails while the rest of the box is up → mesh stays up (mTLS is Pramana-issued, §9) → an
already-armed Kavach keeps functioning on both layers → no new login/step-up/token-exchange is possible, surfaced
to the family as a clear "sign-in temporarily unavailable" rather than a silent hang → restart the Dwarpal container/
service → confirm health, then confirm a fresh login succeeds → **no reboot, no unseal, no restore required** —
Dwarpal-down is a service-level fault, not a boot-tier fault, because its store slice and signing keys are already
unsealed (§8 step 2).

---

## 11. Acceptance criteria / tests

Concrete assertions this ADR's implementation must satisfy:

1. **Box-death alarm.** With the entire Manas stack stopped (containers down), tripping an armed zone **fires the
   Layer 0 siren**; the local keypad **arms and disarms**. (Bench test with the panel + box powered off.)
2. **Dead-man's-switch.** Stop the beacon; the relay pushes an APNs alert within the **5-min** grace and re-pages
   on back-off; resuming the beacon (or a family ack) silences it. A ≤ 60 s network blip does **not** alert.
3. **Monotonic exit delay (headline).** Start a Kavach exit delay; **jump wall-clock forward** by more than the
   delay; the exit delay **still runs its full monotonic duration** and is **not** skipped/shortened.
4. **HA can't forge all-clear.** Inject a fake HA "disarmed/all-clear" that contradicts the Layer 0 panel state ⇒
   Kavach **rejects** it and raises a tamper alert; disarm still requires Manas step-up (PIN).
5. **Tier stays locked at boot.** *(amended 2026-07-11, round 2 — closes S1/S2)* After cold boot, pre-admin-login
   (i.e. after §8 step 5 — Dwarpal is up and login is *possible* — but before step 8, the login gate), Kavach `LIFE`
   is up and arming via **Layer 0** (from step 6), while a `SENS` read raises `TierLockedException` (0002 §4)
   until admin login (step 8) unlocks it at step 9. Assert this holds under **both** §8 S2 branches — Branch A
   additionally has Kaya/Kavach-Layer-1 up pre-login; Branch B does not; `SENS`/`GEN` stay locked in both.
6. **Unencrypted boot log exists and is payload-free.** Force a boot-step failure; the plaintext boot log names
   the failed step with **no** user data/secret in it; the encrypted Saakshi log is unreadable until admin unlock,
   then the decryption tool reads it.
7. **DR restore drill.** A ZFS snapshot restored onto a scratch target on a **compatible Release** brings up
   Kavach (arms), unlocks `SENS`/`GEN`, loads Neo4j, and answers Qdrant. Restoring onto an **incompatible
   Release** is **refused** with a clear error.
8. **Dependency rejection.** Enable Kavach with Kaya's device path disabled ⇒ startup **refuses** Kavach Layer 1
   with the clear error, Layer 0 stays active — it does **not** silently half-run.
9. **Tamper-zeroize.** A simulated chassis-intrusion signal zeroizes the in-RAM `SENS`/`GEN` DEKs (assert via
   test double, per 0002 §9).
10. **NTS + clock health.** With NTS unreachable, the box keeps its last-good offset, raises a Bodha clock alarm,
    and security countdowns keep running on the monotonic clock.
11. **Boot-order test.** *(added 2026-07-11, round 2 — closes S1/S13)* On a cold boot, the full §8 chain (steps
    1–10) comes up **in order**, each step's health gate observed before the next step starts: machine-trust
    unseal (2) before core datastores (3), datastores before mesh (4), mesh before Dwarpal (5), Dwarpal-up/login-possible
    before Kavach+Layer-0 reconcile (6), Kavach before Kaya/HA (7, per the active S2 branch), Kaya/HA before the
    admin-login gate (8), the gate before `SENS`/`GEN` unlock (9), unlock before finance/health/audit readable
    (10). A step that fails health halts the dependent chain with a typed error (not a half-start) and does not
    let a later step silently proceed.
12. **Dwarpal-down.** *(added 2026-07-11, round 2 — closes S13)* With Dwarpal stopped after a healthy boot: the mTLS
    mesh stays up, an already-armed Kavach (both layers, if up) keeps functioning, no **new** login or step-up
    succeeds, and no new token can be exchanged. Recovery (restart Dwarpal, confirm login resumes without a reboot)
    is documented in the runbook (§10).

---

## 12. Open questions

1. **TSME on the home server — VERIFY.** Confirm the CPU + this firmware actually exposes and lets us
   enable SME/TSME memory encryption. If not, §4's RAM residual is larger; document accordingly.
2. **Layer 0 hardware choice.** A board driving the existing/hardwired siren vs a self-contained
   commodity alarm panel that Kavach observes. Both meet the "dumb, independent, own-power" bar;
   pick on wiring reality + how cleanly Layer 1 can arm/disarm it over a wired line. (Not "a second server.")
3. **Second egress path for the beacon.** Is an LTE/failover uplink worth it to disambiguate box-death from
   ISP-outage in the dead-man's-switch, or is the 5-min grace + ack copy enough? Cost vs clarity.
4. **TPM PCR selection** for `LIFE`/`MTRUST` sealing (§4.4) — jointly owned with [`0002`](0002-key-hierarchy-and-unlock.md)
   §11 and Pramana.
5. **Sensor-signing depth (§6.3).** How many life-safety sensors actually support authenticated/signed state vs
   need corroboration-only; enumerate per device during Kavach build.
6. **Restore-drill automation.** How much of the quarterly drill can be automated on a scratch VM vs needs a
   human to confirm the alarm physically arms.
7. **The `GEN-BOOT` decision (S2) — owned by ADR 0002, not this ADR.** §8 step 7 and §9's Kavach row carry
   **both** branches (A: a `GEN-BOOT` sub-tier TPM-auto-unseals Kaya's connection config + the HA token +
   entity registry, giving pre-login Kavach Layer 1; B: no such sub-tier, Layer 1 is honestly down pre-login)
   pending [`0002`](0002-key-hierarchy-and-unlock.md)'s own open question on `GEN-BOOT`. This ADR does not
   pick a branch; once ADR 0002 decides, this ADR's §8/§9 collapse to the single chosen branch and this
   question closes.

---

## 13. Consequences

- **The single-box life-safety gap is closed at the physical layer.** Box death no longer means "no alarm for
  days" — a dumb, own-powered Layer 0 core alarms and arms/disarms without the box, and the family is **told**
  the box died (dead-man's-switch). This is the honest answer to C1/C2/E15 that "buy a second server" is not.
- **The availability envelope is now written down.** RTO/RPO per data class, what survives box death, what runs
  while locked — the family and any future operator know exactly what to expect, instead of an implicit "it's
  always up."
- **HA is demoted from trusted peer to locked-down appliance** (C5) — it senses and transports; it cannot decide
  to disarm or forge safety. Closes the step-up-bypass.
- **Time is trustworthy and delays are monotonic** (C7) — a clock jump can never skip a security countdown, the
  single most dangerous time bug in an alarm.
- **A locked box is debuggable** (C8) via a payload-free boot log + an admin-gated decrypt tool, without
  weakening the encrypt-everything posture.
- **DR is one primitive, not five** (B4) — ZFS snapshots give cross-store consistency and are the only correct
  Neo4j-Community backup; Release-pinning + restore drills make "we have backups" mean "we can actually come
  back."
- **Composability gains teeth** (D5) — hard deps are declared and invalid combos are refused loudly, so Kavach
  can't be half-configured into a false sense of safety.
- **Added cost:** commodity alarm hardware (a panel + siren + keypad, ~one-time, cheap), a little relay logic for
  the beacon, ZFS discipline, and boot-ordering wiring. Modest, commodity, and all on the **existing** the home server +
  cheap alarm gear — **no new server, no different box.**
- **Constrains other ADRs:** it consumes [`0002`](0002-key-hierarchy-and-unlock.md)'s PCR-seal + zeroize hooks,
  depends on ADR 0003's mTLS/CA for §6/§8, and pins backups to the platform's release-and-updates Release BoM.

---

*Refines [`0001-manas-platform-architecture.md`](0001-manas-platform-architecture.md) §14,
and the platform's deployment topology and threat model documentation. Cross-references
[`0002-key-hierarchy-and-unlock.md`](0002-key-hierarchy-and-unlock.md) (keys),
[`0003-identity-propagation-and-pki.md`](0003-identity-propagation-and-pki.md) (mTLS/CA), the internal
trust/vault/CA design, and the release-and-updates documentation (Release pinning).*
