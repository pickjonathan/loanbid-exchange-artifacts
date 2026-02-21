# Implementation Plan — LoanBid Exchange (AI Marketplace)

## 1) Objective
Deliver V1 of the Israel consumer-loan AI marketplace in 90 days with: indicative pre-offers, anonymized ranking, lender reveal on shortlist, and hybrid monetization readiness.

## 2) Scope Baseline (Locked)
- Geography: Israel
- Segment: Consumer personal loans
- Offer type: Indicative pre-offers (non-binding)
- UX: Anonymized offers first, lender identity after shortlist
- Revenue: Lead fee + funded success fee

## 3) Delivery Workstreams

### WS1 — Product & Compliance Foundation (Days 1–15)
**Goals**
- Finalize V1 user flows and disclosure text
- Define legal/compliance controls for consent, KYC-lite, auditability
- Freeze canonical data contracts

**Key Deliverables**
- Approved borrower/lender journeys
- Consent + disclosure policy pack (versioned)
- Data contracts: `BorrowerRequest`, `LenderOffer`, `Outcome`
- Acceptance test scenarios for compliance boundaries

**Exit Criteria**
- Compliance sign-off on indicative-offer disclosures
- Contract schema v1.0 tagged and shared with all teams

---

### WS2 — Marketplace Core Platform (Days 1–45)
**Goals**
- Build core backend services and event-driven orchestration
- Ensure idempotent bid ingestion and immutable audit events
- Provide ranking with borrower-facing explanations

**Core Components**
1. Borrower Intake Service (request capture, consent ref, KYC-lite hooks)
2. Routing Engine (eligibility + lender appetite rules)
3. Bid Orchestrator (invites, SLA windows, expiry handling)
4. Lender Connector API (signed requests, sandbox adapters)
5. Ranking Service (weighted score + explainability snippets)
6. Audit/Event Ledger (append-only, exportable)

**Technical Gates**
- P95 bid ingestion latency and offer-board response SLOs defined + monitored
- Contract tests for lender API adapters
- Replayable event timeline for request lifecycle

**Exit Criteria**
- End-to-end staging run with synthetic lenders
- No critical defects in malformed/late/expiry edge scenarios

---

### WS3 — Borrower & Lender Experience (Days 15–55)
**Goals**
- Mobile-first borrower flow with low abandonment
- Lender portal/API visibility into submission status and outcomes

**Key Deliverables**
- Borrower intake UI + consent checkpoints
- Anonymized offer board with cost breakdown and ranking reasons
- Shortlist + lender reveal handoff flow
- Lender dashboard: response SLA, conversion metrics

**Exit Criteria**
- UX funnel baseline instrumented end-to-end
- Controlled usability pass with no high-severity blockers

---

### WS4 — Pilot Operations & GTM Activation (Days 31–75)
**Goals**
- Onboard initial lenders and launch controlled borrower pilot
- Validate liquidity and operational SLA performance

**Key Deliverables**
- 3–5 lender integrations live in pilot
- Traffic throttling and quality filters enabled
- Runbooks for incident response and lender escalation

**Pilot Success Metrics**
- Median request→first offer under 10 minutes
- 2.5+ offers per request
- 18%+ shortlist rate on qualified traffic

**Exit Criteria**
- Stable pilot for at least 2 consecutive weeks
- No unresolved critical compliance issues

---

### WS5 — Commercial Readiness (Days 61–90)
**Goals**
- Prepare monetization and scale capacity for launch
- Add lender scorecards and initial fraud heuristics

**Key Deliverables**
- Settlement logic for lead fee + funded success fee
- Lender quality scorecards (response + close reliability)
- Fraud heuristic checks (rules-based V1)
- Launch readiness review + V1.5 backlog (medium checks)

**Exit Criteria**
- 8–12 active lenders ready/active
- Funded conversion baseline established and trackable
- Go/No-Go pack approved

## 4) Milestone Plan
- **M1 (Day 15):** Compliance + schema freeze
- **M2 (Day 30):** Core E2E in staging (synthetic lenders)
- **M3 (Day 60):** Live controlled pilot stable
- **M4 (Day 90):** Commercial launch readiness

## 5) Dependency Map
1. Compliance/disclosure decisions block UI copy and consent flow.
2. Schema freeze blocks lender adapters and QA contract tests.
3. Lender onboarding speed controls marketplace liquidity and KPI attainment.
4. Audit pipeline completion is required before broad pilot expansion.

## 6) QA & Release Gates
- Functional: all critical borrower/lender flows pass
- Contract: lender API compatibility tests pass per connector
- Reliability: peak bid-window load tests meet SLO
- Security/Compliance: consent traceability + immutable logs verified
- Analytics: funnel and SLA telemetry complete and trusted

## 7) Risks & Mitigations
1. **Low lender response coverage**
   - Mitigation: SLA-backed participation terms, routing fallback, staged traffic caps.
2. **Compliance ambiguity in disclosures**
   - Mitigation: early legal review, policy versioning hooks, mandatory copy freeze gate.
3. **Integration variability across lenders**
   - Mitigation: strict contract tests, adapter abstraction, sandbox certification checklist.
4. **Borrower trust/conversion drag**
   - Mitigation: transparent cost explanations, clear “indicative” labeling, UX simplification.
5. **Operational instability under peak windows**
   - Mitigation: load tests before pilot scale, observability + rollback playbooks.

## 8) Team Execution Model
- Weekly governance: KPI review, blocker resolution, lender onboarding status
- Parallelization strategy:
  - Parallel: product/compliance design + backend scaffolding
  - Sequential gates: schema freeze → integrations → pilot scale
- Decision cadence:
  - Daily standup for execution blockers
  - Weekly risk review with go/no-go adjustments

## 9) Launch KPIs (Initial Targets)
- 8+ active lenders
- 2.5+ offers/request
- <10 min median request→first offer
- 18%+ borrower shortlist
- 8%+ funded conversion on qualified requests

## 10) Immediate Next 10 Actions (Week 1)
1. Approve V1 scope lock and non-goals.
2. Finalize schema v1.0 and publish contract docs.
3. Freeze legal disclosure copy (indicative vs final).
4. Create lender integration kit (API spec + sandbox + certification checklist).
5. Stand up event ledger and audit export prototype.
6. Implement intake skeleton with consent reference IDs.
7. Build routing rules MVP with configurable appetite settings.
8. Instrument baseline metrics (request, offer, shortlist, handoff).
9. Draft pilot operating runbook (SLA breach, late bids, outage handling).
10. Confirm first cohort of 3–5 lenders and onboarding timeline.
