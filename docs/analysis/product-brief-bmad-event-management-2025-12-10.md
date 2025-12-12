---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments: []
workflowType: 'product-brief'
lastStep: 5
project_name: 'bmad-event-management'
user_name: 'Natalia'
date: '2025-12-10'
---

# Product Brief: bmad-event-management

**Date:** 2025-12-10
**Author:** Natalia

---

## Executive Summary

The Enterprise Event Management Platform (EEMP) addresses a critical governance and architectural crisis in a large financial institution's application platform. Today, hundreds of event types flow through on-prem Kafka infrastructure with zero standardization, no visibility into publishers and subscribers, and no access control enforcement. This creates regulatory risk (SOX, GDPR compliance gaps), security exposure (potential unauthorized access to PII), operational chaos (platform teams firefighting integration issues), and technical debt (AWS-based publishers and subscribers communicating through on-prem infrastructure).

The solution leverages an upcoming infrastructure migration as a forcing function to solve both technical and governance problems simultaneously: migrate to Confluent Cloud on AWS for managed Kafka infrastructure with built-in schema registry and stream governance, integrate with existing OPA (Open Policy Agent) for attribute-based access control, and build a custom governance layer that provides event catalog visibility, policy-driven subscription management, and comprehensive audit trails. This approach balances enterprise control with developer agility while enabling easy migration of existing events.

---

## Core Vision

### Problem Statement

Financial institutions manage critical business objects (users, profiles, preferences, documents, notifications, account relationships) whose state changes must be reliably captured and published to authorized subscribing applications. Currently, these events flow through fragmented infrastructure with no standardization, governance, or access controls, creating a multi-dimensional crisis:

**Governance Crisis:** Different teams create incompatible event schemas causing integration chaos. The organization literally cannot answer "what events exist in production" or "who subscribes to what." There is no schema registry, no event catalog, no visibility.

**Security & Compliance Risk:** Without access controls, there is risk of sensitive data (PII, account data) reaching unauthorized subscribers. Internal threats cannot be mitigated. Compliance teams cannot produce audit trails required for regulatory examinations (SOX, GDPR, banking regulations). Data classification policies cannot be enforced.

**Operational Inefficiency:** Platform teams firefight constant integration failures. Application teams build custom parsers for every event source. When publishers change schemas, downstream subscribers break with no warning. Troubleshooting requires tracking down individual developers who implemented features.

**Technical Debt:** Publishers and subscribers run on AWS but communicate through on-prem Kafka, creating suboptimal cross-network latency, operational complexity, and cost.

### Problem Impact

**Regulatory Exposure:** Cannot demonstrate compliance with data governance requirements during audits. Risk of violations for improper handling of PII or cross-border data flows.

**Security Vulnerabilities:** No enforcement of data classification or tenant isolation. Potential for insider threats to access sensitive events without authorization.

**Development Velocity:** Every new integration requires custom schema negotiation and parsing logic. Schema changes become breaking changes requiring coordinated deployments across teams.

**Operational Cost:** Platform teams spend significant time troubleshooting event-related issues. Infrastructure costs are higher due to inefficient on-prem/cloud architecture.

### Why Existing Solutions Fall Short

**Generic Event Platforms (Kafka, EventBridge, Solace):** Provide infrastructure but lack governance frameworks specific to financial services regulatory requirements. Don't address schema standardization, policy-driven access control, or audit trail requirements out-of-box.

**API Management Platforms:** Focus on synchronous APIs, not asynchronous event streams. Don't provide attribute-based access control for event subscriptions or data classification enforcement.

**Off-the-shelf Schema Registries:** Provide schema validation but lack integration with enterprise policy engines for access control decisions based on data sensitivity, regulatory jurisdiction, and subscriber entitlements.

**Current On-Prem Kafka:** Lacks schema governance, has no access control beyond basic ACLs, provides no visibility/catalog capabilities, and creates architectural inefficiency with AWS-based workloads.

### Proposed Solution

Leverage the forced migration from on-prem Kafka to solve both infrastructure and governance problems simultaneously:

**Confluent Cloud on AWS:** Managed Kafka ecosystem with built-in schema registry, stream governance capabilities, and enterprise-grade operations. Provides the event backbone and schema compliance enforcement.

**OPA Integration:** Integrate existing Open Policy Agent infrastructure to enforce attribute-based access control (ABAC) for event subscriptions. Policies evaluate subscriber attributes (application_id, tenant_id, data_classification_clearance, regulatory_jurisdiction) against event attributes (data_classification, tenant_id, pii_indicator, cross_border_flag) with default-deny posture and full audit logging.

