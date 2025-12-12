# Enterprise Event Management Platform (EEMP) - EventBridge Architecture

**Version:** 1.0
**Date:** 2025-12-10
**Status:** Alternative Architecture
**Author:** Architecture Team

---

## Executive Summary

This document defines an **alternative technical architecture** for the Enterprise Event Management Platform (EEMP) using **Amazon EventBridge** instead of Confluent Cloud Kafka. This architecture leverages AWS-native event infrastructure, providing deep integration with AWS services, simplified cross-account access, and serverless event routing capabilities.

While the primary architecture recommendation uses Confluent Cloud (see [eemp-architecture-2025-12-10.md](eemp-architecture-2025-12-10.md)), this EventBridge-based alternative may be preferred when:
- Organization is heavily AWS-centric with limited Kafka expertise
- Cross-account event distribution is a primary requirement
- Serverless architecture is preferred over managed Kafka
- AWS-native tooling and billing simplification is valued
- Event volume is moderate (< 1M events/day initially)

### Architectural Principles

1. **AWS-Native First:** Leverage AWS services for event infrastructure, reducing external dependencies
2. **Serverless Operations:** Minimize operational overhead with fully managed services
3. **Cross-Account Simplicity:** Use EventBridge native cross-account capabilities
4. **Compliance First:** Immutable audit trails and enriched logging for regulatory examination
5. **Schema Governance:** EventBridge Schema Registry for event schema validation and discovery
6. **Developer Experience:** Self-service capabilities with clear feedback and fast integration cycles

---

## 1. EventBridge vs. Kafka: Platform Comparison

### 1.1 Why Consider EventBridge?

**Amazon EventBridge** is a serverless event bus service that enables event-driven architectures at scale. Unlike Kafka (a streaming platform), EventBridge focuses on **event routing and filtering** with deep AWS integration.

**Key Differentiators:**

| Aspect | EventBridge | Kafka (Confluent Cloud) |
|--------|-------------|-------------------------|
| **Architecture Model** | Serverless event router | Managed streaming platform |
| **Primary Use Case** | Event distribution, filtering, routing | High-throughput streaming, event sourcing |
| **Cross-Account** | Native cross-account event bus sharing | Requires internet/PrivateLink + API keys |
| **AWS Integration** | First-class (Lambda, SQS, SNS, Step Functions, etc.) | External service, requires connectors |
| **Schema Management** | EventBridge Schema Registry (AWS-native) | Confluent Schema Registry (industry standard) |
| **Event Ordering** | Not guaranteed (best-effort) | Guaranteed within partition |
| **Event Replay** | Archive & Replay (90-day retention) | Kafka topic retention (configurable) |
| **Operational Model** | Zero ops (serverless) | Minimal ops (Confluent manages cluster) |
| **Pricing Model** | Pay-per-event ($1/million events) | Cluster-based (fixed CKU cost) |
| **Throughput Limits** | Soft limits (10K events/sec default, can increase) | High throughput (100K+ events/sec) |

### 1.2 EventBridge Architecture Decision Matrix

| Criteria | Weight | EventBridge | Confluent Kafka | Reasoning |
|----------|--------|-------------|-----------------|-----------|
| **AWS Integration** | 20% | **10/10** | 6/10 | EventBridge native to AWS ecosystem |
| **Cross-Account Simplicity** | 15% | **10/10** | 5/10 | Built-in cross-account event bus |
| **Operational Overhead** | 15% | **10/10** | 9/10 | True serverless vs. managed cluster |
| **Event Ordering** | 10% | 3/10 | **10/10** | Kafka guarantees ordering, EventBridge does not |
| **Schema Governance** | 15% | 7/10 | **10/10** | Confluent has mature schema registry |
| **Event Replay** | 10% | 8/10 | **10/10** | Both support replay, Kafka more flexible |
| **Cost (Low Volume)** | 5% | **9/10** | 5/10 | EventBridge cheaper at < 1M events/day |
| **Cost (High Volume)** | 5% | 4/10 | **9/10** | Kafka better at > 10M events/day |
| **Throughput Scalability** | 5% | 7/10 | **10/10** | Kafka handles higher throughput |

**Weighted Scores:**
- **EventBridge: 8.2/10** (for AWS-centric, moderate-volume use cases)
- **Confluent Kafka: 7.9/10** (for high-volume, streaming-first use cases)

### 1.3 When to Choose EventBridge Architecture

**Choose EventBridge if:**
- ✓ Your organization is heavily AWS-centric (80%+ workloads on AWS)
- ✓ Event volume is moderate (< 5M events/day)
- ✓ Cross-account event distribution is a primary requirement
- ✓ You prefer serverless architecture (no cluster management)
- ✓ Team lacks Kafka expertise
- ✓ Integration with AWS services (Lambda, SQS, Step Functions) is critical
- ✓ Event ordering within a single event type is not critical
- ✓ You value AWS-native billing and cost allocation

**Choose Kafka (Confluent Cloud) if:**
- ✓ Event volume is high (> 10M events/day)
- ✓ Event ordering is critical for business logic
- ✓ You need event streaming patterns (continuous processing, windowing)
- ✓ You require mature schema governance with strong compatibility enforcement
- ✓ You need Kafka ecosystem tools (ksqlDB, Kafka Streams)
- ✓ Multi-cloud or hybrid cloud is a requirement
- ✓ You need fine-grained control over retention policies per event type

---

