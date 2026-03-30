# Modernized UNet System Requirements
## A Next-Generation Organ Procurement and Transplantation Network Data Platform

**Version:** 1.0 — Draft for Discussion
**Date:** March 30, 2026
**Scope:** Requirements for a modernized replacement of the UNOS UNet℠ platform that operates the core data infrastructure of the U.S. Organ Procurement and Transplantation Network (OPTN)

---

## 1. Executive Summary

The current UNet system, launched in 1999 and incrementally extended over 25+ years, is the operational backbone of the U.S. organ transplant system. It manages the national waiting list, organ donor registration, algorithmic matching, organ offer and acceptance workflows, data collection, organ labeling and transport tracking, and regulatory reporting for over 250 transplant centers, 57 organ procurement organizations (OPOs), and dozens of histocompatibility laboratories.

Despite its critical role, the system suffers from well-documented limitations: monolithic architecture, closed proprietary internals, heavy reliance on manual data entry, limited API exposure, fragmented subsystems (Waitlist, DonorNet, TIEDI, TransNet), an acknowledged multi-year technology backlog, and an architecture that impedes third-party innovation. Congressional investigations, the 2022 NASEM report, and the 2023 Securing the U.S. OPTN Act have mandated fundamental modernization.

This document defines the functional and non-functional requirements for a modernized platform — referred to herein as **"UNet Next"** — that replaces or substantially re-engineers all four legacy UNet subsystems while supporting HRSA's transition to a multi-vendor, government-owned, cloud-native operating model.

---

## 2. Guiding Principles

The following principles should govern all architectural and design decisions:

### 2.1 Patient Safety Above All
Every design trade-off must be resolved in favor of patient safety. The system handles life-or-death decisions in real time; downtime, data errors, or latency in organ matching can directly result in organ loss or patient death.

### 2.2 Government-Owned, Vendor-Agnostic
The platform and all data must be owned by the federal government (HRSA/HHS). No vendor lock-in. All components must use open standards, open APIs, and portable data formats. Any qualified contractor must be able to operate, maintain, or extend any module.

### 2.3 API-First Architecture
Every function available through a user interface must also be available through a documented, versioned, standards-based API. The platform is a set of services, not a monolithic application.

### 2.4 Interoperability by Default
The system must natively speak the languages of healthcare IT: HL7 FHIR, CDA, USCDI, and standard terminology systems (SNOMED CT, LOINC, ICD-10, CPT). Integration with hospital EHRs, OPO systems, and lab information systems must be a first-class design goal, not an afterthought.

### 2.5 Transparency and Auditability
Every action, decision, override, and data change must be logged with immutable audit trails. Allocation decisions must be fully reconstructable. Public-facing dashboards and data must be updated in near-real-time, not monthly.

### 2.6 Resilience and Continuity
The system must be designed to operate continuously — including through partial infrastructure failures, cyberattacks, natural disasters, and vendor transitions — with zero tolerance for unplanned downtime in organ matching and allocation.

---

## 3. Functional Requirements

### 3.1 Organ Matching and Allocation Engine

#### 3.1.1 Core Matching
- **FR-MATCH-001:** The system shall execute organ-to-candidate matching algorithms for all organ types (kidney, liver, heart, lung, pancreas, intestine, VCA, multi-organ) in accordance with current OPTN allocation policies.
- **FR-MATCH-002:** Match runs shall complete within 60 seconds for standard cases and within 120 seconds for complex multi-organ or highly sensitized cases, measured from initiation to rank-ordered list generation.
- **FR-MATCH-003:** The system shall support configurable allocation algorithms that can be modified by authorized policy administrators without requiring code deployments. Policy changes shall be testable in a sandbox environment before production activation.
- **FR-MATCH-004:** All allocation scoring models (MELD/PELD, EPTS, KDPI, Lung CAS, heart status, CPRA) shall be implemented as independently versioned, pluggable modules with full formula documentation and audit trails.
- **FR-MATCH-005:** The system shall support simulation of proposed policy changes against historical data to project impacts on utilization and outcomes before implementation.
- **FR-MATCH-006:** The system shall generate a complete, immutable audit record for every match run, including the full rank-ordered candidate list, each candidate's calculated scores, and the policy version used.

