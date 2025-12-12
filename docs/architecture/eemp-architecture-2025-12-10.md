# Enterprise Event Management Platform (EEMP) - Technical Architecture

**Version:** 1.0
**Date:** 2025-12-10
**Status:** Draft
**Author:** Architecture Team

---

## Executive Summary

This document defines the technical architecture for the Enterprise Event Management Platform (EEMP), a mission-critical platform designed to bring governance, security, and operational excellence to event-driven integration in a financial services environment. The architecture leverages Confluent Cloud on AWS for managed Kafka infrastructure, integrates with existing Open Policy Agent (OPA) for attribute-based access control, and introduces a custom governance layer to provide event catalog visibility, policy-driven subscription management, and comprehensive audit trails.

### Architectural Principles

1. **Security by Default:** All access decisions evaluated through OPA with default-deny posture
2. **Compliance First:** Immutable audit trails and enriched logging for regulatory examination
3. **Developer Experience:** Self-service capabilities with clear feedback and fast integration cycles
4. **Operational Excellence:** Managed services where possible, automated validation, and resilient design
5. **Migration-Friendly:** Support existing Kafka patterns while enabling modernization
6. **Extensibility:** Architecture supports evolution (data classification, tenant isolation, cross-border controls)

---

## 1. Kafka Platform Comparison & Selection

### 1.1 Platform Options Analysis

This section provides a detailed comparison of the three primary Kafka platform options considered for EEMP: Confluent Cloud, AWS Managed Streaming for Apache Kafka (MSK), and self-managed Apache Kafka on-premises.

#### Option 1: Confluent Cloud (SELECTED)

**Overview:**
Fully managed Kafka service by Confluent (creators of Apache Kafka) with integrated schema registry, stream governance, and enterprise features.

**Key Capabilities:**
- Managed Kafka cluster with automatic scaling, patching, and upgrades
- Built-in Schema Registry with backward/forward compatibility enforcement
- Stream Governance Suite (data lineage, quality rules, audit)
- ksqlDB for stream processing (optional)
- Cluster Linking for multi-region replication
- 99.95% uptime SLA for Dedicated clusters
- Global multi-cloud support (AWS, Azure, GCP)

**Pros:**
- **Zero Operational Overhead:** Confluent manages infrastructure, patching, scaling, monitoring
- **Integrated Schema Registry:** Native schema versioning and compatibility validation (critical for governance)
- **Stream Governance Tools:** Built-in audit capabilities, data lineage, and quality monitoring aligned with financial services requirements
- **Expert Support:** Direct access to Kafka creators for critical issues
- **Advanced Features:** Cluster Linking, ksqlDB, and features often 6-12 months ahead of Apache Kafka releases
- **Fast Time-to-Market:** Deploy production cluster in hours, not weeks
- **Elasticity:** Auto-scaling capabilities without manual intervention
- **Security:** Encryption at rest/in transit by default, RBAC, audit logging
- **Multi-Region Ready:** Easy expansion to multi-region for disaster recovery (post-MVP)

**Cons:**
- **Higher Cost:** 2-3x more expensive than MSK, 4-5x more than on-prem (when factoring ops overhead)
- **Vendor Lock-in:** Proprietary features (Stream Governance) create switching costs
- **Less Control:** Cannot customize JVM settings, Kafka broker configurations beyond exposed parameters
- **Network Latency:** Internet-based connectivity from AWS VPC (mitigated with VPC peering)

**Cost Estimate (MVP - Dedicated Cluster):**
- 2 CKU (Confluent Units): $1.50/hour × 730 hours = **$1,095/month**
- Storage (500 GB): $0.10/GB = **$50/month**
- Schema Registry: Included
- Data transfer (1 TB ingress/egress): $50/month
- **Total: ~$1,200/month** ($14,400/year)

**Best For:**
- Organizations prioritizing speed-to-market and minimal operational overhead
- Financial services requiring enterprise-grade governance and audit capabilities
- Teams without deep Kafka expertise
- Projects where schema governance is critical

---

#### Option 2: AWS Managed Streaming for Kafka (MSK)

**Overview:**
AWS-managed Kafka service running Apache Kafka on EC2 instances, integrated with AWS ecosystem.

**Key Capabilities:**
- Managed Apache Kafka brokers (automated patching, recovery)
- AWS native integrations (IAM, CloudWatch, VPC, KMS)
- MSK Serverless (autoscaling) or Provisioned (fixed capacity)
- MSK Connect for Kafka Connect integrations
- Compatible with open-source Kafka clients and tools

**Pros:**
- **AWS Native Integration:** Seamless integration with IAM, VPC, CloudWatch, Secrets Manager
- **Lower Cost Than Confluent:** 40-50% cheaper than Confluent Cloud for equivalent capacity
- **Open-Source Compatibility:** Standard Apache Kafka (no vendor lock-in)
- **Control:** More configuration options than Confluent Cloud (broker configs, JVM tuning)
- **VPC-Native:** Private connectivity without internet egress
- **Familiar Billing:** Part of AWS consolidated billing

**Cons:**
- **No Built-in Schema Registry:** Must deploy and manage Confluent Schema Registry separately or use AWS Glue Schema Registry (less mature, limited compatibility enforcement)
- **Limited Governance Tools:** No stream governance, data lineage, or audit capabilities out-of-box
- **Operational Overhead:** Requires management of Schema Registry, monitoring setup, capacity planning
- **Slower Feature Adoption:** Apache Kafka features lag Confluent's commercial releases
- **Manual Scaling:** Provisioned clusters require manual broker addition/removal
- **Support Limitations:** AWS support not Kafka-specialized (escalation to Kafka experts slower)
- **No ksqlDB:** Must deploy separately if stream processing needed

**Cost Estimate (MVP - Provisioned):**
- 3 kafka.m5.large brokers: $0.21/hour × 3 × 730 = **$460/month**
- Storage (500 GB): $0.10/GB = **$50/month**
- Schema Registry (self-managed on EC2 t3.medium): $30/month
- Data transfer (1 TB): $40/month
- **Total: ~$580/month** ($6,960/year)

**Additional Hidden Costs:**
- Engineering time to deploy and manage Schema Registry: **~40 hours/quarter** ($8,000/year at $200/hour blended rate)
- Monitoring setup and dashboard creation: **~20 hours** (one-time)
- Capacity planning and scaling: **~10 hours/quarter** ($4,000/year)
- **True Total Cost of Ownership: ~$19,000/year**

**Best For:**
- Organizations with existing Kafka expertise and dedicated platform teams
- Cost-sensitive projects willing to trade operational overhead for savings
- AWS-centric architectures prioritizing native integrations
- Use cases not requiring advanced schema governance

---

#### Option 3: On-Premises Apache Kafka (Self-Managed)

**Overview:**
Deploy and manage Apache Kafka on company-owned infrastructure (existing on-prem or AWS EC2).

**Key Capabilities:**
- Full control over Kafka deployment (versions, configurations, hardware)
- Use open-source Apache Kafka and ecosystem tools
- Integrate with existing on-prem infrastructure and tooling

**Pros:**
- **Complete Control:** Full access to all Kafka configurations, JVM tuning, broker placement
- **No Cloud Egress Costs:** Data stays within corporate network or VPC
- **Predictable Costs:** Fixed infrastructure costs (if using existing hardware)
- **Data Sovereignty:** Full control over data location and residency
- **Customization:** Can apply custom patches, plugins, and monitoring

**Cons:**
- **Significant Operational Burden:** Requires dedicated platform team for deployment, patching, upgrades, monitoring, capacity planning
- **Infrastructure Management:** Must provision and manage servers, storage, networking, high availability
- **No Schema Registry by Default:** Must deploy and manage Confluent Schema Registry (licensed) or Apicurio (open-source)
- **Governance Gap:** No built-in audit, lineage, or governance tools (must build custom)
- **Slow Scaling:** Adding capacity requires hardware procurement, provisioning, rebalancing
- **Single Point of Failure Risk:** Requires expertise to achieve enterprise HA/DR
- **Cross-Network Latency:** If publishers/consumers on AWS, on-prem introduces latency and network complexity
- **Compliance Burden:** Must self-implement audit logging, encryption, access controls to meet regulatory requirements
- **Upgrade Risk:** Manual upgrades with downtime or complex rolling upgrade procedures

**Cost Estimate (MVP - On-Prem/EC2):**
- 3 EC2 instances (m5.xlarge): $0.192/hour × 3 × 730 = **$421/month**
- EBS storage (500 GB): $50/month
- Schema Registry (EC2 t3.medium): $30/month
- Network egress (if on-prem): Variable (potentially high for AWS publishers/consumers)
- **Total Infrastructure: ~$500/month** ($6,000/year)

**Hidden Operational Costs (Critical):**
- Platform engineering (1 FTE @ 25% allocation): **$40,000/year**
- Initial setup and configuration: **~120 hours** ($24,000 one-time)
- Ongoing monitoring, alerting, capacity planning: **~20 hours/month** ($48,000/year)
- Upgrades and patching: **~40 hours/quarter** ($16,000/year)
- Incident response and troubleshooting: **~10 hours/month** ($24,000/year)
- **True Total Cost of Ownership: ~$140,000/year** (after initial setup)

**Best For:**
- Organizations with existing Kafka expertise and dedicated platform teams (3+ engineers)
- Strict data residency requirements preventing cloud usage
- Very high-volume use cases where cloud costs become prohibitive (>100 TB/month)
- Organizations with existing on-prem infrastructure and low cloud adoption

---

### 1.2 Decision Matrix

| Criteria | Weight | Confluent Cloud | AWS MSK | On-Prem Kafka |
|----------|--------|-----------------|---------|---------------|
| **Governance & Compliance** | 25% | **10/10** - Built-in schema governance, audit, lineage | 5/10 - Manual schema registry, no governance tools | 3/10 - Must build custom governance |
| **Operational Simplicity** | 20% | **10/10** - Fully managed, zero ops overhead | 6/10 - Managed Kafka but manual schema registry | 2/10 - Full operational burden |
| **Time to Market** | 15% | **10/10** - Deploy in hours | 7/10 - Deploy in 1-2 days | 3/10 - Deploy in 2-4 weeks |
| **Cost (3-year TCO)** | 15% | 6/10 - $43K over 3 years | **8/10** - $21K TCO | 4/10 - $420K TCO |
| **Schema Management** | 10% | **10/10** - Native, integrated, compatibility enforcement | 5/10 - Separate deployment, limited features | 5/10 - Separate deployment, open-source limitations |
| **Security & Audit** | 10% | **10/10** - Encryption, RBAC, audit by default | 7/10 - AWS security, manual audit setup | 5/10 - Must implement all controls |
| **Migration Effort** | 5% | 8/10 - API compatibility, dual-publish support | 8/10 - Standard Kafka | **10/10** - Already on-prem (but staying is not an option) |
| **Scalability** | 5% | **10/10** - Auto-scaling, multi-region ready | 7/10 - Manual scaling for provisioned | 5/10 - Hardware-dependent |
| **Vendor Lock-in Risk** | 5% | 5/10 - Proprietary features create switching cost | **9/10** - Open-source compatible | **10/10** - Full control |

**Weighted Scores:**
- **Confluent Cloud: 8.5/10** (Winner)
- AWS MSK: 6.4/10
- On-Prem Kafka: 3.9/10

---

### 1.3 Decision Rationale

**Selected Platform: Confluent Cloud**

**Primary Justifications:**

1. **Governance is Non-Negotiable:**
   - Financial services regulatory requirements (SOX, GDPR) demand robust schema governance, audit trails, and data lineage
   - Confluent's Stream Governance Suite provides these capabilities out-of-box
   - Building equivalent governance on MSK or on-prem would require 6+ months of custom development ($200K+ engineering cost)

2. **Minimize Operational Risk:**
   - EEMP is a mission-critical platform affecting hundreds of applications
   - Confluent's 99.95% SLA and expert support reduce operational risk
   - Platform team can focus on business logic (governance layer) rather than infrastructure management

3. **Speed to Market:**
   - Forced migration from on-prem Kafka creates hard deadline
   - Confluent Cloud enables deployment in weeks vs. months (MSK) or quarters (on-prem rebuild)
   - MVP success depends on demonstrating value quickly to stakeholders

4. **Total Cost of Ownership:**
   - While Confluent has higher infrastructure costs ($14K/year), TCO is lowest when factoring engineering time
   - MSK TCO: $21K infrastructure + schema registry management + governance development = **$50K+/year**
   - On-prem TCO: $6K infrastructure + platform engineering (1+ FTE) = **$140K+/year**
   - Confluent TCO: **$14K/year** (governance included, zero ops overhead)

5. **Team Capability Match:**
   - Current team has limited Kafka operations expertise
   - Hiring dedicated Kafka platform engineers would cost $150K+/year per FTE
   - Confluent managed service eliminates this need

6. **Future-Proofing:**
   - Post-MVP roadmap includes multi-region deployment, advanced stream processing, data classification
   - Confluent provides clear upgrade path (Cluster Linking, ksqlDB, enhanced governance)
   - MSK/on-prem would require significant re-architecture for these features