## 2. EventBridge System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EEMP Platform (AWS)                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────┐         ┌──────────────────────────────────────┐  │
│  │  Event Publishers │────────>│    Governance Layer (Lambda + ECS)  │  │
│  │  (Platform APIs)  │         │                                      │  │
│  └──────────────────┘         │  ┌────────────────────────────────┐  │  │
│                                │  │  Publisher Lambda              │  │  │
│                                │  │  - Schema Validation           │  │  │
│  ┌──────────────────┐         │  │  - Event Enrichment            │  │  │
│  │  Event Catalog   │<───────>│  │  - EventBridge PutEvents       │  │  │
│  │  Portal (React)  │         │  └────────────────────────────────┘  │  │
│  │                  │         │                                      │  │
│  │  - Event Discovery│         │  ┌────────────────────────────────┐  │  │
│  │  - Subscription  │         │  │  Subscription Service (ECS)    │  │  │
│  │  - Schema Browser│<───────>│  │  - OPA Integration             │  │  │
│  └──────────────────┘         │  │  - Rule Creation               │  │  │
│                                │  │  - Cross-Account Setup         │  │  │
│                                │  └────────────────────────────────┘  │  │
│  ┌──────────────────┐         │                                      │  │
│  │ Event Consumers  │         │  ┌────────────────────────────────┐  │  │
│  │ (Lambda, Apps)   │<────────│  │  Catalog Service (ECS)         │  │  │
│  └──────────────────┘         │  │  - Event Registry              │  │  │
│                                │  │  - Schema Management           │  │  │
│         │                      │  └────────────────────────────────┘  │  │
│         │                      │                                      │  │
│         │                      └──────────────────────────────────────┘  │
│         │                                                                 │
│         ▼                                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              Amazon EventBridge (Platform Account)               │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │  Custom Event Bus: eemp-platform-events                    │  │   │
│  │  │                                                             │  │   │
│  │  │  - Receives all events via PutEvents API                   │  │   │
│  │  │  - Event pattern matching and filtering                    │  │   │
│  │  │  - Routes to targets (Lambda, SQS, cross-account buses)    │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │  EventBridge Schema Registry                               │  │   │
│  │  │  - Auto-discovery of event schemas                         │  │   │
│  │  │  - Schema versioning (OpenAPI 3.0 format)                  │  │   │
│  │  │  - Code bindings generation (Java, Python, TypeScript)     │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │  Event Archive & Replay                                    │  │   │
│  │  │  - Archive: 90-day retention (compliance)                  │  │   │
│  │  │  - Replay: Reprocess events for debugging/recovery         │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Supporting Infrastructure                      │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  - RDS PostgreSQL (Metadata, Subscriptions, Audit Index)         │   │
│  │  - S3 (Audit Trail Archive, Schema Backup)                       │   │
│  │  - DynamoDB (Event Deduplication, Idempotency)                   │   │
│  │  - SQS (Dead Letter Queues for failed deliveries)                │   │
│  │  - OPA Cluster (Policy Decisions) - Existing Infrastructure      │   │
│  │  - CloudWatch (Monitoring, Alerting, Logs)                       │   │
│  │  - Secrets Manager (Credentials, API Keys)                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                    Cross-Account Event Distribution                       │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Application Account 1 (222222222222)                                     │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  Custom Event Bus: eemp-subscriber-bus                           │    │
│  │  - Receives events from Platform Account via EventBridge Rule    │    │
│  │  - Routes to Lambda, SQS, or application targets                 │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│         │                                                                  │
│         ▼                                                                  │
│  Lambda / SQS → Application Logic                                         │
│                                                                            │
│  Application Account 2 (333333333333)                                     │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  Custom Event Bus: eemp-subscriber-bus                           │    │
│  │  - Receives events from Platform Account                         │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│         │                                                                  │
│         ▼                                                                  │
│  EC2 Application → Polls SQS → Processes Events                           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Overview

| Component | Purpose | Technology | Hosting |
|-----------|---------|------------|---------|
| **Publisher Lambda** | Schema validation, event enrichment, PutEvents to EventBridge | Lambda (Node.js/Python) | AWS Lambda |
| **Subscription Service** | OPA integration, EventBridge rule creation, cross-account setup | ECS Fargate (Node.js/TS) | ECS Fargate |
| **Catalog Service** | Event registry, schema management, discovery API | ECS Fargate (Node.js/TS) | ECS Fargate |
| **Audit Lambda** | Process EventBridge events for audit trail enrichment | Lambda (Node.js/Python) | AWS Lambda |
| **Event Catalog Portal** | Web UI for discovery, subscription, schema browsing | React + TypeScript | S3 + CloudFront |
| **EventBridge Bus** | Event routing, filtering, cross-account distribution | Amazon EventBridge | AWS Managed |
| **Schema Registry** | Schema storage, versioning, code generation | EventBridge Schema Registry | AWS Managed |
| **Event Archive** | Long-term event storage for replay and compliance | EventBridge Archive | AWS Managed |
| **Metadata Store** | Event catalog, subscriptions, publisher registry | PostgreSQL RDS | AWS RDS |
| **Audit Store** | Immutable audit trail, enriched event logs | S3 + PostgreSQL | AWS S3 + RDS |
| **Deduplication Store** | Event idempotency tracking | DynamoDB | AWS DynamoDB |
| **OPA Cluster** | Policy evaluation engine (ABAC) | Open Policy Agent | Existing Infrastructure |

---

## 3. EventBridge Event Bus Architecture

### 3.1 Event Bus Design

**Decision: Custom Event Bus with Cross-Account Distribution**

**Architecture:**

```yaml
platform_account:
  event_bus:
    name: eemp-platform-events
    type: CUSTOM  # Not default bus (isolation)

  event_structure:
    source: "com.company.eemp"  # Custom source prefix
    detail_type: "user.created" | "user.updated" | "document.created" | ...
    detail:
      # Event-specific payload
      event_id: "uuid"
      correlation_id: "uuid"
      timestamp: "ISO-8601"
      payload: { ... }

  rules:
    - name: "route-to-application-account-1"
      event_pattern:
        source: ["com.company.eemp"]
        detail-type: ["user.created", "user.updated"]
      targets:
        - cross_account_event_bus:
            account: "222222222222"
            region: "us-east-1"

    - name: "audit-all-events"
      event_pattern:
        source: ["com.company.eemp"]
      targets:
        - lambda: audit-enrichment-function
        - s3: audit-archive-bucket (via Firehose)
```

**Event Structure:**

```json
{
  "version": "0",
  "id": "event-uuid-1234",
  "detail-type": "user.created",
  "source": "com.company.eemp",
  "account": "111111111111",
  "time": "2025-12-10T10:30:00Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "event_id": "uuid-1234",
    "event_type": "user.created",
    "schema_version": "1.0",
    "correlation_id": "trace-abc-123",
    "actor": {
      "type": "service",
      "id": "user-profile-service"
    },
    "source": {
      "system": "user-profile-service",
      "region": "us-east-1",
      "account_id": "222222222222"
    },
    "payload": {
      "user_id": "12345",
      "email": "user@example.com",
      "created_at": "2025-12-10T10:30:00Z"
    },
    "metadata": {
      "data_classification": 2,
      "pii_indicator": true,
      "tenant_id": "tenant-001"
    }
  }
}
```

**Key Design Decisions:**

1. **Custom Event Bus (not default):**
   - **Rationale:** Isolation from other AWS service events, dedicated throughput limits
   - **Consequence:** Must explicitly specify event bus ARN in PutEvents calls

2. **Standardized Event Envelope:**
   - **EventBridge Standard Fields:** `version`, `id`, `detail-type`, `source`, `time`
   - **Custom Detail Fields:** `event_id`, `correlation_id`, `payload`, `metadata`
   - **Rationale:** Leverage EventBridge native filtering on `detail-type` and `source`

3. **Detail-Type as Event Type:**
   - **Pattern:** `detail-type` = `user.created`, `document.updated`, etc.
   - **Rationale:** Enables EventBridge rule pattern matching on event type
   - **Example Rule:** Match only `user.*` events for user service subscribers

### 3.2 Cross-Account Event Distribution

**EventBridge Native Cross-Account Pattern:**