#### 3.1.2 Organ Offers and Acceptance
- **FR-OFFER-001:** The system shall present organ offers to transplant centers electronically (via web, mobile, and API) with all associated donor clinical data, images, and lab results in a unified view.
- **FR-OFFER-002:** Transplant centers shall be able to accept or decline organ offers electronically with structured decline reason codes. The system shall enforce capture of a decline reason for every refusal.
- **FR-OFFER-003:** The system shall support configurable offer acceptance windows with escalation rules (e.g., auto-advance to next candidate after timeout, notify OPO coordinator).
- **FR-OFFER-004:** The system shall support simultaneous/parallel organ offers to multiple centers for hard-to-place organs, with configurable parallel offer rules per organ type.
- **FR-OFFER-005:** The system shall track and report the full chain of organ offers — every center contacted, response time, acceptance/decline decision, and decline reason — for every organ, with data available for real-time monitoring and retrospective analysis.
- **FR-OFFER-006:** The system shall provide predictive analytics on organ acceptance probability based on historical patterns, donor characteristics, and center-specific acceptance behavior.

#### 3.1.3 Allocation Out of Sequence (AOOS)
- **FR-AOOS-001:** The system shall detect and flag every instance where an organ is allocated to a recipient who was not the highest-ranked candidate on the match run, in real time.
- **FR-AOOS-002:** The system shall require structured justification documentation for every AOOS event and route flagged events to the appropriate oversight body.
- **FR-AOOS-003:** The system shall provide a real-time AOOS dashboard showing rates by OPO, transplant center, organ type, and time period.

### 3.2 Candidate Waiting List Management

- **FR-WAIT-001:** The system shall maintain a single, authoritative national waiting list for each organ type, with real-time synchronization across all access points.
- **FR-WAIT-002:** Transplant centers shall be able to register, update, inactivate, reactivate, and remove candidates via UI and API.
- **FR-WAIT-003:** The system shall calculate and display allocation priority scores in real time as clinical data are updated.
- **FR-WAIT-004:** The system shall support multiple-listing (candidates listed at multiple centers) with accurate time accrual, priority transfer, and deduplication logic.
- **FR-WAIT-005:** The system shall record and display the complete history of every candidate's status changes, clinical updates, and priority score recalculations.
- **FR-WAIT-006:** The system shall support automated clinical data feeds from hospital EHR systems via FHIR APIs to reduce manual data entry for lab values, vitals, and clinical status updates.
- **FR-WAIT-007:** The system shall provide patient-facing read access to their own waiting list status, position information (where permitted by policy), and educational materials, through a secure patient portal.
- **FR-WAIT-008:** The system shall generate automated alerts and notifications to transplant centers when candidate clinical data are expiring, missing, or inconsistent.

### 3.3 Donor Management

- **FR-DONOR-001:** OPOs shall be able to register, update, and manage deceased donor records via UI and API, including demographics, cause of death, medical/social history, serology, and organ-specific clinical data.
- **FR-DONOR-002:** The system shall support structured upload and association of donor imaging (CT, echocardiogram, bronchoscopy, biopsy results) and lab values, accessible to all transplant centers evaluating the donor.
- **FR-DONOR-003:** The system shall support living donor registration, evaluation tracking, paired kidney exchange programs, and non-directed (altruistic) donor workflows.
- **FR-DONOR-004:** The system shall calculate donor quality indices (KDPI, DRI, etc.) automatically as donor data are entered or updated, and display them to OPOs and transplant centers in real time.
- **FR-DONOR-005:** The system shall support the Vascular Composite Allograft (VCA) donor workflow with organ-specific data elements.
- **FR-DONOR-006:** The system shall track the full referral-to-procurement pipeline: hospital referral, OPO evaluation, family authorization, donor management, organ recovery, and disposition of each organ — capturing data at every stage.
- **FR-DONOR-007:** The system shall integrate with external death registries (NTIS Death Master File, state vital records) for outcome verification and donor identification.

