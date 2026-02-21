# Product Manager Review — Implementation Instructions (LoanBid Exchange)

Date: 2026-02-21  
Owner: Product Management  
Context reviewed: PRD v2, Architecture v2, GTM v2, 90-day roadmap, implementation plan

## Executive Assessment
The current plan is strong and coherent. The biggest execution risk is **trying to launch “commercial readiness” before product-market and lender-liquidity stability are proven**. Recommendation: keep V1 narrow, make launch gates strict, and defer monetization complexity and medium-risk checks until after pilot quality thresholds are consistently met.

---

## 1) Refined MVP Scope (What to Build Now)

### Must Have (MVP release-critical)
1. **Borrower intake (mobile-first)**
   - Amount, purpose, tenor, lightweight profile
   - Consent capture with versioned disclosure (indicative vs final terms)
2. **KYC-lite boundary checks**
   - Identity basic checks and anti-fraud minimum controls
3. **Rule-based routing + bid window orchestration**
   - Lender eligibility/appetite rules
   - SLA timers, offer expiry handling
4. **Standardized indicative offer schema**
   - APR/effective rate, fees, tenor, monthly estimate, conditions, validity
5. **Anonymized offer board + explainability**
   - Rank by transparent weighted model
   - Show “why this is ranked here” snippets
6. **Shortlist and lender identity reveal flow**
   - Controlled reveal only post-shortlist
7. **Immutable audit trail + analytics instrumentation**
   - Event logging for consent, ranking, reveal, and outcomes
8. **Lender integration kit + contract tests**
   - Sandbox + certification checklist before production onboarding

### Should Have (Pilot-critical but can launch slightly after MVP go-live)
1. Lender portal with response SLA and conversion dashboard
2. Basic lender quality scorecards (response rate, expiry rate, shortlist-to-fund)
3. Manual ops console for exception handling (late bids, malformed offers)

### Could Have (Post-MVP, V1.5)
1. Open banking/bank statement ingestion
2. Medium-depth checks and fraud model enhancement
3. Dynamic ranking personalization by borrower preference

---

## 2) Explicit Non-Goals (Protect Scope)
1. No binding final underwriting decision inside marketplace in V1
2. No multi-country expansion in first release
3. No advanced ML underwriting/risk automation in V1
4. No borrower-side negotiation/chat with lenders in V1
5. No complex revenue settlement automation before pilot conversion reliability is proven
6. No broad acquisition scale-up before lender response SLA is stable for 2+ consecutive weeks

---

## 3) KPI Framework (with launch thresholds)

## North Star (V1)
- **Qualified borrower request → funded loan conversion**

## Operational KPIs (weekly monitored)
1. **Median request-to-first-offer time**: target < 10 min (gate), stretch < 5 min
2. **Offers per qualified request**: target >= 2.5 (gate), stretch >= 3.5
3. **Borrower shortlist rate**: target >= 18% (gate), stretch >= 25%
4. **Qualified funded conversion**: target >= 8% (gate), stretch >= 12%
5. **Offer expiry before borrower view**: target <= 5%
6. **Malformed/invalid lender offer submissions**: <= 1% of submissions
7. **Consent/disclosure traceability completeness**: 100%
8. **Borrower drop-off (intake start to submit)**: <= 35% in pilot

## Commercial KPIs (post-launch readiness)
1. Revenue per qualified request (lead + success fee)
2. Lender ROI proxy (funded outcomes / billed leads)
3. Repeat lender participation rate week-over-week

---

## 4) Launch Gates (Go/No-Go)

## Gate A — Compliance & Data Contract Freeze (Day ~15)
- Signed approval of disclosure copy and policy versioning process
- Schema v1.0 frozen for `BorrowerRequest`, `LenderOffer`, `Outcome`
- Audit export demonstrates complete request timeline replay

