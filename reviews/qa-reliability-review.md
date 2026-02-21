# QA & Reliability Implementation Instructions — LoanBid Exchange (V1)

## 1) QA Charter & Quality Risks

### Charter
Provide release confidence for an Israel-first, consumer-loan marketplace where borrower trust, compliance traceability, and bid-window reliability are critical to business viability.

### Highest-Risk Failure Modes
1. **Compliance/consent gaps** (cannot prove lawful data use and disclosure state at decision time).
2. **Bid-window instability** (late/missing offers causing poor liquidity and conversion drop).
3. **Ranking correctness defects** (incorrect sorting, stale offers shown, non-explainable outcomes).
4. **Integration drift across lender adapters** (schema mismatches, signature/validation failures).
5. **Telemetry blind spots** (cannot trust funnel/SLA metrics for go/no-go decisions).

---

## 2) Test Strategy (Pyramid + Stage Ownership)

Target distribution for V1:
- **70% Unit tests** (core logic isolation)
- **20% Integration tests** (service boundaries + contracts)
- **10% E2E tests** (critical journeys only)

### A) Unit Test Scope
- Routing eligibility rules and lender appetite matching.
- Ranking score computation and tie-break behavior.
- Offer expiry/validity math (UTC-safe, boundary times).
- Idempotency keys and duplicate bid suppression logic.
- Consent/disclosure state machine and policy version binding.

### B) Integration Test Scope
- Borrower intake → orchestrator → offer board API chain.
- Lender connector contract tests (happy + malformed + auth/signature failures).
- Event ledger append-only guarantees and replay timeline integrity.
- Shortlist/reveal flow: lender identity remains hidden pre-shortlist and revealed post-shortlist only.
- Outcome ingestion (accepted/funded/declined) and settlement event generation.

### C) End-to-End Test Scope
Critical paths only:
1. Borrower submits request, receives anonymized offers, shortlists, identity reveal, lender handoff.
2. Lender submits indicative pre-offer within SLA; appears on board with explanation snippet.
3. Expired offer never shown as active and cannot be shortlisted.
4. Disclosure copy visible and policy version recorded for every borrower request.

### D) Non-Functional Testing
- **Load**: peak bid-window ingestion and concurrent offer-board reads.
- **Soak**: 24h stability with synthetic lender traffic.
- **Resilience**: connector timeout/partial outage, message retry behavior, idempotent recovery.
- **Security/Compliance checks**: PII tokenization boundary and audit export completeness.

---

## 3) Reliability SLOs, SLIs, and Error Budgets

## V1 Production SLOs (must be instrumented before pilot expansion)
1. **Request → First Offer Latency**
   - SLI: time from `borrower_request_created` to first valid offer on board.
   - SLO: **P50 < 10 min**, **P95 < 20 min** (qualified traffic).
2. **Bid Ingestion API Latency**
   - SLI: lender `POST /offers` server processing latency.
   - SLO: **P95 < 800 ms**, **P99 < 1500 ms**.
3. **Offer Board API Latency**
   - SLI: borrower fetch ranked offer board.
   - SLO: **P95 < 1200 ms**, **P99 < 2500 ms**.
4. **Bid Orchestration Availability**
   - SLI: successful invite/collect cycle completion per request window.
   - SLO: **99.9% monthly**.
5. **Audit Event Durability**
   - SLI: required compliance events persisted and queryable.
   - SLO: **99.99% event write success**, **0 tolerated missing compliance events**.

### Error Budget Policy
- If any monthly SLO consumes >50% budget mid-cycle: freeze non-essential feature releases.
- If >100% consumed: reliability-only sprint + executive go/no-go review.

---

## 4) Release Gates (Stage-by-Stage)

## Gate A — Schema & Compliance Freeze (M1 / Day 15)
Must pass:
- Canonical schemas (`BorrowerRequest`, `LenderOffer`, `Outcome`) version-tagged.
- Contract test harness published; baseline tests passing.
- Indicative-vs-final disclosure text approved and policy-versioned.
- Consent traceability test: every request maps to `consentRef` + disclosure version.

