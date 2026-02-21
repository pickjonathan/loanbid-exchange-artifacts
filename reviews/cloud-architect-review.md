# Cloud Architecture Implementation Instructions — LoanBid Exchange (AWS, Israel V1)

## 0) Executive Summary
The product and implementation plans are directionally strong, but the architecture needs a concrete AWS reference design with clear boundaries, scaling controls, and operational gates. This review defines an AWS-first implementation blueprint for V1 (indicative pre-offers), with production-ready decisions for reliability, governance, and disaster recovery.

---

## 1) Architecture Principles (V1)
1. **Event-driven core, API-first edges** for lender integrations and borrower UX responsiveness.
2. **PII minimization by default** with explicit boundary controls between intake, matching/ranking, and analytics.
3. **Multi-AZ baseline** for all stateful and edge services in production.
4. **Deterministic auditability** for consent, ranking decisions, and reveal/handoff actions.
5. **Scale with predictable latency** during bid windows (spiky marketplace traffic).

---

## 2) AWS Service Choices (Recommended)

## 2.1 Edge, API, and Traffic Protection
- **Amazon Route 53**: DNS + health-check-based failover records.
- **Amazon CloudFront + AWS WAF + AWS Shield Standard**: global edge caching and L7 protection for borrower UI/API endpoints.
- **Amazon API Gateway (HTTP API)**: external API front door for borrower and lender endpoints.
- **AWS Certificate Manager (ACM)**: TLS certificate management.

## 2.2 Application Compute
- **Amazon ECS on Fargate** for core microservices:
  - Borrower Intake Service
  - Routing Engine
  - Bid Orchestrator
  - Ranking Service
  - Settlement/Outcome Service
- **AWS Lambda** for event-driven glue tasks:
  - notification fanout
  - expiry sweeps
  - audit export packaging
  - lightweight policy checks

## 2.3 Messaging and Workflow
- **Amazon EventBridge**: domain event bus (`request.created`, `offer.submitted`, `shortlist.created`, etc.).
- **Amazon SQS**: decoupled queues per connector and retry/DLQ strategy.
- **AWS Step Functions**: bid-window orchestration (invite timers, deadlines, reveal transitions).

## 2.4 Data Stores
- **Amazon Aurora PostgreSQL (Multi-AZ)**: transactional system of record for requests, offers metadata, shortlist state, commercial records.
- **Amazon DynamoDB**: idempotency keys, fast state lookup (offer window state, connector throttles).
- **Amazon ElastiCache for Redis**: low-latency ranking board cache and session/short-lived tokens.
- **Amazon S3 (versioned)**: immutable audit export files, document artifacts, analytics landing zone.

## 2.5 Security, Secrets, and Compliance Telemetry
- **AWS KMS (CMKs)**: encryption for RDS, DynamoDB, S3, and sensitive payload fields.
- **AWS Secrets Manager**: lender API credentials and signing keys.
- **AWS CloudTrail + AWS Config + Security Hub + GuardDuty**: governance and threat detection baseline.
- **Amazon CloudWatch + AWS X-Ray**: logs, metrics, traces, SLO dashboards.

## 2.6 Analytics and Reporting
- **Amazon Kinesis Data Firehose** (optional at V1 scale) or EventBridge→S3 pipelines.
- **AWS Glue Data Catalog + Amazon Athena** for compliance and business analytics queries.
- **Amazon QuickSight** for KPI dashboards (offers/request, time-to-first-offer, shortlist/funded conversion).

---

## 3) Reference Deployment Topology

## 3.1 Network Layout
- Single AWS Region for V1 (e.g., `eu-central-1` or approved Israel-adjacent region per legal/commercial decision).
- **One VPC across 3 AZs**:
  - Public subnets: ALB/NAT only (if ALB used behind CloudFront/API).
  - Private app subnets: ECS tasks, Lambda ENIs.
  - Private data subnets: Aurora, Redis.
- No public DB endpoints. No inbound SSH to workloads.

## 3.2 Environment Strategy
- Separate AWS accounts: **dev / staging / prod / security-log-archive**.
- Separate VPCs per environment; no shared runtime resources between staging and prod.
- Infrastructure via Terraform with pinned provider/module versions.