```
┌─────────────────────────────────────────────────────────────────┐
│  Platform Account (111111111111)                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  EventBridge Rule: route-to-app-account-1              │    │
│  │                                                         │    │
│  │  Event Pattern:                                        │    │
│  │    source: ["com.company.eemp"]                        │    │
│  │    detail-type: ["user.created", "user.updated"]       │    │
│  │                                                         │    │
│  │  Target:                                               │    │
│  │    EventBus ARN: arn:aws:events:us-east-1:222222222222│    │
│  │                  :event-bus/eemp-subscriber-bus        │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            │ EventBridge Cross-Account Delivery
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Application Account 1 (222222222222)                           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Event Bus: eemp-subscriber-bus                        │    │
│  │  - Policy allows PutEvents from Platform Account       │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                     │
│                           ▼                                     │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  EventBridge Rule: process-user-events                 │    │
│  │                                                         │    │
│  │  Event Pattern:                                        │    │
│  │    source: ["com.company.eemp"]                        │    │
│  │    detail-type: ["user.created", "user.updated"]       │    │
│  │                                                         │    │
│  │  Targets:                                              │    │
│  │    - Lambda: notifications-handler                     │    │
│  │    - SQS: user-events-queue                            │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Cross-Account Setup (Automated by Subscription Service):**

**Step 1: Application Account creates Event Bus with Resource Policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPlatformAccountToPutEvents",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:222222222222:event-bus/eemp-subscriber-bus"
    }
  ]
}
```

**Step 2: Platform Account creates Rule targeting Application Account Event Bus**

```python
# Executed by EEMP Subscription Service
import boto3

events_client = boto3.client('events')

# Create rule in Platform Account
events_client.put_rule(
    Name='route-to-app-account-222222222222-user-events',
    EventBusName='eemp-platform-events',
    EventPattern=json.dumps({
        'source': ['com.company.eemp'],
        'detail-type': ['user.created', 'user.updated']
    }),
    State='ENABLED'
)

# Add target (Application Account Event Bus)
events_client.put_targets(
    Rule='route-to-app-account-222222222222-user-events',
    EventBusName='eemp-platform-events',
    Targets=[
        {
            'Id': '1',
            'Arn': 'arn:aws:events:us-east-1:222222222222:event-bus/eemp-subscriber-bus',
            'RoleArn': 'arn:aws:iam::111111111111:role/EEMPEventBridgeRole'
        }
    ]
)
```

**Step 3: Application Account creates local rules for processing**

Application team creates rules in their account to route events to their targets (Lambda, SQS, etc.)

**Benefits of EventBridge Cross-Account:**
- **No API Keys:** Uses IAM roles and resource policies
- **Native AWS:** No external authentication mechanisms
- **Simple Setup:** 2-step process (resource policy + rule creation)
- **Automatic Retry:** EventBridge handles retries for cross-account delivery
- **Audit Trail:** CloudTrail logs all cross-account event deliveries

**Comparison to Kafka Cross-Account:**

| Aspect | EventBridge | Kafka (Confluent Cloud) |
|--------|-------------|-------------------------|
| **Setup Complexity** | Simple (resource policy + rule) | Complex (API keys, Secrets Manager) |
| **Authentication** | IAM-based | API key-based |
| **Credential Management** | No credentials to rotate | 90-day rotation required |
| **Network** | AWS backbone (private) | Internet or PrivateLink |
| **Cost** | No cross-account fees | Potential egress costs |

### 3.3 Event Filtering and Routing

**EventBridge Rule Patterns (Examples):**

**1. Route specific event types to subscriber:**

```json
{
  "source": ["com.company.eemp"],
  "detail-type": ["user.created", "user.updated"]
}
```

**2. Filter by payload fields (content-based routing):**

```json
{
  "source": ["com.company.eemp"],
  "detail-type": ["user.updated"],
  "detail": {
    "metadata": {
      "data_classification": [1, 2]
    }
  }
}
```

**3. Filter by tenant (multi-tenancy):**

```json
{
  "source": ["com.company.eemp"],
  "detail": {
    "metadata": {
      "tenant_id": ["tenant-001"]
    }
  }
}
```

**Advanced Filtering (EventBridge supports):**
- Prefix matching: `{"detail-type": [{"prefix": "user."}]}`
- Numeric ranges: `{"detail": {"payload": {"age": [{"numeric": [">=", 18]}]}}}`
- Exists checks: `{"detail": {"payload": {"email": [{"exists": true}]}}}`
- Anything-but: `{"detail-type": [{"anything-but": "user.deleted"}]}`

**Server-Side Filtering Benefits:**
- **Reduced Network Traffic:** Events filtered before delivery to subscribers
- **No Client-Side Logic:** Subscribers receive only relevant events
- **Cost Optimization:** Pay only for matched events delivered to targets
- **Comparison to Kafka:** Kafka requires client-side filtering (consumer must read all events from topic)

---

## 4. EventBridge Schema Registry Integration

### 4.1 Schema Registry Architecture

**EventBridge Schema Registry** provides automatic schema discovery and versioning for events flowing through EventBridge.

**Architecture:**

```
┌────────────────────────────────────────────────────────────────┐
│                  EventBridge Schema Registry                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Registry: eemp-schemas                                        │
│                                                                 │
│  Schema Discovery:                                             │
│  - Automatic: EventBridge infers schema from events (enabled)  │
│  - Manual: Upload OpenAPI 3.0 or JSONSchema definitions        │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Schema Versioning Model                                  │ │
│  │                                                            │ │
│  │  Schema Name: com.company.eemp@user.created               │ │
│  │  Version: 1 (auto-incremented on schema change)           │ │
│  │  Format: OpenAPI 3.0 (EventBridge default)                │ │
│  │                                                            │ │
│  │  Example:                                                  │ │
│  │    user.created (v1) → Schema inferred from first event   │ │
│  │    user.created (v2) → Schema updated when field added    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Code Bindings Generation:                                     │
│  - Java classes                                                │
│  - Python dataclasses                                          │
│  - TypeScript interfaces                                       │
│                                                                 │
│  Schema Storage:                                               │
│  - Primary: EventBridge Schema Registry (AWS-managed)          │
│  - Backup: S3 (daily export for disaster recovery)            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

**Key Features:**

1. **Automatic Schema Discovery:**
   - EventBridge observes events and infers schemas
   - No manual registration required (unlike Confluent Schema Registry)
   - Schemas auto-versioned when structure changes

2. **OpenAPI 3.0 Format:**
   - Schemas stored as OpenAPI 3.0 specifications
   - More verbose than Avro but human-readable
   - Native JSON Schema support

3. **Code Generation:**
   - Download SDKs for Java, Python, TypeScript
   - Auto-generated from schema definitions
   - Example:
     ```bash
     aws schemas get-code-binding-source \
       --registry-name eemp-schemas \
       --schema-name com.company.eemp@user.created \
       --language TypeScript
     ```

**Schema Management Workflow:**

```
Publisher Team
    │
    ▼
1. Define Event Schema (Optional)
   - OpenAPI 3.0 or JSONSchema format
   - Upload to Schema Registry via API/Console
    │
    ▼
2. Publish Event to EventBridge
   - PutEvents API call
    │
    ▼
3. EventBridge Schema Discovery (Automatic)
   - Observes event structure
   - Creates/updates schema in registry
   - Versions schema if structure changed
    │
    ▼
4. Schema Available in Catalog
   - EEMP Catalog Service queries Schema Registry
   - Displays schema in Event Catalog Portal
   - Subscribers can download code bindings