### 3.4 Transplant and Outcome Tracking

- **FR-TX-001:** Transplant centers shall report transplant events, including recipient details, donor-recipient match information, ischemia times, surgical details, and initial outcomes, via UI and API.
- **FR-TX-002:** The system shall generate and manage follow-up data collection schedules for transplant recipients (e.g., 6-month, 1-year, 2-year follow-up forms) with automated reminders and escalation for delinquent submissions.
- **FR-TX-003:** The system shall support living donor follow-up data collection at mandated intervals (6 months, 1 year, 2 years post-donation) with the same automation.
- **FR-TX-004:** The system shall accept automated outcome data feeds from transplant center EHRs via FHIR to reduce manual entry burden for follow-up reporting.
- **FR-TX-005:** The system shall calculate and publish risk-adjusted program-specific outcomes (graft survival, patient survival, waiting list mortality, transplant rates) using validated statistical models.
- **FR-TX-006:** The system shall track graft failure events, patient death events, re-transplantation events, and cause-of-failure/death data.

### 3.5 Organ Transport and Logistics

- **FR-TRANS-001:** The system shall provide end-to-end organ tracking from recovery through transplant, including real-time GPS tracking where available.
- **FR-TRANS-002:** The system shall generate compliant packaging and labeling for all deceased donor organs and specimens per OPTN policy, replacing or modernizing TransNet functionality.
- **FR-TRANS-003:** Transplant centers shall electronically verify organ check-in, confirming organ identity match to intended recipient at the point of care.
- **FR-TRANS-004:** The system shall calculate and display estimated cold ischemia times based on recovery time, transport method, and distance.
- **FR-TRANS-005:** The system shall support integration with third-party organ transport services (commercial couriers, charter flights, drones) for automated logistics coordination and status tracking.
- **FR-TRANS-006:** The system shall log and report all instances of organs lost, delayed, damaged, or misdirected during transport.

### 3.6 Data Collection (TIEDI Modernization)

- **FR-DATA-001:** The system shall support electronic data collection for all OPTN-mandated forms (Deceased Donor Registration, Living Donor Registration, Transplant Recipient Registration, Follow-Up forms, Histocompatibility forms, Potential Transplant Recipient forms) through a unified data entry platform.
- **FR-DATA-002:** All forms shall be available as structured, API-accessible data schemas — not just web-based form UIs — enabling transplant centers to submit data directly from their EHRs.
- **FR-DATA-003:** The system shall enforce real-time data validation with clear, actionable error messages, preventing submission of incomplete or inconsistent records.
- **FR-DATA-004:** The system shall support configurable data submission deadlines with automated reminders, escalation workflows, and data lock provisions per OPTN Policy 18.
- **FR-DATA-005:** The system shall support bulk data import/export in standard formats (CSV, JSON, FHIR bundles, XML) with validation.
- **FR-DATA-006:** The system shall support data correction workflows with full audit trails, approval requirements, and version history for every record.
- **FR-DATA-007:** The system shall minimize redundant data entry by pre-populating form fields from previously submitted data and integrated external systems.

### 3.7 Compliance and Member Management

- **FR-COMP-001:** The system shall maintain a registry of all OPTN members (transplant centers, OPOs, histocompatibility labs) with their membership status, key personnel, program types, and compliance history.
- **FR-COMP-002:** The system shall track and report OPTN policy compliance metrics per member, including data submission timeliness, AOOS rates, offer response times, and outcome reporting completeness.
- **FR-COMP-003:** The system shall support the Membership and Professional Standards Committee (MPSC) workflow: flagging performance concerns, generating review packets, tracking corrective actions, and documenting peer review outcomes.
- **FR-COMP-004:** The system shall generate automated compliance reports for HRSA, CMS, and the OPTN Board on configurable schedules.

### 3.8 Public Data and Transparency