## 3.3 Service Partitioning
- **Core domain services** isolated by bounded context:
  - intake
  - routing
  - orchestration
  - ranking
  - compliance/audit
  - settlement
- **Connector adapter tier** isolated from core domain (blast-radius and partner-specific failures contained).

---

## 4) Scalability & Performance Instructions

## 4.1 Autoscaling Policy
- ECS services scale by:
  - CPU/Memory utilization
  - SQS queue depth (connector queues)
  - custom metric: active bid windows
- Target baseline:
  - prod: min 3 tasks/service across >=2 AZs
  - staging: min 1 task/service

## 4.2 Bid-Window Load Design
- Use Step Functions + EventBridge timers rather than in-process schedulers.
- Each lender connector gets its own SQS queue + DLQ.
- Enforce per-lender rate limits and circuit breakers to avoid systemic queue buildup.

## 4.3 Data Performance
- Aurora:
  - reader endpoint for analytics/reads where needed
  - partition high-volume event tables by date/request domain
- Redis cache for ranked offer board responses with short TTL and explicit invalidation on new valid offers.
- DynamoDB for high-QPS idempotency checks (single-digit ms).

## 4.4 SLO-Linked Capacity Targets
- `POST /offers` ingestion: P95 < 800ms.
- Offer board API: P95 < 1200ms.
- Request→first-offer median < 10 minutes.
- Orchestration availability: 99.9% monthly.

---

## 5) Governance, Security, and Operating Controls

## 5.1 Account & IAM Governance
- Use AWS Organizations + SCPs:
  - deny disabling CloudTrail/Config/GuardDuty
  - deny public RDS snapshots and public DB access
  - enforce approved regions
- IAM Identity Center for human access; no long-lived IAM user keys.
- Least-privilege roles per service; quarterly access recertification.

## 5.2 Policy as Code
- Terraform + CI checks (`terraform validate`, tfsec/checkov, infracost).
- Mandatory tagging for all resources: `Environment`, `Team`, `Service`, `CostCenter`, `ManagedBy`.
- Config rules/Conformance packs for encryption, logging, public access controls.

## 5.3 Observability & Auditability
- Standard structured event schema with correlation IDs.
- Immutable audit trail persisted to S3 with object lock (compliance mode where required).
- CloudWatch alarms wired to PagerDuty/incident channel:
  - API error spikes
  - queue age/backlog
  - missing compliance events
  - unusual ranking score shifts

---

## 6) Data Boundaries and Information Flow

## 6.1 Domain Data Zones
1. **PII Zone (Restricted)**
   - borrower identity/contact and consent artifacts
   - stored encrypted, tightly scoped IAM access
2. **Decision Zone (Controlled)**
   - offer attributes, ranking factors, SLA states, lender anon IDs
   - no direct borrower identifiers exposed
3. **Analytics Zone (Pseudonymized)**
   - aggregated KPIs, cohort trends, lender scorecards
   - irreversible tokenization/anonymization before broad analyst access

## 6.2 Boundary Enforcement
- Tokenize borrower identifiers at intake; pass tokens to routing/ranking.
- Lender identity masked until shortlist event transition.
- Field-level allowlists for lender payloads (minimum necessary sharing).
- Separate KMS keys for PII zone vs analytics zone.

## 6.3 Data Retention Controls
- Retention tiers:
  - operational hot data in Aurora (policy-defined duration)
  - audit records in S3 (legal retention)
  - analytics aggregates in Athena/QuickSight
- Automated lifecycle and deletion workflows with evidence logs.

---

## 7) Disaster Recovery (DR) Strategy

## 7.1 V1 DR Posture
- **Primary strategy:** warm-standby foundation with backup/restore fallback.
- Multi-AZ in-region for high availability.
- Cross-region backups for recovery from regional outage.

## 7.2 Service-Level DR Controls
- Aurora: automated backups + cross-region snapshot copy; tested restore playbooks.
- DynamoDB: PITR enabled; global table optional at V1.5 if RTO pressure increases.
- S3: versioning + cross-region replication for audit bucket (if regulatory policy allows).
- ECR/task definitions/IaC artifacts replicated or reproducible in secondary region.