**Acceptable Trade-offs:**

1. **Higher Infrastructure Cost:**
   - $14K/year premium over MSK infrastructure cost is acceptable given $100K+ savings in engineering time
   - Cost scales with usage (not flat FTE cost like on-prem)

2. **Vendor Lock-in:**
   - Stream Governance features create switching cost, but migration risk is low
   - Core Kafka and Avro remain open standards (data portability maintained)
   - Business value of governance outweighs lock-in risk

3. **Less Control:**
   - Cannot customize broker JVM settings or apply custom patches
   - Trade-off acceptable: financial services prioritizes stability over customization

**Migration from On-Prem:**

The decision to move to Confluent Cloud also solves the architectural inefficiency of AWS-based publishers/consumers communicating through on-prem Kafka:
- Eliminates cross-network latency (on-prem → AWS)
- Reduces operational complexity (dual infrastructure footprint)
- Enables decommissioning of on-prem Kafka infrastructure (cost savings)

---

### 1.4 Post-Decision Validation

**Risks and Mitigations:**

| Risk | Mitigation |
|------|------------|
| **Confluent Cloud Outage** | 99.95% SLA, multi-AZ deployment, automated failover. Post-MVP: Multi-region active-passive. |
| **Cost Overruns** | Set up billing alerts, monitor CKU utilization, plan capacity based on actual usage patterns. |
| **Feature Gaps** | Conduct POC during first 2 weeks to validate schema governance, OPA integration, audit capabilities. |
| **Vendor Dependency** | Maintain standard Kafka client libraries, avoid proprietary APIs where possible, document migration path. |

**Success Criteria for Platform Choice:**

After 90 days of production use:
- ✓ Schema Registry 100% operational with zero manual intervention
- ✓ Zero Kafka infrastructure incidents requiring platform team intervention
- ✓ Stream Governance audit trails meet compliance team requirements
- ✓ Time-to-integrate for new events < 1 hour (vs. 2 weeks on on-prem)
- ✓ Total platform engineering time < 10 hours/month (vs. 80+ hours on on-prem)

If success criteria not met, re-evaluate MSK as alternative (Confluent Cloud → MSK migration is straightforward).

---

## 2. System Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EEMP Platform (AWS)                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────┐         ┌──────────────────────────────────────┐  │
│  │  Event Publishers │────────>│    Governance Layer (ECS Fargate)   │  │
│  │  (Platform APIs)  │         │                                      │  │
│  └──────────────────┘         │  ┌────────────────────────────────┐  │  │
│                                │  │  Publisher Service             │  │  │
│                                │  │  - Schema Validation           │  │  │
│  ┌──────────────────┐         │  │  - Event Enrichment            │  │  │
│  │  Event Catalog   │<───────>│  │  - Audit Logging               │  │  │
│  │  Portal (React)  │         │  └────────────────────────────────┘  │  │
│  │                  │         │                                      │  │
│  │  - Event Discovery│         │  ┌────────────────────────────────┐  │  │
│  │  - Subscription  │         │  │  Subscription Service          │  │  │
│  │  - Schema Browser│<───────>│  │  - OPA Integration             │  │  │
│  └──────────────────┘         │  │  - Access Provisioning         │  │  │
│                                │  │  - Audit Logging               │  │  │
│                                │  └────────────────────────────────┘  │  │
│  ┌──────────────────┐         │                                      │  │
│  │ Event Consumers  │<────────│  ┌────────────────────────────────┐  │  │
│  │ (Applications)   │         │  │  Catalog Service               │  │  │
│  └──────────────────┘         │  │  - Event Registry              │  │  │
│                                │  │  - Schema Management           │  │  │
│                                │  │  - Publisher/Subscriber Views  │  │  │
│         │                      │  └────────────────────────────────┘  │  │
│         │                      │                                      │  │
│         │                      │  ┌────────────────────────────────┐  │  │
│         │                      │  │  Audit Service                 │  │  │
│         │                      │  │  - Enrichment Pipeline         │  │  │
│         │                      │  │  - Query API                   │  │  │
│         ▼                      │  │  - Compliance Reporting        │  │  │
│  ┌──────────────────────────┐ │  └────────────────────────────────┘  │  │
│  │   Confluent Cloud (AWS)  │ │                                      │  │
│  │                          │ └──────────────────────────────────────┘  │
│  │  ┌────────────────────┐ │                                            │
│  │  │ Kafka Cluster      │<────── Publishers & Consumers ──────────────┤
│  │  │ Topic:             │ │                                            │
│  │  │ platform.events    │ │                                            │
│  │  └────────────────────┘ │                                            │
│  │                          │                                            │
│  │  ┌────────────────────┐ │                                            │
│  │  │ Schema Registry    │<────── Schema Validation ───────────────────┤
│  │  │ (Confluent Cloud)  │ │                                            │
│  │  └────────────────────┘ │                                            │
│  │                          │                                            │
│  │  ┌────────────────────┐ │                                            │
│  │  │ Stream Governance  │ │                                            │
│  │  │ (Audit Metrics)    │ │                                            │
│  │  └────────────────────┘ │                                            │
│  └──────────────────────────┘                                            │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Supporting Infrastructure                      │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  - RDS PostgreSQL (Metadata, Subscriptions, Audit Index)         │   │
│  │  - S3 (Immutable Audit Trail, Schema Archive)                    │   │
│  │  - ElastiCache Redis (Session, Caching)                          │   │
│  │  - OPA Cluster (Policy Decisions) - Existing Infrastructure      │   │
│  │  - CloudWatch (Monitoring, Alerting)                             │   │
│  │  - Secrets Manager (Credentials, API Keys)                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Overview

| Component | Purpose | Technology | Hosting |
|-----------|---------|------------|---------|
| **Publisher Service** | Schema validation, event enrichment, publishing to Kafka | Node.js/TypeScript | ECS Fargate |
| **Subscription Service** | OPA integration, access provisioning, subscription lifecycle | Node.js/TypeScript | ECS Fargate |
| **Catalog Service** | Event registry, schema management, discovery API | Node.js/TypeScript | ECS Fargate |
| **Audit Service** | Enrichment pipeline, query API, compliance reporting | Node.js/TypeScript | ECS Fargate |
| **Event Catalog Portal** | Web UI for discovery, subscription, schema browsing | React + TypeScript | S3 + CloudFront |
| **Kafka Cluster** | Event backbone (single topic: `platform.events`) | Confluent Cloud | AWS (Confluent-managed) |
| **Schema Registry** | Schema storage, validation, versioning | Confluent Cloud | AWS (Confluent-managed) |
| **Metadata Store** | Event catalog, subscriptions, publisher registry | PostgreSQL RDS | AWS RDS |
| **Audit Store** | Immutable audit trail, enriched event logs | S3 + PostgreSQL | AWS S3 + RDS |
| **OPA Cluster** | Policy evaluation engine (ABAC) | Open Policy Agent | Existing Infrastructure |

---

## 3. Confluent Cloud Architecture

### 3.1 Kafka Cluster Design

**Decision: Single-Topic Architecture for MVP**

**Rationale:**
- Simplifies infrastructure management and reduces Confluent Cloud costs
- All events share common governance model (schema validation, access control, audit)
- Event-type filtering handled at consumer level (client-side filtering via SDK)
- Aligns with financial services pattern of centralized audit trail
- Enables efficient migration from on-prem Kafka without complex topic mapping

**Configuration:**

```yaml
kafka_cluster:
  provider: confluent_cloud
  region: us-east-1
  availability: single-region (MVP), multi-region (future)
  cloud_provider: AWS
  cluster_type: Dedicated
  cku: 2 (MVP baseline, scale based on throughput)

  topic_configuration:
    name: platform.events
    partitions: 12  # Based on expected throughput and consumer parallelism
    replication_factor: 3  # Confluent Cloud default for durability
    retention_ms: 604800000  # 7 days (compliance requirement)
    min_insync_replicas: 2  # Ensure write durability
    compression_type: lz4  # Balance compression ratio and CPU

  performance_targets:
    throughput: 10,000 events/sec (MVP baseline)
    p99_latency: < 100ms
    availability: 99.9%
```

**Topic Partitioning Strategy:**
- Partition by `event_type` hash to ensure ordering within event type
- 12 partitions provide headroom for 12+ consumer instances per consumer group
- Future scaling: Add partitions as throughput grows (Kafka supports partition expansion)

**Key Decision: Single Topic vs. Multiple Topics**

| Aspect | Single Topic (CHOSEN) | Multiple Topics |
|--------|----------------------|-----------------|
| **Governance** | Unified schema validation, consistent audit | Per-topic policies, fragmented audit |
| **Cost** | Lower (fewer Kafka resources) | Higher (each topic = separate resources) |
| **Consumer Complexity** | Client-side filtering required | Native topic subscription |
| **Migration Effort** | Simple mapping | Complex topic routing logic |
| **Future Flexibility** | Can split topics later if needed | Harder to consolidate |

**Future Consideration:** If specific event types require different retention policies or SLAs, introduce dedicated topics (e.g., `platform.events.high-retention` for audit events requiring 5-year retention).

---

### 3.1.1 Multi-Account and Hybrid Connectivity Architecture

**Context:**
The EEMP platform runs in a dedicated AWS account (Platform Account), while event publishers and subscribers are distributed across:
- Multiple AWS accounts (Application Accounts)
- On-premises data centers
- Potentially other cloud providers (future)

This distributed architecture requires careful design for connectivity, authentication, and network security.

#### Cross-Account AWS Architecture

**Deployment Model:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Organization Structure                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Platform Account (Account ID: 111111111111)               │    │
│  │                                                             │    │
│  │  ┌──────────────────────────────────────────────────────┐  │    │
│  │  │  EEMP Governance Layer (VPC)                         │  │    │
│  │  │  - Publisher Service (ECS)                           │  │    │
│  │  │  - Subscription Service (ECS)                        │  │    │
│  │  │  - Catalog Service (ECS)                             │  │    │
│  │  │  - RDS PostgreSQL                                    │  │    │
│  │  └──────────────────────────────────────────────────────┘  │    │
│  │                                                             │    │
│  │  ┌──────────────────────────────────────────────────────┐  │    │
│  │  │  Confluent Cloud Kafka Cluster                       │  │    │
│  │  │  - Hosted by Confluent (not in customer VPC)         │  │    │
│  │  │  - Internet endpoint: pkc-xxx.aws.confluent.cloud    │  │    │
│  │  │  - PrivateLink endpoint (optional, post-MVP)         │  │    │
│  │  └──────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────┘    │
│                           │                                         │
│                           │ Cross-Account Access                    │
│                           ▼                                         │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Application Account 1 (Account ID: 222222222222)         │    │
│  │                                                             │    │
│  │  Publishers/Subscribers:                                   │    │
│  │  - User Profile Service (Publisher)                        │    │
│  │  - Notifications Service (Subscriber)                      │    │
│  │                                                             │    │
│  │  Connectivity:                                             │    │
│  │  → Internet → Confluent Cloud Kafka (TLS + SASL/PLAIN)    │    │
│  │  → Internet → EEMP APIs (HTTPS + JWT or mTLS)             │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Application Account 2 (Account ID: 333333333333)         │    │
│  │                                                             │    │
│  │  Publishers/Subscribers:                                   │    │
│  │  - Document Service (Publisher)                            │    │
│  │  - Analytics Service (Subscriber)                          │    │
│  │                                                             │    │
│  │  Connectivity: (same as Account 1)                         │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    On-Premises Data Center                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Publishers/Subscribers:                                            │
│  - Legacy Mainframe Integration Service                             │
│  - On-Prem Document Processing                                      │
│                                                                      │
│  Connectivity Options:                                              │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Option 1: Direct Internet (MVP)                            │    │
│  │ → Corporate Proxy → Internet → Confluent Cloud            │    │
│  │ → Corporate Proxy → Internet → EEMP APIs                  │    │
│  │                                                             │    │
│  │ Security: TLS + SASL/PLAIN, IP allowlisting               │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ Option 2: AWS Direct Connect (Post-MVP)                    │    │
│  │ → Direct Connect → AWS Transit Gateway → Platform VPC     │    │
│  │                                                             │    │
│  │ Security: Private network, no internet exposure            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

#### Connectivity Patterns

**Pattern 1: AWS Cross-Account Publishers/Subscribers (MVP)**

```
Application Service (Any AWS Account)
    │
    │ 1. Publish/Consume Events
    │    - Direct HTTPS connection to Confluent Cloud
    │    - Authentication: Confluent Cloud API Key (SASL/PLAIN over TLS)
    │    - No VPC peering required (Confluent Cloud is internet-accessible)
    │
    ▼
Confluent Cloud Kafka Cluster
    - Endpoint: pkc-xxxxx.us-east-1.aws.confluent.cloud:9092
    - Security: TLS 1.2+ encryption
    - Authentication: Per-subscriber API keys (managed by EEMP Subscription Service)
    - Authorization: Confluent Cloud ACLs (READ/WRITE to platform.events topic)
```

