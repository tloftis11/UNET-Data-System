# UNet Next — System Architecture
## AWS-Native Architecture for the Modernized OPTN Data Platform

**Version:** 1.0 — Draft for Discussion
**Date:** March 30, 2026
**Companion Document:** Modernized UNet System Requirements v1.0
**Cloud Platform:** AWS GovCloud (US)
**AI Platform:** Amazon Bedrock

---

## 1. Architecture Overview

UNet Next is decomposed into 12 domain services, a shared platform layer, and a cross-cutting infrastructure layer — all deployed on AWS GovCloud (US) to meet FedRAMP High requirements. The architecture follows an event-driven microservices pattern where each service owns its data, exposes versioned APIs, and communicates asynchronously through a central event backbone powered by Amazon MSK (Managed Streaming for Apache Kafka).

Amazon Bedrock serves as the AI/ML foundation, providing managed access to foundation models for predictive analytics, clinical data extraction, natural language processing, and decision support — without requiring the platform to train or host custom models on dedicated GPU infrastructure.

### 1.1 Design Tenets

1. **Life-critical path isolation.** The organ matching and allocation pipeline runs in a dedicated, hardened compute path with no shared dependencies on analytics, reporting, or administrative functions. If every dashboard goes dark, matching still runs.
2. **Event sourcing as the system of record.** All state changes to candidates, donors, match runs, and transplants are captured as immutable events in MSK. Current state is a projection of the event stream. This enables full auditability, replay, and downstream consumption by SRTR/TDS without batch transfers.
3. **API-first, UI-second.** Every service exposes a gRPC internal API and a REST/FHIR external API. The web and mobile UIs are thin clients consuming these APIs. Member institutions interact through the same APIs.
4. **Progressive autonomy for AI.** Bedrock-powered features start as advisory (human-in-the-loop), with paths to semi-automated workflows only after validated performance thresholds are met and clinical governance approves.

---

## 2. AWS Account and Network Topology

### 2.1 Account Structure (AWS Organizations)

The platform uses a multi-account strategy aligned with AWS GovCloud best practices:

```
OPTN Root (AWS Organizations)
├── Management Account          — Organizations, billing, SCPs
├── Security Account            — GuardDuty, Security Hub, audit logs
├── Log Archive Account         — Centralized CloudTrail, VPC Flow Logs
├── Shared Services Account     — Transit Gateway, DNS, container registries
├── Production Account          — Production workloads
├── Staging Account             — Pre-production mirror
├── Development Account         — Dev/test environments
├── Sandbox Account             — Policy simulation, integration testing
├── Data Analytics Account      — Data lake, Bedrock, research workloads
└── DR Account (alternate region) — Warm standby for disaster recovery
```

Service Control Policies (SCPs) enforce guardrails: no public S3 buckets, no unencrypted EBS, mandatory tagging, region restrictions (us-gov-west-1 primary, us-gov-east-1 DR).

### 2.2 Network Architecture

```
                         ┌─────────────────────────────┐
                         │       AWS GovCloud           │
                         │                              │
  Internet ──► CloudFront ──► ALB (Public) ──────────┐  │
                         │                            │  │
  VPN/DX ──► Transit GW ─┤                            │  │
  (Member    (Hub)        │   ┌────────────────────┐  │  │
  Institutions)           │   │  Production VPC     │  │  │
                         │   │                    │  │  │
                         │   │  ┌──────────────┐  │  │  │
                         │   │  │ Public Subnet │◄─┘  │  │
                         │   │  │ ALB, NAT GW   │     │  │
                         │   │  └──────┬───────┘     │  │
                         │   │         │              │  │
                         │   │  ┌──────▼───────┐     │  │
                         │   │  │ App Subnet    │     │  │
                         │   │  │ EKS Nodes     │     │  │
                         │   │  │ (Private)     │     │  │
                         │   │  └──────┬───────┘     │  │
                         │   │         │              │  │
                         │   │  ┌──────▼───────┐     │  │
                         │   │  │ Data Subnet   │     │  │
                         │   │  │ RDS, ElastiC. │     │  │
                         │   │  │ MSK (Private) │     │  │
                         │   │  └──────────────┘     │  │
                         │   └────────────────────┘  │  │
                         └─────────────────────────────┘
```

**Key networking components:**

| Component | AWS Service | Purpose |
|---|---|---|
| Edge / CDN | CloudFront | Static assets, public portal, DDoS protection |
| Load Balancing | ALB (Application Load Balancer) | HTTPS termination, path-based routing to services |
| Service Mesh | AWS App Mesh + Envoy | mTLS between services, traffic management, observability |
| DNS | Route 53 | Public and private DNS, health-check-based failover |
| Private Connectivity | AWS Transit Gateway + Direct Connect | Secure connectivity to member institutions, CMS, SRTR |
| Network Firewall | AWS Network Firewall | Stateful inspection, IDS/IPS at VPC ingress/egress |
| VPC Endpoints | PrivateLink | Private access to AWS services (S3, KMS, Bedrock, etc.) |

All inter-service traffic flows through App Mesh with mutual TLS. No service communicates directly via IP; all routing is service-name-based through the mesh.

---

## 3. Domain Services

The platform is decomposed into 12 domain services, each independently deployable and owning its data. Services are grouped into three tiers by criticality.

### 3.1 Tier 1 — Life-Critical Services (99.99% SLA)

These services are on the critical path for organ allocation and placement. They run on dedicated, over-provisioned EKS node groups with priority scheduling, separate from all other workloads.

---

#### 3.1.1 Matching & Allocation Service

**Purpose:** Executes organ-to-candidate matching algorithms, produces rank-ordered candidate lists, manages allocation policy configurations, and detects AOOS events.

**Requirements fulfilled:** FR-MATCH-001 through FR-MATCH-006, FR-AOOS-001 through FR-AOOS-003

**Internal architecture:**

