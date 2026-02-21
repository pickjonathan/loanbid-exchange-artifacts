# Backend Implementation Review & Execution Instructions

## 1) Scope (Backend-Owned)

### In scope for V1 (90 days)
- Borrower Intake API (request capture, consent reference, KYC-lite hook state)
- Lender-facing Bid API + webhook/callback ingestion
- Routing engine (rules-based eligibility/appetite)
- Bid orchestration (invite window, SLA tracking, expiry handling)
- Ranking service (weighted score + explainability payload)
- Anonymization/reveal workflow (lender hidden until shortlist)
- Immutable audit/event ledger
- Outcome tracking + monetization-ready settlement events
- Analytics event stream for funnel + SLA KPIs

### Out of scope (explicit)
- Binding underwriting decisions
- Full bureau/open-banking medium/deep checks (V1.5+)
- Multi-country localization beyond Israel

---

## 2) Architecture Decisions (Actionable)

1. **Service style:** Modular monolith first (NestJS) with strict domain modules; event-driven internally (outbox pattern) to avoid distributed-system overhead in 90-day window.
2. **Primary data store:** PostgreSQL for transactional state + immutable events.
3. **Cache/ephemeral state:** Redis for bid windows, SLA timers, idempotency short-TTL keys.
4. **Async messaging:** Postgres outbox + worker (BullMQ/Redis) for orchestration; upgrade path to Kafka later.
5. **PII boundary:** Tokenize PII in `borrower_profile` and keep offer/ranking pipelines PII-minimized.
6. **Connector pattern:** One canonical lender contract + adapter layer per lender.
7. **Auditability:** Append-only `audit_events` table with hash-chain field (`prev_hash`, `event_hash`) for tamper evidence.
8. **Reliability principle:** Idempotent writes on every external callback and lender submission.

**Suggested module map (NestJS):**
- `intake`
- `routing`
- `orchestration`
- `offers`
- `ranking`
- `lender-connectors`
- `audit`
- `outcomes`
- `settlement`
- `analytics`
- `auth/admin`

---

## 3) API / Contracts (v1.0)

## External APIs

### Borrower Intake
- `POST /v1/borrower/requests`
  - Input: amount, tenorMonths, purpose, profileLight, consentVersionId, consentAcceptedAt
  - Output: requestId, status=`RECEIVED`, nextStep
- `GET /v1/borrower/requests/:id/offers`
  - Output: anonymized ranked offers + explainability + expiry
- `POST /v1/borrower/requests/:id/shortlist`
  - Input: selectedOfferIds[]
  - Output: revealed lender identities + redirect URLs

### Lender API
- `POST /v1/lender/offers` (signed)
  - Input: requestId, lenderExternalRef, apr, fees, tenorMonths, monthlyEstimate, conditions, validUntil
  - Output: offerId, accepted=true
- `POST /v1/lender/outcomes` (signed, async updates)
  - Input: offerId/requestId, status(shortlisted|accepted|funded|declined), fundedAmount?, timestamp
- `GET /v1/lender/requests/open` (optional pull model for smaller lenders)

## Internal Events (canonical)
- `BorrowerRequestCreated`
- `ConsentCaptured`
- `RoutingEvaluated`
- `LenderInvited`
- `OfferSubmitted`
- `OfferExpired`
- `RankingComputed`
- `BorrowerShortlisted`
- `LenderRevealed`
- `OutcomeUpdated`
- `SettlementAccrued`

## Contract rules
- Versioned JSON schema per endpoint/event (`v1.0.0`)
- Idempotency key required on lender write endpoints
- All timestamps ISO-8601 UTC
- Currency fixed to ILS in V1
- Signature header required for lender-originating calls

---

## 4) Data Model (Initial)

