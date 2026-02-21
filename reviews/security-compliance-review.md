# Security & Compliance Implementation Instructions — LoanBid Exchange (Israel V1)

## 0) Executive Summary
The current plan is strong on product scope and audit intent, but it needs an implementation-grade security/compliance baseline before pilot expansion. This review defines a practical control framework for V1 (indicative pre-offers), with explicit ownership, evidence, Israel-focused checklist items, and release acceptance criteria.

---

## 1) Scope, Assumptions, and Risk Posture
**In scope (V1):**
- Borrower intake, consent capture, KYC-lite hooks
- Routing + bid orchestration + ranking explainability
- Lender API adapters and shortlist reveal flow
- Compliance/audit ledger and analytics exports

**Out of scope (V1):**
- Final underwriting decisioning inside marketplace
- Deep bureau/open-banking based underwriting

**Security posture target:**
- Protect borrower PII and financial intent data
- Preserve integrity/fairness of offer ranking and disclosure trails
- Demonstrate lawful processing, consent traceability, and regulator/audit readiness for Israel launch

---

## 2) Threat Model (V1)

## 2.1 Assets to protect
1. Borrower PII (identity/contact/basic profile)
2. Sensitive financial request data (loan amount, purpose, affordability signals)
3. Consent artifacts + disclosure version records
4. Offer/ranking events and explainability outputs
5. Lender credentials/API secrets and signed request keys
6. Revenue settlement records and outcome events

## 2.2 Trust boundaries
- **Boundary A:** Public borrower web/app to platform APIs
- **Boundary B:** Platform internal services (intake/orchestration/ranking/audit)
- **Boundary C:** Platform ↔ lender external integrations
- **Boundary D:** Ops/admin access to production data and logs
- **Boundary E:** Analytics/export boundary (regulator/legal reports)

## 2.3 Priority threat scenarios and required mitigations
1. **Account/identity abuse in intake** (synthetic users, scripted abuse)  
   - Mitigate with bot controls, velocity limits, device/IP risk signals, OTP verification, and anti-automation challenges.
2. **PII over-collection or unauthorized sharing to lenders**  
   - Mitigate with field-level minimization, tokenized request IDs, purpose-scoped data release, and policy-enforced disclosure gates.
3. **Offer tampering/replay/injection by compromised connector**  
   - Mitigate with mTLS + request signing + nonce/timestamp windows + idempotency keys + schema validation.
4. **Ranking manipulation (internal or external)**  
   - Mitigate with immutable event logs, config/version pinning, maker-checker approvals for weight changes, and anomaly detection on score deltas.
5. **Sensitive data leakage via logs/analytics/exports**  
   - Mitigate with logging redaction, data classification tags, least-privilege access, and DLP checks before export.
6. **Insider misuse of borrower records**  
   - Mitigate with RBAC/ABAC, JIT privileged access, session recording for admin actions, and quarterly access recertification.
7. **Compliance failure on indicative-vs-final disclosure**  
   - Mitigate with mandatory disclosure checkpoints, versioned legal text IDs in events, and no progression without affirmative consent.
8. **Availability attack during bid windows**  
   - Mitigate with rate limiting, WAF, queue backpressure, graceful degradation, circuit breakers, and tested runbooks.

---

## 3) Security Control Implementation Plan

## 3.1 Identity, Access, and Secrets
- Enforce SSO + MFA for all staff/admin consoles.
- RBAC by service role; no shared accounts.
- JIT elevation for production admin operations (time-bound approvals).
- Store secrets only in a managed secret vault; rotate connector/API keys at least every 90 days.
- Service-to-service auth via short-lived tokens or mutual TLS.

**Evidence required:** access matrix, MFA enforcement report, key rotation logs.

## 3.2 Application & API Security
- Schema validation at every external boundary (borrower input + lender offers).
- Idempotency keys for bid submission endpoints.
- Signed lender requests (HMAC or asymmetric), strict timestamp skew checks.
- OWASP ASVS-aligned secure coding checks; block high/critical findings pre-release.
- Secure headers, CSRF protection, and strict CORS allowlist.

**Evidence required:** API contract tests, negative test logs, SAST/DAST reports.

