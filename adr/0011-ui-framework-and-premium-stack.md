# ADR 0011 — UI Framework & the Premium Front-End Stack (Akriti)

- **Status:** Accepted (Ananth, 2026-07-12)
- **Date:** 2026-07-12
- **Deciders:** Ananth (owner), Claude (orchestrator)
- **Refines:** ADR 0001 §5.3 (Akriti design system), §2 driver #6 (accessible / voice-complete); resolves the
  open web-stack decision in [`tech-stack.md`](../architecture/tech-stack.md) §3a item 4; supersedes the skeleton
  console's Blazor choice (unit 0.16); updates [`ios-app.md`](../vision/ios-app.md) §3/§8's "Blazor WASM PWA".
- **Evidence:** a decision-grade research pass (web-verified, mid-2026) is the substance behind §§1–5 below.

> **Manas's UI has to be premium — very polished, interactive, animation-rich, engaging charts, never
> "Tableau-plain."** This is a hard owner requirement, not polish-later. It was also the single most painful
> recurring failure in the predecessor projects (Jarvis, Steward), where the UI was **redone or upgraded four
> times and stayed "blah" — static, un-engaging, boring charts.** This ADR fixes that at the root: it names the
> cause (an ecosystem mismatch that every redo preserved), chooses the framework and premium stack, and gates
> the full design system behind a "prove-it" reference screen so redo #5 cannot happen.

---

## 1. Context — why four redos didn't fix it

The prior UIs were .NET-centric solo builds on **Blazor and/or server-rendered Razor**. Four redos on the same
axis is the tell: the failure was structural, not effort. Root causes, ranked:

1. **Ecosystem/tooling mismatch (dominant).** The premium-UI toolchain — spring-physics motion, layout/shared-
   element transitions, interactive data-dense charting, a deep polished-component ecosystem — lives in the
   JS/TS world, specifically React. As of mid-2026 Blazor's animation story is a handful of **CSS-wrapper
   libraries** (BlazeFX, Blazor.Animate, Blazorise Animate); there is **no first-class Motion/Framer-Motion
   equivalent** — no unified spring engine, no `layout` auto-animation, no gesture system, no `AnimatePresence`
   exit choreography. Charting tops out at MudBlazor/Radzen/ApexCharts wrappers with a real interactivity
   ceiling. You cannot out-discipline an ecosystem gap; each redo stayed where premium UI isn't.
2. **No motion system / token system / design system *as a system*.** "Blah" is usually the absence of
   choreography and tokens, not the framework alone. Clean ≠ engaging.