**Benefits:**
- **No VPC Peering Required:** Confluent Cloud has public internet endpoints (over TLS)
- **Simple Network Architecture:** No cross-account VPC peering, route tables, or security group management
- **Account Isolation:** Each application account is independent, no network dependencies
- **Fast Onboarding:** New AWS accounts can connect immediately without network provisioning

**Considerations:**
- **Internet Egress Costs:** Traffic to Confluent Cloud incurs AWS data transfer charges (~$0.09/GB to internet)
- **Latency:** Internet routing may add 5-10ms vs. private connectivity
- **Security:** Requires trusting internet path (mitigated by TLS encryption and authentication)

**Pattern 2: AWS Cross-Account via PrivateLink (Post-MVP Enhancement)**

For higher security requirements or cost optimization:

```
Application Service (Application Account)
    │
    │ Via AWS PrivateLink
    │
    ▼
VPC Endpoint (Application Account VPC)
    │
    │ Cross-Account PrivateLink Connection
    │
    ▼
Confluent Cloud PrivateLink Endpoint
    │
    ▼
Confluent Cloud Kafka Cluster (Private IP)
```

**Benefits:**
- **Private Network:** Traffic never traverses public internet
- **Lower Latency:** Direct AWS backbone routing (1-3ms improvement)
- **Lower Costs:** No internet egress charges (potential $5K-10K/year savings at scale)
- **Compliance:** Meets strict "no internet" security policies

**Costs:**
- Confluent PrivateLink: ~$0.01/GB data processed + $0.01/hour per endpoint
- AWS PrivateLink: $0.01/hour per AZ + $0.01/GB data processed
- **Total added cost:** ~$150-300/month for MVP scale

**Implementation Complexity:**
- Requires VPC endpoint creation in each application account
- Confluent Cloud PrivateLink setup (one-time configuration)
- DNS resolution configuration

**Recommendation:** Defer to post-MVP unless security policy mandates private connectivity.

---

**Pattern 3: On-Premises Publishers/Subscribers**

**Option A: Direct Internet Connectivity (MVP)**

```
On-Prem Application
    │
    │ Through corporate proxy/firewall
    │
    ▼
Corporate Internet Gateway
    │
    │ TLS 1.2+ encrypted connection
    │
    ▼
Confluent Cloud Kafka (Internet Endpoint)
    - URL: pkc-xxxxx.us-east-1.aws.confluent.cloud:9092
    - Authentication: SASL/PLAIN with API key
    - IP Allowlisting: Confluent Cloud allows whitelisting source IPs
```

**Configuration Requirements:**

1. **Firewall Rules (On-Prem):**
   ```
   Outbound HTTPS (443): To EEMP API endpoints (platform-api.company.com)
   Outbound Kafka (9092): To Confluent Cloud (*.aws.confluent.cloud)
   ```

2. **IP Allowlisting (Confluent Cloud):**
   ```yaml
   # Configure in Confluent Cloud cluster settings
   allowed_ips:
     - 203.0.113.0/24  # On-prem NAT gateway IP range
     - 198.51.100.50/32  # Specific firewall IP
   ```

3. **Proxy Configuration (On-Prem Applications):**
   ```properties
   # Kafka client configuration
   bootstrap.servers=pkc-xxxxx.us-east-1.aws.confluent.cloud:9092
   security.protocol=SASL_SSL
   sasl.mechanism=PLAIN
   sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule \
     required username="${KAFKA_API_KEY}" password="${KAFKA_API_SECRET}";

   # If corporate proxy required for internet access:
   # Set JVM properties
   -Dhttps.proxyHost=corporate-proxy.company.com
   -Dhttps.proxyPort=8080
   ```

**Benefits:**
- **Simplest Implementation:** Leverages existing internet connectivity
- **No Infrastructure Changes:** No VPN or Direct Connect required
- **Low Cost:** No additional network circuits

**Considerations:**
- **Corporate Proxy Compatibility:** Kafka protocol over TLS must pass through proxy
  - Some proxies block non-HTTP protocols
  - Requires proxy that supports TCP tunneling (CONNECT method)
- **Internet Dependency:** Connectivity depends on internet availability
- **Latency:** 20-50ms typical for internet routing (acceptable for most use cases)
- **Security Approval:** Security team must approve internet-based data egress

**Option B: AWS Direct Connect + Transit Gateway (Post-MVP)**

For on-prem applications with strict "no internet" requirements:

```
On-Prem Application
    │
    ▼
On-Prem Router
    │
    │ AWS Direct Connect (Dedicated 1Gbps or 10Gbps connection)
    │
    ▼
AWS Direct Connect Gateway
    │
    ▼
AWS Transit Gateway (Shared Services)
    │
    ├──> Platform Account VPC
    │      │
    │      └──> EEMP API Services (Private ALB)
    │
    └──> Confluent Cloud (via PrivateLink)
```

**Configuration:**

1. **Direct Connect Setup:**
   - 1 Gbps Dedicated Connection: $0.30/hour port + data transfer
   - Virtual Interface (VIF) to Transit Gateway

2. **Transit Gateway:**
   - Central hub connecting on-prem, Platform Account, and Confluent PrivateLink
   - Route tables for traffic steering

3. **Network Flow:**
   ```
   On-Prem App → Direct Connect → Transit Gateway → PrivateLink → Confluent Cloud
   ```

**Benefits:**
- **Private Network:** No internet exposure
- **Predictable Latency:** 5-15ms typical
- **High Bandwidth:** Dedicated 1-10 Gbps capacity
- **Compliance:** Meets strictest security requirements

**Costs:**
- Direct Connect Port: ~$216/month (1 Gbps)
- Transit Gateway: $0.05/hour per attachment = ~$36/month
- Data Processing: $0.02/GB
- **Total:** ~$300-500/month + usage

**Complexity:**
- Requires network engineering effort (2-4 weeks setup)
- Ongoing management of Direct Connect circuit
- Coordination with network team

**Recommendation:** Only implement if on-prem subscribers have strict regulatory requirements prohibiting internet connectivity.

---

#### Authentication and Authorization Across Accounts

**Cross-Account Identity Management:**

**Challenge:** How do applications in different AWS accounts authenticate to EEMP APIs and Confluent Cloud?

**Solution 1: API Key-Based Authentication (MVP)**

```
┌────────────────────────────────────────────────────────────────┐
│  Subscription Flow (Cross-Account)                             │
└────────────────────────────────────────────────────────────────┘

1. Application Owner (in any AWS account)
    │
    │ Authenticate to EEMP Portal (OAuth/OIDC SSO)
    │
    ▼
2. EEMP Event Catalog Portal
    │
    │ POST /api/v1/subscriptions
    │ { "application_id": "notifications-service",
    │   "event_types": ["user.created"] }
    │
    ▼
3. EEMP Subscription Service
    │
    │ OPA policy evaluation
    │ ├─> APPROVED
    │ │
    │ ├─> Create Confluent Cloud API Key for subscriber
    │ │   - API Key: XXXXXXXXXXX
    │ │   - API Secret: YYYYYYYYYYYY
    │ │
    │ └─> Store in Secrets Manager (Platform Account)
    │
    ▼
4. Return credentials to Application Owner (ONE TIME ONLY)
    {
      "kafka_api_key": "XXXXXXXXXXX",
      "kafka_api_secret": "YYYYYYYYYYYY",
      "bootstrap_servers": "pkc-xxx.aws.confluent.cloud:9092"
    }
    │
    ▼
5. Application Owner stores credentials in their account
    - AWS Secrets Manager (Application Account 222222222222)
    - Secret Name: eemp/kafka-credentials
    - Secret Value: { api_key, api_secret }
    │
    ▼
6. Application reads from Secrets Manager and connects to Kafka
```

**Key Points:**
- **Account Isolation:** Each application account manages its own Kafka credentials in its own Secrets Manager
- **No Cross-Account IAM Roles:** Simplifies security model (no trust relationships needed)
- **Credential Rotation:** EEMP manages rotation, notifies subscribers to update their Secrets Manager

**Solution 2: Cross-Account IAM Roles (Post-MVP Alternative)**

For AWS-based publishers/subscribers only:

```yaml
# In Application Account (222222222222)
AssumeRolePolicy:
  Version: "2012-10-17"
  Statement:
    - Effect: Allow
      Principal:
        AWS: "arn:aws:iam::111111111111:role/EEMPSubscriptionService"
      Action: "sts:AssumeRole"
      Condition:
        StringEquals:
          "sts:ExternalId": "unique-subscriber-id-12345"

# Application retrieves credentials via AssumeRole
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/EEMPKafkaSubscriber \
  --role-session-name kafka-consumer
```

**Benefits:**
- Temporary credentials (auto-rotating)
- Leverage AWS IAM audit trail

**Drawbacks:**
- Only works for AWS-based subscribers (excludes on-prem)
- Increased complexity (IAM role setup per subscriber)
- Doesn't integrate with Confluent Cloud authentication

**Recommendation:** Stick with API key-based approach for MVP (works across AWS accounts and on-prem).

---

#### Network Security and Access Control

**Firewall and Security Group Rules:**

**Platform Account (EEMP Services):**

```yaml
# ALB Security Group (for EEMP APIs)
Ingress:
  - Port: 443 (HTTPS)
    Source: 0.0.0.0/0  # Public internet (protected by OAuth + WAF)
    # Alternative: Restrict to known corporate IP ranges

# ECS Service Security Group
Ingress:
  - Port: 8080 (HTTP from ALB)
    Source: ALB Security Group

Egress:
  - Port: 443 (HTTPS to Confluent Cloud)
    Destination: 0.0.0.0/0
  - Port: 9092 (Kafka to Confluent Cloud)
    Destination: 0.0.0.0/0
  - Port: 5432 (PostgreSQL)
    Destination: RDS Security Group
```

**Application Accounts (Publishers/Subscribers):**

```yaml
# Application Service Security Group
Egress:
  - Port: 443 (HTTPS to EEMP APIs)
    Destination: 0.0.0.0/0  # Or specific IP ranges
  - Port: 9092 (Kafka to Confluent Cloud)
    Destination: 0.0.0.0/0  # Or Confluent Cloud IP ranges
```

**On-Premises Firewall Rules:**

```
Outbound Rules:
  - Allow TCP 443 to platform-api.company.com (EEMP APIs)
  - Allow TCP 9092 to *.aws.confluent.cloud (Kafka)

IP Allowlisting:
  - Configure Confluent Cloud to only accept connections from:
    - Corporate NAT gateway IPs: 203.0.113.0/24
    - Known AWS NAT gateway IPs (if restricting AWS accounts)
```

---

#### Monitoring and Observability Across Accounts

**Challenge:** How to monitor publishers/subscribers in different AWS accounts?

**Solution: Distributed Tracing with Correlation IDs**

```
Publisher (Account 222222222222)
    │
    │ 1. Publish event with correlation_id
    │    Event: {
    │      "correlation_id": "trace-abc-123",
    │      "event_type": "user.created",
    │      "source": { "account_id": "222222222222" }
    │    }
    │
    ▼
Confluent Cloud Kafka
    │
    │ 2. Event flows to subscriber
    │
    ▼
Subscriber (Account 333333333333)
    │
    │ 3. Consume event, propagate correlation_id
    │    Log: { "correlation_id": "trace-abc-123", "action": "processed" }
```

**Centralized Audit Trail (Platform Account):**

All events logged to EEMP Audit Service include:
- `source.account_id` (which AWS account published)
- `subscriber.account_id` (which AWS account consumed)
- `correlation_id` (for distributed tracing)

**Per-Account Monitoring:**

Each application account maintains its own:
- CloudWatch Logs (application-specific logs)
- CloudWatch Metrics (Kafka consumer lag, error rates)
- X-Ray traces (if using AWS X-Ray)

**Platform-Level Dashboards:**

EEMP provides aggregated views:
- Events published/consumed per AWS account
- Cross-account data flows
- Policy decisions by account

---

#### Cost Allocation and Chargeback

**Challenge:** How to allocate Confluent Cloud costs to different application teams/accounts?

**Solution: Usage-Based Chargeback**

1. **Track Usage per Subscriber:**
   ```sql
   -- Query from EEMP audit trail
   SELECT
     subscriber_account_id,
     event_type,
     COUNT(*) as events_consumed,
     SUM(event_size_bytes) as total_bytes
   FROM audit_events
   WHERE timestamp >= '2025-12-01'
   GROUP BY subscriber_account_id, event_type;
   ```

2. **Calculate Cost Allocation:**
   ```
   Total Confluent Cloud Cost: $1,200/month

   Allocation by data volume:
   - Account 222222222222: 40% of data → $480/month
   - Account 333333333333: 35% of data → $420/month
   - Account 444444444444: 25% of data → $300/month
   ```

3. **AWS Cost Allocation Tags:**
   ```yaml
   # Tag Confluent Cloud invoice line items with cost center
   # Use AWS Cost Explorer to create chargeback reports
   Tags:
     - Key: CostCenter
       Value: Application-Team-A
   ```