## 3.3 Data Protection
- Data classification: Public / Internal / Confidential / Regulated-PII.
- Encrypt in transit (TLS 1.2+) and at rest (managed KMS keys).
- Tokenize borrower direct identifiers before routing/ranking where feasible.
- Field-level encryption for identity numbers and other high-risk fields.
- Pseudonymized analytics datasets by default.

**Evidence required:** data flow diagram with classifications, encryption configs, tokenization design doc.

## 3.4 Consent, Disclosure, and Auditability
- Versioned legal disclosures with immutable `disclosureVersionId` at acceptance time.
- Persist `consentRef` and purpose metadata with every lender-share event.
- Block progression if consent missing/expired/withdrawn.
- Append-only event ledger for intake, routing, ranking, shortlist, reveal, and handoff.
- Regulator-ready export format with integrity checksums.

**Evidence required:** consent traceability test pack, sample audit export, tamper-evidence verification.

## 3.5 Monitoring, Detection, and Response
- Centralized logging with PII redaction and retention policy.
- Security alerts for auth anomalies, failed signatures, unusual ranking shifts, excessive data export.
- 24/7 incident severity model and on-call runbook.
- Breach triage process with legal/compliance escalation.

**Evidence required:** alert catalog, incident runbook, tabletop exercise outcomes.

## 3.6 Third-Party / Lender Integration Governance
- Security onboarding checklist per lender connector.
- Mandatory sandbox certification (contract, signature, replay rejection tests).
- Minimum security clauses in partner agreements (breach notice, data usage restrictions, deletion timelines).
- Connector-level kill switch for abusive/non-compliant partners.

**Evidence required:** signed certification checklist, partner security attestation, test evidence.

---

## 4) Data Governance Instructions

## 4.1 Data lifecycle policy (minimum)
- **Collect:** only fields required for matching, disclosure, and KYC-lite.
- **Use:** strictly for loan matching/ranking/handoff and compliance obligations.
- **Share:** minimum necessary fields to eligible shortlisted lenders only.
- **Retain:** retention schedule by dataset (operational, audit, legal hold).
- **Delete/Anonymize:** enforce deletion jobs; irreversible anonymization for analytics history when lawful.

## 4.2 Data subject rights operations
- Implement intake channel for access/correction/deletion requests.
- SLA-backed workflow for identity verification and response tracking.
- Propagate deletion requests to downstream processors/lenders where required by contract/law.

## 4.3 Data governance artifacts to ship before pilot scale
1. Record of Processing Activities (RoPA-equivalent internal register)
2. Data inventory and lineage map
3. Retention + deletion schedule with owners
4. Cross-border transfer map and safeguards
5. Privacy impact assessment (PIA/DPIA-style) for ranking and profiling logic

---

## 5) Israel-Focused Compliance Checklist (Implementation-Oriented)
> **Note:** This is an implementation checklist, not legal advice. Local counsel should validate final interpretations before launch.

### 5.1 Privacy & data protection (Israel)
- [ ] Map processing against Israel Privacy Protection Law, 1981 and related regulations (including data security obligations).
- [ ] Determine whether database registration duties apply; complete if required.
- [ ] Assign formal privacy/security responsibility (e.g., data security manager role) and governance cadence.
- [ ] Publish Hebrew-first privacy notice with clear purpose, sharing logic, retention, and rights process.
- [ ] Implement consent records that capture timestamp, policy version, channel, and user proof.
- [ ] Enforce data minimization and purpose limitation in code and contracts.

### 5.2 Financial-sector and conduct controls (loan brokerage/marketplace context)
- [ ] Validate licensing/registration obligations applicable to brokering/referral model under Israeli financial services/credit laws.
- [ ] Ensure required consumer disclosures are shown pre-action (indicative vs final terms, fees, lender identity timing).
- [ ] Prevent misleading ranking claims; retain explainability evidence for each presented offer.
- [ ] Maintain complaint-handling workflow and audit trail.

### 5.3 AML/CFT and sanctions (risk-based for V1)
- [ ] Define when platform is an obligated entity vs. support role in KYC/AML chain.
- [ ] Implement sanctions/PEP/adverse-media checks where legally required by role.
- [ ] Keep suspicious activity escalation path and recordkeeping controls.
- [ ] Ensure lender contracts clearly allocate AML responsibilities and evidence-sharing requirements.

