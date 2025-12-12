# EEMP Audit Trail Service - Event Flow Diagram

## Platform Events to Immutable Audit Trail

```mermaid
flowchart TB
    subgraph "Event Sources"
        PS[Publisher Service]
        SS[Subscription Service]
        GS[Governance Service]
        API[API Gateway]
    end

    subgraph "Event Capture Layer"
        PS --> |1. Publish event.published| KDS[Kinesis Data Stream<br/>audit-events-stream]
        SS --> |2. Subscribe event.consumed| KDS
        GS --> |3. Policy event.policy.evaluated| KDS
        API --> |4. API calls api.request.received| KDS
    end

    subgraph "Real-Time Enrichment Pipeline"
        KDS --> |5. Stream events| LE[Lambda Enrichment Function<br/>Batch: 100 events/sec]

        LE --> |6a. Lookup user details| UC[(User Cache<br/>DynamoDB)]
        LE --> |6b. Lookup app details| AC[(Application Cache<br/>DynamoDB)]
        LE --> |6c. Lookup tenant details| TC[(Tenant Cache<br/>DynamoDB)]

        UC -.-> |User metadata| LE
        AC -.-> |App metadata| LE
        TC -.-> |Tenant metadata| LE
    end

    subgraph "Immutable Storage Layer"
        LE --> |7. Write enriched audit| FH[Kinesis Firehose<br/>Buffer: 5MB or 60s]

        FH --> |8. Batch write<br/>Parquet format| S3HOT[S3 Hot Tier<br/>eemp-audit-trail/hot/<br/>year=2025/month=12/day=10/hour=10/<br/>audit-records-{timestamp}.parquet]

        S3HOT --> |9. Object Lock ENABLED<br/>Retention: 90 days<br/>Mode: COMPLIANCE| LOCK[🔒 Immutable Records<br/>Cannot be deleted or modified]
    end

    subgraph "Fast Query Index"
        FH --> |10. Parallel write| DDB[(DynamoDB Index<br/>audit-events-index<br/><br/>PK: audit_id<br/>GSI: actor_id<br/>GSI: event_type<br/>GSI: correlation_id)]

        DDB --> |11. TTL 90 days| EXPIRE[Auto-expire to S3]
    end

    subgraph "Integrity Verification"
        S3HOT --> |12. Compute hash| HASH[Lambda Hash Validator<br/>SHA-256 chain]
        HASH --> |13. Store hash| META[S3 Object Metadata<br/>record-hash<br/>previous-hash<br/>chain-position]
    end

    subgraph "Lifecycle Management"
        S3HOT --> |90 days| S3WARM[S3 Warm Tier<br/>Intelligent-Tiering]
        S3WARM --> |2 years| S3COLD[S3 Cold Tier<br/>Glacier Deep Archive]
        S3COLD --> |7 years| DELETE[Delete after retention]
    end

    subgraph "Query Layer"
        APP[Audit Query API]
        APP --> |Query recent<br/>0-90 days| DDB
        APP --> |Query historical<br/>90d-7yr| ATHENA[Athena<br/>SQL on S3 Parquet]
        ATHENA --> S3HOT
        ATHENA --> S3WARM
        ATHENA --> S3COLD
    end

    style LOCK fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    style S3HOT fill:#51cf66,stroke:#2f9e44,stroke-width:2px
    style S3WARM fill:#ffd43b,stroke:#f59f00,stroke-width:2px
    style S3COLD fill:#74c0fc,stroke:#1c7ed6,stroke-width:2px
    style KDS fill:#845ef7,stroke:#5f3dc4,color:#fff
    style DDB fill:#ff922b,stroke:#d9480f
```

## Event Flow Steps

### 1. Event Capture (Steps 1-4)
- **Platform services** emit audit events to Kinesis Data Stream
- **Event types**: `event.published`, `event.consumed`, `policy.evaluated`, `api.request.received`
- **Kinesis shard**: Partition by `actor_id` for ordering guarantee