```

**Schema Validation:**

Unlike Confluent Schema Registry (which validates events before publishing), EventBridge Schema Registry does NOT enforce validation. Validation must be handled by EEMP Publisher Service:

```typescript
// Publisher Lambda function
import Ajv from 'ajv';

async function validateEvent(eventType: string, payload: any): Promise<boolean> {
  // 1. Fetch schema from EventBridge Schema Registry
  const schema = await getSchemaFromRegistry(eventType);

  // 2. Validate payload against JSONSchema
  const ajv = new Ajv();
  const validate = ajv.compile(schema);
  const valid = validate(payload);

  if (!valid) {
    console.error('Schema validation errors:', validate.errors);
    return false;
  }

  return true;
}
```

### 4.2 Schema Governance Trade-offs

**EventBridge Schema Registry vs. Confluent Schema Registry:**

| Feature | EventBridge Schema Registry | Confluent Schema Registry |
|---------|----------------------------|---------------------------|
| **Auto-Discovery** | ✓ Yes (infers from events) | ✗ No (manual registration) |
| **Schema Format** | OpenAPI 3.0, JSONSchema | Avro, Protobuf, JSON Schema |
| **Validation Enforcement** | ✗ No (manual validation required) | ✓ Yes (broker-level enforcement) |
| **Compatibility Modes** | ✗ No compatibility checking | ✓ Yes (BACKWARD, FORWARD, FULL) |
| **Code Generation** | ✓ Java, Python, TypeScript | ✓ Java, Python, Go, etc. |
| **Versioning** | Auto-versioned on change | Explicit version registration |
| **Cost** | Free (included in EventBridge) | Included in Confluent Cloud |

**Governance Gap: Backward Compatibility**

EventBridge Schema Registry does **not** validate backward compatibility. This means:
- Publishers can introduce breaking changes without warning
- No automatic prevention of incompatible schema evolution
- Must build custom compatibility checking in EEMP Publisher Service

**Mitigation Strategy:**

```typescript
// Custom backward compatibility check (simplified)
async function checkBackwardCompatibility(
  eventType: string,
  newPayload: any
): Promise<{ compatible: boolean; errors?: string[] }> {

  // 1. Get current schema version from Schema Registry
  const currentSchema = await getCurrentSchema(eventType);

  // 2. Get previous schema version
  const previousSchema = await getPreviousSchema(eventType);

  if (!previousSchema) {
    return { compatible: true }; // First version, no compatibility check
  }

  // 3. Use schema-compatibility library (e.g., json-schema-diff)
  const diff = compareSchemas(previousSchema, currentSchema);

  // 4. Check for breaking changes
  const breakingChanges = diff.filter(change =>
    change.type === 'removed_field' ||
    change.type === 'type_changed'
  );

  if (breakingChanges.length > 0) {
    return {
      compatible: false,
      errors: breakingChanges.map(c => c.message)
    };
  }

  return { compatible: true };
}
```

**Recommendation:**
- **For MVP:** Accept manual compatibility checking (lower governance maturity)
- **Post-MVP:** Build custom compatibility validation or consider migrating to Confluent Schema Registry

---

## 5. OPA Integration for EventBridge

### 5.1 OPA Policy Evaluation

**OPA Integration Architecture (Same as Kafka architecture):**

```
┌─────────────────────────────────────────────────────────────────┐
│                    OPA Policy Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Centralized Policy Repository (Git) → OPA Bundle Server        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  OPA Sidecar (Subscription Service ECS Task)           │    │
│  │  - Evaluates subscription access policies              │    │
│  │  - Attribute-based access control (ABAC)               │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Subscription Authorization Policy (Same as Kafka architecture):**

```rego
package eemp.subscription

default allow = false

allow {
    # Check data classification clearance
    input.subscriber.data_classification_clearance >= data.events[input.event_type].data_classification

    # Check tenant isolation
    input.subscriber.tenant_id == data.events[input.event_type].tenant_id

    # Check application is active
    input.subscriber.status == "active"
}
```

**Policy Input:**

```json
{
  "subscriber": {
    "application_id": "notifications-service",
    "aws_account_id": "222222222222",
    "tenant_id": "tenant-001",
    "data_classification_clearance": 3
  },
  "event_types": ["user.created", "user.updated"]
}
```

**Policy Decision → EventBridge Rule Creation:**

```
1. Subscription Request
    │
    ▼
2. OPA Policy Evaluation
    │
    ├──> DENIED → Reject subscription (403 Forbidden)
    │
    ├──> APPROVED
    │      │
    │      ▼
3. Create EventBridge Resources (automated by Subscription Service)
    │
    ├──> Create Event Bus in Application Account (if not exists)
    │      - Via CloudFormation StackSet or Step Function
    │
    ├──> Add Resource Policy to Application Account Event Bus
    │      - Allow Platform Account to PutEvents
    │
    ├──> Create EventBridge Rule in Platform Account
    │      - Name: route-to-{account_id}-{event_types}
    │      - Event Pattern: Match approved event_types
    │      - Target: Application Account Event Bus ARN
    │
    └──> Log Audit Entry
           - Subscription created
           - OPA decision_id
           - EventBridge rule ARN
```

### 5.2 Subscription Lifecycle Management

**Subscription Service Responsibilities:**

1. **Subscription Creation:**
   - OPA policy evaluation
   - EventBridge rule creation (Platform Account)
   - Event Bus setup orchestration (Application Account via CloudFormation)
   - Audit logging

2. **Subscription Revocation:**
   - Delete EventBridge rule in Platform Account
   - Optionally delete Event Bus in Application Account (if no other subscriptions)
   - Audit logging

3. **Policy Changes:**
   - Update EventBridge rule event pattern
   - Example: Subscriber promoted to higher data classification → Update rule to include more event types

**Cross-Account Event Bus Setup (Automated):**

```yaml
# CloudFormation StackSet deployed to Application Account
# Triggered by EEMP Subscription Service

AWSTemplateFormatVersion: '2010-09-09'
Description: 'EEMP Subscriber Event Bus'

Resources:
  EEMPSubscriberEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: eemp-subscriber-bus

  EEMPEventBusPolicy:
    Type: AWS::Events::EventBusPolicy
    Properties:
      EventBusName: !Ref EEMPSubscriberEventBus
      StatementId: AllowPlatformAccountPutEvents
      Statement:
        Effect: Allow
        Principal:
          AWS: !Sub 'arn:aws:iam::${PlatformAccountId}:root'
        Action: events:PutEvents
        Resource: !GetAtt EEMPSubscriberEventBus.Arn

Parameters:
  PlatformAccountId:
    Type: String
    Default: '111111111111'
    Description: Platform Account ID

Outputs:
  EventBusArn:
    Value: !GetAtt EEMPSubscriberEventBus.Arn
    Export:
      Name: EEMPSubscriberEventBusArn
```

---

## 6. Governance Layer Architecture

### 6.1 Publisher Service (Lambda)

**Serverless Publisher Architecture:**

