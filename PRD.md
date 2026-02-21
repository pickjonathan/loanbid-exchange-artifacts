# PRD v2 — LoanBid Exchange (Israel, Consumer Loans)

## 1) Scope & Decisions (approved)
- Segment: **Consumer personal loans**
- Geography: **Israel**
- Offer type in V1: **Indicative pre-offers** (not binding)
- Revenue: **Hybrid** = qualified lead fee + funded-loan success fee
- UX: **Anonymized offers first**, lender identity revealed after shortlist
- Risk depth: **Light pre-check in V1**; deeper checks in later phases

## 2) Problem
Borrowers cannot efficiently compare loan offers; lenders receive inconsistent leads with low conversion predictability.

## 3) Product Goal
Create a neutral exchange where one borrower request can receive multiple standardized lender bids quickly and transparently.

## 4) V1 Functional Requirements
### Borrower
- Loan request intake (amount, purpose, tenor, basic profile)
- Consent capture and KYC-lite flow
- Offer board with anonymized ranked pre-offers
- Shortlist and reveal lender identity
- Click-through to lender underwriting/completion

### Lender
- Bid API/portal for indicative pre-offers
- Eligibility/routing rules by risk appetite
- Response SLA and offer expiry
- Status dashboard and conversion reporting

### Platform
- Rule-based matching engine
- Bid orchestration (timed window)
- Ranking + explainability layer
- Audit/compliance logging

## 5) Non-Goals (V1)
- Binding final credit decisions inside marketplace
- Multi-country launch
- Advanced ML underwriting automation

## 6) Standard Offer Schema
- APR / effective annual rate
- Fees (origination, admin)
- Tenor
- Estimated monthly payment
- Conditions/caveats
- Offer validity window

## 7) Ranking Model (Borrower-visible)
Weighted score (configurable):
1. Total borrowing cost
2. Monthly affordability
3. Approval confidence (indicative)
4. Time-to-money estimate
5. Lender historical reliability score

## 8) KPIs
- Time request → first offer
- Avg offers per request
- Borrower shortlist rate
- Offer accept rate
- Funded conversion
- APR delta vs borrower baseline

## 9) Compliance & Controls (Israel launch)
- Consent + purpose limitation for data sharing
- KYC/AML checks in intake boundary
- Clear disclosure: indicative vs final terms
- Immutable audit trail for offer/ranking events

## 10) Phased Risk Depth
- **V1:** light pre-check (self-reported + identity basic)
- **V1.5:** medium checks (bank statements/open-banking where available)
- **V2:** deeper verification (bureau + income verification + policy-driven fraud controls)
