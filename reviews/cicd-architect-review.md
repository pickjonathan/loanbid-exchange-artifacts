# CI/CD Implementation Instructions — LoanBid Exchange (Israel V1)

## 0) Executive Summary
The delivery plan is feasible in 90 days if CI/CD is treated as a launch-critical product capability, not a support function. This review defines a GitHub Actions + AWS implementation blueprint with hard quality/security gates, staged release promotion, and rollback-safe deployments aligned to the V1 scope (indicative pre-offers, anonymized ranking, lender reveal on shortlist).

---

## 1) CI/CD Goals and Principles

### Delivery goals
1. Ship safely to production with predictable release cadence.
2. Prevent compliance/security regressions from reaching pilot/prod.
3. Preserve lender integration stability with contract-driven promotion.
4. Enable rapid rollback for bid-window and borrower-flow incidents.

### Non-negotiable principles
- **CI/CD platform:** GitHub Actions only.
- **Cloud target:** AWS only.
- **Auth to AWS:** OIDC role assumption only (no static AWS keys).
- **Artifact promotion:** immutable Docker images by digest.
- **Deploy policy:** staging first, production only after approval + green gates.

---

## 2) Pipeline Topology (Recommended Workflows)

1. **`ci.yml` (PR + main)**
   - Lint, typecheck, unit tests, integration tests, contract tests.
2. **`build-scan.yml` (main + tags)**
   - Build image, SBOM generation, vulnerability scans, sign/attest, push to ECR.
3. **`deploy-staging.yml` (main merge)**
   - Deploy to ECS staging, run smoke + synthetic lender checks.
4. **`deploy-prod.yml` (manual, protected environment)**
   - Promote pre-validated digest to prod after human approval.
5. **`release.yml` (tag/release orchestration)**
   - Semver tagging, release notes, artifact manifest publication.
6. **`rollback.yml` (manual emergency)**
   - Re-deploy last known good image digest/task definition.

Use GitHub Environments: `staging` and `production` with protection rules.

---

## 3) Pipeline Stages (Execution Order)

## Stage 1 — Static Quality Fast Gate (PR-blocking)
- ESLint + TypeScript checks.
- Action/workflow lint (`actionlint`).
- Dockerfile lint (`hadolint`).
- Basic schema validation for `BorrowerRequest`, `LenderOffer`, `Outcome`.

**Fail condition:** any lint/type/schema error.

## Stage 2 — Test Gate (PR-blocking)
- Unit tests for routing/ranking/expiry/idempotency.
- Integration tests (intake→orchestration→offer board).
- Contract tests for lender adapters (positive + negative + malformed payload cases).
- Coverage thresholds (suggested):
  - Global line coverage >= 80%
  - Domain-critical modules (`ranking`, `orchestration`, `audit`) >= 90%

**Fail condition:** any test failure or coverage below threshold.

## Stage 3 — Security & Supply Chain Gate (main + release)
- Dependency scan (`npm audit` or equivalent policy scanner).
- Container vulnerability scan (Trivy) on built image.
- Secrets scan (repo + image layers).
- SBOM generation (CycloneDX or SPDX) and artifact retention.
- Optional artifact signing/attestation (recommended for regulated readiness).

**Fail condition:** any HIGH/CRITICAL vulnerability unless approved exception exists.

## Stage 4 — Build & Publish Artifact Gate
- Docker multi-stage build.
- Tagging strategy:
  - `sha-<shortsha>` for every build
  - `vMAJOR.MINOR.PATCH` for releases
- Push to Amazon ECR after successful scans.
- Export image digest for downstream deploy workflows.

**Fail condition:** build/push failure or digest mismatch.

## Stage 5 — Staging Deploy Verification Gate
- Deploy digest to ECS staging.
- Run post-deploy smoke tests:
  1. Borrower request creation
  2. Synthetic lender offer ingestion
  3. Ranked anonymized board retrieval
  4. Shortlist + reveal path
- Validate key SLO probes (latency + error rate + event write health).

**Fail condition:** smoke failure, health checks red, or SLO floor breach.

## Stage 6 — Production Promotion Gate
- Manual approval in GitHub `production` environment.
- Confirm staging deployment is based on same immutable digest.
- Deploy using ECS rolling/canary strategy.
- Post-deploy verification + business KPI watch window.

**Fail condition:** approval missing, digest drift, health/KPI checks failing.

---

## 4) Quality Gates (Hard Release Criteria)