```
Publisher Application (any AWS account)
    │
    │ POST /api/v1/events/publish
    │ {
    │   "event_type": "user.created",
    │   "payload": { ... }
    │ }
    │
    ▼
API Gateway (Platform Account)
    │
    │ Invokes Lambda (async)
    │
    ▼
Publisher Lambda Function
    │
    ├──> 1. Validate event_type exists in catalog
    │         │
    │         └──> NOT FOUND → Return 400 Bad Request
    │
    ├──> 2. Fetch schema from Schema Registry
    │
    ├──> 3. Validate payload against schema (JSONSchema validation)
    │         │
    │         └──> INVALID → Return 400 Bad Request
    │
    ├──> 4. Enrich event with standard envelope
    │         - event_id (UUID)
    │         - correlation_id
    │         - actor, source metadata
    │
    ├──> 5. Call EventBridge PutEvents API
    │         │
    │         └──> FAILURE → Retry (3x), then DLQ
    │
    └──> 6. Log audit entry (event published)
         │
         └──> Return 200 OK
```

**Lambda Function Code (Simplified):**

```typescript
// publisher-lambda/index.ts
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const eventBridgeClient = new EventBridgeClient({ region: 'us-east-1' });

export async function handler(event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> {
  const { event_type, payload, correlation_id, metadata } = JSON.parse(event.body);

  // 1. Validate event_type exists
  const eventConfig = await getEventConfig(event_type);
  if (!eventConfig) {
    return { statusCode: 400, body: 'Unknown event type' };
  }

  // 2. Fetch and validate schema
  const schema = await getSchemaFromRegistry(event_type);
  const valid = await validatePayload(payload, schema);
  if (!valid) {
    return { statusCode: 400, body: 'Schema validation failed' };
  }

  // 3. Enrich event
  const enrichedEvent = {
    event_id: uuid.v4(),
    event_type,
    schema_version: eventConfig.schema_version,
    correlation_id: correlation_id || uuid.v4(),
    timestamp: new Date().toISOString(),
    actor: extractActorFromContext(event),
    source: extractSourceMetadata(),
    payload,
    metadata: metadata || {}
  };

  // 4. Publish to EventBridge
  const putEventsCommand = new PutEventsCommand({
    Entries: [
      {
        Source: 'com.company.eemp',
        DetailType: event_type,
        Detail: JSON.stringify(enrichedEvent),
        EventBusName: 'eemp-platform-events',
        Time: new Date()
      }
    ]
  });

  const result = await eventBridgeClient.send(putEventsCommand);

  // 5. Check for failures
  if (result.FailedEntryCount > 0) {
    console.error('EventBridge PutEvents failed:', result.Entries[0].ErrorCode);
    return { statusCode: 500, body: 'Failed to publish event' };
  }

  // 6. Audit log
  await logAuditEvent({
    type: 'event.published',
    event_type,
    event_id: enrichedEvent.event_id,
    publisher: enrichedEvent.actor.id
  });

  return {
    statusCode: 200,
    body: JSON.stringify({
      event_id: enrichedEvent.event_id,
      eventbridge_entry_id: result.Entries[0].EventId
    })
  };
}
```

**Benefits of Lambda Publisher vs. ECS:**
- **Cost:** Pay only when events published (no idle costs)
- **Scalability:** Auto-scales to thousands of concurrent requests
- **Operational Simplicity:** No container orchestration
- **Cold Start Mitigation:** Provisioned concurrency (optional) for latency-sensitive publishers

**Trade-offs:**
- **Cold Starts:** Initial request may have 100-500ms latency (mitigated with provisioned concurrency)
- **Execution Time Limit:** 15-minute max (acceptable for event publishing)

### 6.2 Subscription Service (ECS)

**Why ECS for Subscription Service (not Lambda)?**

- Long-running operations (CloudFormation StackSet deployment can take 2-5 minutes)
- Complex orchestration logic (OPA evaluation → Rule creation → Cross-account setup)
- Persistent connections to OPA sidecar
- Better suited for API server pattern

**Subscription Workflow:**

```
1. Subscriber Request
    │
    ▼
2. OPA Policy Evaluation (via sidecar)
    │
    ├──> DENIED → 403 Forbidden
    │
    └──> APPROVED
         │
         ▼
3. Check if Application Account Event Bus exists
    │
    ├──> NOT EXISTS → Deploy CloudFormation StackSet
    │                  - Create Event Bus
    │                  - Add Resource Policy
    │                  - Wait for completion (2-5 min)
    │
    └──> EXISTS → Continue
         │
         ▼
4. Create EventBridge Rule in Platform Account
    │
    ├──> Event Pattern: Match subscribed event_types
    ├──> Target: Application Account Event Bus ARN
    ├──> Role: EventBridge cross-account role
    │
    ▼
5. Store Subscription in Database
    │
    ├──> subscription_id
    ├──> application_id
    ├──> aws_account_id
    ├──> subscribed_event_types
    ├──> eventbridge_rule_arn
    ├──> status: "active"
    │
    ▼
6. Log Audit Entry
    │
    └──> Return 201 Created
         {
           "subscription_id": "uuid",
           "event_bus_arn": "arn:aws:events:...",
           "status": "active"
         }
```

**Cross-Account Setup via CloudFormation StackSet:**

```typescript
import { CloudFormationClient, CreateStackInstancesCommand } from '@aws-sdk/client-cloudformation';

async function setupSubscriberEventBus(accountId: string): Promise<string> {
  const cfnClient = new CloudFormationClient({ region: 'us-east-1' });

  // Deploy StackSet to Application Account
  const command = new CreateStackInstancesCommand({
    StackSetName: 'EEMPSubscriberEventBus',
    Accounts: [accountId],
    Regions: ['us-east-1'],
    ParameterOverrides: [
      {
        ParameterKey: 'PlatformAccountId',
        ParameterValue: '111111111111'
      }
    ]
  });

  const result = await cfnClient.send(command);

  // Wait for stack creation (2-5 minutes)
  await waitForStackSetCompletion(result.OperationId);

  // Return Event Bus ARN
  return `arn:aws:events:us-east-1:${accountId}:event-bus/eemp-subscriber-bus`;
}
```

### 6.3 Audit Service (Lambda + Kinesis Firehose)

**EventBridge → Audit Trail Flow:**

```
EventBridge (eemp-platform-events)
    │
    │ All events matched by audit rule
    │
    ▼
EventBridge Rule: audit-all-events
    │
    ├──> Target 1: Lambda (audit-enrichment)
    │       │
    │       └──> Enrich with actor details, policy context
    │            └──> Write to S3 (via Kinesis Firehose)
    │            └──> Write to PostgreSQL (audit index)
    │
    └──> Target 2: Kinesis Firehose (direct S3 archival)
             │
             └──> S3 Bucket: s3://eemp-audit-trail/year=2025/month=12/day=10/
```

**Audit Lambda (Enrichment):**