**Alternative:** Flat-fee model where platform costs are absorbed by platform team (common for shared services).

---

### 3.2 Schema Registry Integration

**Architecture:**

```
┌────────────────────────────────────────────────────────────────┐
│                  Confluent Schema Registry                      │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Subject Naming Strategy: TopicNameStrategy                    │
│  Subject: platform.events-value                                │
│                                                                 │
│  Schema Format: Avro (JSON Schema for future consideration)   │
│  Compatibility Mode: BACKWARD (default)                        │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Schema Versioning Model                                  │ │
│  │                                                            │ │
│  │  Schema ID: Auto-assigned by registry                     │ │
│  │  Version: Auto-incremented on registration                │ │
│  │  Compatibility: Backward compatibility enforced            │ │
│  │                                                            │ │
│  │  Example:                                                  │ │
│  │    user.created (v1) → Schema ID: 1001                    │ │
│  │    user.created (v2) → Schema ID: 1002 (backward compat)  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Schema Storage:                                               │
│  - Primary: Confluent Cloud Schema Registry                   │
│  - Backup: S3 (daily archive for disaster recovery)           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

**Key Decisions:**

1. **Schema Format: Avro (MVP)**
   - **Rationale:** Industry standard for Kafka, efficient binary serialization, strong typing, backward compatibility enforcement
   - **Alternative Considered:** JSON Schema (more developer-friendly but less efficient)
   - **Migration Path:** Support JSON Schema in future for specific use cases

2. **Subject Naming Strategy: TopicNameStrategy**
   - **Rationale:** All events on same topic share a schema namespace
   - **Implication:** Event envelope includes `event_type` discriminator for routing
   - **Alternative:** TopicRecordNameStrategy (one subject per event type) - considered for post-MVP

3. **Compatibility Mode: BACKWARD**
   - **Rationale:** Consumers using old schema can read new data (fields added are optional)
   - **Enforcement:** Schema Registry rejects incompatible schema registrations
   - **Validation:** Publisher Service validates compatibility before registration

**Schema Registration Workflow:**

```
Publisher Team
    │
    ▼
Define Avro Schema (*.avsc)
    │
    ▼
Register via Terraform/API ──> Publisher Service
    │                               │
    │                               ▼
    │                        Validate Schema Structure
    │                        (Required fields, naming conventions)
    │                               │
    │                               ▼
    │                        Check Backward Compatibility
    │                        (Schema Registry API)
    │                               │
    │                               ├──> REJECT (incompatible)
    │                               │
    │                               ▼
    │                        Register in Schema Registry
    │                        (Get Schema ID)
    │                               │
    │                               ▼
    │                        Store Metadata in Catalog DB
    │                        (Event Type, Publisher, Version, Schema ID)
    │                               │
    │                               ▼
    └───────────────────────────> SUCCESS (Schema ID returned)
```

### 3.3 Confluent Cloud Features Utilized

| Feature | Purpose | Configuration |
|---------|---------|---------------|
| **Schema Registry** | Schema validation, versioning, compatibility | Enabled, BACKWARD compatibility |
| **Stream Governance** | Audit metrics, lineage tracking (future) | Enabled for compliance reporting |
| **Cluster Linking** | Future multi-region replication | Not enabled in MVP |
| **ksqlDB** | Future real-time analytics, filtering | Not enabled in MVP |
| **Confluent Metrics** | Cluster health, throughput monitoring | Enabled, exported to CloudWatch |
| **Role-Based Access Control (RBAC)** | Confluent Cloud resource permissions | Configured for service accounts |

**Authentication & Authorization:**

```yaml
confluent_cloud_access:
  authentication: API Key (per service account)

  service_accounts:
    - name: eemp-publisher-service
      permissions: [WRITE to platform.events, READ from Schema Registry]

    - name: eemp-consumer-service
      permissions: [READ from platform.events]

    - name: eemp-catalog-service
      permissions: [READ from Schema Registry, READ cluster metadata]

  api_key_rotation: 90 days (automated via Secrets Manager)
  secrets_storage: AWS Secrets Manager
```

---

## 4. Open Policy Agent (OPA) Integration

### 4.1 OPA Architecture

**Integration Model: Sidecar + Centralized Policy Management**

```
┌─────────────────────────────────────────────────────────────────┐
│                    OPA Policy Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Centralized Policy Repository (Git)                       │ │
│  │                                                             │ │
│  │  - eemp_policies/                                          │ │
│  │    - subscription_access.rego (ABAC rules)                │ │
│  │    - data_classification.rego (PII, sensitive data)       │ │
│  │    - publisher_authorization.rego                         │ │
│  │    - audit_enrichment_rules.rego                          │ │
│  │                                                             │ │
│  │  Policy Lifecycle: GitOps (PR → Review → Merge → Deploy)  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           │                                      │
│                           ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  OPA Bundle Server (Existing Infrastructure)               │ │
│  │  - Serves compiled policy bundles                          │ │
│  │  - Versioned policy distribution                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           │                                      │
│            ┌──────────────┴──────────────┐                      │
│            ▼                              ▼                      │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │  OPA Sidecar     │          │  OPA Sidecar     │            │
│  │  (Subscription   │          │  (Publisher      │            │
│  │   Service)       │          │   Service)       │            │
│  │                  │          │                  │            │
│  │  - Local cache   │          │  - Local cache   │            │
│  │  - Bundle sync   │          │  - Bundle sync   │            │
│  │  - Policy eval   │          │  - Policy eval   │            │
│  └──────────────────┘          └──────────────────┘            │
│           ▲                              ▲                       │
│           │ Policy Decision              │                       │
│           │                              │                       │
│  ┌────────┴────────┐          ┌─────────┴────────┐             │
│  │  Subscription   │          │  Publisher       │             │
│  │  Service (ECS)  │          │  Service (ECS)   │             │
│  └─────────────────┘          └──────────────────┘             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Key Decisions:**

1. **Deployment Model: Sidecar Pattern**
   - **Rationale:** Low-latency policy evaluation (local calls), resilience (no network dependency for policy decisions)
   - **Alternative Considered:** Centralized OPA cluster (higher latency, network dependency)
   - **Implementation:** OPA container runs alongside each service container in ECS task

2. **Policy Distribution: Bundle Server**
   - **Rationale:** Leverages existing OPA infrastructure, version-controlled policy updates
   - **Update Frequency:** Poll every 60 seconds for policy bundle updates
   - **Fallback:** If bundle server unreachable, use last cached policy (logged as warning)

### 4.2 ABAC Policy Model

**Subscription Authorization Policy:**

```rego
package eemp.subscription

# Default deny
default allow = false

# Allow subscription if all conditions met
allow {
    # Check application has clearance for data classification
    input.subscriber.data_classification_clearance >= data.events[input.event_type].data_classification

    # Check tenant isolation (subscriber can only access own tenant's events)
    input.subscriber.tenant_id == data.events[input.event_type].tenant_id

    # Check no cross-border restrictions
    not cross_border_violation

    # Check application is active and not suspended
    input.subscriber.status == "active"
}

# Cross-border data flow check
cross_border_violation {
    data.events[input.event_type].data_residency_requirements[_] == "no_cross_border"
    input.subscriber.deployment_region != data.events[input.event_type].region
}

# Audit context enrichment (always executed)
audit_context = {
    "policy_decision": allow,
    "evaluation_timestamp": time.now_ns(),
    "policy_version": data.eemp.policy_version,
    "rules_evaluated": [
        {"rule": "data_classification", "result": data_classification_check},
        {"rule": "tenant_isolation", "result": tenant_isolation_check},
        {"rule": "cross_border", "result": !cross_border_violation}
    ]
}

data_classification_check {
    input.subscriber.data_classification_clearance >= data.events[input.event_type].data_classification
}

tenant_isolation_check {
    input.subscriber.tenant_id == data.events[input.event_type].tenant_id
}
```

**Policy Input Structure:**

```json
{
  "subscriber": {
    "application_id": "notifications-service",
    "tenant_id": "tenant-001",
    "data_classification_clearance": 3,
    "deployment_region": "us-east-1",
    "status": "active"
  },
  "event_type": "user.profile.updated",
  "requested_at": "2025-12-10T10:30:00Z"
}
```

**Policy Data (External Data):**

```json
{
  "events": {
    "user.profile.updated": {
      "data_classification": 2,
      "tenant_id": "tenant-001",
      "data_residency_requirements": [],
      "pii_indicator": true,
      "region": "us-east-1"
    }
  },
  "eemp": {
    "policy_version": "1.2.0"
  }
}
```

### 4.3 Policy Decision Audit Trail

**Every OPA decision is logged with enriched context:**

```json
{
  "timestamp": "2025-12-10T10:30:00.123Z",
  "decision_id": "uuid-1234",
  "policy_decision": true,
  "policy_version": "1.2.0",
  "input": {
    "subscriber": {
      "application_id": "notifications-service",
      "tenant_id": "tenant-001"
    },
    "event_type": "user.profile.updated"
  },
  "rules_evaluated": [
    {"rule": "data_classification", "result": true},
    {"rule": "tenant_isolation", "result": true},
    {"rule": "cross_border", "result": true}
  ],
  "evaluation_duration_ms": 5
}
```

**Storage:**
- **Real-time:** PostgreSQL `policy_decisions` table (indexed by timestamp, application_id, event_type)
- **Long-term Archive:** S3 (immutable, partitioned by date for compliance queries)

---

## 5. Governance Layer Architecture

### 5.1 Publisher Service

**Responsibilities:**
1. Schema validation before publication
2. Event enrichment (correlation_id, actor, source metadata)
3. Kafka publishing with schema registry integration
4. Audit logging

**API Design:**

```typescript
// POST /api/v1/events/publish
interface PublishEventRequest {
  event_type: string;              // e.g., "user.created"
  correlation_id?: string;         // Optional, generated if not provided
  payload: Record<string, any>;    // Event-specific data
  metadata?: {
    actor?: string;                // User or service that triggered event
    source?: string;               // Source system
  };
}

interface PublishEventResponse {
  success: boolean;
  event_id: string;                // Generated UUID
  kafka_offset: number;            // Kafka partition offset
  schema_id: number;               // Schema Registry ID used
  timestamp: string;
}
```

**Publishing Flow:**

```
Publisher Application
    │
    ▼
POST /api/v1/events/publish
    │
    ▼
Publisher Service
    │
    ├──> 1. Validate event_type exists in catalog
    │         │
    │         ├──> NOT FOUND → 400 Bad Request
    │         │
    │         ▼
    ├──> 2. Retrieve schema from Schema Registry
    │         │
    │         ▼
    ├──> 3. Validate payload against schema
    │         │
    │         ├──> INVALID → 400 Bad Request (schema errors)
    │         │
    │         ▼
    ├──> 4. Enrich event with standard envelope
    │         │  - event_id (UUID)
    │         │  - timestamp (ISO 8601)
    │         │  - correlation_id (passed or generated)
    │         │  - actor, source (from metadata or auth context)
    │         │
    │         ▼
    ├──> 5. Serialize with Avro (using schema_id)
    │         │
    │         ▼
    ├──> 6. Publish to Kafka (platform.events topic)
    │         │
    │         ├──> FAILURE → 500 Internal Error (retry logic)
    │         │
    │         ▼
    └──> 7. Log audit entry (event published)
         │
         ▼
    200 OK (event_id, offset, schema_id)
```

**Schema Validation:**

```typescript
// Schema validation using Confluent Schema Registry client
async function validateEvent(
  eventType: string,
  payload: Record<string, any>
): Promise<{ valid: boolean; schemaId?: number; errors?: string[] }> {

  // 1. Get latest schema for event_type from local cache or Schema Registry
  const schema = await getSchema(eventType);

  if (!schema) {
    return { valid: false, errors: [`Unknown event type: ${eventType}`] };
  }

  // 2. Validate payload against Avro schema
  const validationResult = avro.validate(payload, schema.avroSchema);

  if (!validationResult.valid) {
    return {
      valid: false,
      errors: validationResult.errors.map(e => e.message)
    };
  }

  return { valid: true, schemaId: schema.id };
}
```

**Event Enrichment:**

```typescript
interface EnrichedEvent {
  // Standard envelope (added by Publisher Service)
  event_id: string;           // UUID v4
  event_type: string;         // From request
  timestamp: string;          // ISO 8601 UTC
  correlation_id: string;     // From request or generated
  schema_version: number;     // Schema Registry version

  // Metadata (enriched from context)
  actor: {
    type: string;            // "user" | "service" | "system"
    id: string;              // User ID or service account ID
  };
  source: {
    system: string;          // Source application/service
    region: string;          // AWS region
    environment: string;     // "prod" | "staging" | "dev"
  };

  // Event payload (from publisher)
  payload: Record<string, any>;
}
```

### 5.2 Subscription Service

**Responsibilities:**
1. Subscription request intake
2. OPA policy evaluation
3. Kafka consumer group provisioning
4. Access credential management
5. Subscription lifecycle management

**API Design:**