A candidate can be promoted only when all are true:
1. Zero failing checks in `ci.yml`.
2. Contract suite green for all active lender connectors.
3. Security scans have no unapproved HIGH/CRITICAL findings.
4. Staging smoke tests pass on latest artifact digest.
5. Compliance assertions pass:
   - consent/disclosure fields present,
   - audit events emitted for core lifecycle,
   - no PII fields in ranking payloads.
6. No open Sev-1/Sev-2 defects tied to release scope.

---

## 5) Release Strategy

### Branching and promotion
- `main` is always releasable.
- PRs require all required checks before merge.
- Every merge to `main` triggers staging deployment.
- Production deploy is a manual promotion of the same staging-validated digest.

### Versioning
- Semantic versioning: `vMAJOR.MINOR.PATCH`.
- Suggested cadence for V1: weekly release train, with emergency patch path.
- Release notes must include:
  - schema/version changes,
  - connector additions/changes,
  - migration requirements,
  - risk notes.

### Environment strategy
- **staging:** auto-deploy on merge to main.
- **production:** protected, manual approval, change window recommended.

---

## 6) Rollback Strategy (Required Runbook Behavior)

### Primary rollback mechanism
- Re-deploy last known-good ECS task definition/image digest.
- Keep at least last 10 production-ready digests and task defs.

### Rollback triggers
- Borrower flow smoke fails post-deploy.
- Bid ingestion error spike beyond threshold.
- Offer board latency/error budget burn indicates instability.
- Compliance telemetry missing required consent/audit events.

### Rollback execution target
- Decision-to-rollback <= 10 minutes.
- Service restoration <= 15 minutes from trigger.

### Data safety rules during rollback
- No destructive DB rollback for shared mutable data in hot path.
- Prefer forward-fix migrations; down migrations only for pre-approved reversible changes.
- Settlement/outcome processing must be idempotent to tolerate replay after rollback.

---

## 7) Security Scans and Compliance Automation

Minimum required automated checks in CI/CD:
1. **SAST** for application code.
2. **Dependency vulnerability scanning** (Node packages).
3. **Container image scan** (Trivy, block HIGH/CRITICAL).
4. **Secrets scanning** on repo and build context.
5. **IaC/workflow linting** for CI definitions and deployment files.
6. **Compliance assertions as tests**:
   - consent reference required,
   - disclosure version required,
   - audit event presence on state transitions.

Recommended policy:
- HIGH/CRITICAL: block by default.
- MEDIUM: allow with ticket + due date.
- Any exception requires explicit risk acceptance record.

---

## 8) Acceptance Criteria (CI/CD Sign-Off)

CI/CD implementation is accepted when:
1. All six workflows (`ci`, `build-scan`, `deploy-staging`, `deploy-prod`, `release`, `rollback`) exist and run successfully.
2. Required checks are enforced at branch protection level.
3. Staging auto-deploy works on merge to `main` using immutable digest.
4. Production requires manual approval and reuses the same digest promoted from staging.
5. Rollback workflow restores last known-good release in controlled drill.
6. Scan reports, SBOM, test results, and deployment metadata are retained as artifacts.
7. Dashboard evidence exists for release health (latency, error rate, offers/request, audit-event completeness).

---

## 9) 90-Day Milestone Alignment (from Plan to Pipeline)

- **M1 (Day 15):** CI fast gates + contract test harness + compliance test assertions live.
- **M2 (Day 30):** Build/scan/publish and staging auto-deploy operational; synthetic E2E green.
- **M3 (Day 60):** Production promotion + rollback drills completed; security gates fully enforced.
- **M4 (Day 90):** Release train stable, KPI-linked go/no-go process in place, audit-ready evidence artifacts automated.

---

## 10) Immediate Next 10 CI/CD Actions (Week 1)

1. Create `.github/workflows/ci.yml` with lint/type/unit/integration/contract jobs.
2. Add strict branch protection on `main` (required checks + no bypass).
3. Implement `build-scan.yml` with Trivy + SBOM + ECR push.
4. Configure AWS OIDC IAM roles for build and deploy (staging/prod separation).
5. Add `deploy-staging.yml` with smoke tests and synthetic lender probe.
6. Add `deploy-prod.yml` tied to protected `production` environment approvals.
7. Implement `rollback.yml` (select prior digest/task def and redeploy).
8. Add artifact retention policy for test reports, scan results, and SBOM.
9. Add deployment notifications and status reporting for go/no-go calls.
10. Run first game day: deploy-fail-rollback exercise and document timings.

---

## Final Review Conclusion
Proceed with the current 90-day plan **only if the above CI/CD gates are enforced as launch blockers**. The marketplace’s business outcomes (liquidity, trust, and compliance) depend directly on release safety, connector stability, and auditable promotion/rollback mechanics.