```
                    ┌──────────────────────────────────────┐
                    │     Matching & Allocation Service      │
                    │                                        │
  Donor Registered  │  ┌─────────────┐  ┌───────────────┐  │
  Event (MSK) ─────►│  │ Match Run   │  │ Policy Engine │  │
                    │  │ Orchestrator│◄─┤ (Rules DSL)   │  │
                    │  │             │  │               │  │
                    │  └──────┬──────┘  └───────┬───────┘  │
                    │         │                  │          │
                    │  ┌──────▼──────┐  ┌───────▼───────┐  │
                    │  │ Scoring     │  │ Policy Config │  │
                    │  │ Modules     │  │ Store         │  │
                    │  │ (MELD,KDPI, │  │ (Versioned)   │  │
                    │  │  CAS,EPTS,  │  │               │  │
                    │  │  CPRA)      │  └───────────────┘  │
                    │  └──────┬──────┘                      │
                    │         │                              │
                    │  ┌──────▼──────┐  ┌───────────────┐  │
                    │  │ Rank Engine │  │ AOOS Detector │  │
                    │  │             │──►│               │  │
                    │  └──────┬──────┘  └───────┬───────┘  │
                    │         │                  │          │
                    │    Match Run          AOOS Alert      │
                    │    Completed Event    Event            │
                    └──────────┬─────────────┬──────────────┘
                               │             │
                               ▼             ▼
                              MSK Event Backbone
```

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (dedicated node group) | m6i.4xlarge instances, 3 AZ, min 6 pods, HPA enabled |
| Primary Database | Amazon Aurora PostgreSQL (Global Database) | Multi-AZ, read replicas, provisioned IOPS |
| Policy Config Store | DynamoDB (Global Tables) | Versioned policy rules, sub-ms reads |
| Scoring Cache | ElastiCache Redis (cluster mode) | Pre-computed candidate scores, sub-ms lookups |
| Event Output | MSK (Kafka) | `match-run-completed`, `aoos-detected` topics |
| Audit Store | S3 + Amazon QLDB | Immutable match run records with cryptographic verification |

**Policy Engine design:** Allocation policies are expressed in a domain-specific rules language (based on Open Policy Agent / Rego or a custom DSL) stored in DynamoDB with full version history. Policy administrators author rules through a policy management UI; changes are validated in the Sandbox account against historical match data before promotion. The scoring modules (MELD, KDPI, EPTS, Lung CAS, CPRA, heart status) are independently versioned Lambda functions invoked by the Match Run Orchestrator, enabling individual scoring model updates without redeploying the core engine.

**Performance design:** Candidate data for active waitlist patients is pre-loaded into ElastiCache Redis in a denormalized format optimized for match run queries. When a match run initiates, the engine reads donor data, pulls the candidate set from cache, applies policy rules, invokes scoring modules in parallel, and produces the ranked list. Target: < 60s for standard runs (single organ, < 50,000 candidates evaluated), < 120s for complex (multi-organ, highly sensitized).

---

#### 3.1.2 Organ Offer Service

**Purpose:** Manages the organ offer workflow from match run completion through final acceptance — presenting offers to transplant centers, tracking responses, enforcing acceptance windows, managing parallel offers, and escalating timeouts.

**Requirements fulfilled:** FR-OFFER-001 through FR-OFFER-006

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (dedicated node group) | Co-located with Matching service node group |
| Workflow Engine | AWS Step Functions (Express) | Offer state machines with timeout/escalation logic |
| Database | Aurora PostgreSQL | Offer chain history, decline reasons |
| Real-time Notifications | Amazon SNS + Amazon Pinpoint | Push, SMS, email to transplant center on-call teams |
| WebSocket Push | API Gateway WebSocket API | Real-time offer updates to connected UI clients |
| Media Storage | S3 | Donor images, lab PDFs associated with offers |

**Offer workflow:** When a match run completes, the Offer Service consumes the `match-run-completed` event and initiates a Step Functions state machine for each organ. The state machine walks down the ranked list, presenting offers to centers via push notification (SNS/Pinpoint), WebSocket, and API. Each center gets a configurable acceptance window (default: 60 minutes for kidney, 30 minutes for heart). On timeout, the offer auto-advances. On decline, the structured reason code is captured and the next center is notified. For hard-to-place organs, the state machine supports parallel offers to N centers simultaneously with first-accepted-wins logic.

---

#### 3.1.3 Waitlist Service

**Purpose:** Maintains the authoritative national waiting list for all organ types. Manages candidate registration, status changes, clinical data updates, priority score calculations, and multiple-listing logic.

**Requirements fulfilled:** FR-WAIT-001 through FR-WAIT-008

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (dedicated node group) | Shared Tier 1 node group |
| Primary Database | Aurora PostgreSQL (Global Database) | Single source of truth for all candidates |
| Event Store | MSK | `candidate-registered`, `candidate-status-changed`, `score-recalculated` |
| Score Cache | ElastiCache Redis | Pre-computed priority scores fed to Matching service |
| FHIR Ingest | API Gateway + Lambda | FHIR R4 endpoint for EHR clinical data feeds |
| Alerts | Amazon EventBridge + SNS | Expiring labs, missing data, inconsistency alerts |

**Data model:** Candidates are stored as event-sourced entities. Every registration, status change, lab update, and score recalculation is an immutable event. The current state view is a materialized projection in Aurora, while the full event history is retained in MSK (compacted topics) and archived to S3. Multiple-listing is handled through a `patient-identity` aggregate that links candidate records across centers, with time accrual rules applied at the aggregate level.

---

### 3.2 Tier 2 — Mission-Critical Services (99.95% SLA)

These services support organ procurement, transport, and data collection. They are essential to daily operations but not on the real-time matching critical path.

---

#### 3.2.1 Donor Management Service

**Purpose:** Manages the full donor lifecycle — referral intake, OPO evaluation, donor registration, clinical data management, organ recovery coordination, and living donor workflows.

**Requirements fulfilled:** FR-DONOR-001 through FR-DONOR-007

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (Tier 2 node group) | m6i.2xlarge, 3 AZ |
| Primary Database | Aurora PostgreSQL | Donor records, referral pipeline, recovery data |
| Document/Image Store | S3 + CloudFront | Donor imaging, biopsy results, lab reports |
| Donor Quality Calc | Lambda | KDPI, DRI calculations triggered on data update |
| External Integration | EventBridge + SQS | Death registry feeds (NTIS DMF, state vital records) |
| Event Output | MSK | `donor-registered`, `donor-updated`, `organ-recovered` |

**Referral-to-procurement pipeline:** The service tracks every stage: hospital referral → OPO evaluation → family authorization → donor management → organ recovery → organ disposition. Each stage transition is an event. The Bedrock AI layer (see Section 5) assists with clinical data extraction from uploaded donor records.

---

#### 3.2.2 Transport & Logistics Service

**Purpose:** End-to-end organ tracking from recovery through transplant. Labeling, packaging compliance, GPS tracking, cold ischemia time calculation, transport provider integration, and organ check-in verification.

**Requirements fulfilled:** FR-TRANS-001 through FR-TRANS-006

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (Tier 2 node group) | Shared Tier 2 pool |
| Database | DynamoDB | High-throughput location updates, organ tracking state |
| Location Ingestion | IoT Core + Kinesis Data Streams | GPS device telemetry from transport containers |
| Label Generation | Lambda + S3 | PDF/ZPL label generation per OPTN labeling policy |
| Ischemia Calculator | Lambda | Real-time cold ischemia time based on recovery timestamp + current time |
| Transport Provider API | API Gateway | Standardized API for courier/charter/drone integrations |
| Event Output | MSK | `organ-in-transit`, `organ-arrived`, `organ-checked-in`, `transport-incident` |