3. **Solo-dev gravity toward the familiar** (C# everything) kept the UI inside Blazor/Razor, where a solo dev
   must hand-build what a React dev installs — then runs out of bandwidth and ships the boring default.
4. **Server-render staleness** where Razor/MVC was used (no client-side spring physics, no optimistic UI).
5. **Charting ceiling** specifically — stock wrapper charts yield "boring charts" almost deterministically.
6. **Blazor WASM interop tax** — rich animation crosses a JS-interop boundary anyway, through a fragile bridge.

**The fix must therefore do three things, not one:** move to the ecosystem where premium UI lives, install an
explicit motion + token + charting system, and **prove the bar is cleared on one screen before committing the
whole design system.**

## 2. The bar — what "not blah" concretely means

Graded against best-in-class references (Linear, Vercel/Stripe dashboards, Tremor/Retool, Framer):
**spring-physics motion** (not linear tweens), **layout/shared-element transitions**, `AnimatePresence`-class
enter/exit, staggered reveals; **micro-interactions** on hover/press/focus; **interactive data-dense charts**
(hover crosshair, brush/zoom, drill-down, live-update with animated transitions, legend toggles — not static
SVG); **skeleton/optimistic loading**, never a bare spinner; a **⌘K command palette**; and the **glass-box**
pattern the owner names — *number → the calculation → the button* as a first-class interaction. All of it while
honoring **WCAG 2.2 AA + `prefers-reduced-motion` + voice-completeness** (ADR 0001 driver #6).

## 3. Decision — the stack

**Akriti is built on Vite + React 19 + TypeScript (SPA).** Runner-up SvelteKit is genuinely excellent and
lighter, but its order-of-magnitude-thinner package ecosystem **re-creates the exact condition that produced
"blah" four times** — so it is #2, not co-equal. Blazor is rejected for the UI layer (its premium ceiling is
intrinsic, per §1); it remains the backend. Full stack:

| Layer | Choice | Why |
|---|---|---|
| Framework | **Vite + React 19 + TypeScript** (SPA, not Next.js) | single household box, no SEO/edge/SSR need; served as static assets by **Kestrel** (one origin, one auth path); best-documented + most AI-assisted UI stack in 2026 (lowers solo-dev cost) |
| Components / tokens | **shadcn/ui (Radix primitives) + Tailwind v4** | Radix = accessible by construction (serves "WCAG 2.2 AA by construction"); Tailwind CSS-variable tokens = Akriti's token layer; light/dark via `prefers-color-scheme` + `data-theme` |
| Motion | **Motion v12** (`motion/react`, ex-framer-motion) | spring physics, `layout`, `AnimatePresence`, gestures; **global `MotionConfig reducedMotion="user"`** honors `prefers-reduced-motion` app-wide by config, not per-component discipline |
| Charts | **Recharts v3 / Tremor** default; **Apache ECharts** (Canvas) for data-dense/live; **visx** for bespoke; Nivo fallback | Recharts `accessibilityLayer` (ARIA + keyboard nav, WCAG 2.2 AA goal); ECharts for Artha tick data / 10k+ points; a **configurable charting policy**, not one lib (Manas ethos) |
| GraphQL / data | **urql + graphql-ws**; graphcache if earned; **TanStack Router** | urql natively uses graphql-ws — the protocol HC16 already speaks and the skeleton already uses; lighter than Apollo; makes the deferred StrawberryShake decision moot |
| Persona avatar / mascot | **Rive** state-machine avatar; Lottie for decorative icon motion | purpose-built for interactive mascots; one `.riv` per `mascotSet`; **the `[anim:x]` cue → a Rive state-machine input**; web + iOS runtimes run the *same file* — unifies persona.md's "one cue vocabulary, many renderers" across console + iOS (the box's 320×240 face stays its own frame renderer) |

## 4. The architectural freedom (why this changes nothing load-bearing)

**The frontend is a graphql-ws client over the C#/.NET 10 backend, so the framework choice is fully decoupled
from the local-first / deterministic-C# thesis.** GraphQL is the API contract (ADR 0001 §6); the agent transport
is graphql-ws streaming `Delta / ToolStatus / StepUpPending / Ack / Final{resultKind, renderedText, animCue}`
(agent-spec §D). A React client consumes those identically to a Blazor client — **the wall, step-up, the
deterministic decisions, and on-box privacy all stay server-side and unchanged.** Integration costs, all bounded:

- **Build/serve:** Vite → static assets served by **Kestrel from `wwwroot`** (one origin — no separate Node
  server, no second auth surface). SSR is deliberately *not* adopted (a single box has no SEO/edge need and a
  Node SSR server next to the C# box contradicts the single-box posture); streaming SSR stays a later,
  reversible optimization if ever wanted.
- **Auth over graphql-ws:** the token rides the `connection_init` payload (urql `connectionParams`) — the same
  pattern the skeleton adopted (WASM can't set WS upgrade headers). The two HC16 footguns already learned
  (`[Subscribe(With=)]` + `[EventMessage]`; synchronous subscription-authz throw) are backend/contract and
  survive the frontend swap untouched (tech-stack §3a).
- **iOS WKWebView:** a Vite React PWA renders natively in WebKit and is **lighter than Blazor WASM in a webview**
  (no multi-MB mono-wasm over the Tailscale relay). This *improves* the ios-app.md §3 embed story.
- **Offline/local-first PWA:** service worker (Workbox) for app-shell caching; the box is the data source.

## 5. WCAG 2.2 AA *with* rich motion; the child surface

Accessibility is achieved by construction, not per-component vigilance: (1) Radix/shadcn primitives ship focus
management + ARIA + keyboard operability; (2) Motion's `reducedMotion="user"` disables transform/layout
animation globally while keeping opacity; (3) charts are keyboard-navigable (Recharts `accessibilityLayer` /
Nivo ARIA) **and every chart has a queryable GraphQL/tabular equivalent** (satisfying voice-completeness and the
"not just a picture" bar). **The ⌘K palette is the visual twin of voice-completeness** — every capability
reachable by palette maps to one reachable by voice.

**Child-facing (Bonnie/the child):** warmer tokens, larger touch targets, simpler layouts, Bonnie's Rive set; the
child-strict register is already enforced server-side (persona.md §5) so the UI only renders it. **Hard rule:
motion must never nag, manipulate, or create engagement loops on a child — no dark patterns; reduced-motion is
honored for the child identically.**

## 6. The "PROVE-IT" spike — the gate that prevents redo #5

Before any full design-system commitment, build **one genuinely-premium reference screen**: the **Abhyasa
grade-trend dashboard**, exercising every axis at once —
- an **animated, interactive grade-trend chart** (Recharts v3: hover crosshair, brush, drill-into-a-term,
  animated data transitions; one panel swapped to ECharts to prove the escape hatch);
- a **live streaming Sarathi chat** panel on the real graphql-ws `Delta`/`Final` stream (incremental token
  render — proves the transport end-to-end);
- the **persona avatar** (Bonnie for the child / Anik for adult) as a **Rive** component driven by `Final.animCue`;
- **skeleton loading, ⌘K palette, one glass-box interaction** (a grade → its calculation → the action);
- a **reduced-motion pass** and a **WKWebView-over-Tailscale render pass**.

**Acceptance = "does this clear the Linear/Stripe bar (§2), embedded on a phone, accessible?"** Green → commit
Akriti on this stack. Not green → we learned it on one screen, not on redo #5. This is a Phase-1 gate (SDD).

## 7. Consequences

**Positive.** The premium bar becomes reachable because we're finally in the ecosystem where it lives; the
root cause of four failures is named and fixed, not repeated; accessibility is by-construction; the persona cue
system unifies across surfaces via Rive; nothing about the deterministic/local-first thesis changes.

**Costs / tradeoffs (honest).** A **second toolchain** (Node/npm/Vite alongside .NET) and a real (bounded)
React ramp for a .NET-centric owner — mitigated by React being the most AI-assisted, best-documented UI stack,
and by the project already accepting Swift for iOS. A **larger npm supply-chain surface** than NuGet — mitigated
by the same discipline as `Directory.Packages.props` (pinned versions, committed lockfile, dependency review,
preferring copy-in shadcn components you own over deep trees). React ships **more JS than Svelte** — the one real
edge for the WKWebView-over-Tailscale phone case, judged not to outweigh the ecosystem gap.

**What would change the decision:** a flat refusal of any second frontend toolchain *and* no AI assistance →
the honest fallback is Blazor .NET 10 + MudBlazor + ApexCharts + JS-interop Rive at a **documented lower
ceiling** (not a co-equal option). Android becoming P0 doesn't change the answer.

**Constrains / doc updates.** ADR 0001 §5.3 (Akriti tech named); tech-stack §3a item 4 (resolved: React, urql,
retire Blazor skeleton, promote pins); ios-app.md §3/§8 ("Blazor WASM PWA" → "React PWA"); SDD Phase-1 (the
prove-it spike gates unit 1.9 Akriti foundation; the Blazor `Manas.Console` is replaced by a Vite/React
`Manas.Console`). Library picks (Recharts-vs-ECharts, urql-vs-Apollo, Rive-vs-Lottie) stay swappable behind the
component/chart/data seams; the framework is firm.

---

*Refines [`0001-manas-platform-architecture.md`](0001-manas-platform-architecture.md) §5.3; resolves
[`../architecture/tech-stack.md`](../architecture/tech-stack.md) §3a; cross-references
[`../architecture/persona.md`](../architecture/persona.md) §3 (cue vocabulary → Rive),
agent-spec §D (graphql-ws events),
[`../vision/ios-app.md`](../vision/ios-app.md) §3 (WKWebView embed).*