```typescript
export async function handler(event: EventBridgeEvent): Promise<void> {
  // Event is already in EventBridge envelope format
  const auditEntry = {
    audit_id: uuid.v4(),
    event_id: event.detail.event_id,
    event_type: event['detail-type'],
    timestamp: event.time,
    correlation_id: event.detail.correlation_id,

    // Enrichment
    actor: await enrichActor(event.detail.actor),
    source: event.detail.source,

    // Policy context (if subscription event)
    policy_decision: event.detail.policy_decision,
    opa_decision_id: event.detail.opa_decision_id,

    // Compliance metadata
    compliance: {
      data_classification: event.detail.metadata.data_classification,
      pii_indicator: event.detail.metadata.pii_indicator,
      tenant_id: event.detail.metadata.tenant_id
    }
  };

  // Write to PostgreSQL for fast queries
  await writeToAuditDatabase(auditEntry);

  // Write to S3 via Firehose for long-term retention
  await firehose.putRecord({
    DeliveryStreamName: 'eemp-audit-stream',
    Record: { Data: JSON.stringify(auditEntry) }
  });
}
```

---

## 7. Data Flow Patterns

### 7.1 Event Publishing Flow

```
Publisher Application (Account 222222222222)
    │
    │ POST https://api.eemp.company.com/v1/events/publish
    │ {
    │   "event_type": "user.created",
    │   "payload": { "user_id": "123", "email": "user@example.com" }
    │ }
    │
    ▼
API Gateway (Platform Account)
    │
    ▼
Publisher Lambda
    │
    ├──> Validate schema
    ├──> Enrich event
    │
    ▼
EventBridge PutEvents API
    │
    ▼
EventBridge Bus: eemp-platform-events
    │
    │ Event Pattern Matching
    │
    ├──> Rule 1: route-to-account-222222222222
    │      └──> Target: Event Bus (Account 222222222222)
    │
    ├──> Rule 2: route-to-account-333333333333
    │      └──> Target: Event Bus (Account 333333333333)
    │
    └──> Rule 3: audit-all-events
         └──> Target: Audit Lambda + Firehose
```

### 7.2 Event Consumption Flow

```
EventBridge (Account 222222222222: eemp-subscriber-bus)
    │
    │ Receives event from Platform Account
    │
    ▼
Local EventBridge Rule: process-user-events
    │
    │ Event Pattern: { "detail-type": ["user.created"] }
    │
    ├──> Target 1: Lambda (notifications-handler)
    │       │
    │       └──> Send notification to user
    │
    └──> Target 2: SQS (user-events-queue)
             │
             └──> EC2 Application polls SQS
                  └──> Process event (update cache, database, etc.)
```

### 7.3 Subscription Request Flow

```
Developer (Account 222222222222)
    │
    │ Authenticate to EEMP Portal (OAuth)
    │
    ▼
POST /api/v1/subscriptions
{
  "application_id": "notifications-service",
  "aws_account_id": "222222222222",
  "event_types": ["user.created", "user.updated"]
}
    │
    ▼
Subscription Service (ECS)
    │
    ├──> 1. OPA Policy Evaluation
    │         └──> APPROVED
    │
    ├──> 2. Check Event Bus in Account 222222222222
    │         └──> NOT EXISTS → Deploy CloudFormation StackSet
    │              └──> Wait 2-5 minutes
    │              └──> Event Bus Created
    │
    ├──> 3. Create EventBridge Rule (Platform Account)
    │         Name: route-to-222222222222-user-events
    │         Pattern: { "detail-type": ["user.created", "user.updated"] }
    │         Target: arn:aws:events:...:222222222222:event-bus/eemp-subscriber-bus
    │
    ├──> 4. Store Subscription in Database
    │
    └──> 5. Audit Log
    │
    ▼
201 Created
{
  "subscription_id": "uuid",
  "status": "active",
  "event_bus_arn": "arn:aws:events:us-east-1:222222222222:event-bus/eemp-subscriber-bus",
  "setup_instructions": {
    "next_steps": [
      "Create EventBridge rules in your account (222222222222) to route events to your targets",
      "Example: Route user.created events to Lambda or SQS"
    ]
  }
}
```

---

## 8. Cost Analysis: EventBridge vs. Kafka

### 8.1 EventBridge Pricing Model

**Pricing (as of 2025):**
- **Custom Events:** $1.00 per million events published
- **Cross-Account Events:** No additional cost
- **Schema Discovery:** Free (included)
- **Event Archive:** $0.10 per GB/month stored
- **Event Replay:** $0.10 per GB replayed

**Example Cost Calculation (MVP Scale):**

```
Assumptions:
- 10,000 events/day
- Average event size: 2 KB
- 30-day month

Events per month: 10,000 × 30 = 300,000 events
Data volume: 300,000 × 2 KB = 600 MB

Cost Breakdown:
- Event publishing: 300,000 / 1,000,000 × $1.00 = $0.30
- Archive storage (90 days): 600 MB × 3 × $0.10/GB = $0.18
- Schema Registry: $0 (free)
- CloudWatch Logs (audit): ~$5/month

Total EventBridge Cost: ~$6/month
```

**Scaling Analysis:**

| Events/Day | Events/Month | EventBridge Cost | Confluent Cloud Cost |
|------------|--------------|------------------|----------------------|
| 10K | 300K | **$6/month** | $1,200/month (2 CKU minimum) |
| 100K | 3M | **$8/month** | $1,200/month |
| 1M | 30M | **$35/month** | $1,200/month |
| 10M | 300M | **$305/month** | $1,500/month (3 CKU) |
| 100M | 3B | **$3,005/month** | $3,000/month (10 CKU) |

**Break-Even Point:** ~100M events/month (3.3M events/day)

**Key Insight:** EventBridge is **dramatically cheaper** for low-to-moderate event volumes (< 10M events/day), but costs scale linearly with volume. Kafka has high fixed cost but better unit economics at scale.

### 8.2 Total Cost of Ownership (TCO) Comparison

**EventBridge TCO (MVP - 10K events/day):**

```
Infrastructure Costs:
- EventBridge: $6/month
- Lambda (Publisher): $5/month (100K invocations × 512MB × 200ms)
- API Gateway: $10/month (300K requests)
- ECS (Subscription Service): $30/month (1 Fargate task)
- RDS PostgreSQL: $50/month (db.t3.small)
- S3 (Audit Archive): $5/month
- CloudWatch: $10/month

Total Infrastructure: $116/month ($1,392/year)

Operational Costs:
- Platform engineering: ~5 hours/month (serverless, minimal ops)
- Cost: $1,000/month × 5% = $50/month

Total TCO: ~$166/month ($2,000/year)
```

**Confluent Cloud TCO (MVP - 10K events/day):**

```
Infrastructure Costs:
- Confluent Cloud: $1,200/month (2 CKU)
- ECS (Publisher Service): $30/month
- ECS (Subscription Service): $30/month
- RDS PostgreSQL: $50/month
- S3 (Audit Archive): $5/month
- CloudWatch: $10/month

Total Infrastructure: $1,325/month ($15,900/year)

Operational Costs:
- Platform engineering: ~10 hours/month (schema management, monitoring)
- Cost: $1,000/month × 10% = $100/month

Total TCO: ~$1,425/month ($17,100/year)
```