**Custom Governance Layer:** Build platform services for:
- **Event Catalog:** Visibility into all event types, publishers, subscribers, and schemas
- **Subscription Management:** Self-service or workflow-based subscription requests with OPA policy evaluation
- **Schema Evolution Management:** Backward compatibility validation and subscriber impact analysis
- **Audit Trail:** Comprehensive logging of all policy decisions and event access for regulatory examination

**Migration Strategy:** Easy transition path for existing events through schema discovery/generation from current events, dual-publish capability during migration, and consumer compatibility with existing Kafka client patterns.

### Key Differentiators

**Financial Services-Grade Governance:** Purpose-built for regulatory compliance with audit trails, data classification enforcement, and policy-driven access control that maps to banking regulatory requirements.

**Leverage Existing Investments:** Integrates with existing OPA policy engine rather than introducing new policy technology. Uses proven Kafka patterns that development teams already understand.

**Migration-Friendly:** Designed specifically to support easy transition from current fragmented state without requiring simultaneous publisher/subscriber rewrites.

**Platform Extensibility:** Architecture supports evolution over time - can add data classification enforcement, tenant isolation guarantees, cross-border controls, and new policy types as requirements mature.

**Operational Efficiency:** Moves event infrastructure into AWS alongside publishers/subscribers, eliminating cross-network inefficiency while reducing operational burden through managed services.

**Developer Self-Service with Guardrails:** Balances enterprise control (policy enforcement, schema validation, audit) with developer agility (self-service subscriptions, clear schema contracts, automated approvals where policies allow).

## Target Users

### Primary Users

#### 1. Application Development Teams (Event Consumers)

**Persona: Sarah Chen, Senior Backend Developer**

Sarah works on the Customer Notifications service team, building features that need to react to user profile changes, document status updates, and preference modifications across the bank's platform. Her team of 5 engineers builds microservices that integrate with various platform events.

**Current Pain:** Today, when Sarah needs to consume events, she has to track down the team that publishes them, reverse-engineer their schema from example messages, write custom parsing logic, and hope nothing changes. When schemas do change unexpectedly, her service breaks in production with cryptic deserialization errors. She has no visibility into what events exist or which ones are stable enough to depend on.

**Success Vision:** Sarah discovers the events she needs by browsing a searchable event catalog with clear documentation and schema definitions. She requests a subscription through a self-service portal, and if OPA policies allow it, she gets instant access with connection details and schema validation libraries. All events follow a standard envelope structure (event_id, timestamp, correlation_id, source metadata), so she can build reusable consumer patterns once and apply them across all event types. When publishers evolve schemas, her code continues working because backward compatibility is automatically enforced. She receives notifications when events she consumes are deprecated, with clear migration paths and timelines.

**"Aha!" Moment:** Sarah realizes the platform's value when she needs to add a new feature that consumes document lifecycle events. She searches the catalog, finds "document.status.changed," reviews the schema, subscribes via UI, and has her integration working in under an hour—compared to the 2-week integration cycle she's used to.

**Key Needs:**
- Discoverable event catalog with search and filtering
- Self-service subscription with transparent policy decisions (clear error messages if denied)
- Standardized event envelope and naming conventions
- Automatic schema validation and backward compatibility guarantees
- Proactive notifications about schema deprecation and breaking changes
- Code generation tools (SDKs, type definitions) from schemas

#### 2. Platform Engineering Teams (Event Publishers)

**Persona: Marcus Rodriguez, Platform Engineer (User Profile Service)**

Marcus owns the User Profile service, one of the core platform services managing customer identity and profile data. His team publishes events for profile creation, updates, KYC status changes, and contact information modifications. They currently publish 15+ event types directly to Kafka topics with ad-hoc schemas.

**Current Pain:** Marcus doesn't know who consumes his events or how they're being used. When his team needs to add fields to events or fix schema bugs, they have no way to assess impact. They've caused production incidents by changing event structure without realizing downstream dependencies existed. They receive complaints about schema inconsistencies but have no tooling to prevent it. Troubleshooting requires manual coordination with consuming teams.

