# Artha — Vision (Family Finance)

> *Renamed from the working name Steward — decided 2026-07-12 (ADR 0001 §3). The existing Steward .NET codebase re-homes under this name in Phase 4.*

> **Status:** Vision / intent (feeds later ADRs `0002+`). **Date:** 2026-07-11.
> **Scope:** the Manas finance service. Governed by [ADR 0001](../adr/0001-manas-platform-architecture.md);
> nothing here may contradict it without an explicit superseding ADR.
> **This is a DEEPEN-AND-EXTEND document, not a greenfield one.** Steward already exists as a mature
> .NET solution (Plaid + statement ingest, statistical recurring detection, fraud/waste/missed-payment/
> yield-gap analyzers, debt payoff, spend coach, budget/cashflow/net-worth/goal/retirement/tax engines,
> scouts, a tiered `ActionDispatcher`, and per-user walls). This vision **catalogues what is real today,
> then layers the genuinely-missing depth** (envelopes, a first-class shared-vs-personal money model,
> kids' allowance/chores, richer bill radar and forecasting) — and folds it onto the Manas foundation
> (Kernel walls, Medha, Sangam, Saakshi, MCP tools, the polyglot stores).

---

## 1. Intent

**Artha watches the family's money and helps them act on it — never leaving the house to do so.**
It ingests the household's full financial picture (transactions, balances, cards, loans, holdings, bills)
from Plaid and statement imports; runs **deterministic, testable C# math** to detect what is costing the
family money and what could be doing better (fraud, waste, missed payments, idle cash, yield gaps,
budget overruns, cashflow crunches, subscription creep); plans the mathematically-best move (debt payoff,
cash sweeps, budget rebalances, savings-goal funding); and **acts on the family's behalf in tiers** within
limits the family sets — auto for the safe and small, one-tap for the reversible, in-app confirm with
step-up for the consequential. Every dollar figure is computed by deterministic code and shown with its
work (glass-box); the **LLM only understands intent, converses, and frames guidance** — it never computes
a number and is never the security boundary. **Every user's money is walled from every other's** at the
data layer (adults are private from each other; the child sees only her own allowance world). Artha is
a **steward, not a licensed advisor**: it surfaces facts, opportunity-cost math, and commoditised
industry consensus — it does not give personalised investment, tax, or legal advice, and it defers
high-stakes decisions to a human.

**North stars.** (1) *Beat the spreadsheet* — if a competent user could do it as easily in Excel, it
doesn't ship. (2) *Decision → calculation → number → button* — every surface answers a real decision,
shows the math, and ends in an action. (3) *Deterministic money* — same input, same dollar output,
always. (4) *Local-first & private* — data and inference stay on the home server; any metered external call
passes `ISpendGuard`. (5) *Walls that survive the agent* — Sarathi is a deputy carrying the caller's
identity, never a super-user.

---

## 2. Capability areas

Legend — **Kind:** `[deterministic]` pure C# math/rules (the bulk), `[LLM]` natural-language
understanding/explanation/coaching only (never computes money), `[integration:name]` needs an external
provider. **Feasibility:** `now` = buildable on current stack / already largely built, `needs-3rd-party`
= gated on an external grant/provider. **Priority:** P0 = core money-watching (protect & inform),
P1 = planning & optimisation depth, P2 = nice-to-have / advanced.

Many rows are marked *(exists)* — already implemented in the current Steward solution and carried
forward; *(extend)* — a real capability today that this vision deepens; *(new)* — genuinely missing.

### 2.1 Ingestion & the financial picture

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Plaid transaction & balance sync | Incremental cursor-based pull of transactions, balances, accounts *(exists)* | `[integration:Plaid]` | now | P0 |
| Plaid Liabilities | Auto APR / minimum / due-date / statement balance for cards & loans *(exists)* | `[integration:Plaid]` | now | P0 |
| Plaid Investments | Read-only holdings, cost basis, positions for brokerages/401(k) where covered *(extend)* | `[integration:Plaid]` | now | P1 |
| CSV / OFX / QFX statement import | Fallback for issuers Plaid can't reach *(exists, CSV)* | `[deterministic]` | now | P0 |
| PDF statement parsing | Extract transactions + card metadata (APR/limit/due) from statement PDFs *(exists)* | `[deterministic]` | now | P1 |
| Exotic-statement / messy-PDF repair | LLM assists only when deterministic parse fails; extracted facts re-validated by code *(exists, IExplainer)* | `[LLM]` | now | P1 |
| Email bill/receipt intake (Sangam) | email readonly → detect bills, receipts, payment confirmations; persist **facts only**, not bodies *(exists, Sources.Mail → move behind Sangam)* | `[integration:email via Sangam]` | now | P1 |
| USPS Informed Delivery digest | Daily mail-image scan to see bills *before they physically arrive* *(exists, MailVisionScanner)* | `[integration:email]`+`[LLM]` vision | now | P2 |
| Forwarded-SMS payment confirmation | Match "your payment posted" texts to expected obligations *(exists)* | `[deterministic]` | now | P2 |
| Manual accounts & holdings | Hand-entered accounts, cash, crypto, real estate, valuables Plaid can't see *(exists, ManualHolding)* | `[deterministic]` | now | P0 |
| Merchant normalisation & categorisation | Deterministic merchant cleanup + rule-based categories; LLM only for the long tail *(exists)* | `[deterministic]` (+`[LLM]` tail) | now | P0 |
| Multi-currency / FX | Normalise foreign charges via FX source *(exists, Frankfurter)* | `[integration:Frankfurter/FRED]` | now | P2 |
| Consent & source lifecycle | Recorded consent per source (Plaid, email), token vault, revoke/forget *(extend → Manas Kernel + Sangam)* | `[deterministic]` | now | P0 |

### 2.2 Watchdog — detection & alerts (protect the money)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Recurring-payment detection | Statistical clustering (merchant + amount + cadence), confidence-scored, **no LLM** *(exists)* | `[deterministic]` | now | P0 |
| Missed-payment radar | Predict/detect a due obligation that didn't post *(exists)* | `[deterministic]` | now | P0 |
| Card-testing / scam & anomaly | Small-probe fraud pattern, first-time big charge from a new merchant, unusual amount from a known one *(exists)* | `[deterministic]` | now | P0 |
| Duplicate-charge detection | Same merchant/amount posted twice *(exists)* | `[deterministic]` | now | P0 |
| Fee detector | Overdraft / ATM / FX / late fees you actually paid, with "how to avoid next time" *(exists)* | `[deterministic]` | now | P0 |
| Overlapping-subscription waste | Two services in the same category (2 music, 3 streaming) *(exists, WastedMoney)* | `[deterministic]` | now | P1 |
| Idle-cash / yield-gap alert | Cash sitting below HYSA yield; quantified annual opportunity cost *(exists, YieldGap)* | `[deterministic]` | now | P1 |
| Opportunity-cost of over-large emergency fund | Beyond-buffer cash that could be invested (framed as observation, not advice) *(exists)* | `[deterministic]` | now | P2 |
| Credit-utilisation watch | Per-card & aggregate utilisation crossing thresholds before statement cut *(exists)* | `[deterministic]` | now | P1 |
| Refund watch | Track expected refunds/returns until they post *(exists, RefundWatch)* | `[deterministic]` | now | P2 |
| Paycheck sentinel | Expected income didn't arrive / changed materially *(exists, PaycheckSentinel)* | `[deterministic]` | now | P1 |
| Windfall handler | Detect an unusual inflow and prompt an allocation decision *(exists)* | `[deterministic]` | now | P2 |
| Alert explanation & rewriting | Turn a deterministic alert into a plain, kind sentence *(exists, IExplainer)* | `[LLM]` | now | P1 |
| Severity-tiered notifications | Email / SMS (Twilio) / push (Pushover) / Slack / Discord, thresholded per channel *(exists → route via Sangam)* | `[integration:Twilio/Pushover/…]` | now | P0 |

### 2.3 Budgeting & envelopes

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Category budgets vs actuals | Per-category monthly budget with live actuals & variance *(exists, BudgetAnalyzer/Comparer)* | `[deterministic]` | now | P0 |
| **Envelope / zero-based budgeting** | YNAB-style: assign every dollar to an envelope; spend draws it down; overspend visibly borrows *(new — build on BudgetLine)* | `[deterministic]` | now | P1 |
| Rollover envelopes | Unspent rolls to next month; sinking-fund envelopes (car, gifts, annual bills) *(new)* | `[deterministic]` | now | P1 |
| Auto-budget suggestion | Propose budgets from trailing 3–6 month actuals; user tunes *(new; LLM only phrases the suggestion)* | `[deterministic]` (+`[LLM]` phrasing) | now | P1 |
| Safe-to-spend today | Discretionary money left after committed bills + envelopes + buffer *(exists, SafeToSpendCalculator)* | `[deterministic]` | now | P0 |
| Personal-CPI / lifestyle-inflation | The family's own inflation rate vs headline CPI *(exists, PersonalCpiCalculator)* | `[deterministic]`+`[integration:FRED]` | now | P2 |
| Budget-breach coaching | When an envelope is nearly empty, explain trade-offs before the next swipe *(extend SpendCoach)* | `[LLM]` over `[deterministic]` | now | P1 |

### 2.4 Cashflow, forecasting & net worth

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Cashflow forecast | Project balances forward from recurring inflows/outflows + known bills *(exists, CashflowForecaster)* | `[deterministic]` | now | P0 |
| Low-balance / overdraft prediction | Warn before a forecast dip below zero or below buffer *(extend)* | `[deterministic]` | now | P1 |
| Float optimiser | Time bill payments to avoid low points without late fees *(exists, FloatOptimizer)* | `[deterministic]` | now | P2 |
| Net-worth tracking | Assets − liabilities across all accounts + manual holdings, over time *(exists, NetWorthCalculator + FinancialSnapshot)* | `[deterministic]` | now | P0 |
| Net-worth history & trend | Snapshot series, month-over-month deltas, sparklines *(exists, FinancialSnapshot)* | `[deterministic]` | now | P1 |
| Scenario simulator | "What if we buy a house / have a big expense / lose an income" projections *(exists, ScenarioSimulator)* | `[deterministic]` | now | P1 |
| FIRE / fire-drill calculator | Runway if income stops; coast/lean-FIRE math *(exists, FireDrillCalculator)* | `[deterministic]` | now | P2 |

### 2.5 Bills, obligations & payments

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Bill radar | Aggregate all obligations (recurring + emailed + statement due dates) with deterministic urgency ranking *(exists, Steward.Bills)* | `[deterministic]` | now | P0 |
| Upcoming-bill calendar | Timeline of what's due, how much, from which account; feeds cashflow *(extend)* | `[deterministic]` | now | P1 |
| Bill-pay execution | Actually pay a biller (ACH) via a money-movement partner *(exists, Sources.Method — Tier-B/C gated)* | `[integration:Method]` | needs-3rd-party | P1 |
| Bill-pay verification | Confirm the payment posted (via statement / email / SMS) and close the loop *(exists, BillPayVerifier)* | `[deterministic]` | now | P1 |
| Autopay audit | Detect which bills are on autopay vs manual; flag risky manual ones *(new)* | `[deterministic]` | now | P2 |

### 2.6 Savings goals & planning

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Savings goals | Named goals (emergency fund, vacation, house, the child's college) with target + date *(exists, Goal)* | `[deterministic]` | now | P0 |
| Goal-progress & required contribution | Track funding, compute the monthly contribution to hit the date *(exists, GoalProgressCalculator)* | `[deterministic]` | now | P0 |
| Surplus allocation | Route each month's surplus across goals/debt/invest by a chosen strategy *(exists, SurplusAllocator/PlanSurplusLevers)* | `[deterministic]` | now | P1 |
| Goal auto-funding (sweep) | Auto-move to a goal sub-account when surplus exists (Tier A/B) *(extend)* | `[integration:Plaid Transfer/Method]` | needs-3rd-party | P1 |
| Retirement / 401(k)-match check | Are we capturing the full employer match? contribution-rate math *(exists, RetirementCalculator)* | `[deterministic]` | now | P1 |
| Next-best-action synthesis | Rank all recommendations into one prioritised "do this next" list *(exists, NextBestActionSynthesizer)* | `[deterministic]` (+`[LLM]` phrasing) | now | P1 |

### 2.7 Debt & credit optimisation

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Debt payoff planner | Avalanche / snowball / avalanche-with-balance-transfer, month-by-month schedule *(exists, DebtPayoffCalculator)* | `[deterministic]` | now | P0 |
| Balance-transfer evaluator | Model a BT offer's fee vs interest saved; break-even *(exists)* | `[deterministic]` | now | P1 |
| Debt-funding planner | Which account funds extra payments without breaching buffers *(exists, DebtFundingPlanner/FundingSourceOptimizer)* | `[deterministic]` | now | P1 |
| Utilisation optimiser | Which card to pay down first to lift the score before an application *(exists, UtilizationOptimizer)* | `[deterministic]` | now | P1 |
| Which-card-to-swipe coach | Best card for a purchase given utilisation, rewards, APR, payoff plan *(exists, SpendCoach)* | `[deterministic]` | now | P1 |

### 2.8 Subscription management

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Subscription inventory | Auto-list every recurring charge with cadence, amount, last seen *(exists, from RecurringPaymentDetector)* | `[deterministic]` | now | P0 |
| Price-creep scout | Detect when a subscription's price silently rose *(exists, SubscriptionPriceCreepScout)* | `[deterministic]` | now | P1 |
| Unused / ghost subscription flag | Recurring charge with no matching usage signal → "cancel?" *(new; usage signal limited — flag by staleness/heuristic honestly)* | `[deterministic]` | now | P1 |
| Free-trial → paid conversion watch | Warn before a trial converts (from email/receipt intake) *(new)* | `[deterministic]`+`[integration:email]` | now | P2 |
| Cancel assistance | Draft a cancellation, surface the cancel link/steps — **user completes it** (no auto-cancel of contracts) | `[LLM]` draft only | now | P2 |

### 2.9 Investments (read-only first; tiered trades strictly gated)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Holdings & allocation view | Read-only positions, cost basis, gain/loss, asset-class allocation *(exists, Holding/InvestmentObservation)* | `[integration:Plaid Investments]` | now | P1 |
| Portfolio drift observation | Allocation drifted from a target the *user* set; flagged as observation, not advice *(extend)* | `[deterministic]` | now | P2 |
| Idle-brokerage-cash / sweep note | Uninvested cash in a brokerage earning nothing *(exists, InvestmentObservation)* | `[deterministic]` | now | P2 |
| Investment scout (commoditised consensus) | Surfaces broad industry consensus (e.g. low-cost index funds) as **observation**, never a personalised buy *(exists, InvestmentScout)* | `[deterministic]` | now | P2 |
| Tiered trade execution | Any brokerage trade is **always Tier C** (in-app confirm + step-up); v1 leaves execution off unless the user explicitly enables a broker link *(exists as Tier-C seam)* | `[integration:broker]` | needs-3rd-party | P2 |
| Tax-loss-harvest *candidate flagging* | Flag lots at a loss as candidates to review — **not** a recommendation to sell *(new; observation only)* | `[deterministic]` | now | P2 |

> **Hard line (compliance):** Artha never gives personalised investment advice or a personalised
> "buy/sell X". It shows the user's own data, opportunity-cost math, and *commoditised* consensus, always
> labelled as observation. Anything resembling a securities recommendation is out of scope (see §6).

### 2.10 Tax-prep support (organise, don't advise)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Deduction/category tagging | Tag likely-deductible transactions (charity, medical, business) into buckets *(extend, CategoryRule)* | `[deterministic]` | now | P1 |
| Marginal-rate estimator | Estimate marginal/effective rate for planning math (e.g. Roth vs traditional framing) *(exists, MarginalRateEstimator)* | `[deterministic]` | now | P2 |
| Year-end tax summary / export | Aggregate deductible buckets, 1099-style interest/dividends, into a CPA-ready export (CSV/PDF) *(new)* | `[deterministic]` | now | P1 |
| Estimated-tax / withholding sanity check | Rough "are we under-withheld" flag for planning *(exists, TaxAdvisor — framed as estimate)* | `[deterministic]` | now | P2 |
| Document shoebox | Store W-2/1099/receipt PDFs (MinIO) tagged by tax year *(new)* | `[deterministic]`+MinIO | now | P2 |

> Tax features **organise and estimate** for planning. Artha is not a tax preparer and gives no tax
> advice; the export is for the family or their CPA. (See §6.)

### 2.11 Shared vs personal money (the household model)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Per-user wall | Every read/write scoped to the signed-in user by the Kernel global query filter *(exists → Manas Kernel)* | `[deterministic]` | now | P0 |
| Account ownership & sharing scope | Each account is **personal** (owner-only) or **shared** (joint, visible to both adults) — explicit, not inferred *(new; extend Account/AccountGroup + Household)* | `[deterministic]` | now | P0 |
| Shared-expense split & settle-up | Track who paid shared bills, compute balances between Ananth and the other adult, suggest settle-up *(new)* | `[deterministic]` | now | P1 |
| Joint budgets & goals | Envelopes/goals owned by the *household* (both adults contribute & see) vs personal ones *(new; extend Goal/BudgetLine ownership)* | `[deterministic]` | now | P1 |
| Household net worth (opt-in aggregate) | Combined view **only** where both adults consent to share; each adult's private accounts stay walled *(new)* | `[deterministic]` | now | P1 |
| Cross-user privacy guarantee | Adults are walled from each other's *personal* money even at the household view; verified by property-based "wall never leaks" tests *(extend)* | `[deterministic]` | now | P0 |

> **Design note.** Manas walls are per-*user*, not per-household. "Shared" is modelled as an explicit
> sharing scope on an account/budget/goal — a joint object both adults own — never as a relaxation of the
> wall. The other adult's private savings never appear in Ananth's view, and vice-versa, unless the object is
> explicitly shared. This is the finance analogue of ADR 0001's per-user wall surviving the agent.

### 2.12 Kids' money — the child (allowance & chores)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Kid money account (virtual) | A parent-funded virtual ledger for the child — **not** a real bank/Plaid account; balances tracked in Artha *(new)* | `[deterministic]` | now | P1 |
| Allowance schedule | Recurring allowance credited automatically (e.g. weekly) from a parent's account as an internal transfer *(new)* | `[deterministic]` | now | P1 |
| Chores → earnings | Chores (optionally sourced from the Kaya/Abhyasa/Todoist side) mark complete → credit the child's ledger; parent approves *(new; Tier-B parent approval)* | `[deterministic]` | now | P1 |
| Spend / Save / Give buckets | Split the child's money into three envelopes to teach allocation *(new)* | `[deterministic]` | now | P2 |
| Savings goal for a kid | The child sets a goal (a toy); progress bar + "you need N more weeks" *(new; reuses Goal engine)* | `[deterministic]` | now | P2 |
| Child-safe money coaching | Age-appropriate, encouraging explanations of saving/spending; **child-safety-gated** persona *(new; reuse Vatara child-safety gate)* | `[LLM]` child-safe | now | P2 |
| Parent oversight & wall | Parents see and manage the child's ledger; the child sees **only her own** money, never household finances *(new; Kernel wall + role)* | `[deterministic]` | now | P0 |

> The child's world is a **teaching sandbox**, not banking. No real money movement is initiated by the child;
> everything is a parent-funded internal ledger with parent approval (Tier B) on credits. Real
> kid-banking cards (e.g. Greenlight) are explicitly **out of scope** (§6).

### 2.13 Coaching, conversation & reporting (the LLM skin)

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Conversational advisor | Ask "can we afford X / where did the money go / what should I pay first" → routes to deterministic tools, LLM phrases the answer *(exists, AdvisorService + 17 tools)* | `[LLM]` over `[deterministic]` | now | P0 |
| Numeric provenance guard | Every number the advisor speaks is traced to a deterministic tool output; the LLM can't invent figures *(exists, NumericProvenance)* | `[deterministic]` guard | now | P0 |
| Weekly / monthly review | Auto-generated plain-language digest of what happened + what to do *(exists, WeeklyReview)* | `[LLM]` over `[deterministic]` | now | P1 |
| Pre-purchase coaching | "Should I buy this $X?" → card, utilisation impact, interest, envelope hit, a hard "don't" if it breaches a cap *(exists, SpendCoach)* | `[deterministic]` (+`[LLM]` phrasing) | now | P1 |
| Anti-sycophancy / honesty guard | Delivery is personalised; **truth is not** — it will say "don't buy this" *(carry Vatara guardrail)* | `[LLM]` policy | now | P1 |
| Voice-complete access | Every capability reachable by voice through Sarathi (Akriti a11y, blind-first) *(new integration with platform Sarathi)* | `[integration:Sarathi/MCP]` | now | P1 |
| Dashboard & drill-downs | Glass-box surfaces: number → the calculation → the button *(exists, Dashboard/Money views)* | `[deterministic]` render | now | P0 |

### 2.14 Action, autonomy & safety spine

| Feature | Description | Kind | Feasibility | Priority |
|---|---|---|---|---|
| Tiered ActionDispatcher | Tier A auto (small, reversible) · Tier B one-tap HMAC-signed approval · Tier C in-app confirm + step-up *(exists)* | `[deterministic]` | now | P0 |
| Policy engine & caps | Per-user autonomy policy, spending caps, per-vendor & monthly cost caps via `ISpendGuard` *(exists)* | `[deterministic]` | now | P0 |
| Idempotency + outbox on every actuation | Exactly-once money movement / notification; no double-execute on retry *(extend → Kernel outbox)* | `[deterministic]` | now | P0 |
| Value ledger | Record the realised $ impact of each executed action (proof the family saved money) *(exists, ValueLedgerEntry)* | `[deterministic]` | now | P1 |
| Audit trail (tamper-evident) | Append-only, hash-chained who/what/before→after for every change *(extend → Saakshi)* | `[deterministic]` | now | P0 |
| Step-up on high-risk | Money movement / trades require a strong factor via Dwarpal; the LLM is never the gate *(extend → Dwarpal)* | `[deterministic]` | now | P0 |

---

## 3. Data-model sketch

Grounded in Steward's **existing** entities (`Steward.Core/Entities`), re-homed on `Manas.Kernel`'s
base entity (`TenantId` + `OwnerId` + provenance + audit interceptor + `[Encrypted]` value-converters).
`{new}` marks entities this vision adds.

```
Tenant / Household
  Household(id, tenantId, name)
  HouseholdMember(userId, householdId, role: Adult|Child, displayName)
  HouseholdProfile(filingStatus, dependents, riskPosture, buffers…)     // planning inputs

Identity & ownership (walls)
  AppUser(id, tenantId, role)                                            // Kernel identity
  SharingScope {new} (enum on ownable objects: Personal | Shared)        // per-user wall + explicit share
  AccountGroup(id, ownerId, sharingScope {new}, colour)                  // joint vs personal grouping

Accounts & instruments
  Account(id, ownerId, sharingScope {new}, type, institution,
          [Encrypted]mask, currency, plaidItemId?)
  PlaidItem(id, ownerId, cursor, consentAt, status)                      // + revoke/forget
  CreditCardDebt(accountId, apr, minimum, dueDate, statementBalance, limit)
  Holding / ManualHolding(accountId, symbol, qty, costBasis, assetClass)
  ValueLedgerEntry(actionId, realisedImpactUsd, at)                      // proof of savings

Transactions & recurrence
  Transaction(id, ownerId, accountId, [Encrypted]amount, merchant,
              normalizedMerchant, category, postedAt, pending, deductibleTag {new})
  RecurringPayment(id, ownerId, merchant, amount, cadence, confidence, lastSeen)
  CategoryRule(id, ownerId, matcher, category, deductibleBucket {new})

Budgets, goals, plan
  BudgetLine(id, ownerId|householdId, sharingScope {new}, category, period, amount)
  Envelope {new} (id, ownerId|householdId, name, assigned, spent, rollover: bool,
                  sinkingTargetDate?)                                    // zero-based/envelope
  Goal(id, ownerId|householdId, sharingScope {new}, name, target, byDate, funded)
  PlanItem / PlannedAction(id, ownerId, kind, tier, status, idempotencyKey {new})
  SpendingCap(id, ownerId, scope, limit, window)
  AutonomyPolicy(id, ownerId, tierRules, caps)

Bills & obligations
  BillCandidate(id, ownerId, source: recurring|email|statement, amount, dueDate, biller)
  RefundWatch(id, ownerId, merchant, expectedAmount, since)
  PaymentConfirmation(id, ownerId, billId, source, confirmedAt)

Kids' money {new}
  KidLedger {new} (id, childUserId, fundingParentId, balance)
  KidBucket {new} (ledgerId, kind: Spend|Save|Give, balance)
  Allowance {new} (childUserId, amount, cadence, fromAccountId, nextRunAt)
  ChoreEarning {new} (childUserId, choreRef, amount, status: pending|approved, approvedBy)

Detection & advice output
  Alert(id, ownerId, kind, severity, payload, explainedText?)
  Recommendation / ActionRequest(id, ownerId, kind, projectedImpact, tier)
  FinancialSnapshot(id, ownerId|householdId, at, assets, liabilities, netWorth)   // history

Market & cost
  MarketRate(source, key, value, at)                                    // HYSA APY, FX, benchmarks
  MeteredSpend(vendor, cost, at)                                        // ISpendGuard ledger
  AuditEvent → Saakshi (hash-chained, encrypted, redacted)             // platform-level
```

**Rules baked into the model:** ownership is `OwnerId` (per-user wall, Kernel filter) **plus** an
explicit `SharingScope` for the joint case — a shared object is co-owned by the household, never a
relaxed wall. Sensitive fields (`amount`, `mask`, tokens) are `[Encrypted]` via `Manas.Crypto`. Every
actuation carries an `idempotencyKey` and rides the outbox. Kids' ledgers are internal (no Plaid
account), child-walled, parent-approved.

---

## 4. Integrations needed (all via Sangam; all metered by `ISpendGuard` where they cost)

| Integration | Purpose | Trust tier | Notes / real limits |
|---|---|---|---|
| **Plaid — Transactions/Balances/Liabilities/Investments** | The primary read source | read | Free Sandbox/Dev; Production is paid per-item. **Read-only by default.** Investments coverage varies by institution. |
| **Plaid Transfer** *(optional)* | ACH between the family's *own* linked accounts (goal sweeps) | act (Tier A/B) | Requires Plaid money-movement approval; **account-to-account you own**, not arbitrary billers. Gate hard. |
| **Method** *(existing)* | Pay third-party **billers** (loans/cards/utilities) by ACH | act (Tier B/C) | Steward already integrates Method for bill-pay; this is how real bill payment happens (Plaid can't pay arbitrary US billers). |
| **Statement files (CSV/OFX/QFX/PDF)** | Fallback ingest for issuers Plaid can't reach | read | Deterministic parsers; PDF metadata extractor exists. |
| **Email (readonly, via Sangam)** | Bill/receipt/confirmation intake, USPS Informed Delivery digest | read | Facts extracted in memory; **only facts persisted**, never bodies. |
| **Market data — FRED / Frankfurter** *(existing)* | HYSA APY reference, CPI, FX rates | read | Public/free; cached via `MarketRateProvider`. |
| **Notification channels — Twilio / Pushover / SMTP / Slack / Discord** *(existing)* | Severity-tiered alerts & one-tap approvals | act (outbound) | Route through Sangam; approval links are HMAC-signed. |
| **Brokerage trade API** *(deferred)* | Tier-C trade execution *if* the user explicitly enables it | act (Tier C only) | Out of the box **off**; strong compliance surface. Prefer to stay read-only. |
| **Local LLM via Medha** | Intent, explanation, coaching, tail categorisation, exotic-PDF repair | cognition | Ollama on-box; `$0`; every call traced. Never computes money. |
| **MinIO (self-hosted)** | Statement/tax-document blob storage | store | Platform store; encrypted at rest. |

**No integration bypasses the walls or the spend guard.** Every acting integration is Tier-gated and
idempotent; every metered call is priced and capped or refused (fail-closed).

---

## 5. Interconnections with other Manas services

Artha is one mind among many. It exposes its data + actions as **MCP tools** (`artha.get_budget`,
`artha.get_networth`, `artha.get_bills_due`, `artha.coach_purchase`, `artha.record_kid_chore`,
`artha.propose_action`…) and consumes others' tools — always carrying the caller's verified identity,
never as a super-user.

| With | Interconnection | Direction |
|---|---|---|
| **Sarathi** | The one conversational front door. "Can we afford a trip?" / "what's due this week?" → Sarathi calls Artha tools; the LLM phrases, Artha computes. | Sarathi → Artha |
| **Dwarpal** | Step-up for money movement/trades; risk-based factor on Tier-C actions; low-assurance voice can *request* but not *execute* high-risk. | Artha ← Dwarpal |
| **Sangam** | All external grants (Plaid, email, Twilio, Pushover) live in Sangam's OAuth vault; Artha reaches them as governed connectors, reading vs acting as distinct tiers. | Artha → Sangam |
| **Saakshi** | Every detection, recommendation, action, LLM call and wall-denial is logged (redacted/encrypted, hash-chained). The value ledger + audit spine live here. | Artha → Saakshi |
| **Medha** | All LLM calls (advisor, explainer, coaching, tail categorisation) go through the shared gateway with QoS + spend/trace. | Artha → Medha |
| **Kutumba** | Household/members/roles (Ananth and the other adult as adults; the child) + the tenant seam. Artha reads *who exists* and their roles; it doesn't own identity. | Artha ← Kutumba |
| **Kaya** | Optional context: a big utility bill spike can cross-reference thermostat/energy usage; "the house is empty, is it worth heating?" framing. Read-only, opt-in. | Artha ↔ Kaya |
| **Abhyasa / Todoist (chores)** | Chore completion (from the school/tasks side) can trigger a chore-earning credit for the child — parent-approved. Artha is the money authority; Abhyasa/tasks own the chore. | Artha ← Abhyasa/tasks |
| **Ojas (health)** | Loose, opt-in: a large medical expense can be tagged deductible / flagged for an HSA/FSA envelope. **No health detail crosses into finance** beyond the amount the user chooses to tag. | Artha ↔ Ojas (minimal) |
| **Bodha** | Artha reports its own status/health (sources connected, last sync, analyzers enabled) to the platform self-model. | Artha → Bodha |

**Guardrail on interconnection:** cross-service reads carry the acting user and hit *the other service's
API*, never its store. Health→finance and home→finance links are opt-in and minimal by design — the
walls and the "no service reads another's store" rule (ADR 0001 §5.4) hold across every edge.

---

## 6. Out of scope + why

| Not doing | Why |
|---|---|
| **Personalised licensed investment advice** (personalised "buy/sell security X", allocation prescriptions, market timing) | Requires being a Registered Investment Adviser; huge liability. Artha gives **observations + commoditised consensus + the user's own opportunity-cost math**, always labelled — never tailored advice. (README disclaimer stands.) |
| **Tax advice / acting as a tax preparer** | Regulated (Circular 230 territory). Artha **organises and estimates** for planning and produces a CPA-ready export; the family or their CPA decides and files. |
| **Personalised insurance/legal advice** | Same liability class. Scouts surface *comparative options* as information, not a recommendation to switch. |
| **Being a bank / holding funds / issuing cards** | Money-transmission & banking licensing. Artha *initiates* movement through licensed partners (Method / Plaid Transfer) between the family's own accounts / known billers — it never custodies money. |
| **Real kid debit cards / kid neobank (Greenlight-style)** | Licensing + KYC for a minor; out of scope. The child's money is an **internal teaching ledger**, parent-funded, no real card. |
| **Auto-cancelling contracts / auto-negotiating bills on the user's behalf** | High blast-radius, often needs identity/authority we shouldn't automate. Artha drafts the cancellation and surfaces the steps; **the user completes it**. |
| **The LLM computing any dollar figure or being a security boundary** | Core ADR 0001 rule. Numbers are deterministic and provenance-checked; walls/step-up/tiers are deterministic code. A hallucinated figure or a curious child fails at the data layer. |
| **Arbitrary autonomous money movement without tiers** | Every actuation is Tier-gated, capped (`ISpendGuard` + `SpendingCap`), idempotent, step-up'd for high-risk, and value-ledgered. No silent transfers. |
| **Cloud storage of financial data** | Local-first (ADR 0001 §2). Data and inference stay on the home server; the cloud is a blind relay only. Off-site backup is **client-side encrypted**. |
| **Credit-repair / credit-report pulls / lending** | Regulated (FCRA); not the product. Artha reads what Plaid/statements show and optimises utilisation with the user's own data. |
| **Cryptocurrency trading/custody** | Out of scope beyond a **manual holding** line for net-worth completeness. |

**Compliance/liability posture (summary).** Artha is *software, not a financial/tax/investment/legal
advisor*. It surfaces information, deterministic math, and industry-consensus observations, and acts only
within explicit user-set tiers through licensed partners between the family's own accounts. High-stakes
decisions defer to a human. This posture, plus local-first data residency and per-user walls, keeps the
liability surface minimal — and a legal review precedes any productisation (ADR 0001 §12).

---

*Deepen-and-extend map: ~70% of the capability rows above already exist in the current Steward solution
and are carried forward; the genuinely new depth is **envelopes/zero-based budgeting**, a **first-class
shared-vs-personal money model**, **the child's allowance/chores ledger**, **year-end tax export + document
shoebox**, **autopay/ghost-subscription audits**, and the **platform re-homing** (Kernel walls, Sangam
connectors, Saakshi audit, Medha LLM gateway, MCP tools, idempotency/outbox). Everything is realistically
feasible on the .NET-10 / the home server stack, deterministic-first, with the LLM kept to a thin conversational
skin.*