**TCO Comparison Summary:**

| Event Volume | EventBridge TCO/Year | Confluent Cloud TCO/Year | Savings (EventBridge) |
|--------------|----------------------|--------------------------|----------------------|
| 10K/day | **$2,000** | $17,100 | **$15,100 (88% cheaper)** |
| 100K/day | **$2,500** | $17,100 | **$14,600 (85% cheaper)** |
| 1M/day | **$6,000** | $17,100 | **$11,100 (65% cheaper)** |
| 10M/day | **$45,000** | $20,000 | -$25,000 (Kafka cheaper) |

**Conclusion:** EventBridge is **significantly cheaper** for MVP and moderate scale (< 1M events/day). Kafka becomes cost-competitive at high scale (> 10M events/day).

---

## 9. Cross-Account and On-Premises Considerations

### 9.1 Cross-Account EventBridge (AWS Accounts)

**Native Cross-Account Support:**

EventBridge provides **first-class cross-account event delivery**, making it simpler than Kafka for distributed AWS architectures.

**Setup Steps (Automated by EEMP):**

```
1. Application Account (222222222222):
   - Create Custom Event Bus: eemp-subscriber-bus
   - Add Resource Policy: Allow Platform Account (111111111111) to PutEvents

2. Platform Account (111111111111):
   - Create EventBridge Rule: route-to-222222222222
   - Event Pattern: Match subscribed event types
   - Target: arn:aws:events:...:222222222222:event-bus/eemp-subscriber-bus
   - Role: IAM role with events:PutEvents permission

3. Application Account (222222222222):
   - Create local EventBridge Rules to route events to Lambda, SQS, etc.
```

**Benefits Over Kafka:**
- **No Credentials:** IAM-based authentication (no API keys to rotate)
- **No Network Complexity:** Uses AWS backbone (no internet/PrivateLink setup)
- **No Cost:** Cross-account delivery is free
- **Fast Setup:** 2-step process (resource policy + rule creation)

### 9.2 On-Premises Integration

**Challenge:** EventBridge does not support on-premises subscribers directly (unlike Kafka which is internet-accessible).

**Solution: Hybrid Architecture with SQS Bridge**

```
┌─────────────────────────────────────────────────────────────────┐
│  Platform Account (AWS)                                         │
│                                                                  │
│  EventBridge Rule: route-to-onprem-queue                        │
│  - Event Pattern: Match events for on-prem subscribers          │
│  - Target: SQS Queue (onprem-events-queue)                      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  SQS Queue: onprem-events-queue                        │    │
│  │  - Retention: 14 days                                  │    │
│  │  - DLQ: onprem-events-dlq                              │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                     │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            │ HTTPS (AWS SDK)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  On-Premises Data Center                                        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  SQS Polling Application                               │    │
│  │  - AWS SDK for SQS (ReceiveMessage API)               │    │
│  │  - Long polling (20-second wait time)                  │    │
│  │  - Processes events, DeleteMessage on success          │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                     │
│                           ▼                                     │
│  On-Prem Application Logic                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Configuration:**

```python
# On-premises SQS consumer (Python example)
import boto3
import json

sqs = boto3.client('sqs', region_name='us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/111111111111/onprem-events-queue'

def poll_events():
    while True:
        # Long polling (20-second wait)
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,
            MessageAttributeNames=['All']
        )

        if 'Messages' not in response:
            continue

        for message in response['Messages']:
            # Parse EventBridge event
            event = json.loads(message['Body'])
            event_type = event['detail-type']
            payload = event['detail']['payload']

            # Process event
            process_event(event_type, payload)

            # Delete message from queue
            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=message['ReceiptHandle']
            )

if __name__ == '__main__':
    poll_events()
```

**Pros:**
- **Standard AWS SDK:** On-prem app uses familiar SQS polling pattern
- **No Special Firewall Rules:** Outbound HTTPS (443) to SQS endpoints
- **Reliable Delivery:** SQS provides at-least-once delivery with DLQ for failures
- **Cost-Effective:** SQS pricing is low (first 1M requests/month free)

**Cons:**
- **Polling Overhead:** On-prem app must continuously poll SQS
- **Latency:** SQS polling adds 1-20 seconds latency (depends on polling interval)
- **Not True Push:** Unlike Kafka consumer which receives events in near real-time

**Alternative: EventBridge API Destinations (for HTTP endpoints)**

If on-prem application exposes an HTTP endpoint:

```
EventBridge Rule: route-to-onprem-http
- Target: API Destination
- Endpoint: https://onprem.company.com/events
- Authentication: OAuth or API Key
```

**Pros:** True push model (EventBridge calls on-prem HTTP endpoint)
**Cons:** Requires on-prem endpoint to be internet-accessible (firewall holes)

### 9.3 Cross-Account Cost Allocation

**EventBridge Advantage:** AWS native cost allocation tags work seamlessly.

```yaml
# EventBridge Rule tags (Platform Account)
Tags:
  - Key: CostCenter
    Value: Application-Team-A
  - Key: Subscriber
    Value: account-222222222222
```

**Cost Allocation Dashboard (AWS Cost Explorer):**
- Filter by tag: `Subscriber = account-222222222222`
- View EventBridge costs attributed to specific subscriber accounts
- Chargeback monthly based on event volume

**Comparison to Kafka:**
- Kafka requires custom usage tracking and chargeback calculation
- EventBridge uses native AWS billing tags

---

## 10. Migration Strategy

### 10.1 Migration from On-Prem Kafka to EventBridge

**Phase 1: Dual-Publish (Week 1-2)**

```
Publisher Application
    │
    ├──> On-Prem Kafka (existing consumers)
    │
    └──> EEMP Publisher API → EventBridge (new consumers)