- **FR-PUB-001:** The system shall publish a public-facing data portal with current national, regional, state, and center-level transplant statistics updated at least weekly (target: daily).
- **FR-PUB-002:** The system shall provide interactive data visualizations and dashboards accessible to patients, researchers, policymakers, and the general public without requiring authentication.
- **FR-PUB-003:** The system shall support a public API for programmatic access to de-identified aggregate transplant data.
- **FR-PUB-004:** The system shall publish OPTN computer system performance metrics (uptime, match run speed, system availability) in real time.
- **FR-PUB-005:** The system shall support the patient-facing "find a transplant center" tool with current performance data, wait times, volume, and outcome comparisons.

---

## 4. Non-Functional Requirements

### 4.1 Availability and Reliability

- **NFR-AVAIL-001:** The organ matching and allocation engine shall maintain 99.99% uptime (no more than 52 minutes of unplanned downtime per year). This is a life-critical, real-time system.
- **NFR-AVAIL-002:** The system shall be deployed in an active-active multi-region cloud architecture with automatic failover. No single point of failure shall exist in the matching pipeline.
- **NFR-AVAIL-003:** The system shall maintain full operational capability during planned maintenance windows through rolling deployments with zero-downtime releases.
- **NFR-AVAIL-004:** The system shall include comprehensive disaster recovery capabilities with a Recovery Point Objective (RPO) of 0 seconds (zero data loss) and a Recovery Time Objective (RTO) of less than 15 minutes.
- **NFR-AVAIL-005:** The system shall implement circuit breakers, bulkheads, and graceful degradation patterns so that failure in non-critical subsystems (e.g., analytics dashboards) does not impact critical operations (e.g., organ matching).

### 4.2 Performance

- **NFR-PERF-001:** Match run execution: < 60 seconds for standard, < 120 seconds for complex.
- **NFR-PERF-002:** Organ offer notification delivery: < 5 seconds from match run completion to transplant center notification.
- **NFR-PERF-003:** UI page load times: < 2 seconds for standard data views; < 5 seconds for complex reports.
- **NFR-PERF-004:** API response times: < 500ms for read operations (p95); < 2 seconds for write operations (p95).
- **NFR-PERF-005:** The system shall support a minimum of 10,000 concurrent authenticated users without performance degradation.
- **NFR-PERF-006:** The system shall support at least 500 simultaneous match runs without performance degradation, anticipating growth toward 60,000+ transplants per year.

### 4.3 Security

- **NFR-SEC-001:** The system shall comply with FISMA High baseline controls, NIST SP 800-53 Rev 5, and HHS cybersecurity requirements.
- **NFR-SEC-002:** All data in transit shall be encrypted using TLS 1.3. All data at rest shall be encrypted using AES-256 or equivalent.
- **NFR-SEC-003:** The system shall implement zero-trust network architecture with microsegmentation, mutual TLS between services, and continuous identity verification.
- **NFR-SEC-004:** The system shall support multi-factor authentication (MFA) for all users with role-based access control (RBAC) and attribute-based access control (ABAC) for fine-grained permissions.
- **NFR-SEC-005:** The system shall maintain comprehensive security event logging with real-time SIEM integration and automated threat detection.
- **NFR-SEC-006:** The system shall undergo annual penetration testing, quarterly vulnerability assessments, and continuous automated scanning.
- **NFR-SEC-007:** All PHI and PII shall be handled in compliance with HIPAA, the Privacy Act, and applicable state health privacy laws.
- **NFR-SEC-008:** The system shall support hardware security modules (HSMs) or equivalent for cryptographic key management.
- **NFR-SEC-009:** The system shall implement immutable audit logs stored in a write-once, tamper-evident data store with a minimum 10-year retention.

### 4.4 Scalability

- **NFR-SCALE-001:** The system architecture shall support horizontal scaling of all stateless services.
- **NFR-SCALE-002:** The system shall be designed to handle a 100% increase in transplant volume (from ~45,000 to 90,000+ annually) without architectural changes.
- **NFR-SCALE-003:** Data storage shall scale elastically to accommodate the full historical OPTN dataset (1987–present) plus projected growth.
- **NFR-SCALE-004:** The system shall support addition of new organ types, allocation algorithms, and data elements without requiring core platform changes.