```typescript
// POST /api/v1/subscriptions
interface CreateSubscriptionRequest {
  application_id: string;           // Subscriber identity
  event_types: string[];            // Event types to subscribe to
  consumer_group_id?: string;       // Optional, generated if not provided
  metadata?: {
    team: string;
    purpose: string;
  };
}

interface CreateSubscriptionResponse {
  subscription_id: string;
  status: "approved" | "denied" | "pending_manual_review";
  consumer_group_id: string;
  connection_details?: {            // Only if approved
    bootstrap_servers: string;
    topic: string;
    api_key: string;                // Confluent Cloud API key (encrypted)
    api_secret: string;             // Encrypted, rotate after first retrieval
    schema_registry_url: string;
  };
  denied_event_types?: {            // If partially denied
    event_type: string;
    reason: string;
  }[];
  opa_decision_id: string;          // Audit trail reference
}

// GET /api/v1/subscriptions/{subscription_id}
interface GetSubscriptionResponse {
  subscription_id: string;
  application_id: string;
  event_types: string[];
  status: "active" | "suspended" | "revoked";
  created_at: string;
  consumer_group_id: string;
  last_consumed_at?: string;
}

// DELETE /api/v1/subscriptions/{subscription_id}
// Revokes access, removes Kafka ACLs
```

**Subscription Workflow:**

```
Consumer Application
    │
    ▼
POST /api/v1/subscriptions
    │
    ▼
Subscription Service
    │
    ├──> For each event_type in request:
    │      │
    │      ▼
    │    1. Check event_type exists in catalog
    │         │
    │         ├──> NOT FOUND → Skip (add to denied_event_types)
    │         │
    │         ▼
    │    2. Retrieve event metadata (data_classification, tenant_id, etc.)
    │         │
    │         ▼
    │    3. Build OPA policy input
    │         │  {
    │         │    "subscriber": { application_id, tenant_id, clearance, ... },
    │         │    "event_type": "user.created"
    │         │  }
    │         │
    │         ▼
    │    4. Call OPA sidecar: POST /v1/data/eemp/subscription/allow
    │         │
    │         ├──> DENIED → Add to denied_event_types
    │         │
    │         ▼
    │    5. APPROVED → Add to approved_event_types
    │
    ├──> If all event_types denied:
    │         └──> Return 403 Forbidden (with OPA decision_id)
    │
    ├──> If at least one approved:
    │      │
    │      ▼
    │    6. Generate Confluent Cloud API key for consumer
    │         │  (via Confluent Cloud API)
    │         │
    │         ▼
    │    7. Store subscription in database
    │         │  - subscription_id (UUID)
    │         │  - application_id
    │         │  - approved event_types
    │         │  - consumer_group_id
    │         │  - api_key reference (Secrets Manager)
    │         │  - status: "active"
    │         │
    │         ▼
    │    8. Log audit entry (subscription created)
    │         │
    │         ▼
    └──> 9. Return 201 Created
              - connection_details (API key, bootstrap servers, topic)
              - denied_event_types (if partial approval)
              - opa_decision_id
```

**OPA Policy Evaluation:**

```typescript
async function evaluateSubscriptionAccess(
  subscriberId: string,
  eventType: string
): Promise<{ allowed: boolean; decision: any; decision_id: string }> {

  // 1. Retrieve subscriber metadata from identity service
  const subscriber = await getSubscriberMetadata(subscriberId);

  // 2. Retrieve event metadata from catalog
  const event = await getEventMetadata(eventType);

  // 3. Build OPA input
  const opaInput = {
    subscriber: {
      application_id: subscriber.id,
      tenant_id: subscriber.tenant_id,
      data_classification_clearance: subscriber.clearance_level,
      deployment_region: subscriber.region,
      status: subscriber.status
    },
    event_type: eventType,
    requested_at: new Date().toISOString()
  };

  // 4. Call OPA sidecar
  const response = await fetch('http://localhost:8181/v1/data/eemp/subscription', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ input: opaInput })
  });

  const result = await response.json();

  // 5. Extract decision and audit context
  const allowed = result.result.allow;
  const auditContext = result.result.audit_context;
  const decision_id = uuid.v4();

  // 6. Log policy decision to audit trail
  await logPolicyDecision({
    decision_id,
    timestamp: new Date().toISOString(),
    policy_decision: allowed,
    policy_version: auditContext.policy_version,
    input: opaInput,
    rules_evaluated: auditContext.rules_evaluated
  });

  return { allowed, decision: result.result, decision_id };
}
```

### 5.3 Catalog Service

**Responsibilities:**
1. Event type registry (CRUD operations)
2. Schema metadata management
3. Publisher/subscriber relationship tracking
4. Search and discovery API

**Data Model:**

```sql
-- Event Types Table
CREATE TABLE event_types (
  event_type_id UUID PRIMARY KEY,
  event_type VARCHAR(255) UNIQUE NOT NULL,  -- e.g., "user.created"
  display_name VARCHAR(255),
  description TEXT,
  data_classification INT,  -- 1=Public, 2=Internal, 3=Confidential, 4=Restricted
  schema_id INT,  -- Current active schema version (Schema Registry ID)
  publisher_id VARCHAR(255),  -- Application/service that publishes
  status VARCHAR(50),  -- "active" | "deprecated" | "sunset"
  deprecated_at TIMESTAMP,
  sunset_date TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_event_type ON event_types(event_type);
CREATE INDEX idx_publisher ON event_types(publisher_id);
CREATE INDEX idx_status ON event_types(status);

-- Schemas Table (mirrors Schema Registry)
CREATE TABLE schemas (
  schema_id INT PRIMARY KEY,  -- Schema Registry ID
  event_type_id UUID REFERENCES event_types(event_type_id),
  version INT,
  avro_schema JSONB,
  compatibility_mode VARCHAR(50),
  registered_at TIMESTAMP DEFAULT NOW(),
  registered_by VARCHAR(255)
);

CREATE INDEX idx_event_type_version ON schemas(event_type_id, version);

-- Subscriptions Table
CREATE TABLE subscriptions (
  subscription_id UUID PRIMARY KEY,
  application_id VARCHAR(255) NOT NULL,
  event_types TEXT[],  -- Array of event type strings
  consumer_group_id VARCHAR(255),
  status VARCHAR(50),  -- "active" | "suspended" | "revoked"
  api_key_secret_arn VARCHAR(500),  -- AWS Secrets Manager ARN
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_consumed_at TIMESTAMP
);

CREATE INDEX idx_application ON subscriptions(application_id);
CREATE INDEX idx_status_subs ON subscriptions(status);

-- Subscription Event Types (many-to-many)
CREATE TABLE subscription_event_types (
  subscription_id UUID REFERENCES subscriptions(subscription_id),
  event_type_id UUID REFERENCES event_types(event_type_id),
  subscribed_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (subscription_id, event_type_id)
);

-- Audit Decisions Table (Policy decisions index)
CREATE TABLE policy_decisions (
  decision_id UUID PRIMARY KEY,
  timestamp TIMESTAMP NOT NULL,
  policy_decision BOOLEAN NOT NULL,
  policy_version VARCHAR(50),
  subscriber_application_id VARCHAR(255),
  event_type VARCHAR(255),
  input JSONB,
  rules_evaluated JSONB,
  evaluation_duration_ms INT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_timestamp ON policy_decisions(timestamp);
CREATE INDEX idx_subscriber ON policy_decisions(subscriber_application_id);
CREATE INDEX idx_event_type_decision ON policy_decisions(event_type);
```

**API Design:**

```typescript
// GET /api/v1/catalog/events
interface ListEventsRequest {
  search?: string;              // Search event_type or description
  data_classification?: number; // Filter by classification
  publisher_id?: string;        // Filter by publisher
  status?: string;              // Filter by status
  limit?: number;
  offset?: number;
}

interface ListEventsResponse {
  events: EventType[];
  total: number;
  limit: number;
  offset: number;
}

interface EventType {
  event_type: string;
  display_name: string;
  description: string;
  data_classification: number;
  schema_version: number;
  schema_id: number;
  publisher_id: string;
  status: string;
  subscriber_count: number;
  created_at: string;
}

// GET /api/v1/catalog/events/{event_type}
interface GetEventDetailsResponse {
  event_type: string;
  display_name: string;
  description: string;
  data_classification: number;
  schema_id: number;
  schema_version: number;
  schema: {
    avro_schema: object;      // Full Avro schema definition
    sample_payload: object;   // Example event payload
  };
  publisher: {
    id: string;
    name: string;
  };
  subscribers: {
    application_id: string;
    subscribed_at: string;
  }[];
  status: string;
  deprecated_at?: string;
  sunset_date?: string;
  created_at: string;
  updated_at: string;
}

// GET /api/v1/catalog/schemas/{event_type}
interface GetSchemasResponse {
  event_type: string;
  schemas: SchemaVersion[];
}

interface SchemaVersion {
  version: number;
  schema_id: number;
  avro_schema: object;
  compatibility_mode: string;
  registered_at: string;
  registered_by: string;
}
```

### 5.4 Audit Service

**Responsibilities:**
1. Enrichment pipeline for audit events
2. Immutable audit trail storage
3. Query API for compliance reporting
4. Export capabilities for regulatory examination

**Audit Data Flow:**

```
Event Source (Publisher, Subscription, OPA)
    │
    ▼
Kinesis Data Stream (audit-events)
    │
    ▼
Audit Enrichment Lambda
    │
    ├──> Enrich with context:
    │      - Actor details (from IAM/directory)
    │      - Source system metadata
    │      - Correlation_id propagation
    │      - Timestamp normalization
    │
    ▼
Write to dual storage:
    │
    ├──> PostgreSQL (audit_events table)
    │      - Indexed for fast queries
    │      - 90-day retention
    │
    └──> S3 (immutable archive)
         - Partitioned by date: s3://audit-trail/year=2025/month=12/day=10/
         - Encrypted at rest (SSE-KMS)
         - Lifecycle: Glacier after 1 year, retain 7 years
```

**Audit Event Schema:**

```typescript
interface AuditEvent {
  // Core identification
  audit_id: string;                  // UUID
  event_type: string;                // "event.published" | "subscription.created" | "policy.evaluated"
  timestamp: string;                 // ISO 8601 UTC
  correlation_id: string;            // For tracing related events

  // Actor information (enriched)
  actor: {
    type: string;                    // "user" | "service" | "system"
    id: string;                      // User ID or service account
    name?: string;                   // Human-readable name
    ip_address?: string;             // Source IP
  };

  // Source system (enriched)
  source: {
    system: string;                  // Application/service name
    region: string;                  // AWS region
    environment: string;             // "prod" | "staging"
    version?: string;                // Application version
  };

  // Event-specific payload
  payload: {
    // For "event.published":
    published_event_type?: string;
    published_event_id?: string;
    kafka_offset?: number;
    schema_id?: number;

    // For "subscription.created":
    subscription_id?: string;
    subscriber_application_id?: string;
    subscribed_event_types?: string[];

    // For "policy.evaluated":
    policy_decision?: boolean;
    policy_version?: string;
    rules_evaluated?: any[];
    decision_id?: string;
  };

  // Compliance metadata
  compliance: {
    data_classification?: number;
    pii_indicator?: boolean;
    regulatory_scope?: string[];    // ["SOX", "GDPR", etc.]
  };
}
```

**Query API:**

```typescript
// POST /api/v1/audit/query
interface AuditQueryRequest {
  // Time range (required)
  start_time: string;               // ISO 8601
  end_time: string;                 // ISO 8601

  // Filters (optional)
  event_types?: string[];           // Filter by audit event types
  actor_id?: string;                // Filter by actor
  correlation_id?: string;          // Trace all related events
  published_event_type?: string;    // Filter by published event type
  subscriber_application_id?: string;

  // Pagination
  limit?: number;
  offset?: number;
}

interface AuditQueryResponse {
  events: AuditEvent[];
  total: number;
  limit: number;
  offset: number;
}

// POST /api/v1/audit/export
interface AuditExportRequest {
  start_time: string;
  end_time: string;
  format: "csv" | "json" | "parquet";
  filters?: AuditQueryRequest;
}

interface AuditExportResponse {
  export_id: string;
  status: "processing";
  estimated_completion: string;
}

// GET /api/v1/audit/export/{export_id}
interface AuditExportStatusResponse {
  export_id: string;
  status: "processing" | "completed" | "failed";
  download_url?: string;            // Pre-signed S3 URL (expires in 1 hour)
  created_at: string;
  completed_at?: string;
}
```

---

## 6. Data Flow Patterns

### 6.1 Event Publishing Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Event Publishing Flow                              │
└─────────────────────────────────────────────────────────────────────────┘

Publisher Application (e.g., User Profile Service)
    │
    │ POST /api/v1/events/publish
    │ {
    │   "event_type": "user.created",
    │   "payload": { "user_id": "123", "email": "user@example.com" }
    │ }
    │
    ▼