## Core tables
- `borrower_requests` (id, amount_ils, tenor_months, purpose, profile_light_json, consent_ref, status, created_at)
- `borrower_pii_tokens` (request_id, pii_token, vault_ref, created_at)
- `lenders` (id, legal_name, anon_label, status, api_mode, sla_seconds)
- `lender_appetite_rules` (id, lender_id, rule_json, priority, active)
- `lender_invitations` (id, request_id, lender_id, invited_at, expires_at, status)
- `offers` (id, request_id, lender_id, apr_bps, fees_ils, tenor_months, monthly_estimate_ils, conditions_json, valid_until, submitted_at, state)
- `rankings` (id, request_id, computed_at, weights_json, version)
- `ranking_items` (ranking_id, offer_id, score_total, score_breakdown_json, reason_text)
- `shortlists` (id, request_id, borrower_action_at)
- `shortlist_items` (shortlist_id, offer_id, revealed_at)
- `outcomes` (id, request_id, offer_id, lender_id, status, funded_amount_ils, updated_at)
- `settlement_entries` (id, lender_id, request_id, outcome_id, fee_type[lead|success], amount_ils, status)
- `audit_events` (id, aggregate_type, aggregate_id, event_type, payload_json, occurred_at, prev_hash, event_hash)
- `idempotency_keys` (key, scope, first_seen_at, response_fingerprint)

## Required indexes
- offers(request_id, state)
- offers(valid_until)
- lender_invitations(request_id, lender_id)
- outcomes(request_id, status)
- audit_events(aggregate_type, aggregate_id, occurred_at)
- idempotency_keys(key, scope) unique

---

## 5) Milestones & Backend Deliverables

### M1 (Day 15) — Schema + compliance freeze
- DB schema v1 migrations merged
- OpenAPI + JSON schemas published
- Consent/disclosure version fields enforced in API
- Signed-request verification middleware working

### M2 (Day 30) — Core E2E staging
- Intake → routing → invite → offer ingest → ranking → shortlist happy path
- Synthetic lenders integrated via adapter test harness
- Audit ledger populated for every lifecycle step
- Contract tests green for canonical lender API

### M3 (Day 60) — Live pilot stable
- 3–5 real lenders onboarded
- SLA timers/expiry behavior verified in production-like load
- KPI dashboards: first-offer latency, offers/request, shortlist rate
- Incident/runbook readiness complete

### M4 (Day 90) — Commercial readiness
- Outcome ingestion + settlement accrual live
- Lender reliability score inputs captured
- Fraud heuristics v1 (rules) active
- Go-live checklist signed (security, compliance, reliability)

---

## 6) Key Risks & Mitigations

1. **Lender integration inconsistency**
   - Mitigation: strict contract certification, sandbox replay suite, adapter fallback queue.
2. **Race conditions in bid window/expiry**
   - Mitigation: Redis-based distributed locks + single source of truth in DB states.
3. **Ranking disputes / transparency issues**
   - Mitigation: persist weight version + reason snippets per offer; expose explanation API.
4. **Compliance trace gaps**
   - Mitigation: enforce event emission in service layer transaction; reject state change without audit write.
5. **Low offer liquidity at pilot start**
   - Mitigation: routing thresholds, controlled traffic caps, lender SLA monitoring + escalation.

---

## 7) Acceptance Criteria (Backend)

### Functional
- For a valid request, system returns anonymized ranked offers when >=1 offer received within window.
- Lender identity remains hidden until shortlist action succeeds.
- Late/expired offers are excluded from active board and marked with terminal state.
- Outcome updates transition state machine legally (no invalid jumps).

### Non-functional
- P95 `POST /borrower/requests` < 400ms (excluding external KYC providers)
- Median request→first-offer < 10 min in pilot conditions
- 99.9% idempotency correctness on duplicate lender submissions
- Full request lifecycle reconstructable from audit events

### Compliance/Security
- Consent version + timestamp persisted for 100% of borrower requests
- All lender-originated writes are signature-verified and logged
- PII never present in ranking/analytics payloads
- Immutable audit export available for legal/regulator review

---

## 8) Immediate Backend Task List (Next 2 Weeks)

1. Finalize OpenAPI spec for borrower + lender endpoints.
2. Implement base NestJS modules and shared validation/error envelope.
3. Create initial Postgres migrations for core tables + indexes.
4. Implement idempotency middleware + persistence.
5. Build signed-request auth for lender endpoints.
6. Implement routing engine MVP with DB-driven rule config.
7. Implement offer ingestion pipeline + expiry worker.
8. Build ranking v1 (weighted formula + reason generator).
9. Add append-only audit event service with hash-chain.
10. Stand up contract/integration test harness with synthetic lender adapters.

---

## 9) Recommended Definition of Done (per feature)
- API documented in OpenAPI + examples
- Validation and auth enforced at boundary
- Migration included (if schema changed)
- Unit + integration tests passing
- Audit events emitted and verifiable
- Dashboards/logs updated for the new flow
- Runbook note added for failure modes