### 4.5 Interoperability

- **NFR-INTEROP-001:** The system shall expose all clinical data exchange capabilities through HL7 FHIR R4 (or later) APIs.
- **NFR-INTEROP-002:** The system shall support USCDI (U.S. Core Data for Interoperability) data classes and elements.
- **NFR-INTEROP-003:** The system shall use standard terminology: SNOMED CT for clinical terms, LOINC for lab observations, ICD-10-CM for diagnoses, CPT for procedures, RxNorm for medications.
- **NFR-INTEROP-004:** The system shall support SMART on FHIR for application-level authorization and EHR integration.
- **NFR-INTEROP-005:** The system shall support ADT (Admit/Discharge/Transfer) event feeds and CDS Hooks for real-time clinical decision support integration.
- **NFR-INTEROP-006:** The system shall provide a developer portal with comprehensive API documentation, sandbox environments, SDKs, and sample code for member institution integration.
- **NFR-INTEROP-007:** The system shall support bi-directional data exchange with CMS systems, the SRTR, state health departments, and other authorized federal/state agencies.

### 4.6 Usability

- **NFR-UX-001:** The system shall provide responsive web interfaces that function on desktop, tablet, and mobile devices for all primary workflows.
- **NFR-UX-002:** The system shall provide native mobile applications (iOS and Android) for time-critical workflows: organ offer review/acceptance, donor data entry in the field, and transport tracking.
- **NFR-UX-003:** All user interfaces shall meet WCAG 2.1 AA accessibility standards.
- **NFR-UX-004:** The system shall support localization for at minimum English and Spanish.
- **NFR-UX-005:** The system shall provide context-sensitive help, embedded training modules, and searchable documentation.
- **NFR-UX-006:** The system shall provide configurable notification preferences (email, SMS, push, in-app) per user role, with guaranteed delivery and escalation for time-critical organ offer notifications.

---

## 5. Architecture Requirements

### 5.1 Cloud-Native, Government-Owned

- **AR-CLOUD-001:** The system shall be deployed on FedRAMP High authorized cloud infrastructure (e.g., AWS GovCloud, Azure Government, GCP Assured Workloads).
- **AR-CLOUD-002:** All infrastructure shall be defined as code (IaC) using standard tools (Terraform, CloudFormation, or equivalent) with version-controlled configurations.
- **AR-CLOUD-003:** The system shall use container orchestration (Kubernetes or equivalent) for all application workloads.
- **AR-CLOUD-004:** The government shall own all source code, data, configurations, infrastructure definitions, and deployment pipelines. No proprietary vendor code shall be embedded in the core platform.

### 5.2 Microservices Architecture

- **AR-MICRO-001:** The system shall be decomposed into independently deployable, loosely coupled services organized around business capabilities (matching, waitlist, donor management, data collection, transport, compliance, analytics, identity).
- **AR-MICRO-002:** Services shall communicate through well-defined APIs (synchronous REST/gRPC) and event-driven messaging (asynchronous pub/sub) as appropriate to the interaction pattern.
- **AR-MICRO-003:** Each service shall own its data store (database-per-service pattern) with no direct database sharing between services.
- **AR-MICRO-004:** The system shall implement an API gateway for external access with rate limiting, authentication, request routing, and API versioning.

### 5.3 Event-Driven Architecture

- **AR-EVENT-001:** The system shall implement an enterprise event bus (Apache Kafka, AWS EventBridge, or equivalent) for cross-service communication and real-time data streaming.
- **AR-EVENT-002:** Key domain events shall be published to the event bus including but not limited to: candidate registered, candidate status changed, donor registered, match run initiated, match run completed, organ offered, organ accepted, organ declined, transplant reported, follow-up submitted, AOOS detected.
- **AR-EVENT-003:** All events shall be persisted in an immutable event store with a minimum 10-year retention, enabling full event replay and historical reconstruction.
- **AR-EVENT-004:** The SRTR, TDS, and authorized third-party systems shall be able to subscribe to relevant event streams for near-real-time data access, replacing the current monthly batch data transfer model.