## Gate B — Technical Readiness (Day ~30)
- Staging E2E with synthetic lenders passes all critical flows
- Contract tests passing for at least 3 lender adapters (or simulators)
- Peak bid-window load test meets defined p95 latency SLOs

## Gate C — Pilot Stability (Day ~60)
- 3–5 lenders live and SLA-compliant for 2 consecutive weeks
- KPI minima met: <10 min first offer, >=2.5 offers/request, >=18% shortlist
- No unresolved Sev-1 compliance/security issues

## Gate D — Commercial Launch Readiness (Day ~90)
- 8+ active lenders available (not just signed)
- Qualified funded conversion >=8% on controlled traffic cohort
- Runbooks validated (incident, outage, dispute, lender escalation)
- Executive sign-off packet includes KPI trendline, risk register, and rollback triggers

---

## 5) Roadmap Sequencing Recommendations

## Sequence Principle
Prioritize **marketplace liquidity + trust + conversion signal** before monetization optimization.

## Recommended sequence
1. **Now (Days 1–30):** consent/legal freeze, schema freeze, orchestration core, anonymized comparison UX, telemetry baseline
2. **Next (Days 31–60):** controlled pilot with 3–5 lenders, fix liquidity gaps, tighten ranking explainability, reduce drop-off
3. **Next+ (Days 61–90):** selective lender scale (to 8+ active), introduce simple billing mechanics, publish lender scorecards
4. **Later (Post-90):** medium checks (open banking), fraud sophistication, monetization refinement, broader borrower acquisition

## Critical sequencing changes vs current draft
- Defer complex settlement logic until pilot conversion floor is validated
- Make lender quality scorecards available before aggressive volume ramp
- Keep GTM spend throttled by real-time liquidity and shortlist performance

---

## 6) Decision Log Recommendations (to add immediately)

Add the following decision entries to the program log:

| Date | Decision | Rationale | Trigger to Revisit |
|---|---|---|---|
| 2026-02-21 | Freeze V1 to indicative-only offers | Minimize regulatory and integration complexity | Revisit after 3 months of funded-conversion data |
| 2026-02-21 | Keep anonymized ranking until shortlist | Preserve neutral marketplace behavior and reduce brand bias | Revisit if trust metrics drop or lender-brand pull proves dominant |
| 2026-02-21 | Gate acquisition by liquidity KPIs | Prevent poor borrower experience during low supply periods | Revisit when active lenders >=12 with stable SLA |
| 2026-02-21 | Defer medium-risk checks to V1.5 | Protect 90-day timeline and reduce implementation risk | Revisit after pilot fraud/quality baseline |
| 2026-02-21 | Use rules-first ranking with explainability | Faster to ship and auditable in regulated context | Revisit after sufficient labeled outcomes for ML |
| 2026-02-21 | Launch with prime + near-prime ICP only | Improve early conversion reliability and lender confidence | Revisit after stable funded conversion >=10% |

---

## 7) Implementation Instructions to Product + Engineering
1. Treat disclosure and consent versioning as release-blocking artifacts, not copy tasks.
2. Build instrumentation before pilot traffic: every funnel stage must be measurable on day 1.
3. Enforce connector certification: no lender moves to production without passing schema and error-path tests.
4. Add borrower-facing ranking reason snippets in MVP (trust-critical).
5. Define explicit rollback triggers (e.g., offers/request <2.0 for 3 consecutive days).
6. Run weekly joint review (Product + Ops + Compliance + Eng) with one KPI scoreboard and one risk register.

---

## 8) Top Risks to Watch (PM view)
1. **Liquidity illusion risk:** signed lenders but low real response rate.
2. **Disclosure confusion risk:** borrower misunderstands indicative status and drops.
3. **Integration fragility risk:** malformed offers at peak windows degrade trust quickly.
4. **Premature scale risk:** paid acquisition outruns lender response capacity.

Mitigation priority: liquidity controls, explainability UX, strict connector certification, and staged demand throttling.