**Organ check-in:** At the receiving transplant center, the organ is scanned (barcode/QR from the generated label). The service verifies the organ ID against the accepted match, confirms the intended recipient, and logs the check-in event. Any mismatch triggers an immediate alert to the OPO coordinator and OPTN safety team.

---

#### 3.2.3 Data Collection Service (TIEDI Next)

**Purpose:** Manages all OPTN-mandated electronic data collection forms — submission, validation, deadlines, data locks, corrections, and bulk import/export. Replaces the legacy TIEDI system.

**Requirements fulfilled:** FR-DATA-001 through FR-DATA-007

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (Tier 2 node group) | Shared Tier 2 pool |
| Primary Database | Aurora PostgreSQL | Form submissions, validation state, correction history |
| Form Schema Registry | S3 + DynamoDB | Versioned JSON Schema definitions for all form types |
| Validation Engine | Lambda | Real-time field-level and cross-field validation rules |
| Deadline/Lock Scheduler | EventBridge Scheduler + Step Functions | Automated deadline reminders, escalation, data lock enforcement |
| Bulk Import/Export | S3 + AWS Batch | CSV, JSON, FHIR bundle, XML processing |
| FHIR Ingest | API Gateway + Lambda | Direct EHR-to-form data submission via FHIR |
| Event Output | MSK | `form-submitted`, `form-locked`, `form-correction-requested` |

**Pre-population:** When a new form is generated (e.g., a follow-up form triggered 6 months post-transplant), the service pre-populates fields from the candidate's existing data, the donor record, and the transplant event record — reducing manual entry. The Bedrock AI layer assists with mapping unstructured clinical notes to structured form fields (see Section 5).

---

#### 3.2.4 Transplant & Outcomes Service

**Purpose:** Records transplant events, manages follow-up schedules, collects outcome data, calculates risk-adjusted program-specific metrics, and tracks graft/patient survival.

**Requirements fulfilled:** FR-TX-001 through FR-TX-006

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (Tier 2 node group) | Shared Tier 2 pool |
| Primary Database | Aurora PostgreSQL | Transplant records, follow-up schedules, outcomes |
| Follow-up Scheduler | EventBridge Scheduler + Step Functions | 6mo/1yr/2yr follow-up form generation and reminders |
| Outcomes Calculator | AWS Batch + Lambda | Risk-adjusted survival calculations (batch processing) |
| CMS Data Integration | SQS + Lambda | Medicare claims, death records ingestion |
| FHIR Ingest | API Gateway + Lambda | Automated outcome feeds from transplant center EHRs |
| Event Output | MSK | `transplant-reported`, `follow-up-submitted`, `graft-failure-reported` |

---

### 3.3 Tier 3 — Operational Services (99.9% SLA)

These services support compliance, member management, public data, notifications, and the patient portal. Important for system operations and transparency but not on the organ placement critical path.

---

#### 3.3.1 Compliance & Member Service

**Purpose:** OPTN member registry, policy compliance tracking, MPSC workflow support, and automated compliance reporting.

**Requirements fulfilled:** FR-COMP-001 through FR-COMP-004

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (Tier 3 node group) | Shared Tier 3 pool |
| Database | Aurora PostgreSQL | Member registry, compliance history, MPSC cases |
| Report Generation | AWS Batch + Lambda | Scheduled and on-demand compliance reports |
| Document Store | S3 | MPSC review packets, corrective action documents |
| Workflow Engine | Step Functions | MPSC peer review workflow state machines |
| Event Input | MSK | Consumes events from all services to compute compliance metrics |

---

#### 3.3.2 Public Data & Transparency Service

**Purpose:** Public-facing data portal, interactive dashboards, public API, system health metrics, and the "find a transplant center" tool.

**Requirements fulfilled:** FR-PUB-001 through FR-PUB-005

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Web Application | CloudFront + S3 (static site) | React SPA for public dashboards |
| Public API | API Gateway (REST) | Rate-limited, unauthenticated, de-identified aggregate data |
| Data Materialization | Lambda + DynamoDB | Pre-computed aggregate views refreshed daily/weekly |
| Visualization Backend | EKS (Tier 3) | QuickSight Embedded or custom chart APIs |
| System Health | CloudWatch Synthetics + Route 53 Health Checks | Public status page data |
| CDN | CloudFront | Edge caching for high-traffic public pages |

---

#### 3.3.3 Notification Service

**Purpose:** Centralized notification delivery across all channels — push, SMS, email, in-app, and WebSocket — with guaranteed delivery for time-critical organ offer notifications and configurable preferences per user.

**Requirements fulfilled:** NFR-UX-006, FR-WAIT-008, FR-OFFER-001

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Compute | EKS (Tier 3 node group) | Shared Tier 3 pool |
| Channel Dispatch | Amazon SNS (push/SMS), Amazon SES (email), Pinpoint (campaigns) | Multi-channel with delivery receipts |
| WebSocket | API Gateway WebSocket API | Real-time in-app notifications |
| Preference Store | DynamoDB | Per-user notification channel preferences |
| Dead Letter / Retry | SQS (DLQ) + Step Functions | Guaranteed delivery with escalation on failure |
| Event Input | MSK | Consumes notification-triggering events from all services |

**Guaranteed delivery for organ offers:** Organ offer notifications follow a cascade: push notification first, with SMS fallback at 2 minutes if no read receipt, and phone call escalation (via Amazon Connect) at 5 minutes if still unacknowledged. Delivery status is tracked end-to-end and reported to the Offer Service.

---

#### 3.3.4 Patient Portal Service

**Purpose:** Secure, patient-facing portal for waiting list status, educational materials, and transplant center comparison tools.

**Requirements fulfilled:** FR-WAIT-007, FR-PUB-005

**AWS services:**

| Component | Service | Configuration |
|---|---|---|
| Web Application | CloudFront + S3 | React SPA, WCAG 2.1 AA compliant |
| API Backend | API Gateway + Lambda | Patient-scoped read-only APIs |
| Authentication | Cognito (patient identity pool) | MFA, identity verification, consent management |
| Content | S3 + DynamoDB | Educational materials, FAQs (English + Spanish) |

---

## 4. Shared Platform Layer

These components are shared across all domain services.

### 4.1 Identity & Access Management

**Requirements fulfilled:** NFR-SEC-003, NFR-SEC-004

