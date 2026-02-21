# Infrastructure Implementation Instructions — LoanBid Exchange (AWS)

## 1) Infrastructure Objectives (V1)

Build an AWS-native, production-ready platform that supports Israel-first marketplace launch in 90 days with:
- Secure borrower/lender data boundaries (PII tokenization + auditability)
- Low-latency bid ingestion and offer-board reads
- High reliability during bid windows and pilot ramp
- Fast, controlled rollout from dev → staging → prod
- Cost discipline suitable for pre-scale marketplace economics

---

## 2) Target AWS Architecture (Reference)

## Core Runtime
- **Compute**: Amazon ECS Fargate (primary services)
- **Ingress**: Route53 + AWS WAF + ALB
- **Edge/static UI**: CloudFront + S3 (frontend artifacts)
- **Async/eventing**: EventBridge (domain events) + SQS (durable queues, retries, DLQs)
- **Data**:
  - Aurora PostgreSQL (primary transactional store, Multi-AZ in prod)
  - DynamoDB (idempotency keys, workflow state, connector throttling metadata)
  - S3 (immutable audit export bundles, analytics raw events)
  - ElastiCache Redis (optional for ranking cache/session acceleration)
- **Secrets/config**: AWS Secrets Manager + SSM Parameter Store
- **Observability**: CloudWatch Logs/Metrics/Alarms, X-Ray tracing, CloudTrail, AWS Config

## Service Mapping (from product architecture)
- Borrower Intake Service → ECS service + Aurora + consent/audit event publishing
- Routing Engine → ECS service + rules in Aurora/DynamoDB + EventBridge triggers
- Bid Orchestrator → ECS worker service + SQS queues + scheduler/timers
- Lender Connector API → ECS service + per-lender adapter modules + idempotency via DynamoDB
- Ranking Service → ECS service + cache + explanation payload generation
- Compliance/Audit Ledger → append-only audit stream to S3 (+ optional QLDB later if required)
- Analytics pipeline → EventBridge/S3 landing + Athena/QuickSight (phase-appropriate)

## Network & Security Baseline
- VPC across 3 AZs (prod), 2 AZs (dev/staging acceptable)
- Public subnets: ALB/NAT only; Private subnets: ECS/RDS/Redis
- Security groups least-privilege, no broad intra-VPC allow rules
- KMS encryption for RDS, S3, logs, Secrets Manager
- TLS everywhere (ACM certs at ALB/CloudFront)
- IAM task roles per service (no shared admin task role)

---

## 3) Environment Strategy & Account Topology

## Recommended account split
- **Shared Services Account**: ECR, central logs archive, security tooling
- **Non-Prod Account**: dev + staging
- **Prod Account**: production only

(If account split is not feasible immediately, isolate by strict IAM boundaries and separate VPCs/state while planning migration by M4.)

## Environment definitions
- **dev**: rapid iteration, synthetic data, relaxed autoscaling floor
- **staging**: production-like topology, synthetic lenders + pre-prod connector certification
- **prod**: HA baseline, strict change control, full alerting/on-call

## Terraform structure
- `infrastructure/terraform/global`: Route53/ACM/ECR/KMS/shared primitives
- `infrastructure/terraform/environments/dev|staging|prod`: environment root stacks
- Remote state: S3 + DynamoDB locking + versioning + KMS encryption
- Mandatory CI sequence: `fmt` → `validate` → `tflint/checkov` → `plan -out` → manual approval → apply

---

## 4) Deployment & Release Architecture

## CI/CD (GitHub Actions + OIDC)
1. Build/test application
2. Build container image (multi-stage, non-root), scan (Trivy/Inspector)
3. Push immutable tag to ECR (no `latest` in ECS task definitions)
4. Terraform plan/apply infra changes (separate workflow)
5. ECS rolling deploy with health checks

## Deployment policy
- **Dev/Staging**: rolling deployments with automatic rollback on ALB/ECS health failures
- **Prod**: blue/green via CodeDeploy for high-risk services (Bid Orchestrator, Connector API, Ranking)
- Database migrations: backward-compatible first, destructive changes only after code cutover and backup checkpoint

---

## 5) Observability Implementation Instructions

## Golden signals and SLO instrumentation
- **Latency**: bid ingestion P95/P99, offer board read P95/P99
- **Traffic**: requests/min, offers submitted/min, lender callback rate
- **Errors**: 5xx rate by service, connector failure rate by lender
- **Saturation**: ECS CPU/memory, DB connections, queue depth/age

## Required telemetry
- Structured JSON logs with correlation IDs (`requestId`, `offerId`, `lenderAnonId`)
- End-to-end distributed tracing across intake → routing → orchestration → ranking
- Custom business metrics in CloudWatch:
  - request→first-offer latency
  - offers/request
  - shortlist rate
  - expiry rejection count
  - consent traceability coverage

## Alerting (minimum)
- High 5xx rate per service (5-min and 30-min windows)
- SQS DLQ non-zero and queue age breach
- ECS task restart storm / desired vs running mismatch
- Aurora CPU, free storage, replica lag (if used), connection exhaustion
- Missing audit events threshold breach (compliance-critical)

