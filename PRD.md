# PRD v1 — LoanBid Exchange

## 1. Problem
Borrowers struggle to compare loan options across lenders; lenders struggle to acquire high-intent applicants with consistent data.

## 2. Product Vision
A neutral exchange where borrowers create one loan request and multiple lenders submit competing bids in a standardized format.

## 3. Users
- Borrowers (SME first)
- Financial institutions / lenders
- Marketplace operations/compliance team

## 4. MVP Scope
### Borrower
- Request form (amount, term, purpose, revenue/income, docs)
- Consent + KYC flow
- Offer comparison table
- Accept offer flow

### Lender
- Bid API/portal
- Eligibility filters
- Bid SLA + expiry
- Offer status dashboard

### Platform
- Rule-based matching engine
- Bid collection/orchestration
- Ranking + explanation layer
- Audit logs + admin panel

## 5. Out of Scope (MVP)
- Full dynamic auctions with multi-round visibility
- Cross-border, multi-jurisdiction launch
- Consumer + SME simultaneously

## 6. Bid Schema (standard)
- APR
- Origination fee
- Tenor
- Monthly installment
- Prepayment penalty
- Required covenants/conditions
- Offer expiry timestamp

## 7. Ranking Model (borrower-visible)
Score dimensions:
1. Total cost of credit
2. Monthly affordability
3. Approval confidence
4. Time-to-funds
5. Lender quality score

## 8. Success Metrics
- Median time request->first bid
- Bids/request
- Offer acceptance rate
- Funded rate
- APR improvement vs baseline
- Lender repeat participation rate

## 9. Compliance & Risk
- KYC/AML checks
- Consent ledger
- Fair lending monitoring
- Explainable ranking decisions
- Immutable audit trail

## 10. Business Model
Primary: success fee on funded loans.
Secondary: lender subscription for API + analytics.