| Component | AWS Service | Purpose |
|---|---|---|
| Workforce Identity | AWS IAM Identity Center (SSO) | Federated SSO for OPTN member staff, HRSA, OPO users |
| Patient Identity | Amazon Cognito | Patient portal authentication |
| Service Identity | IAM Roles for Service Accounts (IRSA) | Pod-level identity in EKS, no long-lived credentials |
| Authorization | Amazon Verified Permissions (Cedar) | Fine-grained RBAC/ABAC policy evaluation |
| Key Management | AWS KMS (FIPS 140-2 Level 3) + CloudHSM | Encryption keys, signing keys, HSM for critical operations |
| Certificate Management | AWS Private CA + ACM | mTLS certificates for service mesh |
| Secrets | AWS Secrets Manager | Database credentials, API keys, rotated automatically |

**Authorization model:** Amazon Verified Permissions uses the Cedar policy language to express fine-grained access rules. Policies encode role-based constraints (e.g., "OPO coordinators at OPO-X can view/edit donors managed by OPO-X") and attribute-based constraints (e.g., "Transplant surgeons at Center-Y can view organ offers for candidates at Center-Y"). Every API call is evaluated against Cedar policies at the API Gateway and service level.

### 4.2 API Gateway Layer

**Requirements fulfilled:** AR-MICRO-004, NFR-INTEROP-001, NFR-INTEROP-006

| Component | AWS Service | Purpose |
|---|---|---|
| External API Gateway | Amazon API Gateway (REST + HTTP) | Public and member-facing APIs, rate limiting, API keys |
| FHIR API Gateway | API Gateway + Lambda (HAPI FHIR Facade) | HL7 FHIR R4 endpoints for EHR integration |
| Internal API Gateway | AWS App Mesh (Envoy) | Service-to-service routing, mTLS, retries, circuit breaking |
| Developer Portal | API Gateway Developer Portal + S3 | API documentation, sandbox access, SDK downloads |
| API Versioning | API Gateway stages + custom domain mappings | v1, v2, etc. with deprecation lifecycle |

**FHIR Implementation:** A HAPI FHIR facade running on Lambda provides conformant FHIR R4 endpoints. It translates between FHIR resources and internal domain models. Supported FHIR resources include: Patient (candidates), Condition, Observation (lab values), Procedure (transplant events), Specimen (organs), Organization (transplant centers, OPOs), and ServiceRequest (organ offers). SMART on FHIR authorization is supported for EHR-launched applications.

### 4.3 Event Backbone

**Requirements fulfilled:** AR-EVENT-001 through AR-EVENT-004, AR-DATA-004

| Component | AWS Service | Configuration |
|---|---|---|
| Event Streaming | Amazon MSK (Apache Kafka) | 3-broker minimum per AZ, 6 AZ total, replication factor 3 |
| Schema Registry | AWS Glue Schema Registry | Avro schemas, backward compatibility enforcement |
| Event Archival | MSK → Kinesis Data Firehose → S3 | All events archived to S3 Glacier for 10-year retention |
| CDC Streams | MSK Connect + Debezium | Change Data Capture from Aurora databases |
| External Subscriptions | MSK + Amazon EventBridge Pipes | SRTR/TDS event feeds via managed Kafka consumers |

**Topic structure:**

```
optn.matching.match-run-initiated
optn.matching.match-run-completed
optn.matching.aoos-detected
optn.offers.organ-offered
optn.offers.organ-accepted
optn.offers.organ-declined
optn.offers.offer-timed-out
optn.waitlist.candidate-registered
optn.waitlist.candidate-status-changed
optn.waitlist.score-recalculated
optn.donors.donor-registered
optn.donors.donor-updated
optn.donors.organ-recovered
optn.transport.organ-in-transit
optn.transport.organ-arrived
optn.transport.organ-checked-in
optn.transport.transport-incident
optn.transplants.transplant-reported
optn.transplants.follow-up-submitted
optn.transplants.graft-failure-reported
optn.data-collection.form-submitted
optn.data-collection.form-locked
optn.compliance.member-flagged
optn.system.health-check
```

All events include: `event_id` (UUID), `event_type`, `timestamp` (UTC), `source_service`, `correlation_id`, `actor_id`, `payload`, `schema_version`. Events are persisted indefinitely in S3 (Glacier after 1 year) and retained in MSK for 30 days for real-time consumption.

### 4.4 Observability Stack

**Requirements fulfilled:** OPS-002 through OPS-006

| Component | AWS Service | Purpose |
|---|---|---|
| Metrics | Amazon CloudWatch Metrics + Prometheus (AMP) | Service-level metrics, SLI/SLO dashboards |
| Logging | CloudWatch Logs + OpenSearch Serverless | Structured JSON logs, full-text search |
| Distributed Tracing | AWS X-Ray + OpenTelemetry | End-to-end request tracing across all services |
| Alerting | CloudWatch Alarms + SNS + PagerDuty integration | Tiered alerting (P1-P4) with escalation |
| Synthetic Monitoring | CloudWatch Synthetics | Canary scripts simulating critical user journeys |
| Status Page | Route 53 Health Checks + CloudWatch | Public-facing status page (integrated with Statuspage.io or equivalent) |
| Audit Logging | CloudTrail (Organization Trail) | All AWS API calls, management events, data events |
| SIEM | Amazon Security Lake + OpenSearch | Security event aggregation, correlation, threat detection |

**SLI/SLO definitions:**

| Service | SLI | SLO |
|---|---|---|
| Matching Engine | Match run latency (p99) | < 120 seconds, 99.99% of runs |
| Offer Service | Notification delivery latency | < 5 seconds, 99.99% |
| Waitlist Service | API availability | 99.99% (measured per 5-minute window) |
| All Tier 2 Services | API availability | 99.95% |
| All Tier 3 Services | API availability | 99.9% |
| Event Backbone | End-to-end event delivery | < 500ms, 99.99% |

---

## 5. AI/ML Layer — Amazon Bedrock

Amazon Bedrock provides the AI foundation for UNet Next. All AI features are advisory (human-in-the-loop) and follow the progressive autonomy tenet. No AI system makes allocation decisions — allocation is governed solely by the deterministic policy engine in the Matching Service.