**Success Vision:** Marcus defines schemas once using Avro format and registers them through the platform API (integrated into his team's CI/CD pipeline via Terraform). The platform automatically validates that new schemas are backward compatible and shows him which subscribers would be affected by changes. He can see in the catalog exactly who subscribes to each event type. When his team needs to deprecate an old event version, the platform shows active consumers, allows him to mark it deprecated with a 90-day sunset date, and automatically notifies subscribers. When the sunset date arrives, the platform blocks new subscriptions to the deprecated version and provides visibility into any remaining legacy consumers.

**"Aha!" Moment:** Marcus realizes the platform's value when he needs to add a new "preferred_language" field to the user.profile.updated event. The platform validates backward compatibility automatically, shows him that 12 applications currently subscribe, and confirms the change is safe. His team deploys confidently, knowing they won't break downstream services.

**Key Needs:**
- Schema registration via API/IaC for CI/CD integration
- Automatic backward compatibility validation with clear error messages
- Visibility into active subscribers per event type
- Safe deprecation workflow with sunset dates and subscriber notifications
- Audit log of all schema changes and publications
- Testing/validation capability before production publication

### Secondary Users

#### 3. Platform Governance Team

Manages the EEMP platform itself, defines and maintains OPA policies, monitors subscription requests, and provides support when developers encounter access issues. Needs dashboards showing policy violations, subscription trends, and platform health metrics.

#### 4. Compliance & Audit Teams

Requires ability to generate audit reports showing who accessed what events during specific time periods, validate that access controls align with regulatory requirements, and produce evidence for SOX/GDPR examinations. Needs immutable audit trails and policy decision logs.

#### 5. Security & Risk Teams

Monitors for anomalous access patterns, validates data classification enforcement, and reviews policy configurations. Needs alerts for policy violations and dashboards showing event data flows across security boundaries.

### User Journey

**Event Consumer Journey (Sarah's Path):**

1. **Discovery:** Sarah's team needs to react to document signing events. She browses the EEMP event catalog, searches for "document," and finds "document.lifecycle.signed" with complete schema documentation and sample payloads.

2. **Subscription Request:** She clicks "Subscribe" in the portal, selects her application identity, and submits. OPA evaluates her application's clearance against the event's data classification. Since she has appropriate clearance, subscription is auto-approved instantly with Kafka connection details and consumer group configuration.

3. **Integration:** She downloads the auto-generated Java classes from the schema registry, implements her consumer using standard Kafka patterns, and validates against schema in her dev environment.

4. **Ongoing Use:** When the document team adds a new optional field "signer_ip_address," Sarah receives a notification. Her code continues working without changes because the field is optional and backward compatible.

5. **Deprecation Handling:** Six months later, the document team deprecates v1 of the event in favor of v2 with restructured metadata. Sarah receives a 90-day warning with migration guide, updates her code during the next sprint, and transitions smoothly before the sunset date.

**Event Publisher Journey (Marcus's Path):**

1. **Schema Definition:** Marcus's team adds a new "user.preferences.consent.updated" event to track GDPR consent changes. A developer writes an Avro schema following EEMP standards.

2. **Registration:** In their Terraform IaC, they define the schema resource with data classification "PII" and register it via platform API. The platform validates schema structure, assigns a version, and registers it in Confluent Schema Registry.

3. **Publication:** Their User Profile service publishes the first event. The Kafka producer automatically validates against the registered schema, rejecting any malformed events before they reach the broker.

4. **Monitoring:** Marcus uses the EEMP catalog UI to see that 3 applications have subscribed to the new event. The audit log shows when each subscription was created and which OPA policies allowed it.

5. **Evolution:** Three months later, they need to add "consent_source" field. The platform validates backward compatibility, confirms it's safe, and auto-increments the schema version. Existing subscribers continue working without code changes.

6. **Deprecation:** A year later, they want to consolidate consent events. They mark v1 deprecated with 90-day sunset. The platform automatically notifies all 8 current subscribers with migration documentation and tracks their transition to v2.

## Success Metrics

The Enterprise Event Management Platform succeeds when it transforms event-driven integration from a high-friction, risky operation into a governed, self-service capability that developers trust and compliance teams can audit.

### User Success Metrics

**Event Consumers (Application Development Teams):**
- **Fast Discovery & Integration:** Developers find relevant events and complete integration in under 1 hour (compared to 2-week integration cycles today)
- **Self-Service Adoption:** 80%+ of subscription requests automatically approved via OPA policy evaluation with instant access provisioning
- **Zero Schema Surprises:** Developers receive proactive notifications for all schema deprecations with clear migration paths and timelines
- **Reusable Patterns:** Standardized event envelope enables developers to build consumer logic once and reuse across all event types

**Event Publishers (Platform Engineering Teams):**
- **Safe Schema Evolution:** Publishers confidently evolve schemas with automatic backward compatibility validation and real-time subscriber impact analysis
- **Visibility & Control:** Publishers see exactly who subscribes to their events and can track schema version adoption across all consumers
- **Zero Breaking Changes:** 100% of schema changes validated before deployment, eliminating unintentional production incidents
- **Streamlined Deprecation:** Publishers safely retire old event versions with automated subscriber notification and migration tracking

### Business Objectives

**Governance & Compliance Excellence:**
- Achieve complete event catalog coverage with 100% of active event types registered with validated schemas
- Establish audit-ready posture with enriched, immutable audit trails for all regulatory examinations (SOX, GDPR, banking regulations)
- Enforce policy-driven access control with 100% of subscriptions evaluated by OPA with full decision logging

**Migration & Technical Modernization:**
- Successfully migrate all existing events from on-prem Kafka to Confluent Cloud on AWS with zero data loss
- Eliminate cross-network inefficiency by co-locating event infrastructure with AWS-based publishers and subscribers
- Reduce operational burden through managed services while maintaining 99.9%+ platform availability

**Developer Velocity & Platform Adoption:**
- Reduce time-to-integration from 2 weeks to under 1 hour through self-service catalog and subscription management
- Enable platform extensibility supporting future requirements (data classification enforcement, tenant isolation, cross-border controls) without architectural rework
- Establish platform as the standard for event-driven integration across the financial institution

### Key Performance Indicators

**Governance & Quality KPIs:**
- **Event Catalog Coverage:** 100% of active event types registered in catalog with validated schemas
- **Policy Enforcement:** 100% of subscription requests evaluated by OPA with audit trail of all policy decisions
- **Event Enrichment:** 100% of audit log entries include required context (actor, timestamp, correlation_id, source system, OPA policy evaluation results)
- **Schema Compliance:** Zero malformed events reach Kafka (all rejected at publication via automatic schema validation)
- **Backward Compatibility:** Zero unintentional breaking schema changes deployed to production

**Developer Experience KPIs:**
- **Discovery Efficiency:** 90%+ of event searches return relevant results on first attempt
- **Integration Velocity:** Average time from event discovery to working integration < 1 hour
- **Self-Service Success Rate:** 80%+ of subscription requests auto-approved without manual intervention
- **Subscription Latency:** Auto-approved subscriptions provisioned in < 5 minutes, manual approvals < 24 hours

**Operational KPIs:**
- **Platform Availability:** 99.9%+ uptime for event catalog, subscription management, and schema registry services
- **Event Delivery Performance:** p99 latency < 100ms for event delivery through Confluent Cloud
- **Migration Success:** 100% of existing on-prem events migrated to Confluent Cloud within [target timeframe]
- **Audit Readiness:** Complete audit trail available for 100% of events with enrichment data for regulatory examination

**Security & Risk KPIs:**
- **Access Control Coverage:** 100% of events classified with data sensitivity levels
- **Policy Violation Detection:** Real-time alerting on all policy violations or anomalous access patterns
- **Compliance Reporting:** Ability to generate audit reports showing event access by application, time period, and data classification within 1 hour

## MVP Scope

### Core Features

**1. Confluent Cloud Event Infrastructure**
- Single Kafka topic (`platform.events`) deployed on Confluent Cloud (AWS)
- Confluent Schema Registry configured with 10 standardized event type schemas
- Migration from on-prem Kafka infrastructure complete with zero data loss
- Dual-publish capability during migration cutover period

**2. Standardized Event Model (10 Lifecycle Events)**
- **Standard Event Envelope:** All events include `event_type`, `event_id`, `timestamp`, `correlation_id`, `source`, `actor`, and enrichment context
- **Event Types:**
  - User lifecycle: `user.created`, `user.updated`
  - Profile lifecycle: `profile.created`, `profile.updated`
  - Document lifecycle: `document.created`, `document.updated`
  - User Preference lifecycle: `preference.created`, `preference.updated`
  - Notification lifecycle: `notification.created`, `notification.updated`
- Avro schema for each event type registered in Confluent Schema Registry
- Automatic schema validation on publish (malformed events rejected)
- Backward compatibility validation for schema evolution

**3. Event Catalog & Discovery**
- Event catalog UI/portal displaying all 10 event types
- Search and browse functionality by event type
- Schema documentation and sample payloads for each event type
- Visibility into active subscribers per event type

**4. Publisher Capabilities**
- Publisher SDK/library that validates events against registered schemas based on `event_type`
- Schema registration API (Terraform/IaC compatible)
- All events published to single topic with proper `event_type` discriminator
- 5 core platform services migrated to publish through new platform

**5. Subscription Management with Event-Type Filtering**
- Self-service subscription request UI/API where consumers specify which event types they need
- OPA policy integration evaluating access per event type (e.g., "Can App X subscribe to `user.created`?")
- Auto-provisioning of Kafka connection details for approved subscriptions
- Consumer SDK with client-side filtering by `event_type` (consumers read from topic, filter in application code)
- Subscription audit trail tracking which apps subscribe to which event types

**6. Governance & Audit Trail**
- Event enrichment pipeline adding required context before audit logging (actor, timestamp, correlation_id, source system)
- OPA policy decision logging for all subscription requests
- Immutable audit trail storage for regulatory examination
- Audit query capability for compliance team (who accessed which event types, when)

**7. Migration Success Criteria**
- Existing publishers migrated from on-prem to Confluent Cloud
- Existing consumers successfully consuming from Confluent Cloud with event-type filtering
- Zero data loss during migration
- On-prem Kafka infrastructure decommissioned for these 10 events

### Out of Scope for MVP

**Deferred to Post-MVP:**

**Advanced Filtering & Routing:**
- Platform-managed filtering proxy/gateway (MVP uses client-side filtering)
- Content-based routing beyond event_type
- Complex subscription filters (e.g., "only user.updated events where country=US")

**Schema Deprecation Workflow:**
- Automated schema deprecation notifications to subscribers
- Sunset date enforcement and migration tracking
- Schema version adoption dashboard

**Data Classification & Tenant Isolation:**
- Automatic data classification enforcement based on event content
- Tenant isolation guarantees and cross-tenant access prevention
- Cross-border data sovereignty controls

**Advanced Catalog Features:**
- Publisher impact analysis showing downstream subscriber dependencies
- Real-time subscriber health monitoring
- Code generation tools (auto-generated SDKs, type definitions)

**Additional Event Types:**
- Expansion beyond the initial 10 lifecycle events
- Account relationship events
- Transaction events
- Other business object domains

**Advanced Governance:**
- Anomaly detection and suspicious access pattern alerting
- Automated compliance reporting dashboard
- Policy simulation/testing tools

**Multi-Region & High Availability:**
- Multi-region Confluent Cloud deployment
- Geo-redundancy and disaster recovery
- Cross-region event replication

### MVP Success Criteria

The MVP is considered successful when:

**Technical Validation:**
- All 10 event types successfully publishing to Confluent Cloud with schema validation
- At least 5 consumer applications successfully subscribed and filtering events by type
- Zero malformed events reaching Kafka (100% schema compliance)
- Zero data loss during migration from on-prem
- Platform availability >99% during MVP period

**User Validation:**
- Event consumers can discover, subscribe, and integrate new event types in under 2 hours (vs. 2 weeks baseline)
- 70%+ of subscription requests auto-approved via OPA policies
- Publishers can evolve schemas without breaking consumers (backward compatibility working)
- Compliance team can query audit trail and produce reports within 1 hour

**Business Validation:**
- Complete audit trail available for all 10 event types
- OPA policy enforcement working with full decision logging
- Migration completed within target timeframe with zero incidents
- Platform team receives positive feedback from 3+ publisher and consumer teams

**Go/No-Go Decision:**
If MVP success criteria are met, proceed with:
- Migrating additional event types (expand from 10 to 50+)
- Implementing schema deprecation workflow
- Adding platform-managed filtering capabilities
- Rolling out data classification enforcement

### Future Vision

**6-12 Months Post-MVP:**
- **Complete Event Catalog:** 100+ event types covering all platform business objects
- **Advanced Governance:** Full data classification enforcement, tenant isolation, cross-border controls operational
- **Developer Experience:** Code generation tools, interactive schema browser, consumer health monitoring
- **Enterprise Scale:** Multi-region deployment, 99.99% availability SLA, support for 1000+ subscribing applications

**12-24 Months:**
- **Event Mesh Evolution:** Support for multiple event patterns (pub/sub, streaming, request/response, event sourcing)
- **AI-Powered Insights:** Anomaly detection, usage pattern analysis, automated schema suggestions
- **External Integration:** Secure event exposure to partner organizations and external systems
- **Platform Ecosystem:** Event marketplace where teams can discover and compose event-driven solutions