┌───────────────────────────────────────┐
│      Publisher Service (ECS)          │
│                                       │
│  1. Authenticate request (JWT/mTLS)  │
│  2. Validate event_type exists       │
│  3. Get schema from Schema Registry  │
│  4. Validate payload against schema  │
│  5. Enrich event envelope:           │
│     - event_id (UUID)                │
│     - timestamp                      │
│     - correlation_id                 │
│     - actor, source                  │
│  6. Serialize with Avro              │
└───────────────────────────────────────┘
    │
    │ Avro-serialized event
    │
    ▼
┌───────────────────────────────────────┐
│   Confluent Cloud Kafka Cluster      │
│                                       │
│   Topic: platform.events              │
│   Partition: hash(event_type) % 12   │
│                                       │
│   Event stored with:                  │
│   - Key: event_id                     │
│   - Value: Avro payload (schema_id)  │
│   - Headers: correlation_id, source  │
└───────────────────────────────────────┘
    │
    │ Publish success (offset returned)
    │
    ▼
┌───────────────────────────────────────┐
│      Publisher Service                │
│                                       │
│  7. Log audit event:                  │
│     - Send to Kinesis audit stream   │
│     - Event: "event.published"       │
│     - Metadata: event_id, offset     │
│                                       │
│  8. Return 200 OK to caller           │
└───────────────────────────────────────┘
    │
    ▼
Publisher Application receives:
{
  "event_id": "uuid-1234",
  "kafka_offset": 56789,
  "schema_id": 1001,
  "timestamp": "2025-12-10T10:30:00Z"
}
```

### 6.2 Event Consumption Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Event Consumption Flow                             │
└─────────────────────────────────────────────────────────────────────────┘

Consumer Application (e.g., Notifications Service)
    │
    │ Kafka Consumer (using Confluent Kafka Client)
    │ - bootstrap.servers: <Confluent Cloud>
    │ - security.protocol: SASL_SSL
    │ - sasl.mechanism: PLAIN
    │ - sasl.username: <API Key>
    │ - sasl.password: <API Secret>
    │ - group.id: notifications-service-consumer
    │ - auto.offset.reset: earliest
    │
    ▼
┌───────────────────────────────────────┐
│   Confluent Cloud Kafka Cluster      │
│                                       │
│   Topic: platform.events              │
│   Consumer Group: notifications-svc   │
│                                       │
│   Consumer assigned partitions:       │
│   [0, 3, 6, 9] (example distribution) │
└───────────────────────────────────────┘
    │
    │ Consume messages (poll loop)
    │
    ▼
┌───────────────────────────────────────┐
│      Consumer Application             │
│                                       │
│  1. Receive Avro-serialized message  │
│  2. Fetch schema from Schema Registry│
│  3. Deserialize using schema_id      │
│  4. Extract event_type from envelope │
│  5. CLIENT-SIDE FILTERING:            │
│     - Check if event_type in          │
│       ["user.created", "user.updated"]│
│     - If NO MATCH → Skip, continue   │
│     - If MATCH → Process event       │
│  6. Business logic processing        │
│  7. Commit offset to Kafka           │
└───────────────────────────────────────┘
    │
    │ Event processed
    │
    ▼
Consumer Application updates:
- Database records
- Sends notifications
- Updates cache
- etc.
```

**Consumer SDK Filtering Example:**

```typescript
import { KafkaConsumer, SchemaRegistry } from '@eemp/consumer-sdk';

const consumer = new KafkaConsumer({
  bootstrapServers: process.env.KAFKA_BOOTSTRAP_SERVERS,
  apiKey: process.env.KAFKA_API_KEY,
  apiSecret: process.env.KAFKA_API_SECRET,
  consumerGroupId: 'notifications-service',
  schemaRegistryUrl: process.env.SCHEMA_REGISTRY_URL,

  // Filter configuration (client-side filtering)
  eventTypeFilter: ['user.created', 'user.updated', 'profile.updated']
});

consumer.on('event', async (event) => {
  // Only receives events matching eventTypeFilter
  console.log(`Received event: ${event.event_type}`);

  switch (event.event_type) {
    case 'user.created':
      await handleUserCreated(event.payload);
      break;
    case 'user.updated':
      await handleUserUpdated(event.payload);
      break;
    // ...
  }
});

consumer.start();
```

### 6.3 Subscription Request Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Subscription Request Flow                             │
└─────────────────────────────────────────────────────────────────────────┘

Developer (via Event Catalog Portal)
    │
    │ Browse catalog, find "user.created" event
    │ Click "Subscribe"
    │
    ▼
Event Catalog Portal (React SPA)
    │
    │ POST /api/v1/subscriptions
    │ {
    │   "application_id": "notifications-service",
    │   "event_types": ["user.created", "user.updated"]
    │ }
    │
    ▼
┌───────────────────────────────────────┐
│   Subscription Service (ECS)          │
│                                       │
│  For each event_type:                 │
│                                       │
│  1. Get event metadata from Catalog  │
│     - data_classification: 2         │
│     - tenant_id: "tenant-001"        │
│     - pii_indicator: true            │
│                                       │
│  2. Get subscriber metadata           │
│     - clearance_level: 3             │
│     - tenant_id: "tenant-001"        │
│     - region: "us-east-1"            │
│     - status: "active"               │
│                                       │
│  3. Build OPA policy input            │
└───────────────────────────────────────┘
    │
    │ POST /v1/data/eemp/subscription
    │ {
    │   "input": {
    │     "subscriber": { ... },
    │     "event_type": "user.created"
    │   }
    │ }
    │
    ▼
┌───────────────────────────────────────┐
│   OPA Sidecar (localhost:8181)        │
│                                       │
│  Evaluate policy:                     │
│  - Data classification check ✓        │
│  - Tenant isolation check ✓           │
│  - Cross-border check ✓               │
│                                       │
│  Result: ALLOW                        │
└───────────────────────────────────────┘
    │
    │ { "result": { "allow": true, "audit_context": {...} } }
    │
    ▼
┌───────────────────────────────────────┐
│   Subscription Service                │
│                                       │
│  4. Generate Confluent API key        │
│     - Call Confluent Cloud API        │
│     - Create service account          │
│     - Grant READ access to topic      │
│                                       │
│  5. Store API key in Secrets Manager  │
│     - ARN: arn:aws:secretsmanager:... │
│                                       │
│  6. Save subscription to database     │
│     - subscription_id (UUID)          │
│     - application_id                  │
│     - event_types: [...]              │
│     - api_key_secret_arn              │
│     - status: "active"                │
│                                       │
│  7. Log audit event:                  │
│     - Event: "subscription.created"  │
│     - OPA decision_id reference       │
│                                       │
│  8. Return 201 Created                │
└───────────────────────────────────────┘
    │
    │ 201 Created
    │ {
    │   "subscription_id": "uuid-5678",
    │   "status": "approved",
    │   "connection_details": {
    │     "bootstrap_servers": "pkc-xyz.aws.confluent.cloud:9092",
    │     "topic": "platform.events",
    │     "api_key": "***",
    │     "api_secret": "***",
    │     "schema_registry_url": "https://..."
    │   }
    │ }
    │
    ▼
Developer receives credentials
- Configures consumer application
- Deploys with KAFKA_API_KEY, KAFKA_API_SECRET
- Application starts consuming events
```

### 6.4 Schema Evolution Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Schema Evolution Flow                              │
└─────────────────────────────────────────────────────────────────────────┘

Publisher Team (User Profile Service)
    │
    │ Need to add "preferred_language" field to user.updated event
    │
    ▼
Update Avro Schema:
{
  "type": "record",
  "name": "UserUpdated",
  "fields": [
    {"name": "user_id", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "preferred_language", "type": ["null", "string"], "default": null}  // NEW FIELD
  ]
}
    │
    │ Via Terraform or API
    │
    ▼
┌───────────────────────────────────────┐
│      Publisher Service API            │
│  POST /api/v1/schemas/register        │
│                                       │
│  1. Validate schema structure         │
│     - Required fields present?        │
│     - Naming conventions followed?    │
│                                       │
│  2. Check backward compatibility      │
│     - Call Schema Registry API        │
│     - Compare with current version    │
│     - New field has default value? ✓  │
│                                       │
│  COMPATIBILITY CHECK: PASS            │
└───────────────────────────────────────┘
    │
    │ POST /subjects/platform.events-value/versions
    │ { "schema": "..." }
    │
    ▼
┌───────────────────────────────────────┐
│   Confluent Schema Registry           │
│                                       │
│  - Assign schema_id: 1002             │
│  - Version: 2                         │
│  - Store schema                       │
│  - Mark as active                     │
└───────────────────────────────────────┘
    │
    │ Schema registered successfully
    │ { "id": 1002 }
    │
    ▼
┌───────────────────────────────────────┐
│      Publisher Service                │
│                                       │
│  3. Update catalog metadata           │
│     - event_types.schema_id = 1002   │
│     - Insert into schemas table       │
│                                       │
│  4. Log audit event:                  │
│     - Event: "schema.registered"     │
│     - Old version: 1, New: 2          │
│                                       │
│  5. Return success to caller          │
└───────────────────────────────────────┘
    │
    ▼
Publisher Team updates application code:
- Publish events with new field
- Old consumers continue working (field is optional)
- New consumers can use new field
```

**Backward Compatibility Validation:**

```typescript
async function validateBackwardCompatibility(
  eventType: string,
  newSchema: object
): Promise<{ compatible: boolean; errors?: string[] }> {

  // 1. Get current active schema
  const currentSchema = await getActiveSchema(eventType);

  if (!currentSchema) {
    // First schema version, no compatibility check needed
    return { compatible: true };
  }

  // 2. Call Schema Registry compatibility check
  const response = await fetch(
    `${SCHEMA_REGISTRY_URL}/compatibility/subjects/${SUBJECT_NAME}/versions/latest`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/vnd.schemaregistry.v1+json' },
      body: JSON.stringify({ schema: JSON.stringify(newSchema) })
    }
  );

  const result = await response.json();

  if (!result.is_compatible) {
    return {
      compatible: false,
      errors: ['Schema is not backward compatible with current version']
    };
  }

  return { compatible: true };
}
```

---

## 7. Event Catalog Architecture

### 7.1 Catalog Portal (React Application)

**Technology Stack:**
- **Frontend Framework:** React 18 + TypeScript
- **State Management:** React Query (for server state) + Context API (for UI state)
- **UI Component Library:** Material-UI (MUI) v5
- **Routing:** React Router v6
- **API Client:** Axios with interceptors for auth
- **Authentication:** OAuth 2.0 / OIDC (integrate with existing SSO)
- **Hosting:** S3 + CloudFront (static site)

**Key Features:**

1. **Event Discovery**
   - Search bar with autocomplete (event_type, description)
   - Filters: data classification, publisher, status
   - Table view with sortable columns
   - Card view for browsing

2. **Event Details Page**
   - Full schema documentation
   - Sample payloads
   - Publisher information
   - Active subscribers list
   - Subscribe button (triggers subscription flow)

3. **Schema Browser**
   - Interactive Avro schema viewer
   - Field documentation
   - Version history
   - Backward compatibility indicators

4. **Subscription Management**
   - My Subscriptions dashboard
   - Connection details retrieval
   - Subscription status monitoring
   - Revoke subscription capability

5. **Audit Trail Viewer** (for compliance team)
   - Query interface with filters
   - Export functionality
   - Decision log viewer

**Page Structure:**

```
/
├── /catalog                  (Event listing)
├── /catalog/:event_type      (Event details + subscribe)
├── /subscriptions            (My subscriptions dashboard)
├── /subscriptions/:id        (Subscription details + credentials)
├── /schemas/:event_type      (Schema browser)
├── /audit                    (Audit query interface - restricted)
└── /docs                     (Platform documentation)
```

### 7.2 Search and Discovery

**Search Implementation:**

```typescript
// Catalog Service API: GET /api/v1/catalog/events
async function searchEvents(query: string, filters: SearchFilters): Promise<EventType[]> {
  // PostgreSQL full-text search
  const sql = `
    SELECT
      event_type,
      display_name,
      description,
      data_classification,
      schema_id,
      publisher_id,
      status,
      (
        SELECT COUNT(*)
        FROM subscription_event_types set
        JOIN subscriptions s ON set.subscription_id = s.subscription_id
        WHERE set.event_type_id = et.event_type_id
          AND s.status = 'active'
      ) as subscriber_count,
      created_at
    FROM event_types et
    WHERE
      (
        -- Full-text search on event_type and description
        to_tsvector('english', event_type || ' ' || description) @@ to_tsquery('english', $1)
        OR event_type ILIKE $2
      )
      AND ($3::int IS NULL OR data_classification <= $3)
      AND ($4::text IS NULL OR publisher_id = $4)
      AND ($5::text IS NULL OR status = $5)
    ORDER BY
      -- Relevance ranking
      ts_rank(to_tsvector('english', event_type || ' ' || description), to_tsquery('english', $1)) DESC,
      event_type ASC
    LIMIT $6 OFFSET $7
  `;

  const params = [
    query.replace(/\s+/g, ' & '),  // Convert to tsquery format
    `%${query}%`,                  // ILIKE pattern
    filters.data_classification || null,
    filters.publisher_id || null,
    filters.status || null,
    filters.limit || 50,
    filters.offset || 0
  ];

  const result = await db.query(sql, params);
  return result.rows;
}
```