### 5.4 Data Architecture

- **AR-DATA-001:** The system shall maintain a single source of truth for each data entity, with clear data ownership by service.
- **AR-DATA-002:** The system shall implement a data lake or data lakehouse architecture for analytics, combining structured (relational), semi-structured (JSON/FHIR), and unstructured (images, documents) data.
- **AR-DATA-003:** The system shall support event sourcing for critical entities (candidates, donors, match runs) to enable full historical reconstruction.
- **AR-DATA-004:** The system shall implement Change Data Capture (CDC) to stream database changes to downstream consumers (SRTR, TDS, analytics) in near-real-time.
- **AR-DATA-005:** The system shall enforce a master data management (MDM) strategy for shared reference data (organ types, status codes, policy parameters, member registry).
- **AR-DATA-006:** All data schemas shall be versioned and governed through a schema registry with backward compatibility requirements.
- **AR-DATA-007:** The system shall support data lineage tracking — tracing any derived metric or report back to its source records and transformations.

---

## 6. Analytics and Intelligence Requirements

### 6.1 Operational Analytics

- **AN-OPS-001:** The system shall provide real-time operational dashboards for HRSA, OPOs, and transplant centers showing key metrics: active donors, organs in allocation, pending offers, transplants in progress, transport status.
- **AN-OPS-002:** The system shall provide trend analytics for waitlist growth, transplant volumes, organ utilization rates, offer acceptance rates, and cold ischemia times at national, regional, OPO, and center levels.
- **AN-OPS-003:** The system shall track and report organ non-utilization metrics: organs recovered but not transplanted, by organ type, donor characteristics, and reason for non-use.

### 6.2 Predictive Analytics

- **AN-PRED-001:** The system shall provide predictive models for organ acceptance probability to support OPO decision-making during the offer process.
- **AN-PRED-002:** The system shall provide estimated wait time projections for candidates, accounting for blood type, sensitization, geography, and organ type.
- **AN-PRED-003:** The system shall support predictive modeling for post-transplant outcomes to inform clinical decision-making (with appropriate clinical disclaimers).
- **AN-PRED-004:** All predictive models shall be transparent, documented, validated against historical data, and monitored for bias across demographic groups.

### 6.3 Research Data Access

- **AN-RES-001:** The system shall provide a self-service research data request portal with automated de-identification and dataset generation.
- **AN-RES-002:** The system shall support creation of standard analysis datasets (SAFs — Standard Analysis Files) that are refreshed on a defined schedule and made available to approved researchers.
- **AN-RES-003:** The system shall support federated analytics — enabling authorized researchers to run approved analyses against the data without extracting patient-level datasets.
- **AN-RES-004:** The system shall maintain a data dictionary and metadata catalog that is publicly accessible and kept current.

---

## 7. Integration Requirements

### 7.1 SRTR Integration

- **INT-SRTR-001:** The system shall provide the SRTR with near-real-time data feeds (target: < 1 hour latency) replacing the current monthly batch transfer from UNOS.
- **INT-SRTR-002:** Data feeds to the SRTR shall be delivered through the Transplant Data Services (TDS) platform as the authoritative data layer.
- **INT-SRTR-003:** The system shall support SRTR access to de-identified data for independent analysis without requiring SRTR to maintain a separate copy of the full operational database.

### 7.2 CMS Integration

- **INT-CMS-001:** The system shall exchange data with CMS systems for OPO Conditions for Coverage compliance monitoring.
- **INT-CMS-002:** The system shall support data feeds to the IOTA Model and future CMS value-based payment programs.
- **INT-CMS-003:** The system shall integrate with CMS data sources (Medicare claims, death records) for outcome verification and supplemental analysis.

### 7.3 Member Institution Integration