### 5.4 Cybersecurity baseline and regulator readiness
- [ ] Align operational controls with a recognized framework (ISO 27001 / NIST CSF mapping).
- [ ] Conduct penetration test on borrower flow, lender API, and admin surfaces before broad launch.
- [ ] Verify immutable audit export capability for regulator/legal requests.
- [ ] Maintain breach/incident notification decision tree with legal counsel sign-off.

### 5.5 Localization and fairness/transparency
- [ ] Hebrew disclosures are plain-language and prominent on key decision screens.
- [ ] Ranking reason snippets are understandable and non-deceptive.
- [ ] Manual override events are logged with actor, reason, and approval.

---

## 6) Release Acceptance Criteria (Security/Compliance Gates)

## Gate A — Design Freeze (by Day 15)
- Approved threat model and trust boundary diagram.
- Data classification + minimization matrix approved.
- Consent/disclosure text versioning mechanism implemented.

## Gate B — Staging E2E (by Day 30)
- 100% of lender connectors pass signature, replay, and schema contract tests.
- Consent traceability test: any lender-share event can be traced to valid consent in <5 minutes.
- Audit ledger proves append-only behavior and successful integrity verification.
- No open critical/high findings from SAST, dependency audit, and secrets scan.

## Gate C — Pilot Readiness (by Day 60)
- Pen-test complete with no unresolved critical/high exploitable findings.
- Incident runbook + on-call escalation tested via tabletop.
- Access recertification completed for production roles.
- Data export redaction validated; no direct identifier leakage in analytics outputs.

## Gate D — Commercial Launch Readiness (by Day 90)
- Israel legal/compliance checklist signed by counsel + internal compliance owner.
- Retention/deletion automation proven with test evidence.
- Partner security addenda executed with all active lenders.
- Security and compliance KPIs (below) are green for 2 consecutive weeks.

---

## 7) Security & Compliance KPIs (Operational)
- % requests with valid consent artifact linked to all downstream share events (**target: 100%**)
- # unauthorized data-access incidents (**target: 0**)
- Mean time to revoke compromised connector credentials (**target: <30 min**)
- % lender API calls rejected for signature/replay issues (monitor anomaly, threshold alerts)
- Audit export integrity pass rate (**target: 100%**)
- Time to fulfill data subject request within policy SLA (**target: ≥99% on-time**)

---

## 8) Immediate Implementation Backlog (Top 12)
1. Create DFD + trust boundary diagram with asset classification labels.
2. Define and enforce data minimization schema per workflow stage.
3. Implement signed lender request protocol + anti-replay middleware.
4. Add idempotency and strict schema validation to bid ingestion.
5. Build consent/disclosure version registry and immutable linkage in events.
6. Deploy append-only audit ledger integrity checker + export signer.
7. Enable centralized redacted logging and sensitive-field denylist.
8. Set up RBAC/JIT privileged access and quarterly recertification process.
9. Add abuse protections: rate limits, bot controls, anomaly scoring on intake.
10. Implement deletion/retention jobs with test harness and reporting.
11. Build Israel compliance evidence binder (privacy, consumer disclosure, AML role mapping).
12. Run independent pen-test and fix cycle before pilot expansion.

---

## 9) Owner Matrix (Recommended)
- **Security Lead:** threat model, control library, pen-test closure, incident readiness
- **Compliance/Legal Lead (Israel counsel-backed):** legal checklist, disclosures, licensing/role determinations
- **Data Protection Owner:** retention/deletion, rights handling, data inventory
- **Platform Engineering:** API security, encryption, audit ledger, secrets management
- **Product:** UX disclosure placement, explainability text, consent friction optimization
- **Partnerships/Ops:** lender security onboarding and contractual enforcement

---

## 10) Final Review Conclusion
Proceed with the existing 90-day plan **only with the above control gates added as hard release criteria**. The plan is viable for V1 if consent traceability, connector security, immutable auditability, and Israel-specific legal obligations are treated as launch blockers—not post-launch enhancements.