## Dashboards
- Executive launch KPI board
- Service reliability board
- Connector-by-lender quality board
- Compliance evidence board (consent/audit/disclosure version integrity)

---

## 6) Reliability, DR, and Operational Resilience

## Availability targets (infra-aligned)
- Platform target: 99.9% monthly for bid orchestration path
- Zero tolerated loss for compliance-required audit events

## Resilience controls
- SQS + DLQ for all lender/async workflows
- Idempotent consumers using DynamoDB idempotency keys
- Circuit breakers/timeouts for lender adapters
- Graceful degradation: show partial board if subset of lenders fails
- ECS autoscaling by CPU + custom queue depth metrics

## Backup/restore
- Aurora automated backups + PITR enabled
- Daily snapshot retention policy (prod >= 30 days)
- S3 versioning + Object Lock (where legal/compliance requires immutability)
- Quarterly restore drills (RDS restore + audit export verification)

## DR posture (V1 pragmatic)
- Single-region active (eu-central-1 or me-central-1 based on legal/latency decision)
- Cross-region backup copy for RDS snapshots and S3 critical audit artifacts
- Defined RTO/RPO targets:
  - RTO: 4 hours (V1)
  - RPO: 15 minutes for transactional DB (via backups/logs)

---

## 7) Cost Control Instructions (Pre-Scale Discipline)

## Guardrails
- Mandatory resource tagging: `Environment`, `Project`, `Owner`, `CostCenter`, `ManagedBy`
- AWS Budgets alerts at 50%/80%/100% monthly threshold per environment
- Cost anomaly detection enabled

## Expected cost hotspots + actions
1. **NAT Gateway**: minimize egress via VPC endpoints (S3, ECR, CloudWatch Logs, Secrets Manager)
2. **Aurora**: right-size instance class; use storage autoscaling with alarms
3. **Fargate**: start with conservative min tasks in dev/staging; autoscale in prod
4. **CloudWatch Logs**: retention policies by environment (e.g., dev 14d, staging 30d, prod 90d+)
5. **Data transfer**: colocate services and data in same AZ patterns where feasible

## Governance cadence
- Weekly cost review during pilot (WS4)
- Infracost delta required in PRs touching Terraform
- Monthly rightsizing recommendations captured and actioned

---

## 8) Rollout Plan (Mapped to 90-Day Milestones)

## M1 (Day 1–15): Foundation
- Stand up baseline VPC/network/security modules
- Provision dev + staging core stacks (ECS, RDS, SQS, ALB, Secrets)
- Implement CI/CD OIDC roles and ECR repos
- Create baseline dashboards/alarms and log schema

## M2 (Day 16–30): Core E2E Infrastructure Hardening
- Add event bus, DLQs, idempotency stores, tracing
- Run synthetic lender E2E load baseline in staging
- Validate schema/version propagation into logs/events
- Implement backup policies and first restore dry run

## M3 (Day 31–60): Pilot Reliability
- Enable prod stack (initial capacity) with strict IAM/policies
- Execute game days: connector timeout, queue backlog, DB failover scenario
- Tune autoscaling and alert thresholds from pilot traffic
- Establish on-call runbooks and escalation paths

## M4 (Day 61–90): Launch Readiness
- Blue/green for critical services in prod
- Complete cost optimization pass (NAT endpoints, task sizing)
- Final DR validation and compliance evidence export drill
- Freeze launch infra baseline and produce Go/No-Go packet

---

## 9) Acceptance Criteria (Infrastructure Go/No-Go)

Infrastructure is launch-ready only if all are true:
1. **Environment completeness**: dev/staging/prod stacks reproducible via Terraform with no manual drift.
2. **Security baseline**: encryption at rest/in transit, least-privilege IAM roles, WAF and audit logging active.
3. **Observability**: required dashboards + alarms live; SLI metrics validated against sampled raw logs/events.
4. **Reliability evidence**: load + resilience tests meet SLO thresholds; DLQ and retry behaviors verified.
5. **Data durability**: backup/PITR enabled and one successful restore drill documented.
6. **Deployment safety**: rollback procedure tested in staging and once in prod canary/blue-green path.
7. **Cost governance**: budgets/alerts/tags in place; projected monthly spend reviewed and accepted.
8. **Operational readiness**: runbooks complete for outage, lender degradation, DB incidents, and compliance export requests.

---

## 10) Immediate Next Infrastructure Actions (Week 1)

1. Finalize region choice and legal data residency decision for Israel launch.
2. Create Terraform environment skeletons + remote state backends.
3. Provision ECR, baseline ECS cluster, ALB, and Aurora in staging.
4. Define IAM task roles per service and GitHub OIDC deployment roles.
5. Implement structured logging/tracing libraries and correlation ID standard.
6. Create CloudWatch dashboard/alarms for latency, errors, queue health, DB saturation.
7. Establish SQS/DLQ patterns for orchestrator and connector workflows.
8. Configure AWS Budgets, cost anomaly detection, and mandatory tag policies.
9. Draft incident + rollback runbooks and assign on-call ownership.
10. Schedule first synthetic load test and backup/restore drill date.

These instructions should be treated as the infrastructure execution baseline for WS2/WS4/WS5 and audited at each milestone gate.