## Gate B — Core Staging E2E (M2 / Day 30)
Must pass:
- End-to-end synthetic-lender flow green for critical journeys.
- Late/malformed/duplicate/expired offer scenarios pass expected outcomes.
- Replayable event timeline validated for ≥ 99.9% of sampled requests.
- No Sev-1/Sev-2 defects open in routing, ranking, consent, or audit domains.

## Gate C — Pilot Stability (M3 / Day 60)
Must pass:
- 2 consecutive weeks pilot stability.
- SLOs met for latency + orchestration availability.
- Pilot KPI floor maintained: **2.5+ offers/request**, **18%+ shortlist** (qualified traffic).
- Incident runbook drill completed (connector outage + rollback simulation).

## Gate D — Commercial Readiness (M4 / Day 90)
Must pass:
- 8–12 lender connectors certified via contract suite.
- Settlement events reconcile against outcomes with zero critical mismatches.
- Telemetry trust checks complete (dashboard↔raw-event reconciliation within 1%).
- Security/compliance sign-off on audit export and PII boundary controls.

---

## 5) Scenario Matrix (Minimum Required Coverage)

1. **Low Liquidity**: only 0–1 lender responds; board handles sparse offers gracefully.
2. **Late Bids**: bids after SLA window are stored but excluded/flagged from active ranking.
3. **Malformed Offers**: schema violations rejected with precise error code; no corrupt persistence.
4. **Duplicate Submissions**: same idempotency key returns prior result; no duplicate ranking entries.
5. **Offer Expiry Edge**: expiry exactly at boundary timestamp; deterministic inclusion/exclusion.
6. **Ranking Tie Cases**: equal total cost; tie-break by affordability/confidence per policy.
7. **Consent Withdrawn**: routing and lender invites halted; audit evidence complete.
8. **Disclosure Version Change Mid-Day**: new requests bind to new policy; historical requests unchanged.
9. **Connector Timeout**: one lender adapter down; others unaffected; borrower board still renders.
10. **Identity Reveal Guardrail**: lender identity inaccessible before shortlist action.
11. **Outcome Backfill**: late funded outcomes update analytics + settlement idempotently.
12. **Audit Export Drill**: regulator-style export is complete, ordered, and immutable.

---

## 6) Acceptance Criteria (Definition of Done for QA Sign-off)

A release candidate is accepted only when all are true:
1. **Functional correctness**: all critical borrower/lender flows pass E2E in staging and pilot.
2. **Contract integrity**: every active lender connector passes full contract suite (incl. negative tests).
3. **Reliability**: SLO dashboards are live, validated, and within target for the required stabilization window.
4. **Compliance evidence**: consent/disclosure/audit trace is complete per sampled request set (100% pass on required fields).
5. **Data quality**: funnel and SLA metrics reconcile to raw events within ±1%.
6. **Defect bar**: zero open Sev-1/Sev-2 defects; Sev-3 only with approved risk waivers.
7. **Operational readiness**: on-call playbooks tested; rollback + incident comms drill completed.

---

## 7) Execution Plan for QA Team (Practical)

- **Week 1–2**: Build test plan, scenario catalog, contract harness skeleton, synthetic data fixtures.
- **Week 3–4**: Implement integration suite and event-replay assertions; start load baseline.
- **Week 5–6**: E2E critical journeys + resilience tests; wire SLI telemetry checks into CI.
- **Week 7–9**: Pilot monitoring, defect triage, runbook drills, KPI/SLO weekly sign-off packets.
- **Week 10–12**: Commercial gate validation, connector certification sweep, final go/no-go evidence pack.

---

## 8) CI/CD Quality Controls

Mandatory pipeline order:
1. Lint + static analysis
2. Unit tests + coverage thresholds
3. Integration + contract tests (including adapter negative cases)
4. E2E smoke for critical borrower/lender flows
5. Load smoke (short profile) on release candidates
6. Security and compliance checks (policy version + audit event assertions)

Block merge/deploy if any gate fails.

---

## 9) Recommended Metrics Dashboard (Launch-Critical)

- Request→first offer (P50/P95)
- Offers per request (overall + qualified traffic)
- Shortlist rate and lender handoff completion
- Offer expiry rejection rate
- Connector error rate by lender
- Missing/invalid audit event rate
- Consent traceability pass rate
- Ranking explanation coverage rate

This dashboard is a required artifact for Go/No-Go reviews.