### 5.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    AI/ML Platform                          │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Amazon Bedrock                           │ │
│  │                                                       │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │ │
│  │  │ Foundation  │  │ Titan       │  │ Titan        │ │ │
│  │  │ Model (LLM) │  │ Embeddings  │  │ Image        │ │ │
│  │  │             │  │             │  │ (future)     │ │ │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬───────┘ │ │
│  │         │                │                 │         │ │
│  │  ┌──────▼─────────────────▼─────────────────▼──────┐ │ │
│  │  │            Bedrock Guardrails                    │ │ │
│  │  │  (PHI filtering, content policies, PII redact.) │ │ │
│  │  └──────┬──────────────────────────────────────────┘ │ │
│  └─────────┼────────────────────────────────────────────┘ │
│            │                                               │
│  ┌─────────▼────────────────────────────────────────────┐ │
│  │           Bedrock Knowledge Bases                     │ │
│  │  ┌──────────────┐  ┌───────────────┐  ┌───────────┐ │ │
│  │  │ OPTN Policy  │  │ Clinical      │  │ Historical│ │ │
│  │  │ Documents    │  │ Guidelines    │  │ Match     │ │ │
│  │  │              │  │               │  │ Patterns  │ │ │
│  │  └──────────────┘  └───────────────┘  └───────────┘ │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │           Custom Model Training (SageMaker)           │ │
│  │  ┌─────────────────┐  ┌────────────────────────────┐ │ │
│  │  │ Organ Acceptance │  │ Post-Transplant Outcome    │ │ │
│  │  │ Prediction Model │  │ Prediction Model           │ │ │
│  │  └─────────────────┘  └────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### 5.2 Bedrock Use Cases

#### 5.2.1 Clinical Data Extraction and Structuring
**Model:** Bedrock foundation model (LLM)
**Guardrails:** PHI-aware, output validation, human review required
**Integration point:** Donor Management Service, Data Collection Service

When OPOs upload unstructured clinical documents (hospital records, pathology reports, imaging reports), the LLM extracts structured data fields and maps them to OPTN form schemas. The extracted data is presented to the OPO coordinator for validation before submission — the AI pre-fills, the human confirms.

**AWS flow:**
```
S3 (document upload) → Lambda (trigger) → Bedrock (LLM) → 
  Bedrock Guardrails (PHI policy) → Lambda (structure output) → 
    SQS → Data Collection Service (pending human review)
```

#### 5.2.2 Organ Acceptance Prediction
**Model:** Custom ML model trained on SageMaker, hosted on SageMaker Endpoints
**Training data:** Historical offer/acceptance/decline data from OPTN (de-identified)
**Integration point:** Organ Offer Service

Given a donor-organ-candidate triplet, the model predicts the probability that a transplant center will accept the offer. This is displayed to OPO coordinators as advisory information to support their workflow — for example, identifying likely-to-be-declined offers early so parallel offer strategies can be initiated sooner.

**AWS flow:**
```
Offer Service → SageMaker Endpoint (real-time inference) → 
  Acceptance probability score → Offer Service UI (advisory display)
```

**Model governance:** The model is retrained quarterly on updated historical data. Performance is monitored via SageMaker Model Monitor with drift detection. Predictions are logged and compared against actual outcomes. Results are published in the operational analytics dashboards. The model is validated by the OPTN Data Advisory Committee before each production update.

#### 5.2.3 Wait Time Estimation
**Model:** Custom ML model on SageMaker
**Integration point:** Waitlist Service, Patient Portal

Given a candidate's clinical profile (organ type, blood type, CPRA, geography, status), the model estimates the distribution of expected wait times. This is displayed to patients in the Patient Portal with appropriate uncertainty ranges and clinical disclaimers.

#### 5.2.4 Policy Simulation Copilot
**Model:** Bedrock foundation model (LLM) + Bedrock Knowledge Bases
**Integration point:** Matching & Allocation Service (sandbox mode)

Policy analysts can describe proposed allocation policy changes in natural language. The LLM, grounded in OPTN policy documents and historical allocation data via Bedrock Knowledge Bases, helps translate the intent into the policy DSL, identifies potential edge cases, and summarizes the projected impact of simulation runs. All policy changes still require formal governance review and approval.

#### 5.2.5 Compliance Anomaly Detection
**Model:** Bedrock foundation model (LLM) + custom anomaly detection (SageMaker)
**Integration point:** Compliance & Member Service

The system monitors compliance signals across all OPTN members — data submission patterns, offer response times, AOOS rates, outcome reporting completeness — and flags anomalies for MPSC review. The LLM generates natural-language summaries of flagged patterns to assist peer reviewers. Anomaly detection models are trained on historical compliance data to identify statistically significant deviations.

#### 5.2.6 Research Data Assistance
**Model:** Bedrock foundation model (LLM) + Bedrock Knowledge Bases
**Integration point:** Research Data Portal (Analytics Account)

Researchers interacting with the self-service data request portal can describe their research question in natural language. The LLM, grounded in the OPTN/SRTR data dictionary and metadata catalog, helps construct the appropriate data request — identifying relevant variables, suggesting cohort definitions, and flagging potential confounders. The researcher reviews and submits the formalized request.

### 5.3 Bedrock Safety and Governance

| Control | Implementation |
|---|---|
| PHI/PII Protection | Bedrock Guardrails with custom PHI detection policy; all inputs/outputs scrubbed |
| Prompt Injection Defense | Input validation, system prompt hardening, output schema enforcement |
| Auditability | All Bedrock invocations logged to CloudWatch Logs with full input/output (encrypted at rest) |
| Human-in-the-Loop | No AI output is committed to the system of record without explicit human confirmation |
| Model Selection | Only models available in AWS GovCloud authorized for use |
| Data Residency | All Bedrock inference runs in GovCloud; no data leaves GovCloud region |
| Bias Monitoring | Prediction models monitored via SageMaker Model Monitor; quarterly bias audits |
| Clinical Validation | All clinical AI features reviewed by OPTN clinical committee before deployment |

---

## 6. Data Architecture