```

- Publishers send events to both on-prem Kafka and EventBridge
- Validate event parity (compare event content)

**Phase 2: Consumer Migration (Week 3-4)**

- Migrate consumers to EventBridge (via SQS or Lambda targets)
- Monitor consumer lag and error rates
- Rollback capability: Switch back to Kafka if issues

**Phase 3: Publisher Cutover (Week 5)**

- Stop publishing to on-prem Kafka
- All events flow through EventBridge

**Phase 4: On-Prem Decommission (Week 6)**

- Archive on-prem Kafka data for compliance
- Decommission on-prem Kafka cluster

**Migration Validation:**
- Event content comparison (on-prem vs. EventBridge)
- Latency comparison (p99 < 200ms acceptable)
- Zero data loss during cutover

---

## 11. Architectural Decisions Record (ADR)

### ADR-001: EventBridge over Confluent Cloud Kafka

**Decision:** Use Amazon EventBridge as the event infrastructure for EEMP instead of Confluent Cloud Kafka.

**Context:**
- Event volume is moderate (< 1M events/day for MVP)
- Organization is AWS-centric (90% of workloads on AWS)
- Cross-account event distribution is critical requirement
- Team lacks deep Kafka expertise
- Cost optimization is priority for MVP

**Rationale:**

1. **Cost:** EventBridge is 88% cheaper at MVP scale ($2K/year vs. $17K/year)
2. **Cross-Account Simplicity:** Native EventBridge cross-account event bus vs. complex Kafka API key management
3. **Operational Overhead:** True serverless (zero ops) vs. managed cluster monitoring
4. **AWS Integration:** First-class integration with Lambda, SQS, Step Functions
5. **Team Expertise:** AWS-native services easier for current team to operate

**Consequences:**

**Pros:**
- Dramatically lower cost for MVP and moderate scale
- Simple cross-account setup (IAM-based, no credentials)
- Serverless architecture (no cluster management)
- Fast time-to-market (less complexity)

**Cons:**
- No event ordering guarantees (EventBridge is best-effort)
- Weaker schema governance (no backward compatibility enforcement)
- Limited to AWS ecosystem (can't support multi-cloud easily)
- Higher cost at very high scale (> 10M events/day)

**Mitigation:**
- For event ordering: Use separate event buses per partition key if needed (post-MVP)
- For schema governance: Build custom compatibility checking in Publisher Lambda
- For multi-cloud: EventBridge is AWS-only (if multi-cloud becomes requirement, migrate to Kafka)
- For cost at scale: Monitor event volume; migrate to Kafka if > 10M events/day

**Validation Criteria (90 days):**
- Event volume stays < 1M events/day
- No critical failures due to lack of event ordering
- Schema governance issues < 5/month (manual validation acceptable)
- Cost stays < $500/month
- Platform team ops time < 10 hours/month

**Reversal Criteria:**
If event volume exceeds 10M events/day or event ordering becomes critical, migrate to Confluent Cloud Kafka.

---

### ADR-002: SQS Bridge for On-Premises Subscribers

**Decision:** Use SQS queues as bridge for on-premises event subscribers instead of exposing EventBridge directly or using Kafka.

**Rationale:**
- EventBridge cannot deliver events to on-prem directly
- SQS polling is well-understood pattern for on-prem applications
- Avoids complex firewall rules (only outbound HTTPS required)
- SQS provides reliable delivery with DLQ for failures

**Consequences:**
- On-prem apps must poll SQS (1-20 second latency)
- Additional SQS cost (~$0.50/month per on-prem subscriber)
- Not true push model like Kafka consumers

**Alternatives Considered:**
- Kafka (Confluent Cloud): Better for on-prem (true push), but adds cost and complexity
- API Destinations: Requires internet-accessible on-prem HTTP endpoint (security concern)

---

### ADR-003: Lambda for Publisher Service

**Decision:** Use AWS Lambda for Publisher Service instead of ECS Fargate.

**Rationale:**
- Event publishing is short-lived operation (< 1 second)
- Serverless cost model better for variable load
- Auto-scaling built-in (no capacity planning)

**Consequences:**
- Cold starts: 100-500ms latency on first request (mitigated with provisioned concurrency if needed)
- 15-minute execution limit (acceptable for event publishing)

**Alternatives Considered:**
- ECS Fargate: Better for long-running, predictable load (not applicable for publishing)

---

## 12. Future Considerations

### Post-MVP Enhancements

**Phase 2 (3-6 months post-MVP):**

1. **EventBridge Pipes:**
   - Stream processing with filtering, transformation, enrichment
   - Alternative to Kafka Streams for simple use cases

2. **Event Ordering (if required):**
   - Partition events by key (e.g., user_id) across multiple event buses
   - Consumer reads from specific bus based on partition key

3. **Schema Compatibility Validation:**
   - Build custom schema diffing and validation service
   - Integrate with Publisher Lambda to enforce backward compatibility

4. **Multi-Region Deployment:**
   - EventBridge global endpoints (preview feature)
   - Cross-region event replication for disaster recovery

**Phase 3 (6-12 months post-MVP):**

1. **Hybrid Kafka + EventBridge:**
   - High-throughput events → Kafka
   - Cross-account distribution → EventBridge
   - Use EventBridge as router, Kafka as stream processor

2. **Event Sourcing:**
   - Use EventBridge Archive for event sourcing pattern
   - Replay events to rebuild state

3. **AI-Powered Insights:**
   - Use EventBridge + Kinesis Data Analytics for anomaly detection
   - Schema suggestions based on usage patterns

---

## 13. Comparison Summary: EventBridge vs. Confluent Cloud

### When to Choose EventBridge

✓ **Event volume < 10M events/day**
✓ **AWS-centric architecture (80%+ on AWS)**
✓ **Cross-account distribution is primary use case**
✓ **Serverless preference**
✓ **Cost optimization priority**
✓ **Team lacks Kafka expertise**
✓ **Event ordering not critical**

### When to Choose Confluent Cloud Kafka

✓ **Event volume > 10M events/day**
✓ **Event ordering is critical**
✓ **Need streaming patterns (windowing, aggregation)**
✓ **Require mature schema governance (backward compatibility)**
✓ **Multi-cloud or hybrid cloud architecture**
✓ **Team has Kafka expertise**
✓ **Need Kafka ecosystem (ksqlDB, Kafka Streams)**

### Hybrid Approach (Future)

For very large deployments:
- **EventBridge:** Cross-account routing and AWS service integration
- **Kafka:** High-throughput streaming and complex event processing
- **Pattern:** EventBridge routes to Kafka for processing, results published back to EventBridge

---

## 14. Appendix

### A. Technology Choices Summary

| Category | Technology | Justification |
|----------|-----------|---------------|
| **Event Infrastructure** | Amazon EventBridge | Serverless, AWS-native, cross-account simplicity |
| **Schema Management** | EventBridge Schema Registry | Free, auto-discovery, AWS-native |
| **Publisher Service** | AWS Lambda | Serverless, cost-effective for variable load |
| **Subscription Service** | ECS Fargate | Long-running orchestration, OPA integration |
| **Cross-Account Delivery** | EventBridge Native | IAM-based, no credentials, free |
| **On-Prem Bridge** | Amazon SQS | Polling pattern, reliable delivery, familiar |
| **Audit Trail** | Lambda + Kinesis Firehose → S3 | Serverless pipeline, long-term retention |
| **Event Archive** | EventBridge Archive | Native replay capability, 90-day retention |
| **Deduplication** | DynamoDB | Fast idempotency checks |

### B. Cost Comparison Table

| Scale | Events/Day | EventBridge/Year | Confluent Cloud/Year | Winner |
|-------|------------|------------------|----------------------|--------|
| MVP | 10K | **$2,000** | $17,000 | EventBridge (88% cheaper) |
| Small | 100K | **$2,500** | $17,000 | EventBridge (85% cheaper) |
| Medium | 1M | **$6,000** | $17,000 | EventBridge (65% cheaper) |
| Large | 10M | $45,000 | **$20,000** | Kafka (55% cheaper) |
| Very Large | 100M | $360,000 | **$40,000** | Kafka (89% cheaper) |

**Break-even:** ~30M events/day (~1M events/month)

---

**Document End**