- **INT-MEMBER-001:** The system shall provide production-ready EHR integration packages for major EHR platforms (Epic, Cerner/Oracle Health, MEDITECH, Allscripts) with certified FHIR connectors.
- **INT-MEMBER-002:** The system shall support OPO vendor integrations (iTransplant, DonorConnect, Transplant Connect, and others) through standardized APIs.
- **INT-MEMBER-003:** The system shall provide a self-service integration testing environment where member institutions can develop and validate their integrations.
- **INT-MEMBER-004:** The system shall support electronic lab result feeds from histocompatibility laboratories, including HLA typing, crossmatch results, and PRA/CPRA data.

### 7.4 External Systems

- **INT-EXT-001:** The system shall integrate with the National Donor Designation registries (state-level first-person consent registries).
- **INT-EXT-002:** The system shall support integration with organ transport tracking systems (GPS, logistics platforms).
- **INT-EXT-003:** The system shall integrate with the Organ Donation and Transplant Alliance and other stakeholder platforms as appropriate.
- **INT-EXT-004:** The system shall support data exchange with international transplant registries and organizations where governed by treaty or agreement.

---

## 8. Migration and Transition Requirements

### 8.1 Data Migration

- **MIG-DATA-001:** The system shall migrate the complete historical OPTN dataset (October 1, 1987 – present) with full fidelity. No historical data shall be lost or degraded.
- **MIG-DATA-002:** Data migration shall be validated through automated reconciliation comparing record counts, key field values, and computed metrics between legacy and modernized systems.
- **MIG-DATA-003:** The system shall support a parallel-run period where both legacy UNet and UNet Next operate simultaneously, with data synchronization, to validate correctness before cutover.

### 8.2 Transition Operations

- **MIG-OPS-001:** The transition plan shall ensure zero interruption to organ matching, allocation, and placement operations at all times.
- **MIG-OPS-002:** The system shall support phased migration — individual subsystems (e.g., data collection first, then donor management, then matching) may be migrated independently.
- **MIG-OPS-003:** The system shall provide backward-compatible APIs to support member institutions that cannot immediately upgrade their integrations.
- **MIG-OPS-004:** The system shall include comprehensive training programs, including self-paced e-learning, instructor-led training, certification tracks, and a help desk, for all user roles at transplant centers, OPOs, labs, and HRSA.

### 8.3 Vendor Transition Support

- **MIG-VENDOR-001:** All system documentation (architecture, operations runbooks, data dictionaries, API specifications, deployment procedures) shall be maintained at a level of detail sufficient for a new vendor to assume operations within 90 days.
- **MIG-VENDOR-002:** All CI/CD pipelines, infrastructure-as-code, monitoring configurations, and operational procedures shall be version-controlled and transferable.
- **MIG-VENDOR-003:** The system shall not depend on any proprietary vendor tools, frameworks, or services that cannot be substituted.

---

## 9. Compliance and Regulatory Requirements

- **REG-001:** The system shall comply with the National Organ Transplant Act (NOTA) as amended, the OPTN Final Rule (42 CFR Part 121), and the Securing the U.S. OPTN Act (2023).
- **REG-002:** The system shall comply with HIPAA Privacy Rule, Security Rule, and Breach Notification Rule.
- **REG-003:** The system shall comply with the Federal Information Security Modernization Act (FISMA) at the High impact level.
- **REG-004:** The system shall comply with Section 508 of the Rehabilitation Act for accessibility.
- **REG-005:** The system shall support compliance with CMS Conditions for Coverage for OPOs and Conditions of Participation for transplant hospitals.
- **REG-006:** The system shall comply with the 21st Century Cures Act information blocking provisions.
- **REG-007:** The system shall support FedRAMP authorization at the High baseline.
- **REG-008:** The system shall comply with OMB Circular A-130 for federal information management.
- **REG-009:** The system shall implement records management in compliance with NARA (National Archives and Records Administration) requirements.

---

## 10. Testing and Quality Assurance Requirements