**Autocomplete:**

```typescript
// Fast autocomplete using trigram similarity
const autocompleteSQL = `
  SELECT event_type, display_name
  FROM event_types
  WHERE event_type % $1  -- Trigram similarity operator
  ORDER BY similarity(event_type, $1) DESC
  LIMIT 10
`;
```

**Performance:**
- PostgreSQL indexes on `event_type`, `publisher_id`, `status`
- GIN index on `to_tsvector(event_type || description)` for full-text search
- Trigram index on `event_type` for autocomplete
- Response time target: < 200ms for search queries

---

## 8. Security Architecture

### 8.1 Authentication and Authorization

**Authentication:**

| Component | Authentication Method | Details |
|-----------|----------------------|---------|
| **Event Catalog Portal** | OAuth 2.0 / OIDC | Integrate with existing corporate SSO (e.g., Okta, Azure AD) |
| **Governance Services (APIs)** | mTLS + JWT | Service-to-service: mTLS; User requests: JWT from SSO |
| **Kafka Publishers/Consumers** | SASL/PLAIN (API Key) | Confluent Cloud API keys (managed by Subscription Service) |
| **OPA Policy Management** | Git-based RBAC | Policy updates via GitOps, PR approval required |

**Authorization Layers:**

1. **API Gateway Layer** (AWS API Gateway)
   - JWT validation
   - Rate limiting per application_id
   - WAF rules for common attacks

2. **Service Layer** (ECS services)
   - Application-level authorization (check user roles)
   - Subscription ownership validation (users can only see their own subscriptions)

3. **Data Layer** (OPA)
   - Event-level access control (ABAC policies)
   - Policy evaluation for all subscription requests

### 8.2 Data Encryption

| Data at Rest | Encryption Method |
|--------------|-------------------|
| **Kafka Data** | Confluent Cloud (encrypted by default with AWS KMS) |
| **RDS PostgreSQL** | AWS RDS encryption enabled (AES-256) |
| **S3 Audit Trail** | SSE-KMS (customer-managed key) |
| **Secrets Manager** | AWS Secrets Manager (encrypted by default) |

| Data in Transit | Encryption Method |
|-----------------|-------------------|
| **Kafka Connections** | TLS 1.2+ (SASL_SSL) |
| **Service APIs** | HTTPS (TLS 1.2+) |
| **Internal ECS** | mTLS between services |
| **Schema Registry** | HTTPS (TLS 1.2+) |

### 8.3 Secrets Management

**API Key Lifecycle:**

```
1. Subscription Approved
    │
    ▼
2. Subscription Service calls Confluent Cloud API
   - Creates service account for subscriber
   - Generates API key/secret pair
    │
    ▼
3. Store in AWS Secrets Manager
   - Secret name: eemp/subscriptions/{subscription_id}/kafka-credentials
   - Secret value: { "api_key": "...", "api_secret": "..." }
   - Auto-rotation policy: 90 days
    │
    ▼
4. Return to subscriber (ONE TIME ONLY)
   - API secret shown in response
   - After first retrieval, secret is redacted in UI
   - Subscriber must store securely
    │
    ▼
5. Subscriber uses credentials
   - Configure Kafka consumer with api_key, api_secret
   - Connect to Confluent Cloud
    │
    ▼
6. Auto-rotation (every 90 days)
   - Lambda function triggered
   - Generate new API key via Confluent Cloud
   - Update Secrets Manager
   - Email notification to subscriber
   - Grace period: 7 days (both old and new keys valid)
```

**Secret Rotation:**

```typescript
// Lambda function: secrets-rotation-handler
async function rotateKafkaCredentials(subscriptionId: string) {
  // 1. Get current credentials
  const currentSecret = await secretsManager.getSecretValue({
    SecretId: `eemp/subscriptions/${subscriptionId}/kafka-credentials`
  });

  const current = JSON.parse(currentSecret.SecretString);

  // 2. Create new API key in Confluent Cloud
  const newCredentials = await confluentCloud.createApiKey({
    serviceAccountId: current.service_account_id
  });

  // 3. Update Secrets Manager with new credentials
  await secretsManager.putSecretValue({
    SecretId: `eemp/subscriptions/${subscriptionId}/kafka-credentials`,
    SecretString: JSON.stringify({
      api_key: newCredentials.key,
      api_secret: newCredentials.secret,
      service_account_id: current.service_account_id,
      created_at: new Date().toISOString(),
      previous_key: current.api_key  // Keep old key for grace period
    })
  });

  // 4. Schedule deletion of old API key (7 days grace period)
  await scheduleApiKeyDeletion(current.api_key, 7);

  // 5. Notify subscriber
  await notifySubscriberOfRotation(subscriptionId, newCredentials.key);
}
```

### 8.4 Network Security

**VPC Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                          AWS VPC                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Public Subnets (Multi-AZ)                             │    │
│  │  - NAT Gateways                                        │    │
│  │  - Application Load Balancer (ALB)                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Private Subnets (Multi-AZ)                            │    │
│  │                                                         │    │
│  │  - ECS Fargate Tasks (Governance Services)             │    │
│  │  - RDS PostgreSQL (Multi-AZ)                           │    │
│  │  - ElastiCache Redis                                   │    │
│  │                                                         │    │
│  │  Outbound Internet via NAT Gateway                     │    │
│  │  (for Confluent Cloud connectivity)                    │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Security Groups:                                               │
│  - ALB SG: Allow 443 from corporate IP ranges                  │
│  - ECS Service SG: Allow traffic from ALB only                 │
│  - RDS SG: Allow 5432 from ECS Service SG only                 │
│  - Redis SG: Allow 6379 from ECS Service SG only               │
│                                                                  │
│  VPC Endpoints:                                                 │
│  - S3 Gateway Endpoint (for audit trail access)                │
│  - Secrets Manager Endpoint                                    │
│  - CloudWatch Logs Endpoint                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Confluent Cloud Connectivity:**

- **Connection:** ECS tasks → NAT Gateway → Internet → Confluent Cloud (AWS peering available post-MVP)
- **Security:** TLS 1.2+ encryption, SASL/PLAIN authentication
- **Future:** VPC Peering between AWS VPC and Confluent Cloud VPC for private connectivity

---

## 9. Operational Architecture

### 9.1 Deployment Architecture

**Infrastructure as Code:**
- **Tool:** Terraform
- **State Management:** S3 backend with DynamoDB locking
- **Environments:** Dev, Staging, Production (separate AWS accounts)

**Deployment Strategy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Deployment Pipeline (GitHub Actions)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Trigger: Push to main branch                                   │
│                                                                  │
│  1. Build Stage                                                  │
│     - Build Docker images for services                          │
│     - Tag with git SHA                                          │
│     - Push to ECR                                               │
│                                                                  │
│  2. Test Stage                                                   │
│     - Unit tests                                                 │
│     - Integration tests (against dev Kafka)                     │
│     - Security scanning (Snyk, Trivy)                           │
│                                                                  │
│  3. Deploy to Dev (auto)                                         │
│     - Terraform apply (dev environment)                         │
│     - ECS service update (rolling deployment)                   │
│     - Smoke tests                                               │
│                                                                  │
│  4. Deploy to Staging (auto)                                     │
│     - Terraform apply (staging environment)                     │
│     - ECS service update                                        │
│     - E2E tests                                                  │
│                                                                  │
│  5. Deploy to Production (manual approval)                       │
│     - Manual approval gate in GitHub                            │
│     - Terraform apply (production environment)                  │
│     - ECS Blue/Green deployment                                 │
│     - Gradual traffic shift (10% → 50% → 100%)                 │
│     - Automated rollback on error rate spike                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**ECS Service Configuration:**

```yaml
publisher_service:
  cluster: eemp-cluster
  launch_type: FARGATE
  task_cpu: 1024  # 1 vCPU
  task_memory: 2048  # 2 GB
  desired_count: 3  # Multi-AZ placement
  autoscaling:
    min_capacity: 3
    max_capacity: 10
    target_cpu_utilization: 70%
    target_memory_utilization: 80%
  health_check:
    path: /health
    interval: 30s
    timeout: 5s
    healthy_threshold: 2
    unhealthy_threshold: 3

subscription_service:
  # Similar configuration
  task_cpu: 512
  task_memory: 1024
  desired_count: 2
  # ...
```

### 9.2 Monitoring and Observability

**Metrics Collection:**

| Component | Metrics Source | Key Metrics |
|-----------|----------------|-------------|
| **Kafka Cluster** | Confluent Cloud → CloudWatch | Throughput, latency (p50, p99), partition lag, consumer group lag |
| **ECS Services** | CloudWatch Container Insights | CPU, memory, request rate, error rate, latency |
| **RDS** | CloudWatch RDS Metrics | Connections, CPU, IOPS, query latency |
| **API Gateway** | CloudWatch API Gateway Metrics | Request count, 4xx/5xx errors, latency |
| **OPA Sidecar** | Prometheus exporter → CloudWatch | Policy evaluation latency, decision count, cache hit rate |

**Logging:**

```
All logs sent to CloudWatch Logs

Log Groups:
- /eemp/publisher-service
- /eemp/subscription-service
- /eemp/catalog-service
- /eemp/audit-service
- /eemp/opa-sidecar

Log Format: JSON structured logs
{
  "timestamp": "2025-12-10T10:30:00.123Z",
  "level": "INFO",
  "service": "publisher-service",
  "correlation_id": "uuid-1234",
  "event_type": "user.created",
  "message": "Event published successfully",
  "kafka_offset": 56789,
  "latency_ms": 45
}

Retention: 30 days (CloudWatch), 7 years (S3 archive for audit logs)
```

**Distributed Tracing:**

- **Tool:** AWS X-Ray
- **Propagation:** Correlation ID passed through event envelope headers
- **Tracing Scope:** API Gateway → ECS Service → OPA → Kafka → Consumer

**Dashboards:**

1. **Platform Health Dashboard**
   - Kafka cluster health (Confluent Cloud metrics)
   - ECS service health (task count, CPU, memory)
   - RDS health (connections, CPU)
   - API error rates

2. **Event Metrics Dashboard**
   - Events published per minute (by event_type)
   - Events consumed per minute (by consumer group)
   - Consumer lag (by event_type)
   - Schema validation failures

3. **Governance Metrics Dashboard**
   - Subscription requests (approved/denied/pending)
   - OPA policy evaluation latency
   - Active subscriptions (by event_type)
   - Audit query volume

**Alerting:**

```yaml
alerts:
  - name: HighKafkaLatency
    condition: p99_latency > 200ms for 5 minutes
    severity: warning
    notification: PagerDuty (Platform Team)

  - name: ConsumerLagHigh
    condition: consumer_lag > 10000 messages for 10 minutes
    severity: warning
    notification: Slack (#eemp-alerts)

  - name: SchemaValidationFailureSpike
    condition: validation_failures > 100/min for 5 minutes
    severity: critical
    notification: PagerDuty (Platform Team)

  - name: OPAEvaluationFailure
    condition: opa_errors > 10/min for 3 minutes
    severity: critical
    notification: PagerDuty (Security Team)

  - name: ServiceDown
    condition: healthy_task_count = 0 for 2 minutes
    severity: critical
    notification: PagerDuty (Platform Team)
```

### 9.3 Disaster Recovery

**Backup Strategy:**

| Component | Backup Method | RPO | RTO |
|-----------|---------------|-----|-----|
| **Kafka Data** | Confluent Cloud (automatic replication) | < 1 min | < 5 min |
| **RDS PostgreSQL** | Automated snapshots + PITR | < 5 min | < 30 min |
| **S3 Audit Trail** | Cross-region replication | < 1 min | < 5 min |
| **Schema Registry** | Daily S3 archive | 24 hours | < 1 hour |

**Disaster Recovery Plan:**

1. **Region Failure (AWS us-east-1 down)**
   - **Current State (MVP):** Single-region deployment
   - **Recovery:** Restore from backups in alternate region (manual process)
   - **Downtime:** 2-4 hours (acceptable for MVP)
   - **Post-MVP:** Multi-region active-passive setup with automated failover

2. **Confluent Cloud Failure**
   - **Mitigation:** Confluent Cloud has built-in multi-AZ replication
   - **Recovery:** Automatic failover within region (handled by Confluent)
   - **Downtime:** < 5 minutes (Confluent SLA)

3. **Database Corruption**
   - **Recovery:** Restore from RDS snapshot + apply transaction logs (PITR)
   - **Data Loss:** < 5 minutes (RPO based on transaction log frequency)

**Runbook Documentation:**

- Incident response procedures for common failure scenarios
- Service restoration steps
- Escalation paths and on-call rotation

---

## 10. Migration Strategy

### 10.1 Migration from On-Prem Kafka

**Phase 1: Preparation (Week 1-2)**

1. **Schema Discovery**
   - Scan existing Kafka topics for event patterns
   - Generate Avro schemas from representative samples
   - Validate with publisher teams