### 2. Real-Time Enrichment (Steps 5-6)
- **Lambda function** processes events in batches (100 events/sec)
- **Enrichment sources**:
  - User Cache: Add user name, email, department, manager
  - Application Cache: Add app name, owner, purpose
  - Tenant Cache: Add tenant name, industry, region
- **Output**: Fully enriched audit record with domain context

### 3. Immutable Storage (Steps 7-9)
- **Kinesis Firehose** buffers events (5MB or 60 seconds)
- **Parquet format**: Columnar storage, 10x compression
- **S3 Object Lock**: Immutable for 90 days (COMPLIANCE mode)
  - ✅ Cannot be deleted by anyone (including root account)
  - ✅ Cannot be modified or overwritten
  - ✅ Meets regulatory requirements (SOX, SEC Rule 17a-4)

### 4. Fast Query Index (Steps 10-11)
- **DynamoDB** stores lightweight index for fast queries
- **GSI indexes**: Query by actor, event type, correlation ID
- **TTL**: Auto-expire after 90 days (historical queries use Athena)

### 5. Integrity Verification (Steps 12-13)
- **Hash chain**: Each record links to previous record hash
- **Tamper detection**: Any modification breaks the chain
- **S3 metadata**: Store hash for verification

### 6. Lifecycle Management
- **Hot Tier (0-90d)**: S3 Standard, DynamoDB index, fast queries
- **Warm Tier (90d-2yr)**: S3 Intelligent-Tiering, Athena queries
- **Cold Tier (2-7yr)**: Glacier Deep Archive, compliance retention
- **Deletion (>7yr)**: Automatic after retention period

## Key Features

### Immutability Guarantees
```yaml
s3_bucket_configuration:
  object_lock_enabled: true
  object_lock_configuration:
    mode: COMPLIANCE  # Cannot be overridden by anyone
    retention:
      days: 90  # Hot tier retention
      years: 7  # Total retention via lifecycle
    legal_hold: false
```

### Event Ordering
```yaml
kinesis_configuration:
  partition_key: actor_id  # All events from same actor in order
  shard_count: 2  # Auto-scaling based on throughput
  retention_hours: 168  # 7 days replay capability
```

### Audit Record Schema
```json
{
  "audit_id": "uuid-1234",
  "timestamp": "2025-12-10T10:30:00.123Z",
  "event_type": "event.published",
  "actor": {
    "id": "user-12345",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "department": "Engineering"
  },
  "event": {
    "event_id": "evt-uuid-5678",
    "correlation_id": "trace-abc-123",
    "event_type": "user.created"
  },
  "outcome": "success",
  "integrity": {
    "record_hash": "sha256:abc123...",
    "previous_hash": "sha256:def456...",
    "chain_position": 12345
  },
  "storage": {
    "s3_object_key": "s3://eemp-audit-trail/hot/year=2025/month=12/day=10/hour=10/audit-records-1702198200.parquet",
    "immutable_until": "2025-03-10T10:30:00Z"
  }
}
```

## Compliance Benefits

| Requirement | Solution |
|-------------|----------|
| **Immutability** | S3 Object Lock (COMPLIANCE mode) - cannot delete/modify |
| **Tamper Detection** | SHA-256 hash chain - any change breaks chain |
| **Audit Trail** | Every platform event captured with full context |
| **Retention** | 7-year retention with automatic lifecycle management |
| **Discovery** | DynamoDB + Athena - query by actor, event type, time |
| **Enrichment** | Real-time context: user, app, tenant metadata |
| **Ordering** | Kinesis partition key ensures event order per actor |
| **Availability** | 99.99% SLA (S3 + DynamoDB + Kinesis) |

---

**Generated**: 2025-12-10
**Platform**: AWS
**Compliance**: SOX, GDPR, PCI-DSS, SEC Rule 17a-4
```