## 7.3 DR Targets (Recommended)
- **Critical marketplace operations:** RTO 60 minutes, RPO 15 minutes.
- **Analytics/reporting:** RTO 24 hours, RPO 24 hours.

## 7.4 DR Testing Cadence
- Quarterly recovery drill:
  1. restore Aurora from snapshot/PITR
  2. replay event backlog from S3/EventBridge archive
  3. validate shortlist/reveal and audit export integrity
- Evidence retained in runbooks and go/no-go packs.

---

## 8) Implementation Plan Alignment (90-Day)

## M1 (Day 15) — Foundation Freeze
- Terraform landing zone and account model ready.
- VPC, IAM baseline, KMS keys, Secrets Manager, CloudTrail/Config/SecurityHub enabled.
- Canonical event schema and service API contracts frozen.

## M2 (Day 30) — Core Staging E2E
- ECS services + EventBridge/SQS + Aurora/DynamoDB running in staging.
- Synthetic lenders integrated via connector queues.
- SLO dashboards and tracing live.

## M3 (Day 60) — Pilot Reliability
- Prod-grade autoscaling policies tuned from pilot load tests.
- Alerting/runbooks validated (connector outage, queue backlog, DB failover).
- Compliance/audit export pipeline operational.

## M4 (Day 90) — Commercial Readiness
- 8–12 lenders supported with isolated connector capacity controls.
- DR drill completed with documented RTO/RPO evidence.
- Cost and governance guardrails enforced across production accounts.

---

## 9) Acceptance Criteria (Architecture Go/No-Go)

## 9.1 Platform Readiness
- [ ] Multi-AZ deployment active for all production compute and databases.
- [ ] All external APIs protected by WAF + rate limiting + auth/signature controls.
- [ ] Connector queues include DLQs with automated alerting.

## 9.2 Scalability & Reliability
- [ ] Performance tests prove P95 latency SLOs for ingestion and offer board APIs.
- [ ] Request→first-offer KPI median under 10 min in pilot-qualified traffic.
- [ ] No single connector failure can degrade global orchestration availability below 99.9%.

## 9.3 Governance & Security
- [ ] CloudTrail, Config, GuardDuty, Security Hub enabled and monitored.
- [ ] Encryption at rest/in transit enforced for all in-scope data stores and APIs.
- [ ] IAM least-privilege review complete; no wildcard prod permissions without waiver.

## 9.4 Data Boundary Assurance
- [ ] PII tokenization boundary validated end-to-end.
- [ ] Lender identity remains hidden pre-shortlist in all tested flows.
- [ ] Analytics exports contain no direct identifiers outside restricted roles.

## 9.5 DR & Operational Readiness
- [ ] Backup restore and failover drill completed successfully within target RTO/RPO.
- [ ] Incident runbooks tested for queue backlog, connector outage, and DB failover.
- [ ] Audit export integrity verification passes for sampled regulator-style requests.

---

## 10) Priority Architecture Backlog (Top 12)
1. Finalize AWS region and cross-border data position with legal/compliance.
2. Build Terraform landing zone (accounts, SCPs, logging archive).
3. Implement VPC 3-AZ network baseline and private data plane.
4. Stand up ECS/Fargate service mesh baseline with per-service IAM roles.
5. Implement EventBridge event taxonomy and SQS connector isolation model.
6. Deploy Aurora + DynamoDB + Redis with encryption and backup policies.
7. Implement consent/audit immutable pipeline to S3 with object lock.
8. Build SLO dashboards and alert catalog (CloudWatch + tracing).
9. Add autoscaling policies driven by queue depth and active bid windows.
10. Implement DR backup replication and recovery runbooks.
11. Run load, resilience, and failover tests; tune bottlenecks.
12. Produce architecture decision records (ADRs) for all non-default choices.

---

## 11) Final Conclusion
Proceed with the current product roadmap, but treat this architecture baseline as a **hard implementation contract** for pilot and launch gates. The marketplace can meet V1 business goals if AWS service boundaries, scaling controls, governance guardrails, and DR evidence are implemented before commercial launch—not deferred to post-launch hardening.