2. **Infrastructure Setup**
   - Provision Confluent Cloud cluster
   - Deploy governance services (ECS)
   - Configure OPA policies
   - Set up catalog database

3. **Pilot Event Selection**
   - Select 2-3 low-risk event types for pilot
   - Identify publishers and consumers
   - Document migration plan

**Phase 2: Dual-Publish (Week 3-4)**

```
Publisher Application
    │
    ├──> On-Prem Kafka (existing consumers)
    │
    └──> EEMP Publisher Service ──> Confluent Cloud (new consumers)
```

- Publishers send events to BOTH on-prem and Confluent Cloud
- Existing consumers continue using on-prem Kafka (no change)
- New consumers test against Confluent Cloud
- Validate data consistency between on-prem and cloud

**Phase 3: Consumer Migration (Week 5-6)**

- Migrate consumers one-by-one to Confluent Cloud
- Use consumer group offsets to ensure no message loss
- Monitor consumer lag and error rates
- Rollback capability: Switch consumer back to on-prem if issues

**Phase 4: Publisher Cutover (Week 7)**

- Stop publishing to on-prem Kafka
- All events now flow only through EEMP → Confluent Cloud
- Monitor for 48 hours before decommissioning on-prem

**Phase 5: On-Prem Decommission (Week 8)**

- Verify all consumers migrated
- Archive on-prem Kafka data for compliance
- Decommission on-prem Kafka cluster

### 10.2 Migration Validation

**Data Consistency Checks:**

```python
# Validation script (run during dual-publish phase)
def validate_event_consistency():
    # 1. Consume same event from both on-prem and Confluent Cloud
    onprem_event = consume_from_onprem(event_id)
    cloud_event = consume_from_confluent(event_id)

    # 2. Compare payloads (ignoring envelope differences)
    assert onprem_event.payload == cloud_event.payload

    # 3. Verify schema compliance
    assert validate_against_schema(cloud_event, schema)

    # 4. Check latency
    latency = cloud_event.timestamp - onprem_event.timestamp
    assert latency < 100ms  # Acceptable latency difference
```

**Rollback Plan:**

- If critical issues during migration, revert to on-prem Kafka
- Consumer applications keep on-prem Kafka credentials as fallback
- Feature flag to toggle between on-prem and Confluent Cloud

---

## 11. Architectural Decisions Record (ADR)

### ADR-001: Single Kafka Topic Architecture

**Decision:** Use a single Kafka topic (`platform.events`) for all event types in MVP, with client-side filtering by `event_type`.

**Rationale:**
- Simplifies governance (unified schema validation, access control, audit)
- Reduces Confluent Cloud infrastructure costs
- Aligns with financial services pattern of centralized audit trail
- Enables efficient migration from on-prem Kafka

**Consequences:**
- Consumers must implement client-side filtering (handled by SDK)
- All events share same retention policy (7 days for MVP)
- Cannot apply different SLAs per event type
- Future: Can split into multiple topics if needed without architecture change

**Alternatives Considered:**
- Multiple topics per event type (rejected: complexity, cost)
- Topic per publisher (rejected: poor consumer experience)

---

### ADR-002: OPA Sidecar Deployment Model

**Decision:** Deploy OPA as a sidecar container alongside each ECS service, with policy bundles distributed from centralized bundle server.

**Rationale:**
- Low-latency policy evaluation (local calls, no network dependency)
- Resilience (service can operate with cached policies if bundle server unreachable)
- Leverages existing OPA infrastructure and policy management

**Consequences:**
- Each ECS task runs an additional OPA container (resource overhead)
- Policy updates require bundle server deployment
- 60-second policy update latency (bundle polling interval)

**Alternatives Considered:**
- Centralized OPA cluster (rejected: network latency, single point of failure)
- Embedded policy library (rejected: harder policy updates)

---

### ADR-003: Avro Schema Format

**Decision:** Use Apache Avro for event schema definition and serialization in MVP.

**Rationale:**
- Industry standard for Kafka ecosystems
- Efficient binary serialization (lower bandwidth costs)
- Strong typing with backward compatibility enforcement
- Native Confluent Schema Registry support

**Consequences:**
- Requires Avro serialization libraries for publishers/consumers
- Less human-readable than JSON (mitigated by schema documentation in catalog)
- Schema evolution requires backward compatibility validation

**Alternatives Considered:**
- JSON Schema (rejected: less efficient, weaker compatibility guarantees)
- Protocol Buffers (rejected: less Kafka ecosystem integration)

---

### ADR-004: Backward Compatibility as Default

**Decision:** Enforce BACKWARD compatibility mode for schema evolution.

**Rationale:**
- Consumers using old schemas can read new data (safest for financial services)
- Publishers can add optional fields without breaking consumers
- Aligns with principle of consumer protection

**Consequences:**
- Cannot remove fields from schemas (must deprecate entire event version)
- Cannot change field types (must add new field)
- Publishers must ensure new fields have default values

**Alternatives Considered:**
- FULL compatibility (rejected: too restrictive for publishers)
- FORWARD compatibility (rejected: risks breaking consumers)

---

### ADR-005: Event Catalog Portal as Static Site

**Decision:** Host Event Catalog Portal as a React static site on S3 + CloudFront.

**Rationale:**
- Cost-effective (no server infrastructure for frontend)
- Highly available (CloudFront global CDN)
- Simple deployment (build → S3 upload)
- Fast performance (static assets cached at edge)

**Consequences:**
- All business logic must be in backend APIs
- Authentication handled via OAuth tokens (no server-side sessions)
- Cannot use server-side rendering (mitigated: client-side rendering acceptable for internal portal)

**Alternatives Considered:**
- Server-side rendering with Next.js on ECS (rejected: unnecessary complexity and cost)

---

### ADR-006: Multi-Account and Hybrid Connectivity

**Decision:** Support cross-account AWS subscribers and on-premises subscribers via internet-based connectivity to Confluent Cloud using API key authentication (MVP), with optional PrivateLink and Direct Connect for post-MVP.

**Context:**
- EEMP Platform Account hosts governance services
- Publishers/subscribers distributed across multiple AWS accounts
- On-premises systems need event integration
- Security, cost, and operational complexity must be balanced

**Rationale:**

1. **Internet-Based Connectivity (MVP):**
   - Confluent Cloud provides internet-accessible endpoints with TLS encryption
   - No VPC peering or cross-account networking required
   - Simplified onboarding: new AWS accounts connect immediately
   - Works uniformly for AWS and on-premises subscribers
   - Security via TLS 1.2+ encryption + SASL/PLAIN authentication + IP allowlisting

2. **API Key Authentication:**
   - Each subscriber receives unique Confluent Cloud API key/secret
   - Works across AWS accounts and on-premises without IAM complexity
   - Credentials stored in subscriber's own Secrets Manager (account isolation)
   - Automated 90-day rotation with grace period

3. **Account Isolation:**
   - No trust relationships or cross-account IAM roles required
   - Each application account manages own credentials
   - Platform Account tracks subscriptions but doesn't access application accounts

**Consequences:**

**Pros:**
- Simple architecture: no VPC peering, no Direct Connect setup
- Fast time-to-market: subscribers onboard in minutes, not weeks
- Uniform experience: AWS and on-prem subscribers use same connectivity pattern
- Low operational overhead: no complex network management
- Account independence: application teams fully control their credentials

**Cons:**
- Internet egress costs: ~$0.09/GB from AWS to internet (estimated $100-300/month at MVP scale)
- Latency: Internet routing adds 5-10ms vs. private connectivity
- Security perception: Some teams may prefer "air-gapped" connectivity
- Corporate proxy compatibility: On-prem subscribers need proxy that supports TCP tunneling

**Migration Path (Post-MVP):**

When scale, security requirements, or cost optimization justify it:

1. **AWS PrivateLink** (estimated 6-12 months post-MVP):
   - Private connectivity for AWS subscribers
   - 1-3ms latency improvement
   - Eliminates internet egress costs (saves $5K-10K/year at scale)
   - Added cost: ~$150-300/month

2. **Direct Connect** (estimated 12+ months post-MVP):
   - Private connectivity for on-premises subscribers
   - 5-15ms latency, dedicated bandwidth
   - Required for strict regulatory "no internet" policies
   - Added cost: ~$300-500/month + usage

**Alternatives Considered:**

1. **VPC Peering + PrivateLink from Day 1:**
   - Rejected: Adds 2-4 weeks setup time per application account
   - Rejected: Doesn't solve on-prem connectivity
   - Rejected: Premature optimization (internet connectivity sufficient for MVP)

2. **Cross-Account IAM Roles:**
   - Rejected: Only works for AWS, excludes on-prem
   - Rejected: Doesn't integrate with Confluent Cloud authentication
   - Rejected: Complex trust relationship management

3. **Direct Connect from Day 1:**
   - Rejected: 4-8 weeks network engineering effort
   - Rejected: High upfront cost for unproven platform
   - Rejected: Overkill for MVP scale

**Validation Criteria:**

Monitor for 90 days:
- Internet egress costs < $500/month (acceptable)
- p99 latency < 150ms (internet routing overhead acceptable)
- Zero security incidents related to internet connectivity
- Subscriber feedback on connectivity experience

If costs exceed $500/month or security policy changes mandate private connectivity, implement PrivateLink.

---

## 12. Future Considerations

### Post-MVP Enhancements

**Phase 2 (3-6 months post-MVP):**
1. **Platform-Managed Filtering:**
   - Kafka Streams application or ksqlDB for server-side filtering
   - Reduces consumer complexity and network bandwidth
   - Enables content-based routing (e.g., "only user.updated where country=US")

2. **Schema Deprecation Workflow:**
   - Automated notifications to subscribers when schemas deprecated
   - Sunset date enforcement (block new subscriptions to deprecated versions)
   - Migration tracking dashboard

3. **Advanced Data Classification:**
   - Automatic PII detection in event payloads
   - Data masking for non-authorized consumers
   - Cross-border data sovereignty controls

**Phase 3 (6-12 months post-MVP):**
1. **Multi-Region Deployment:**
   - Active-passive or active-active Confluent Cloud clusters
   - Cross-region replication for disaster recovery
   - Regional event routing

2. **Event Mesh Evolution:**
   - Support multiple event patterns (pub/sub, streaming, request/response)
   - Event sourcing capabilities
   - CQRS pattern support

3. **AI-Powered Insights:**
   - Anomaly detection on event access patterns
   - Usage analytics and optimization recommendations
   - Automated schema compatibility suggestions

---

## 13. Appendix

### A. Technology Choices Summary

| Category | Technology | Justification |
|----------|-----------|---------------|
| **Event Backbone** | Confluent Cloud (Kafka) | Managed service, schema registry, stream governance, AWS integration |
| **Schema Format** | Apache Avro | Industry standard, efficient, backward compatibility |
| **Policy Engine** | Open Policy Agent (OPA) | Existing infrastructure, ABAC support, declarative policies |
| **Backend Services** | Node.js + TypeScript | Team expertise, async I/O performance, strong typing |
| **Frontend** | React + TypeScript | Modern SPA framework, component reusability, type safety |
| **Database** | PostgreSQL (RDS) | Relational model for catalog/subscriptions, full-text search, ACID |
| **Cache** | Redis (ElastiCache) | Session management, OPA policy cache, API response cache |
| **Secrets** | AWS Secrets Manager | Managed rotation, integration with ECS, audit trail |
| **Monitoring** | CloudWatch + X-Ray | Native AWS integration, centralized logs and metrics |
| **IaC** | Terraform | Multi-cloud support, state management, team expertise |

### B. API Reference Summary

**Publisher Service:**
- `POST /api/v1/events/publish` - Publish event
- `POST /api/v1/schemas/register` - Register schema

**Subscription Service:**
- `POST /api/v1/subscriptions` - Create subscription
- `GET /api/v1/subscriptions/{id}` - Get subscription details
- `DELETE /api/v1/subscriptions/{id}` - Revoke subscription

**Catalog Service:**
- `GET /api/v1/catalog/events` - List events
- `GET /api/v1/catalog/events/{event_type}` - Get event details
- `GET /api/v1/catalog/schemas/{event_type}` - Get schema versions

**Audit Service:**
- `POST /api/v1/audit/query` - Query audit trail
- `POST /api/v1/audit/export` - Export audit data
- `GET /api/v1/audit/export/{export_id}` - Get export status

### C. Glossary

| Term | Definition |
|------|------------|
| **ABAC** | Attribute-Based Access Control - access decisions based on attributes of subscriber and event |
| **AVRO** | Apache Avro - data serialization format with schema support |
| **Consumer Group** | Kafka concept - group of consumers sharing partition assignments |
| **Event Envelope** | Standard wrapper around event payload (event_id, timestamp, correlation_id, etc.) |
| **OPA** | Open Policy Agent - policy-based control engine |
| **Schema Registry** | Confluent service for storing and versioning event schemas |
| **Backward Compatibility** | New schema can read data written with old schema |
| **Publisher** | Application/service that produces events |
| **Subscriber/Consumer** | Application that consumes events |

---

**Document End**
