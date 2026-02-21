# Frontend Implementation Review — LoanBid Exchange (V1)

## 1) Frontend POV Summary
The plan is strong on business/compliance scaffolding and backend orchestration. For frontend execution, the highest leverage is to make the borrower flow feel fast, trustworthy, and low-friction while accurately communicating that offers are **indicative**. V1 UI should optimize for:
- Completion of intake (low abandonment)
- Fast first offer perception (progress + expectations)
- Confident shortlist action (clear comparisons + ranking explanations)
- Clean handoff to lender underwriting (measured conversion)

---

## 2) UX Flow (Borrower + Lender-facing touchpoints)

### A. Borrower journey (primary)
1. **Entry / intent capture**
   - Landing route with concise value proposition, trust badges, and “Get offers” CTA.
   - Pre-CTA disclosure: “Indicative offers, final terms determined by lender underwriting.”

2. **Step 1: Loan request basics**
   - Fields: amount, purpose, tenor.
   - Instant validation + examples/ranges.
   - Progress indicator (e.g., Step 1 of 4).

3. **Step 2: Basic profile (KYC-lite inputs)**
   - Minimal required personal and financial fields only.
   - Explain why each sensitive field is requested.
   - Save draft locally in-session to reduce drop-off.

4. **Step 3: Consent + disclosures**
   - Explicit, versioned legal copy with checkbox + timestamp.
   - Required acknowledgement for data sharing and indicative nature.

5. **Step 4: Matching in progress**
   - Non-blocking waiting screen with SLA expectation (e.g., “first offers typically in <10 min”).
   - Real-time status chips: invited lenders / offers received / expiry countdown.

6. **Offer board (anonymized ranking)**
   - Card/table hybrid for mobile + desktop.
   - Mandatory normalized fields: APR/effective annual rate, fees, tenor, monthly estimate, conditions, validity window.
   - Ranking explanation snippet per offer (e.g., “Higher due to lower total cost and better reliability score”).
   - Sort/filter controls without breaking default platform ranking.

7. **Shortlist + lender reveal**
   - User selects one or more offers.
   - Confirmation modal clarifies identity reveal and redirection behavior.
   - Post-confirmation: lender identity, next steps, estimated completion time.

8. **Handoff + tracking confirmation**
   - Redirect to lender flow with handoff event confirmed in UI.
   - Fallback path for redirect issues (copy link, retry, support CTA).

### B. Lender touchpoints (V1 lightweight frontend)
- Status dashboard for invitations, response SLA timer, offer submission status, and funnel outcomes.
- Clear empty/error states for no eligible requests / expired windows.

---

## 3) Component Architecture (Next.js App Router)

### Route structure (proposed)
- `app/page.tsx` — marketing + entry CTA
- `app/request/page.tsx` — intake shell
- `app/request/[requestId]/progress/page.tsx` — matching/progress
- `app/request/[requestId]/offers/page.tsx` — anonymized offer board
- `app/request/[requestId]/shortlist/page.tsx` — shortlist + reveal confirm
- `app/request/[requestId]/handoff/page.tsx` — redirect status/fallback
- `app/lender/dashboard/page.tsx` — lender operational dashboard

### Feature component tree
- `components/features/intake/`
  - `LoanRequestWizard`
  - `AmountInput`, `PurposeSelect`, `TenorSelector`
  - `ProfileSection`, `ConsentPanel`, `DisclosureVersionBadge`
- `components/features/offers/`
  - `OfferBoard`
  - `OfferCard`, `OfferTableRow`
  - `RankingExplanation`, `CostBreakdown`, `ValidityCountdown`
  - `OfferFilters`, `OfferSortControl`
- `components/features/shortlist/`
  - `ShortlistSelector`, `RevealConfirmationModal`, `HandoffCTA`
- `components/features/lender/`
  - `InvitationQueue`, `SLAIndicator`, `ConversionSummary`
- `components/shared/`
  - `StepProgress`, `EmptyState`, `ErrorState`, `LoadingSkeleton`, `TrustDisclosure`

### Rendering strategy
- Prefer **Server Components** for routes and static/legal content.
- Use **Client Components** only for interactive pieces: forms, filters, countdowns, selection, polling.
- Stream page shell quickly and hydrate interactive modules lazily.

---

## 4) State & Data Fetching Strategy

### Data boundaries
- Server fetch for initial request/offers snapshot to improve TTFB and SEO where useful.
- Client-side updates for dynamic bid window state.

### Recommended stack
- **TanStack Query** for client fetching/caching/retries/invalidation.
- **React Hook Form + Zod** for intake and consent forms.
- Optional lightweight **Zustand** store for cross-step wizard state (if server actions alone are insufficient).

### Query model (examples)
- `['request', requestId]` — borrower request status
- `['offers', requestId]` — offer list + metadata
- `['shortlist', requestId]` — shortlist state
- `['lenderDashboard', lenderId, dateRange]` — lender metrics

### Live updates
- Prefer SSE/WebSocket if available from orchestrator; fallback to controlled polling (e.g., 5–10s with backoff).
- Stop polling on terminal states (`expired`, `shortlisted`, `handoff_complete`).

### Error handling
- Typed API client with schema parsing at boundary.
- Distinct UI states for:
  - validation errors (field-level)
  - recoverable network errors (retry CTA)
  - terminal business-state errors (window expired / no offers)