### 6.1 Data Stores by Service

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Data Architecture                              │
│                                                                       │
│  Operational Tier                 Analytical Tier                      │
│  ┌────────────────────────┐      ┌────────────────────────────────┐  │
│  │ Aurora PostgreSQL       │      │ S3 Data Lakehouse              │  │
│  │ (Per-service databases) │─CDC─►│ (Apache Iceberg tables)        │  │
│  │                         │      │                                │  │
│  │ • matching_db           │      │ ┌────────────────────────────┐ │  │
│  │ • waitlist_db           │      │ │ Raw Zone                   │ │  │
│  │ • donor_db              │      │ │ (Events + CDC streams)     │ │  │
│  │ • transplant_db         │      │ └────────────┬───────────────┘ │  │
│  │ • datacollection_db     │      │              │ AWS Glue ETL    │  │
│  │ • compliance_db         │      │ ┌────────────▼───────────────┐ │  │
│  │ • offer_db              │      │ │ Curated Zone               │ │  │
│  └────────────────────────┘      │ │ (Conformed dimensions,     │ │  │
│                                   │ │  fact tables)              │ │  │
│  ┌────────────────────────┐      │ └────────────┬───────────────┘ │  │
│  │ DynamoDB                │      │              │                  │  │
│  │ (High-throughput)       │      │ ┌────────────▼───────────────┐ │  │
│  │                         │      │ │ Consumption Zone           │ │  │
│  │ • policy_config         │      │ │ (Aggregates, SAFs,         │ │  │
│  │ • transport_tracking    │      │ │  de-identified datasets)   │ │  │
│  │ • notification_prefs    │      │ └────────────────────────────┘ │  │
│  │ • public_data_cache     │      └────────────────────────────────┘  │
│  └────────────────────────┘                                           │
│                                   ┌────────────────────────────────┐  │
│  ┌────────────────────────┐      │ Athena + QuickSight            │  │
│  │ ElastiCache Redis       │      │ (Query + Visualization)        │  │
│  │ (Caching)               │      └────────────────────────────────┘  │
│  │                         │                                          │
│  │ • candidate_scores      │      ┌────────────────────────────────┐  │
│  │ • active_waitlist       │      │ Amazon OpenSearch Serverless   │  │
│  │ • session_cache         │      │ (Full-text search, log search) │  │
│  └────────────────────────┘      └────────────────────────────────┘  │
│                                                                       │
│  ┌────────────────────────┐      ┌────────────────────────────────┐  │
│  │ Amazon QLDB             │      │ Amazon Redshift Serverless     │  │
│  │ (Immutable ledger)      │      │ (Complex analytical queries,   │  │
│  │                         │      │  SRTR/TDS data warehouse)      │  │
│  │ • match_run_audit       │      └────────────────────────────────┘  │
│  │ • allocation_decisions  │                                          │
│  └────────────────────────┘                                           │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 Data Flow: Operational → Analytical

```
Aurora (service DBs)
    │
    ├─► MSK Connect (Debezium CDC) ─► MSK ─► Kinesis Firehose ─► S3 Raw Zone
    │
    └─► Event-sourced topics (MSK) ──────► Kinesis Firehose ─► S3 Raw Zone
                                                                    │
                                                              AWS Glue ETL
                                                              (Iceberg merge)
                                                                    │
                                                              S3 Curated Zone
                                                              (Iceberg tables)
                                                                    │
                                                    ┌───────────────┼────────────┐
                                                    │               │            │
                                              Athena/Spark    Redshift       TDS/SRTR
                                              (ad hoc)        (warehouse)    (external feed)
```

### 6.3 Transplant Data Services (TDS) Integration

TDS is HRSA's government-owned authoritative data repository. UNet Next feeds TDS via:

1. **Near-real-time event stream:** MSK topics are consumed by an EventBridge Pipe that delivers events to the TDS ingestion endpoint (API Gateway in the TDS AWS account) with < 1 hour latency.
2. **Daily reconciliation batch:** An AWS Glue job produces a daily reconciliation dataset comparing UNet Next operational state to TDS state, flagging any discrepancies.
3. **Historical backfill:** The full historical dataset (1987–present) is available as Apache Iceberg tables in S3, shared with TDS via S3 cross-account access.

### 6.4 SRTR Integration

SRTR receives data through TDS (the authoritative intermediary) plus direct analytical access:

1. **TDS event feeds** (see above) — SRTR consumes from TDS, not directly from UNet Next operational systems.
2. **Redshift data sharing** — The Analytics Account Redshift cluster shares specific schemas with SRTR's AWS account via Redshift data sharing, enabling SRTR to run analytical queries without maintaining a separate copy.
3. **Standard Analysis Files (SAFs)** — Generated weekly by AWS Glue jobs, de-identified per HIPAA Safe Harbor, and deposited in an S3 bucket accessible to approved SRTR analysts.

---

## 7. Compute Platform — Amazon EKS

### 7.1 Cluster Architecture

```
EKS Cluster: optn-prod-primary (us-gov-west-1)
├── System Node Group (m6i.xlarge × 6)
│   └── CoreDNS, App Mesh controller, Karpenter, cluster add-ons
│
├── Tier 1 Node Group (m6i.4xlarge × 12, dedicated)
│   ├── Matching & Allocation Service
│   ├── Organ Offer Service
│   └── Waitlist Service
│
├── Tier 2 Node Group (m6i.2xlarge × 9, Karpenter-managed)
│   ├── Donor Management Service
│   ├── Transport & Logistics Service
│   ├── Data Collection Service
│   └── Transplant & Outcomes Service
│
├── Tier 3 Node Group (m6i.xlarge × 6, Karpenter-managed)
│   ├── Compliance & Member Service
│   ├── Public Data Service
│   ├── Notification Service
│   └── Patient Portal Service
│
└── Spot Node Group (m6i.2xlarge, Karpenter-managed)
    └── Non-critical batch jobs, dev/test workloads
```

**Key EKS configurations:**

| Feature | Configuration |
|---|---|
| Kubernetes Version | Latest stable (auto-upgraded via EKS managed add-ons) |
| Container Runtime | containerd |
| Service Mesh | AWS App Mesh (Envoy sidecar injection) |
| Autoscaling | Karpenter (node), HPA (pod) |
| Secrets | AWS Secrets Manager via Secrets Store CSI Driver |
| Logging | Fluent Bit → CloudWatch Logs |
| Image Registry | Amazon ECR (private, image scanning enabled) |
| Network Policy | Calico (namespace-level isolation) |
| Pod Security | Pod Security Standards (restricted profile for Tier 1) |
| GitOps | Flux CD or ArgoCD for declarative deployment |

### 7.2 CI/CD Pipeline

```
Developer → GitHub (GovCloud) → AWS CodePipeline
    │
    ├─ Source Stage ─► CodeBuild (lint, unit tests, SAST scan)
    │
    ├─ Build Stage ─► CodeBuild (container build, ECR push, image scan)
    │
    ├─ Test Stage ─► CodeBuild (integration tests against staging)
    │
    ├─ Security Stage ─► CodeBuild (DAST scan, dependency audit)
    │
    ├─ Staging Deploy ─► Flux CD → EKS Staging Cluster
    │   └─ Automated smoke tests + canary analysis
    │
    └─ Prod Deploy (manual approval gate) ─► Flux CD → EKS Prod
        └─ Canary deployment (10% → 50% → 100%) with automated rollback
```

**For allocation algorithm changes**, an additional gate is enforced: the policy simulation environment must show passing validation against the historical match test suite, and the OPTN Data Advisory Committee must approve before the production deploy gate opens.

---

## 8. Security Architecture

### 8.1 Zero Trust Model