- **QA-001:** The system shall maintain automated test coverage of at least 90% for unit tests and 80% for integration tests on all allocation algorithm code.
- **QA-002:** All allocation algorithm changes shall undergo formal validation testing using historical match data with results reviewed by clinical subject matter experts before production deployment.
- **QA-003:** The system shall support automated regression testing of all OPTN policy logic with a test suite maintained alongside the policy configuration.
- **QA-004:** The system shall undergo load testing simulating 2x peak historical volume before each major release.
- **QA-005:** The system shall support chaos engineering practices (controlled fault injection) to validate resilience.
- **QA-006:** The system shall implement canary deployments and feature flags for all changes, with automated rollback on anomaly detection.
- **QA-007:** User acceptance testing (UAT) environments shall be available to OPTN member representatives, patient advocates, and HRSA staff prior to major releases.

---

## 11. Operational Requirements

- **OPS-001:** The system shall provide 24/7/365 operational support with defined SLAs: critical incidents (matching system down) acknowledged in 5 minutes, resolved in 30 minutes.
- **OPS-002:** The system shall implement comprehensive observability: distributed tracing, structured logging, metrics collection, and alerting across all services.
- **OPS-003:** The system shall provide real-time system health dashboards accessible to HRSA, OPTN operations staff, and (in summary form) the public.
- **OPS-004:** The system shall implement automated incident response playbooks for common failure scenarios.
- **OPS-005:** The system shall maintain an OPTN-specific 24/7 operations center (or equivalent coverage model) with organ-specific clinical and technical expertise.
- **OPS-006:** The system shall publish a public-facing status page showing current system health and any active incidents.

---

## Appendix A: Traceability to Known Deficiencies

| Known Deficiency | Requirements Addressed |
|---|---|
| Monthly batch data transfer to SRTR | INT-SRTR-001, AR-EVENT-004, AR-DATA-004 |
| Manual data entry burden on OPOs/centers | FR-WAIT-006, FR-TX-004, FR-DATA-002, FR-DATA-007, NFR-INTEROP-001 |
| Closed/proprietary system internals | AR-CLOUD-004, NFR-INTEROP-006, MIG-VENDOR-003 |
| Multi-year IT backlog | FR-MATCH-003 (configurable algorithms), AR-MICRO-001 (independent deployment) |
| Organ non-utilization (1 in 5 kidneys) | FR-OFFER-004, FR-OFFER-006, AN-OPS-003 |
| AOOS detection and reporting gaps | FR-AOOS-001 through FR-AOOS-003 |
| OPO patchwork systems/phone calls | INT-MEMBER-002, FR-TRANS-001, AR-EVENT-002 |
| No real-time public data | FR-PUB-001 through FR-PUB-004 |
| Vendor lock-in risk | Principle 2.2, AR-CLOUD-004, MIG-VENDOR-001 through MIG-VENDOR-003 |
| Cybersecurity vulnerabilities (OIG report) | NFR-SEC-001 through NFR-SEC-009 |
| Lack of organ transport tracking | FR-TRANS-001, FR-TRANS-005, FR-TRANS-006 |
| Limited third-party innovation | NFR-INTEROP-006, INT-MEMBER-003 |

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| AOOS | Allocation Out of Sequence — when an organ is given to someone other than the highest-ranked candidate |
| CAS | Composite Allocation Score (used for lung allocation) |
| CPRA | Calculated Panel Reactive Antibody (measures sensitization for kidney candidates) |
| DRI | Donor Risk Index |
| EPTS | Estimated Post-Transplant Survival |
| FHIR | Fast Healthcare Interoperability Resources (HL7 standard) |
| HLA | Human Leukocyte Antigen (immune system proteins relevant to transplant matching) |
| HRSA | Health Resources and Services Administration |
| KDPI | Kidney Donor Profile Index |
| MELD/PELD | Model/Pediatric End-Stage Liver Disease (liver allocation scores) |
| OPO | Organ Procurement Organization |
| OPTN | Organ Procurement and Transplantation Network |
| SRTR | Scientific Registry of Transplant Recipients |
| TDS | Transplant Data Services (HRSA's new data platform) |
| TIEDI | Transplant Information Electronic Data Interchange |
| UNOS | United Network for Organ Sharing |
| VCA | Vascularized Composite Allograft (e.g., hand, face transplants) |