---

## 5) Validation & Disclosure Requirements (Frontend-enforced)

### Form validation
- Amount: numeric, min/max by policy, currency formatting.
- Tenor: constrained options + policy bounds.
- Purpose: enum-based selection.
- Profile fields: strict format checks (ID, phone, email where required).

### Consent/disclosure controls
- Mandatory explicit consent checkbox(es), no pre-checked fields.
- Record disclosure version displayed at acceptance time.
- Gate progression until required legal acknowledgements are complete.
- Persistent visual label on offer board: “Indicative pre-offers, subject to final underwriting.”

### Offer integrity checks before render
- Validate offer schema client-side (safe parsing) to avoid broken cards.
- Suppress malformed offers from display and surface audit-safe error telemetry.

---

## 6) Accessibility (must-have for V1)
- Semantic landmarks (`main`, `nav`, `section`, `form`) and heading hierarchy.
- Full keyboard navigation across wizard, offer selection, and modals.
- Visible focus states and accessible error summaries.
- ARIA-live region for dynamic events (new offer arrival, expiry warning).
- Color contrast compliant labels for cost/risk signals (not color-only encoding).
- Screen-reader clear copy for indicative vs final terms.
- Mobile-first tap targets (>=44px), especially shortlist/reveal CTAs.

---

## 7) Analytics & Experimentation Plan

### Core events
- `intake_started`, `intake_step_completed`, `intake_submitted`
- `consent_viewed`, `consent_accepted` (with disclosure version)
- `match_started`, `first_offer_received`, `offers_count_updated`
- `offer_viewed`, `offer_rank_explanation_opened`, `offer_shortlisted`
- `lender_reveal_confirmed`, `handoff_clicked`, `handoff_success`, `handoff_failed`

### Event properties (minimum)
- `requestId`, `timestamp`, `deviceType`, `step`, `offersCount`, `timeSinceStart`, `disclosureVersion`, `rankPosition`

### Funnel dashboards
- Start → submit → first offer → shortlist → handoff → funded callback.
- Segment by traffic source, device, lender cohort, and request band (amount/tenor).

### V1 experiments
- Progress UI variant (linear vs milestone)
- Offer card emphasis (monthly payment first vs total cost first)
- Shortlist CTA wording and placement

---

## 8) Milestones & Frontend Delivery Slices

### M1 (Day 15): Compliance + schema freeze support
- Intake UX wireframes and copy placeholders complete.
- Consent/disclosure component with version support implemented.
- Zod schemas aligned to `BorrowerRequest` v1.0.

### M2 (Day 30): Core E2E staging
- Intake-to-offer-board flow functional with synthetic lender data.
- Offer board shows ranking reasons + validity window.
- Instrumentation baseline shipped (intake/offer/shortlist/handoff events).

### M3 (Day 60): Pilot-ready UX hardening
- Mobile performance pass + accessibility QA pass.
- Robust edge handling (late bids, expiry, empty state).
- Lender dashboard MVP operational.

### M4 (Day 90): Launch readiness
- Conversion-optimized copy and interaction polish.
- Analytics dashboards trusted by product/ops.
- Regression/e2e suite covering critical borrower journey.

---

## 9) Frontend-Specific Risks & Mitigations
1. **Trust drop due to “indicative” confusion**
   - Mitigation: repeated plain-language disclosure at decision points; tooltip + inline examples.
2. **Offer overload causing choice paralysis**
   - Mitigation: clear default ranking, concise explanation snippets, optional comparison mode.
3. **Latency perception during bid window**
   - Mitigation: progressive status updates, expected wait messaging, non-blocking UX.
4. **Inconsistent lender payload quality**
   - Mitigation: strict schema parsing, fallback rendering rules, connector feedback loop.
5. **Mobile abandonment**
   - Mitigation: short steps, input masks, sticky next CTA, resume-friendly flow.

---

## 10) Acceptance Criteria (Frontend)

### Borrower flow
- User can complete intake + consent on mobile in a guided <=4-step flow.
- Blocking validation prevents invalid submission; errors are understandable and accessible.
- Offer board renders normalized fields and ranking explanation for each displayed offer.
- User can shortlist and explicitly confirm lender reveal before handoff.
- Handoff success/failure state is visible with retry/fallback options.

### Compliance/disclosure
- Indicative-vs-final disclosure appears at intake consent and offer decision points.
- Consent capture includes disclosure version and timestamp in emitted event payload.

### Reliability/performance
- Loading, empty, error, and success states exist for all data-driven screens.
- Dynamic updates (offers/status) function without full-page refresh.
- No high-severity accessibility defects in keyboard and screen-reader smoke tests.

### Analytics
- All core funnel events are emitted with required properties and validated in staging.
- Dashboard can compute time request→first offer and shortlist rate from frontend+backend events.

---

## 11) Recommended Immediate Frontend Next Actions (Week 1)
1. Lock UX copy with legal for consent + indicative disclosures.
2. Define frontend contract types from schema v1.0 and generate validators.
3. Build intake wizard skeleton and analytics event map.
4. Implement offer board shell with placeholder ranking explanation component.
5. Set up e2e happy-path test: intake → offers → shortlist → handoff.