```
                    ┌─────────────────────────┐
                    │    Zero Trust Layers      │
                    │                           │
   User ──► MFA ──► IAM Identity Center ──► │  │
                    │  ┌───────────────────┐ │  │
                    │  │ Verified Perms.   │ │  │
                    │  │ (Cedar policies)  │ │  │
                    │  └────────┬──────────┘ │  │
                    │           │             │  │
                    │  ┌────────▼──────────┐ │  │
                    │  │ API Gateway        │ │  │
                    │  │ (JWT validation,   │ │  │
                    │  │  rate limiting)    │ │  │
                    │  └────────┬──────────┘ │  │
                    │           │             │  │
                    │  ┌────────▼──────────┐ │  │
                    │  │ App Mesh (mTLS)    │ │  │
                    │  │ (service-to-       │ │  │
                    │  │  service auth)     │ │  │
                    │  └────────┬──────────┘ │  │
                    │           │             │  │
                    │  ┌────────▼──────────┐ │  │
                    │  │ IAM Roles (IRSA)   │ │  │
                    │  │ (AWS resource      │ │  │
                    │  │  access)           │ │  │
                    │  └───────────────────┘ │  │
                    └─────────────────────────┘
```

### 8.2 Data Protection

| Data State | Mechanism | AWS Service |
|---|---|---|
| In transit (external) | TLS 1.3 | ALB, API Gateway, CloudFront |
| In transit (internal) | mTLS (Envoy/App Mesh) | App Mesh, Private CA |
| At rest (databases) | AES-256 with CMK | KMS (customer-managed keys) |
| At rest (S3) | SSE-KMS with bucket policies | KMS + S3 Bucket Policies |
| At rest (audit logs) | WORM (write-once-read-many) | S3 Object Lock + QLDB |
| PHI in AI pipelines | Redacted before Bedrock invocation | Bedrock Guardrails + Lambda pre-processing |
| Key management | FIPS 140-2 Level 3 HSM | CloudHSM (for signing), KMS (for encryption) |
| Backup encryption | Encrypted with separate CMK | AWS Backup with CMK |

### 8.3 Threat Detection and Response

| Capability | AWS Service | Configuration |
|---|---|---|
| Threat Detection | GuardDuty | Enabled across all accounts, EKS audit log monitoring |
| Vulnerability Management | Inspector | Continuous scanning of ECR images and EC2 instances |
| Security Posture | Security Hub | NIST 800-53 and AWS FSBP standards enabled |
| WAF | AWS WAF | OWASP Top 10 rules on ALB and API Gateway |
| DDoS Protection | Shield Advanced | Enabled on CloudFront, ALB, API Gateway |
| SIEM | Security Lake + OpenSearch | Centralized security event lake with automated correlation |
| Incident Response | Systems Manager Incident Manager | Automated runbooks for common security incidents |

---

## 9. Disaster Recovery and Business Continuity

### 9.1 Multi-Region Strategy

**Primary region:** us-gov-west-1
**DR region:** us-gov-east-1
**Strategy:** Warm standby with rapid promotion for Tier 1 services; pilot light for Tier 2/3.

| Component | DR Strategy | RPO | RTO |
|---|---|---|---|
| Aurora PostgreSQL | Global Database (async replication) | < 1 second | < 5 minutes (promote replica) |
| DynamoDB | Global Tables (active-active) | 0 seconds | 0 seconds (already active) |
| ElastiCache Redis | Global Datastore | < 1 second | < 5 minutes |
| MSK (Kafka) | MirrorMaker 2 cross-region replication | < 10 seconds | < 15 minutes |
| S3 | Cross-Region Replication | < 15 minutes | 0 (already replicated) |
| EKS | Standby cluster in DR region, pre-warmed Tier 1 nodes | N/A | < 15 minutes (Flux CD deploy) |
| API Gateway | Route 53 failover routing | N/A | < 1 minute (DNS failover) |

### 9.2 Failure Modes and Recovery

| Failure Scenario | Detection | Response |
|---|---|---|
| Single AZ outage | ALB health checks | Automatic: traffic redistributed to healthy AZs |
| Full region outage | Route 53 health checks | Automated: DNS failover to DR region, DR cluster promoted |
| Database corruption | Anomaly detection on write patterns | Manual: point-in-time recovery from Aurora backups |
| Kafka topic corruption | Consumer lag monitoring | Manual: replay from S3 event archive |
| Ransomware/compromise | GuardDuty + Security Hub alerts | Automated: isolate affected accounts, restore from immutable backups |
| Vendor transition | Planned | All IaC, configs, docs in government-owned repos; new vendor deploys from source |

---

## 10. Infrastructure as Code

All infrastructure is defined in Terraform, stored in a government-owned Git repository.

```
terraform/
├── modules/
│   ├── eks-cluster/           # EKS cluster + node groups
│   ├── aurora-global/         # Aurora PostgreSQL Global Database
│   ├── msk-cluster/           # MSK (Kafka) cluster
│   ├── api-gateway/           # API Gateway configurations
│   ├── networking/            # VPCs, subnets, Transit Gateway
│   ├── security/              # KMS keys, IAM policies, GuardDuty
│   ├── observability/         # CloudWatch, X-Ray, OpenSearch
│   ├── s3-data-lake/          # S3 buckets, lifecycle policies
│   └── bedrock/               # Bedrock model access, guardrails, KB
├── environments/
│   ├── dev/                   # Development account
│   ├── staging/               # Staging account
│   ├── production/            # Production account
│   │   ├── us-gov-west-1/     # Primary region
│   │   └── us-gov-east-1/     # DR region
│   ├── sandbox/               # Policy simulation sandbox
│   └── analytics/             # Data analytics account
├── policies/
│   ├── scps/                  # Service Control Policies
│   ├── cedar/                 # Verified Permissions Cedar policies
│   └── opa/                   # OPA policies for Kubernetes
└── pipelines/
    └── codepipeline.tf        # CI/CD pipeline definitions
```

---

## 11. Requirements Traceability Matrix

| Requirement ID | Service(s) | AWS Service(s) |
|---|---|---|
| FR-MATCH-001–006 | Matching & Allocation | EKS, Aurora, DynamoDB, ElastiCache, QLDB, MSK |
| FR-OFFER-001–006 | Organ Offer | EKS, Step Functions, Aurora, SNS, Pinpoint, S3 |
| FR-AOOS-001–003 | Matching & Allocation | EKS, Aurora, MSK, CloudWatch |
| FR-WAIT-001–008 | Waitlist | EKS, Aurora, ElastiCache, MSK, EventBridge, API GW |
| FR-DONOR-001–007 | Donor Management | EKS, Aurora, S3, Lambda, EventBridge, MSK |
| FR-TX-001–006 | Transplant & Outcomes | EKS, Aurora, Batch, EventBridge, SQS, Lambda |
| FR-TRANS-001–006 | Transport & Logistics | EKS, DynamoDB, IoT Core, Kinesis, Lambda, MSK |
| FR-DATA-001–007 | Data Collection | EKS, Aurora, S3, DynamoDB, Lambda, Batch, MSK |
| FR-COMP-001–004 | Compliance & Member | EKS, Aurora, S3, Batch, Step Functions, MSK |
| FR-PUB-001–005 | Public Data | CloudFront, S3, API GW, DynamoDB, Lambda |
| AN-OPS-001–003 | All services → Analytics | MSK, Glue, S3, Athena, QuickSight, Redshift |
| AN-PRED-001–004 | AI/ML Layer | Bedrock, SageMaker, S3, Lambda |
| AN-RES-001–004 | Research Data Portal | Athena, S3, Glue, Redshift, Bedrock |
| NFR-AVAIL-001–005 | All services | EKS (multi-AZ), Aurora Global DB, Route 53, MSK |
| NFR-SEC-001–009 | Security Layer | KMS, CloudHSM, GuardDuty, Security Hub, WAF, QLDB |
| NFR-INTEROP-001–007 | FHIR Gateway + all services | API GW, Lambda (HAPI FHIR), EKS |
| INT-SRTR-001–003 | Analytics + Event Backbone | MSK, EventBridge Pipes, Redshift, S3 |
| INT-CMS-001–003 | Transplant & Outcomes | SQS, Lambda, API GW |
| INT-MEMBER-001–004 | FHIR Gateway + all services | API GW, Lambda, EKS |
| MIG-DATA-001–003 | Migration tooling | DMS, Glue, S3, Step Functions |
| MIG-VENDOR-001–003 | IaC + docs | Terraform, CodeCommit/GitHub, S3 |

---

## 12. Cost Estimation Framework

Approximate monthly cost for the production environment (order of magnitude):

| Category | Key Services | Est. Monthly Cost |
|---|---|---|
| Compute (EKS) | ~27 EC2 instances across node groups | $25,000–$35,000 |
| Databases (Aurora) | 7 Aurora clusters, Global DB, read replicas | $30,000–$45,000 |
| Caching (ElastiCache) | Redis cluster mode, global datastore | $5,000–$8,000 |
| Streaming (MSK) | 6-broker Kafka cluster | $8,000–$12,000 |
| Storage (S3) | Data lake, event archive, documents, media | $5,000–$15,000 |
| Analytics (Redshift, Glue, Athena) | Serverless Redshift, Glue jobs | $10,000–$20,000 |
| AI/ML (Bedrock + SageMaker) | LLM inference, SageMaker endpoints | $8,000–$15,000 |
| Networking | Transit GW, Direct Connect, NAT, CloudFront | $8,000–$12,000 |
| Security | GuardDuty, Security Hub, WAF, Shield, Inspector | $5,000–$8,000 |
| Observability | CloudWatch, X-Ray, OpenSearch | $5,000–$10,000 |
| Other (Secrets, KMS, Cognito, etc.) | Supporting services | $3,000–$5,000 |
| **Total estimated** | | **$112,000–$185,000/mo** |

Note: This excludes DR region standby costs (~30–40% of primary), development/staging environments (~20% of primary each), AWS GovCloud premium (~5–10% over commercial), and Reserved Instance / Savings Plan discounts (30–50% reduction on compute/databases with 1-year or 3-year commitments). Actual costs will depend on transaction volumes, data growth, and Bedrock invocation rates.

---

## Appendix A: AWS Service Summary

| AWS Service | Role in UNet Next |
|---|---|
| Amazon EKS | Primary compute platform for all 12 domain services |
| Amazon Aurora PostgreSQL | Operational databases (per-service) with Global Database for DR |
| Amazon DynamoDB | High-throughput key-value stores (policy config, transport tracking, caches) |
| Amazon ElastiCache Redis | Candidate score cache, session cache, active waitlist cache |
| Amazon MSK (Kafka) | Central event backbone, CDC streams, SRTR/TDS feeds |
| Amazon S3 | Data lake, event archive, document/image storage, static web hosting |
| AWS Lambda | Event-driven compute (FHIR translation, validation, scoring, notifications) |
| AWS Step Functions | Workflow orchestration (offer state machines, compliance workflows, data locks) |
| Amazon API Gateway | External REST/FHIR APIs, WebSocket, public API, developer portal |
| Amazon Bedrock | Foundation model inference (LLM for NLP, data extraction, policy copilot) |
| Amazon SageMaker | Custom ML model training, hosting, monitoring (acceptance prediction, outcomes) |
| Amazon Redshift Serverless | Analytical data warehouse, SRTR data sharing |
| AWS Glue | ETL, data catalog, schema registry, data quality |
| Amazon Athena | Ad hoc SQL queries on S3 data lake |
| Amazon QuickSight | Business intelligence dashboards (HRSA, OPO, center-level) |
| Amazon OpenSearch Serverless | Log search, full-text search across OPTN data |
| Amazon QLDB | Immutable audit ledger for match runs and allocation decisions |
| AWS IoT Core | GPS device telemetry ingestion for organ transport tracking |
| Amazon Kinesis | Real-time data streaming (transport telemetry, Firehose to S3) |
| Amazon Cognito | Patient portal authentication, MFA |
| AWS IAM Identity Center | Workforce SSO for OPTN members, OPOs, HRSA staff |
| Amazon Verified Permissions | Fine-grained RBAC/ABAC using Cedar policy language |
| AWS KMS + CloudHSM | Encryption key management, FIPS 140-2 Level 3 HSM |
| AWS App Mesh | Service mesh (mTLS, traffic management, observability) |
| Amazon CloudFront | CDN, static site hosting, DDoS protection |
| AWS WAF + Shield Advanced | Web application firewall, DDoS mitigation |
| Amazon GuardDuty | Threat detection across all accounts |
| AWS Security Hub | Security posture management (NIST 800-53 compliance) |
| Amazon Security Lake | Centralized security event lake |
| AWS CloudTrail | API audit logging across all accounts |
| Amazon Route 53 | DNS, health-check failover, multi-region routing |
| AWS Transit Gateway | Hub-and-spoke network connectivity |
| AWS Direct Connect | Dedicated connectivity to member institutions |
| Amazon SNS + SES + Pinpoint | Multi-channel notification delivery |
| Amazon Connect | Phone call escalation for unacknowledged organ offers |
| AWS CodePipeline + CodeBuild | CI/CD pipeline |
| AWS Batch | Batch processing (outcomes calculation, report generation, data migration) |
| Amazon EventBridge | Event routing, scheduling, cross-account event delivery |
| AWS Backup | Centralized backup management across all data stores |
| Terraform (IaC) | All infrastructure defined as code (not an AWS service, but foundational) |
