# Enterprise Event Management Platform (EEMP) - Custom Audit Trail Architecture

**Version:** 1.0
**Date:** 2025-12-10
**Status:** Compliance Architecture
**Author:** Architecture Team

---

## Executive Summary

This document defines a **custom audit trail architecture** for the Enterprise Event Management Platform (EEMP) that provides comprehensive, immutable, and discoverable audit logging for all platform events. This architecture is designed for financial institutions where **AWS CloudTrail is not permitted** due to regulatory or corporate policy constraints.

The solution leverages **Amazon S3** as the primary audit storage layer, combined with enrichment pipelines, indexing mechanisms, and query capabilities to enable:
- **Immutable audit trail** for regulatory compliance (SOX, GDPR, banking regulations)
- **Domain-specific enrichment** with business context (user details, account information, etc.)
- **Fast discoverability** through multiple access patterns (time-based, event-type, actor, correlation)
- **Long-term retention** (7+ years) with cost-optimized storage tiers
- **Audit query API** for compliance reporting and forensic investigation

### Architectural Principles

1. **Immutability:** Once written, audit records cannot be modified or deleted (S3 Object Lock)
2. **Completeness:** Every platform event and policy decision must be audited
3. **Enrichment:** Audit records include full business context, not just technical metadata
4. **Discoverability:** Multiple query patterns supported (time, event type, actor, correlation ID)
5. **Compliance-First:** Meet financial services regulatory requirements (SOX, GDPR, etc.)
6. **Cost Optimization:** Intelligent tiering for long-term retention at minimal cost
7. **Performance:** Sub-second query response for recent data, minutes for historical

---

## 1. Audit Trail Requirements

### 1.1 Regulatory Requirements

**SOX (Sarbanes-Oxley) Compliance:**
- Immutable audit trail of all financial data access and modifications
- 7-year retention period
- Ability to prove "who accessed what, when, and why"
- Audit trail must be tamper-proof (no deletion or modification)

**GDPR (General Data Protection Regulation):**
- Audit trail of all PII access and processing activities
- Right to be forgotten: Ability to identify and report all PII access for specific individuals
- Cross-border data transfer tracking
- Data breach notification: Ability to identify affected records within 72 hours

**Banking Regulations (e.g., PCI-DSS, FFIEC):**
- Audit trail of all payment card data access
- Log all authentication attempts and authorization decisions
- Retain logs for regulatory examination (typically 7 years)
- Provide audit reports within hours for regulatory inquiries

**Internal Audit Requirements:**
- Track all subscription access requests and policy decisions
- Monitor for anomalous access patterns
- Support forensic investigation for security incidents
- Demonstrate segregation of duties

### 1.2 Audit Event Categories

**1. Platform Events (Business Events):**
- Event published: `event.published`
- Event consumed: `event.consumed` (optional, high volume)
- Schema registered: `schema.registered`
- Schema evolved: `schema.evolved`

**2. Governance Events:**
- Subscription requested: `subscription.requested`
- Subscription approved: `subscription.approved`
- Subscription denied: `subscription.denied`
- Subscription revoked: `subscription.revoked`
- Policy evaluated: `policy.evaluated`

**3. Security Events:**
- Authentication success: `auth.success`
- Authentication failure: `auth.failure`
- Authorization denied: `authz.denied`
- API key rotated: `apikey.rotated`
- Suspicious activity detected: `security.anomaly`

**4. Administrative Events:**
- Configuration changed: `config.changed`
- User role modified: `user.role.changed`
- Event type deprecated: `eventtype.deprecated`

### 1.3 Audit Data Model

**Standard Audit Record Structure:**

```json
{
  // Core Identification
  "audit_id": "uuid-1234-5678-9012",
  "timestamp": "2025-12-10T10:30:00.123Z",
  "event_type": "event.published",
  "version": "1.0",

  // Actor Information (WHO)
  "actor": {
    "type": "user | service | system",
    "id": "user-12345 | service-notifications | system",
    "name": "John Doe | Notifications Service",
    "email": "john.doe@company.com",
    "ip_address": "203.0.113.45",
    "user_agent": "Mozilla/5.0...",
    "session_id": "sess-abc-123",
    "authentication_method": "oauth | api_key | mTLS"
  },

  // Source Information (WHERE)
  "source": {
    "system": "user-profile-service",
    "component": "publisher-lambda",
    "aws_account_id": "222222222222",
    "aws_region": "us-east-1",
    "environment": "production | staging | dev",
    "version": "v2.3.1"
  },

  // Event Details (WHAT)
  "event": {
    "event_id": "evt-uuid-9876",
    "event_type": "user.created",
    "schema_version": "1.0",
    "correlation_id": "trace-abc-123",
    "parent_event_id": "evt-uuid-5555",  // For event chains
    "payload_hash": "sha256:abc123...",   // For integrity verification
    "payload_size_bytes": 2048,
    "data_classification": 2,
    "pii_indicator": true,
    "tenant_id": "tenant-001"
  },

  // Outcome (RESULT)
  "outcome": {
    "status": "success | failure | denied",
    "status_code": 200,
    "error_message": "Schema validation failed: missing required field 'email'",
    "error_code": "SCHEMA_VALIDATION_ERROR",
    "duration_ms": 45
  },

  // Policy Decision (for authorization events)
  "policy": {
    "decision": "allow | deny",
    "decision_id": "policy-uuid-7890",
    "policy_version": "1.2.0",
    "rules_evaluated": [
      {"rule": "data_classification", "result": true},
      {"rule": "tenant_isolation", "result": true}
    ],
    "evaluation_duration_ms": 5
  },

  // Domain-Specific Enrichment (business context)
  "enrichment": {
    "user_details": {
      "department": "Engineering",
      "manager": "Jane Smith",
      "cost_center": "CC-1234"
    },
    "account_details": {
      "account_type": "premium",
      "region": "US-East"
    },
    "custom_metadata": {
      "business_unit": "Retail Banking",
      "regulatory_scope": ["SOX", "GDPR"]
    }
  },

  // Compliance Metadata
  "compliance": {
    "retention_years": 7,
    "regulatory_scope": ["SOX", "GDPR", "PCI-DSS"],
    "data_sensitivity": "confidential",
    "cross_border_transfer": false,
    "encryption_key_id": "arn:aws:kms:..."
  },

  // Integrity & Provenance
  "integrity": {
    "record_hash": "sha256:def456...",  // Hash of entire record
    "previous_record_hash": "sha256:abc123...",  // Blockchain-style chaining
    "signature": "RSA-SHA256:...",       // Digital signature (optional)
    "written_at": "2025-12-10T10:30:00.456Z",
    "s3_object_key": "year=2025/month=12/day=10/hour=10/audit-uuid-1234.json",
    "s3_etag": "abc123def456",
    "immutable_until": "2032-12-10T00:00:00Z"  // Object Lock retention
  }
}
```

---

## 2. S3-Based Audit Storage Architecture

### 2.1 S3 Bucket Design

**Multi-Tier Storage Strategy:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    S3 Audit Storage Architecture                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Hot Tier: S3 Standard (0-90 days)                         │    │
│  │  Bucket: eemp-audit-trail-hot                              │    │
│  │                                                             │    │
│  │  - Fast access (sub-second retrieval)                      │    │
│  │  - Frequent queries (compliance, debugging, monitoring)    │    │
│  │  - Object Lock: Governance mode (90 days)                  │    │
│  │  - Versioning: Enabled                                     │    │
│  │  - Encryption: SSE-KMS (customer-managed key)              │    │
│  │                                                             │    │
│  │  Partition Structure:                                      │    │
│  │    year=2025/month=12/day=10/hour=10/                      │    │
│  │      audit-{uuid}.json                                     │    │
│  │      audit-{uuid}.json                                     │    │
│  │      ...                                                    │    │
│  └────────────────────────────────────────────────────────────┘    │
│                           │                                         │
│                           │ Lifecycle Policy (90 days)              │
│                           ▼                                         │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Warm Tier: S3 Intelligent-Tiering (90 days - 2 years)    │    │
│  │  Bucket: eemp-audit-trail-warm                             │    │
│  │                                                             │    │
│  │  - Access within minutes (occasional queries)              │    │
│  │  - Automatic cost optimization (infrequent → archive)      │    │
│  │  - Object Lock: Compliance mode (2 years)                  │    │
│  │  - Same partition structure as Hot tier                    │    │
│  └────────────────────────────────────────────────────────────┘    │
│                           │                                         │
│                           │ Lifecycle Policy (2 years)              │
│                           ▼                                         │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Cold Tier: S3 Glacier Deep Archive (2-7+ years)           │    │
│  │  Bucket: eemp-audit-trail-cold                             │    │
│  │                                                             │    │
│  │  - Low-cost archival ($0.00099/GB/month)                   │    │
│  │  - Retrieval: 12-48 hours (for regulatory examination)     │    │
│  │  - Object Lock: Compliance mode (7+ years)                 │    │
│  │  - Legal Hold: Available for litigation                    │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Aggregated Archive: Parquet Format (parallel tier)        │    │
│  │  Bucket: eemp-audit-trail-analytics                        │    │
│  │                                                             │    │
│  │  - Daily aggregation: JSON → Parquet (compressed)          │    │
│  │  - Partition: year=2025/month=12/day=10/                   │    │
│  │      events.parquet                                        │    │
│  │  - Used for: Athena queries, compliance reports            │    │
│  │  - Cost: 10x compression vs. JSON                          │    │
│  │  - Retention: Same as Cold tier (7+ years)                 │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**S3 Bucket Configuration:**

```yaml
# Hot Tier Bucket (0-90 days)
hot_tier:
  bucket_name: eemp-audit-trail-hot
  region: us-east-1
  versioning: Enabled
  encryption:
    type: SSE-KMS
    kms_key_id: arn:aws:kms:us-east-1:111111111111:key/audit-key-id
  object_lock:
    mode: GOVERNANCE  # Can be removed by privileged users (break-glass)
    retention_days: 90
  lifecycle_policy:
    - transition_to_warm_tier_after_days: 90
  replication:
    destination: eemp-audit-trail-hot-replica (us-west-2)
    rule: Replicate all objects for disaster recovery

# Warm Tier Bucket (90 days - 2 years)
warm_tier:
  bucket_name: eemp-audit-trail-warm
  region: us-east-1
  storage_class: INTELLIGENT_TIERING
  object_lock:
    mode: COMPLIANCE  # Cannot be removed, even by root
    retention_days: 730  # 2 years
  lifecycle_policy:
    - transition_to_cold_tier_after_days: 730

# Cold Tier Bucket (2-7+ years)
cold_tier:
  bucket_name: eemp-audit-trail-cold
  region: us-east-1
  storage_class: GLACIER_DEEP_ARCHIVE
  object_lock:
    mode: COMPLIANCE
    retention_days: 2555  # 7 years
  legal_hold: Enabled  # For litigation

# Analytics Bucket (Parquet aggregation)
analytics_tier:
  bucket_name: eemp-audit-trail-analytics
  region: us-east-1
  storage_class: INTELLIGENT_TIERING
  format: Parquet (snappy compression)
  partition: year/month/day
```

### 2.2 Partition Strategy for Discoverability

**Time-Based Partitioning (Primary):**

```
s3://eemp-audit-trail-hot/
  year=2025/
    month=12/
      day=10/
        hour=10/
          audit-{uuid-1}.json
          audit-{uuid-2}.json
          ...
        hour=11/
          audit-{uuid-3}.json
          ...
      day=11/
        ...
    month=01/
      ...
  year=2026/
    ...
```

**Benefits:**
- **Time-Range Queries:** Easy to query "all events on 2025-12-10"
- **Lifecycle Management:** Automatic transition based on partition age
- **Cost Optimization:** Older partitions moved to cheaper storage
- **Athena Performance:** Partition pruning reduces scan cost

**Event-Type Hive Partitioning (Secondary Index via Metadata):**

For fast event-type queries without scanning entire dataset, use **S3 Object Tagging**:

```json
{
  "TagSet": [
    {"Key": "event_type", "Value": "event.published"},
    {"Key": "actor_type", "Value": "service"},
    {"Key": "data_classification", "Value": "2"},
    {"Key": "tenant_id", "Value": "tenant-001"}
  ]
}
```

**Alternative: Dual-Write Pattern** (if tag-based queries insufficient):

```
s3://eemp-audit-trail-hot/
  by-time/year=2025/month=12/day=10/hour=10/audit-{uuid}.json
  by-event-type/event_type=event.published/year=2025/month=12/day=10/audit-{uuid}.json
  by-actor/actor_type=service/actor_id=notifications-service/year=2025/audit-{uuid}.json
```

**Trade-off:** Increased storage cost (3x duplication) vs. faster queries

**Recommendation:** Start with time-based partitioning + S3 tags. If query performance insufficient, add dual-write for high-priority access patterns.

### 2.3 S3 Object Lock and Immutability

**Object Lock Configuration:**

```python
# Enable Object Lock on bucket creation (cannot be added later)
s3_client.create_bucket(
    Bucket='eemp-audit-trail-hot',
    ObjectLockEnabledForBucket=True
)

# Set default retention policy
s3_client.put_object_lock_configuration(
    Bucket='eemp-audit-trail-hot',
    ObjectLockConfiguration={
        'ObjectLockEnabled': 'Enabled',
        'Rule': {
            'DefaultRetention': {
                'Mode': 'GOVERNANCE',  # GOVERNANCE or COMPLIANCE
                'Days': 90
            }
        }
    }
)

# Write audit record with Object Lock
s3_client.put_object(
    Bucket='eemp-audit-trail-hot',
    Key='year=2025/month=12/day=10/hour=10/audit-uuid-1234.json',
    Body=json.dumps(audit_record),
    ObjectLockMode='GOVERNANCE',
    ObjectLockRetainUntilDate=datetime(2025, 3, 10)  # 90 days from now
)
```

**Object Lock Modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| **GOVERNANCE** | Can be removed by users with `s3:BypassGovernanceRetention` permission | Hot tier (0-90 days) - allows break-glass access |
| **COMPLIANCE** | Cannot be removed by anyone, including root account | Warm/Cold tiers (90 days+) - regulatory compliance |

**Legal Hold (for litigation):**

```python
# Apply legal hold to object (prevents deletion even after retention expires)
s3_client.put_object_legal_hold(
    Bucket='eemp-audit-trail-cold',
    Key='year=2023/month=01/day=15/audit-uuid-5678.json',
    LegalHold={'Status': 'ON'}
)
```

**Integrity Verification:**

```python
# Write audit record with checksum
audit_record_json = json.dumps(audit_record, sort_keys=True)
checksum = hashlib.sha256(audit_record_json.encode()).hexdigest()

# Store checksum in object metadata
s3_client.put_object(
    Bucket='eemp-audit-trail-hot',
    Key=object_key,
    Body=audit_record_json,
    Metadata={
        'record-hash': checksum,
        'previous-record-hash': previous_checksum  # Blockchain-style chaining
    },
    ChecksumAlgorithm='SHA256'
)

# Verify integrity on read
response = s3_client.get_object(Bucket=bucket, Key=object_key)
stored_hash = response['Metadata']['record-hash']
computed_hash = hashlib.sha256(response['Body'].read()).hexdigest()

assert stored_hash == computed_hash, "Audit record integrity compromised!"
```

---

## 3. Audit Enrichment Pipeline

### 3.1 Enrichment Architecture

**Real-Time Enrichment Pipeline:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Audit Enrichment Pipeline                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Event Source (Publisher, Subscription Service, etc.)               │
│         │                                                            │
│         │ 1. Emit raw audit event                                   │
│         │    { "actor_id": "12345", "event_type": "event.published" }│
│         ▼                                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Kinesis Data Stream: audit-events-stream                  │    │
│  │  - Buffer: 100KB or 1 second                               │    │
│  │  - Shard count: Auto-scaling (based on throughput)         │    │
│  └────────────────────────────────────────────────────────────┘    │
│         │                                                            │
│         │ 2. Stream to enrichment Lambda                            │
│         ▼                                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Enrichment Lambda Function                                │    │
│  │                                                             │    │
│  │  Enrichment Steps:                                         │    │
│  │  1. Lookup actor details (user directory, service registry)│    │
│  │  2. Fetch domain-specific data (account info, metadata)   │    │
│  │  3. Add compliance metadata (regulatory scope, retention)  │    │
│  │  4. Calculate record hash for integrity                    │    │
│  │  5. Add partition metadata (year/month/day/hour)          │    │
│  │                                                             │    │
│  │  Data Sources:                                             │    │
│  │  - DynamoDB: User/service metadata cache                   │    │
│  │  - RDS: Business domain data (accounts, relationships)     │    │
│  │  - External APIs: HR system, CRM (with caching)           │    │
│  └────────────────────────────────────────────────────────────┘    │
│         │                                                            │
│         │ 3. Write enriched audit record                            │
│         ▼                                                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  S3 (Hot Tier) + Kinesis Firehose                          │    │
│  │                                                             │    │
│  │  Parallel writes:                                          │    │
│  │  1. S3 JSON (individual records) - for compliance          │    │
│  │  2. Kinesis Firehose → S3 Parquet (aggregated) - for Athena│    │
│  │  3. DynamoDB (index) - for fast queries (optional)         │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Enrichment Lambda Function:**

```python
# audit-enrichment-lambda/handler.py
import boto3
import json
import hashlib
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
rds_client = boto3.client('rds-data')
s3_client = boto3.client('s3')

# Cache tables
user_cache = dynamodb.Table('user-metadata-cache')
service_cache = dynamodb.Table('service-metadata-cache')

def lambda_handler(event, context):
    """
    Enriches raw audit events with business context
    """
    for record in event['Records']:
        # 1. Parse raw audit event from Kinesis
        raw_audit = json.loads(record['kinesis']['data'])

        # 2. Enrich actor information
        actor = enrich_actor(raw_audit['actor'])

        # 3. Fetch domain-specific data
        enrichment = fetch_domain_enrichment(raw_audit)

        # 4. Add compliance metadata
        compliance = add_compliance_metadata(raw_audit)

        # 5. Build enriched audit record
        enriched_audit = {
            **raw_audit,
            'audit_id': generate_uuid(),
            'timestamp': raw_audit.get('timestamp', datetime.utcnow().isoformat()),
            'actor': actor,
            'enrichment': enrichment,
            'compliance': compliance,
            'integrity': calculate_integrity(raw_audit)
        }

        # 6. Write to S3 with partitioning
        write_to_s3(enriched_audit)

        # 7. Write to Parquet (via Firehose)
        write_to_firehose(enriched_audit)

        # 8. Update index (DynamoDB) for fast queries
        update_index(enriched_audit)

def enrich_actor(actor_raw):
    """
    Enrich actor with user/service details
    """
    actor_id = actor_raw['id']
    actor_type = actor_raw['type']

    if actor_type == 'user':
        # Lookup user details from cache (populated from HR/LDAP)
        user_details = user_cache.get_item(Key={'user_id': actor_id}).get('Item', {})
        return {
            **actor_raw,
            'name': user_details.get('name', 'Unknown'),
            'email': user_details.get('email'),
            'department': user_details.get('department'),
            'manager': user_details.get('manager'),
            'cost_center': user_details.get('cost_center')
        }

    elif actor_type == 'service':
        # Lookup service details from registry
        service_details = service_cache.get_item(Key={'service_id': actor_id}).get('Item', {})
        return {
            **actor_raw,
            'name': service_details.get('name', 'Unknown Service'),
            'team': service_details.get('owning_team'),
            'environment': service_details.get('environment')
        }

    return actor_raw

def fetch_domain_enrichment(audit_event):
    """
    Fetch domain-specific business context
    """
    event_type = audit_event.get('event', {}).get('event_type')

    enrichment = {}

    # Example: For user.created events, fetch account details
    if event_type == 'user.created':
        user_id = audit_event['event']['payload'].get('user_id')
        # Query RDS for account details (cached in DynamoDB for performance)
        account_info = query_account_info(user_id)
        enrichment['account_details'] = {
            'account_type': account_info.get('type'),
            'region': account_info.get('region'),
            'created_date': account_info.get('created_date')
        }

    # Example: For subscription events, fetch subscriber details
    elif event_type in ['subscription.approved', 'subscription.denied']:
        subscriber_id = audit_event.get('event', {}).get('subscriber_id')
        subscriber_info = service_cache.get_item(Key={'service_id': subscriber_id}).get('Item', {})
        enrichment['subscriber_details'] = {
            'team': subscriber_info.get('owning_team'),
            'business_unit': subscriber_info.get('business_unit'),
            'data_classification_clearance': subscriber_info.get('clearance_level')
        }

    return enrichment

def add_compliance_metadata(audit_event):
    """
    Add compliance and retention metadata
    """
    event_type = audit_event.get('event', {}).get('event_type')
    data_classification = audit_event.get('event', {}).get('data_classification', 0)

    # Determine regulatory scope based on event type and data classification
    regulatory_scope = []
    if data_classification >= 2:  # PII or confidential data
        regulatory_scope.append('GDPR')
    if event_type.startswith('payment.') or 'card' in event_type:
        regulatory_scope.append('PCI-DSS')
    if data_classification >= 3:  # Financial data
        regulatory_scope.append('SOX')

    # Determine retention period (default: 7 years for financial services)
    retention_years = 7
    if 'PCI-DSS' in regulatory_scope:
        retention_years = max(retention_years, 10)  # PCI requires 10 years

    return {
        'retention_years': retention_years,
        'regulatory_scope': regulatory_scope,
        'data_sensitivity': get_sensitivity_label(data_classification),
        'cross_border_transfer': audit_event.get('event', {}).get('cross_border_flag', False),
        'encryption_key_id': 'arn:aws:kms:us-east-1:111111111111:key/audit-key-id'
    }

def calculate_integrity(audit_event):
    """
    Calculate cryptographic hash for integrity verification
    """
    # Canonical JSON (sorted keys) for consistent hashing
    canonical_json = json.dumps(audit_event, sort_keys=True)
    record_hash = hashlib.sha256(canonical_json.encode()).hexdigest()

    # Retrieve previous record hash for chaining (blockchain-style)
    previous_hash = get_previous_record_hash()

    return {
        'record_hash': record_hash,
        'previous_record_hash': previous_hash,
        'hash_algorithm': 'SHA-256'
    }

def write_to_s3(enriched_audit):
    """
    Write enriched audit record to S3 with partitioning
    """
    timestamp = datetime.fromisoformat(enriched_audit['timestamp'])

    # Time-based partitioning
    object_key = (
        f"year={timestamp.year}/"
        f"month={timestamp.month:02d}/"
        f"day={timestamp.day:02d}/"
        f"hour={timestamp.hour:02d}/"
        f"audit-{enriched_audit['audit_id']}.json"
    )

    # Write to S3 with Object Lock
    s3_client.put_object(
        Bucket='eemp-audit-trail-hot',
        Key=object_key,
        Body=json.dumps(enriched_audit, indent=2),
        ContentType='application/json',
        ServerSideEncryption='aws:kms',
        SSEKMSKeyId='arn:aws:kms:us-east-1:111111111111:key/audit-key-id',
        ObjectLockMode='GOVERNANCE',
        ObjectLockRetainUntilDate=datetime.utcnow() + timedelta(days=90),
        Metadata={
            'record-hash': enriched_audit['integrity']['record_hash'],
            'event-type': enriched_audit.get('event_type', 'unknown'),
            'actor-id': enriched_audit['actor']['id']
        },
        Tagging=f"event_type={enriched_audit.get('event_type')}&"
                f"actor_type={enriched_audit['actor']['type']}&"
                f"data_classification={enriched_audit.get('event', {}).get('data_classification', 0)}"
    )

    # Store S3 metadata in audit record
    enriched_audit['integrity']['s3_object_key'] = object_key
    enriched_audit['integrity']['written_at'] = datetime.utcnow().isoformat()
```

### 3.2 Domain-Specific Enrichment Examples

**Example 1: User Event Enrichment**

```python
def enrich_user_event(audit_event):
    """
    Enrich user-related events with HR and account data
    """
    user_id = audit_event['event']['payload'].get('user_id')

    # Fetch from HR system (cached in DynamoDB)
    hr_data = get_hr_data(user_id)

    # Fetch account relationship data
    account_data = get_account_data(user_id)

    return {
        'user_details': {
            'employee_id': hr_data.get('employee_id'),
            'department': hr_data.get('department'),
            'manager': hr_data.get('manager_name'),
            'hire_date': hr_data.get('hire_date'),
            'location': hr_data.get('office_location')
        },
        'account_details': {
            'account_count': account_data.get('count'),
            'account_types': account_data.get('types'),
            'total_balance': account_data.get('total_balance'),  # If financial event
            'primary_account_id': account_data.get('primary_account')
        }
    }
```

**Example 2: Subscription Event Enrichment**

```python
def enrich_subscription_event(audit_event):
    """
    Enrich subscription events with subscriber and publisher context
    """
    subscriber_id = audit_event['event'].get('subscriber_id')
    event_types = audit_event['event'].get('subscribed_event_types', [])

    # Fetch subscriber service metadata
    subscriber_info = get_service_metadata(subscriber_id)

    # Fetch publisher info for each subscribed event type
    publisher_info = {}
    for event_type in event_types:
        publisher = get_event_publisher(event_type)
        publisher_info[event_type] = {
            'publisher_id': publisher.get('id'),
            'publisher_team': publisher.get('team'),
            'data_classification': publisher.get('data_classification')
        }

    return {
        'subscriber_details': {
            'team': subscriber_info.get('owning_team'),
            'business_unit': subscriber_info.get('business_unit'),
            'cost_center': subscriber_info.get('cost_center'),
            'aws_account_id': subscriber_info.get('aws_account_id')
        },
        'publisher_details': publisher_info,
        'subscription_metadata': {
            'event_count': len(event_types),
            'estimated_volume': estimate_event_volume(event_types)
        }
    }
```

**Example 3: Policy Decision Enrichment**

```python
def enrich_policy_decision(audit_event):
    """
    Enrich policy decision events with full decision context
    """
    decision_id = audit_event['policy']['decision_id']

    # Fetch OPA policy details
    policy_info = get_policy_metadata(decision_id)

    # Fetch request context
    request_context = audit_event.get('policy', {}).get('input', {})

    return {
        'policy_details': {
            'policy_name': policy_info.get('name'),
            'policy_version': policy_info.get('version'),
            'policy_owner': policy_info.get('owner_team'),
            'last_updated': policy_info.get('updated_at')
        },
        'decision_context': {
            'requested_clearance': request_context.get('data_classification_clearance'),
            'granted_clearance': audit_event['policy'].get('granted_clearance'),
            'decision_factors': audit_event['policy'].get('rules_evaluated', []),
            'override_applied': audit_event['policy'].get('override', False)
        }
    }
```

---

## 4. Audit Discovery and Query Mechanisms

### 4.1 Multi-Layer Query Architecture

**Layer 1: DynamoDB Index (Fast, Recent Data)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DynamoDB Audit Index Table                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Table: audit-events-index                                          │
│  Retention: 90 days (hot tier only)                                 │
│                                                                      │
│  Primary Key:                                                        │
│    PK: event_type#{date}  (e.g., "event.published#2025-12-10")     │
│    SK: timestamp#{audit_id}                                         │
│                                                                      │
│  GSI 1 (Actor Index):                                               │
│    PK: actor_id  (e.g., "user-12345")                              │
│    SK: timestamp                                                    │
│                                                                      │
│  GSI 2 (Correlation Index):                                         │
│    PK: correlation_id                                               │
│    SK: timestamp                                                    │
│                                                                      │
│  Attributes:                                                         │
│    - audit_id                                                       │
│    - timestamp                                                      │
│    - event_type                                                     │
│    - actor_id, actor_type                                          │
│    - correlation_id                                                 │
│    - outcome_status                                                │
│    - s3_object_key  (pointer to full record in S3)                 │
│    - ttl  (expire after 90 days)                                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**DynamoDB Query Examples:**

```python
# Query 1: Get all events of a specific type on a date
response = dynamodb.query(
    TableName='audit-events-index',
    KeyConditionExpression='PK = :pk AND begins_with(SK, :date)',
    ExpressionAttributeValues={
        ':pk': 'event.published#2025-12-10',
        ':date': '2025-12-10'
    }
)

# Query 2: Get all events by a specific actor (using GSI 1)
response = dynamodb.query(
    TableName='audit-events-index',
    IndexName='ActorIndex',
    KeyConditionExpression='actor_id = :actor AND #ts BETWEEN :start AND :end',
    ExpressionAttributeNames={'#ts': 'timestamp'},
    ExpressionAttributeValues={
        ':actor': 'user-12345',
        ':start': '2025-12-01T00:00:00Z',
        ':end': '2025-12-31T23:59:59Z'
    }
)

# Query 3: Trace correlated events (using GSI 2)
response = dynamodb.query(
    TableName='audit-events-index',
    IndexName='CorrelationIndex',
    KeyConditionExpression='correlation_id = :corr_id',
    ExpressionAttributeValues={
        ':corr_id': 'trace-abc-123'
    }
)

# Retrieve full record from S3
for item in response['Items']:
    s3_key = item['s3_object_key']
    full_record = s3_client.get_object(
        Bucket='eemp-audit-trail-hot',
        Key=s3_key
    )
    audit_event = json.loads(full_record['Body'].read())
```

**Benefits:**
- **Sub-second queries** for recent data (0-90 days)
- **Multiple access patterns** (by event type, actor, correlation ID)
- **Low cost** (pay per request, no idle cost)

**Trade-offs:**
- **Limited retention** (90 days only, older data must query S3/Athena)
- **Eventual consistency** (slight delay between S3 write and DynamoDB update)

---

**Layer 2: Amazon Athena (SQL Queries on S3)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Athena Query Layer (S3 Parquet)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Database: eemp_audit                                               │
│  Table: audit_events                                                │
│                                                                      │
│  CREATE EXTERNAL TABLE audit_events (                               │
│    audit_id STRING,                                                 │
│    timestamp TIMESTAMP,                                             │
│    event_type STRING,                                               │
│    actor STRUCT<type:STRING, id:STRING, name:STRING, email:STRING>, │
│    source STRUCT<system:STRING, aws_account_id:STRING>,            │
│    event STRUCT<event_id:STRING, payload:STRING>,                  │
│    outcome STRUCT<status:STRING, status_code:INT>,                 │
│    policy STRUCT<decision:STRING, rules_evaluated:ARRAY<STRING>>,  │
│    enrichment STRUCT<...>,                                         │
│    compliance STRUCT<retention_years:INT, regulatory_scope:ARRAY>  │
│  )                                                                  │
│  PARTITIONED BY (year INT, month INT, day INT)                     │
│  STORED AS PARQUET                                                 │
│  LOCATION 's3://eemp-audit-trail-analytics/'                       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**Athena Query Examples:**

```sql
-- Query 1: Count events by type for a specific date
SELECT event_type, COUNT(*) as event_count
FROM audit_events
WHERE year = 2025 AND month = 12 AND day = 10
GROUP BY event_type
ORDER BY event_count DESC;

-- Query 2: Find all failed subscription requests
SELECT
  timestamp,
  actor.email as requester_email,
  event.subscriber_id,
  policy.decision,
  policy.rules_evaluated
FROM audit_events
WHERE event_type = 'subscription.denied'
  AND year = 2025 AND month = 12
ORDER BY timestamp DESC;

-- Query 3: Audit all PII access by a specific user
SELECT
  timestamp,
  event_type,
  event.event_id,
  event.payload,
  source.system
FROM audit_events
WHERE actor.id = 'user-12345'
  AND event.pii_indicator = true
  AND year = 2025 AND month >= 10
ORDER BY timestamp DESC;

-- Query 4: Compliance report - SOX events
SELECT
  DATE_TRUNC('day', timestamp) as date,
  event_type,
  COUNT(*) as event_count,
  COUNT(DISTINCT actor.id) as unique_actors
FROM audit_events
WHERE 'SOX' IN (compliance.regulatory_scope)
  AND year = 2025 AND month = 12
GROUP BY DATE_TRUNC('day', timestamp), event_type
ORDER BY date DESC;

-- Query 5: Cross-border data transfer tracking (GDPR)
SELECT
  timestamp,
  actor.email,
  event_type,
  source.aws_account_id,
  source.region,
  enrichment.user_details.location
FROM audit_events
WHERE compliance.cross_border_transfer = true
  AND year = 2025
ORDER BY timestamp DESC;

-- Query 6: Trace event flow by correlation_id
SELECT
  timestamp,
  event_type,
  actor.id,
  source.system,
  outcome.status
FROM audit_events
WHERE event.correlation_id = 'trace-abc-123'
ORDER BY timestamp ASC;
```

**Athena Query Optimization:**

```sql
-- Create partition metadata (run after new data arrives)
MSCK REPAIR TABLE audit_events;

-- Optimize query cost with partition pruning
SELECT * FROM audit_events
WHERE year = 2025 AND month = 12 AND day = 10  -- Prunes to single partition
  AND event_type = 'event.published';

-- Use approximate functions for large scans
SELECT approx_distinct(actor.id) as unique_actors
FROM audit_events
WHERE year = 2025;
```

**Benefits:**
- **SQL interface** for compliance analysts (familiar query language)
- **Cost-effective** for large datasets (scan only necessary partitions)
- **Flexible schema** (nested structs, arrays for complex data)
- **Long-term retention** (query 7+ years of data in Glacier)

**Trade-offs:**
- **Query latency** (seconds to minutes for large scans)
- **Cost per query** (based on data scanned, mitigate with partitioning)
- **Not real-time** (eventual consistency, data aggregated daily)

---

**Layer 3: OpenSearch (Full-Text Search and Dashboards)**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenSearch Audit Search (Optional)                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Index: audit-events-YYYY-MM-DD                                     │
│  Retention: 90 days (hot tier, indices rotated daily)              │
│                                                                      │
│  Document Structure:                                                │
│  {                                                                  │
│    "audit_id": "uuid-1234",                                        │
│    "timestamp": "2025-12-10T10:30:00.123Z",                        │
│    "event_type": "event.published",                                │
│    "actor": { "id": "...", "name": "...", "email": "..." },       │
│    "event": { "event_id": "...", "payload": {...} },              │
│    "enrichment": { ... },                                          │
│    "full_text": "searchable concatenation of all fields"          │
│  }                                                                  │
│                                                                      │
│  Use Cases:                                                         │
│  - Full-text search ("find all events mentioning user@example.com")│
│  - Real-time dashboards (Kibana/OpenSearch Dashboards)             │
│  - Anomaly detection (ML-powered alerts)                           │
│  - Compliance officer ad-hoc queries                               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**OpenSearch Query Examples:**

```json
// Full-text search
POST /audit-events-*/_search
{
  "query": {
    "multi_match": {
      "query": "user@example.com",
      "fields": ["actor.email", "event.payload.*"]
    }
  },
  "sort": [{"timestamp": "desc"}],
  "size": 100
}

// Complex filter query
POST /audit-events-*/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"event_type": "subscription.denied"}},
        {"range": {"timestamp": {"gte": "2025-12-01"}}}
      ],
      "should": [
        {"term": {"policy.decision": "deny"}}
      ]
    }
  }
}

// Aggregation for dashboards
POST /audit-events-*/_search
{
  "size": 0,
  "aggs": {
    "events_by_hour": {
      "date_histogram": {
        "field": "timestamp",
        "interval": "hour"
      },
      "aggs": {
        "by_event_type": {
          "terms": {"field": "event_type"}
        }
      }
    }
  }
}
```

**Benefits:**
- **Full-text search** (find events by any field content)
- **Real-time dashboards** (Kibana/OpenSearch Dashboards)
- **Fast queries** (millisecond response for indexed data)

**Trade-offs:**
- **Cost** (additional infrastructure: OpenSearch cluster)
- **Complexity** (another system to manage)
- **Limited retention** (90 days typical, expensive for long-term)

**Recommendation:** Use OpenSearch only if full-text search or real-time dashboards are critical requirements. Otherwise, DynamoDB + Athena sufficient for most compliance needs.

---

### 4.2 Audit Query API

**REST API for Audit Queries:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Audit Query API (API Gateway + Lambda)            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Endpoints:                                                         │
│                                                                      │
│  1. GET /api/v1/audit/events                                       │
│     - Query recent events (90 days) from DynamoDB                  │
│     - Parameters: event_type, actor_id, start_time, end_time       │
│                                                                      │
│  2. POST /api/v1/audit/query                                       │
│     - Complex queries via Athena                                   │
│     - Request body: SQL query or filter parameters                 │
│     - Async: Returns query_id, poll for results                    │
│                                                                      │
│  3. GET /api/v1/audit/events/{audit_id}                            │
│     - Retrieve specific audit record from S3                       │
│                                                                      │
│  4. GET /api/v1/audit/trace/{correlation_id}                       │
│     - Trace all events in a correlation chain                      │
│                                                                      │
│  5. POST /api/v1/audit/reports/compliance                          │
│     - Generate compliance reports (SOX, GDPR, etc.)                │
│     - Async job, export to S3                                      │
│                                                                      │
│  6. GET /api/v1/audit/integrity/verify                             │
│     - Verify integrity of audit trail (hash chain validation)      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**API Implementation Example:**

```python
# audit-query-api/query_handler.py
from aws_lambda_powertools import Logger, Tracer
from aws_lambda_powertools.event_handler import APIGatewayRestResolver

app = APIGatewayRestResolver()
logger = Logger()
tracer = Tracer()

@app.get("/audit/events")
@tracer.capture_method
def query_recent_events():
    """
    Query recent audit events from DynamoDB index
    """
    # Parse query parameters
    event_type = app.current_event.get_query_string_value('event_type')
    actor_id = app.current_event.get_query_string_value('actor_id')
    start_time = app.current_event.get_query_string_value('start_time')
    end_time = app.current_event.get_query_string_value('end_time')

    # Query DynamoDB
    if event_type:
        results = query_by_event_type(event_type, start_time, end_time)
    elif actor_id:
        results = query_by_actor(actor_id, start_time, end_time)
    else:
        return {'error': 'Must specify event_type or actor_id'}, 400

    # Fetch full records from S3
    enriched_results = []
    for item in results:
        full_record = fetch_from_s3(item['s3_object_key'])
        enriched_results.append(full_record)

    return {
        'count': len(enriched_results),
        'events': enriched_results
    }

@app.post("/audit/query")
@tracer.capture_method
def execute_athena_query():
    """
    Execute complex query via Athena (async)
    """
    request_body = app.current_event.json_body

    # Build Athena query from request
    if 'sql' in request_body:
        sql_query = request_body['sql']
    else:
        # Build SQL from filters
        sql_query = build_query_from_filters(request_body['filters'])

    # Submit async Athena query
    query_execution_id = submit_athena_query(sql_query)

    return {
        'query_id': query_execution_id,
        'status': 'QUEUED',
        'poll_url': f'/audit/query/{query_execution_id}/results'
    }, 202

@app.get("/audit/events/<audit_id>")
@tracer.capture_method
def get_audit_event(audit_id: str):
    """
    Retrieve specific audit record
    """
    # Query DynamoDB index for S3 key
    index_record = dynamodb.get_item(
        TableName='audit-events-index',
        Key={'audit_id': audit_id}
    ).get('Item')

    if not index_record:
        return {'error': 'Audit record not found'}, 404

    # Fetch full record from S3
    s3_key = index_record['s3_object_key']
    full_record = s3_client.get_object(
        Bucket='eemp-audit-trail-hot',  # Try hot tier first
        Key=s3_key
    )

    audit_event = json.loads(full_record['Body'].read())

    # Verify integrity
    stored_hash = full_record['Metadata']['record-hash']
    computed_hash = hashlib.sha256(
        json.dumps(audit_event, sort_keys=True).encode()
    ).hexdigest()

    if stored_hash != computed_hash:
        logger.error(f"Integrity check failed for audit_id={audit_id}")
        audit_event['integrity_warning'] = 'Hash mismatch detected'

    return audit_event

@app.get("/audit/trace/<correlation_id>")
@tracer.capture_method
def trace_correlated_events(correlation_id: str):
    """
    Trace all events in a correlation chain
    """
    # Query DynamoDB GSI 2 (Correlation Index)
    response = dynamodb.query(
        TableName='audit-events-index',
        IndexName='CorrelationIndex',
        KeyConditionExpression='correlation_id = :corr_id',
        ExpressionAttributeValues={':corr_id': correlation_id}
    )

    # Fetch full records from S3
    events = []
    for item in response['Items']:
        full_record = fetch_from_s3(item['s3_object_key'])
        events.append(full_record)

    # Sort by timestamp to show event flow
    events.sort(key=lambda e: e['timestamp'])

    return {
        'correlation_id': correlation_id,
        'event_count': len(events),
        'events': events,
        'event_flow': build_event_flow_visualization(events)
    }

def lambda_handler(event, context):
    return app.resolve(event, context)
```

---

## 5. Compliance Reporting and Export

### 5.1 Compliance Report Types

**Report 1: SOX Access Audit Report**

```sql
-- Athena query for SOX compliance report
-- Shows all access to financial data with actor details

SELECT
  DATE_FORMAT(timestamp, '%Y-%m-%d') as date,
  actor.name as actor_name,
  actor.email as actor_email,
  actor.department,
  event_type,
  event.event_id,
  source.system as source_system,
  outcome.status,
  enrichment.account_details.account_type
FROM audit_events
WHERE 'SOX' IN (compliance.regulatory_scope)
  AND year = 2025 AND month BETWEEN 10 AND 12  -- Q4 2025
  AND event.data_classification >= 3  -- Financial data
ORDER BY timestamp DESC;
```

**Export to CSV for regulatory submission:**

```python
def generate_sox_report(start_date, end_date):
    """
    Generate SOX compliance report and export to S3
    """
    query = f"""
        SELECT ... (query above)
        WHERE timestamp BETWEEN '{start_date}' AND '{end_date}'
    """

    # Execute Athena query
    query_execution_id = athena.start_query_execution(
        QueryString=query,
        ResultConfiguration={
            'OutputLocation': 's3://eemp-compliance-reports/sox/',
            'EncryptionConfiguration': {
                'EncryptionOption': 'SSE_KMS',
                'KmsKey': 'arn:aws:kms:...'
            }
        }
    )

    # Wait for completion
    wait_for_query_completion(query_execution_id)

    # Get results location
    result_location = get_query_results_location(query_execution_id)

    return {
        'report_id': query_execution_id,
        'download_url': generate_presigned_url(result_location),
        'expires_in': 3600  # 1 hour
    }
```

**Report 2: GDPR Data Access Report**

```sql
-- Identify all PII access for a specific individual (GDPR Article 15)

SELECT
  timestamp,
  actor.email as accessor,
  event_type,
  event.event_id,
  source.system,
  outcome.status,
  event.payload as data_accessed
FROM audit_events
WHERE event.pii_indicator = true
  AND (
    event.payload LIKE '%user-12345%'  -- User ID
    OR event.payload LIKE '%john.doe@example.com%'  -- Email
  )
  AND year = 2025
ORDER BY timestamp DESC;
```

**Report 3: Cross-Border Transfer Audit (GDPR Article 44)**

```sql
-- Track all cross-border data transfers

SELECT
  timestamp,
  actor.id,
  event_type,
  source.region as source_region,
  enrichment.transfer_destination_region,
  compliance.regulatory_scope
FROM audit_events
WHERE compliance.cross_border_transfer = true
  AND year = 2025 AND month = 12
ORDER BY timestamp DESC;
```

### 5.2 Automated Compliance Reporting

**Scheduled Report Generation (EventBridge + Lambda):**

```yaml
# EventBridge Rule: Generate monthly SOX report
sox_monthly_report:
  schedule: cron(0 5 1 * ? *)  # 5 AM on 1st of every month
  target: lambda:generate-sox-report
  input:
    report_type: SOX
    period: last_month

# Lambda function generates report and emails to compliance team
def generate_and_email_sox_report(event):
    # 1. Generate report via Athena
    report_location = generate_sox_report(
        start_date=get_last_month_start(),
        end_date=get_last_month_end()
    )

    # 2. Convert to PDF (optional)
    pdf_report = convert_csv_to_pdf(report_location)

    # 3. Email to compliance team
    ses.send_email(
        Source='audit@company.com',
        Destination={'ToAddresses': ['compliance@company.com']},
        Message={
            'Subject': {'Data': f'SOX Compliance Report - {get_last_month_name()}'},
            'Body': {
                'Text': {'Data': f'Attached is the SOX compliance report for {get_last_month_name()}.'}
            }
        },
        Attachments=[pdf_report]
    )
```

---

## 6. Cost Analysis

### 6.1 Storage Cost Breakdown

**Assumptions:**
- 10,000 events/day
- Average event size: 5 KB (after enrichment)
- Retention: 7 years

**Cost Calculation:**

```
Daily Data Volume:
- Events: 10,000 × 5 KB = 50 MB/day
- Monthly: 50 MB × 30 = 1.5 GB/month
- Yearly: 1.5 GB × 12 = 18 GB/year
- 7-year total: 18 GB × 7 = 126 GB

Tier 1: Hot (S3 Standard, 0-90 days)
- Storage: 4.5 GB × $0.023/GB = $0.10/month

Tier 2: Warm (S3 Intelligent-Tiering, 90 days - 2 years)
- Storage: 27 GB × $0.0125/GB = $0.34/month

Tier 3: Cold (S3 Glacier Deep Archive, 2-7 years)
- Storage: 90 GB × $0.00099/GB = $0.09/month

Parquet Analytics Tier (all 7 years, 10x compression):
- Storage: 12.6 GB × $0.0125/GB = $0.16/month

Total Storage Cost: ~$0.70/month ($8.40/year)
```

**Scaling Analysis:**

| Events/Day | Daily Data | 7-Year Total | Monthly Storage Cost |
|------------|------------|--------------|---------------------|
| 10K | 50 MB | 126 GB | **$0.70** |
| 100K | 500 MB | 1.26 TB | **$7.00** |
| 1M | 5 GB | 12.6 TB | **$70.00** |
| 10M | 50 GB | 126 TB | **$700.00** |

### 6.2 Query and Enrichment Costs

**DynamoDB (Index):**
```
Write Cost (90-day retention):
- 10,000 writes/day × $1.25 per million = $0.0125/day = $0.38/month

Read Cost (assume 1,000 queries/day):
- 1,000 reads/day × $0.25 per million = $0.00025/day = $0.01/month

Storage (90 days of data):
- 4.5 GB × $0.25/GB = $1.13/month

Total DynamoDB: ~$1.50/month
```

**Athena (Query):**
```
Assume 100 compliance queries/month, each scanning 1 GB:
- 100 queries × 1 GB × $5.00/TB = $0.50/month
```

**Lambda (Enrichment):**
```
10,000 invocations/day × 512 MB × 500ms:
- 10,000 × 30 × $0.0000002 = $0.06/month
```

**Kinesis Data Stream:**
```
1 shard (< 1 MB/sec ingestion):
- $0.015/hour × 730 hours = $11/month
```

**Total Infrastructure Cost (10K events/day):**
```
S3 Storage: $0.70/month
DynamoDB Index: $1.50/month
Athena Queries: $0.50/month
Lambda Enrichment: $0.06/month
Kinesis Stream: $11.00/month
KMS (encryption): $1.00/month

Total: ~$15/month ($180/year)
```

**Comparison to CloudTrail:**
- **CloudTrail Cost (if permitted):** $2.00 per 100K events = $6/month for 300K events/month
- **Custom Audit Trail:** $15/month for same volume BUT includes:
  - Domain-specific enrichment
  - Multiple query interfaces (DynamoDB, Athena, API)
  - 7-year retention with tiering
  - Custom compliance reports
  - Full control over data residency

**Conclusion:** Custom audit trail is cost-competitive with CloudTrail while providing far more flexibility and enrichment capabilities.

---

## 7. Operational Considerations

### 7.1 Monitoring and Alerting

**CloudWatch Alarms:**

```yaml
alarms:
  - name: AuditEnrichmentFailureRate
    metric: Lambda Errors (audit-enrichment-lambda)
    threshold: > 5 errors in 5 minutes
    action: SNS notification to platform team

  - name: S3WriteFailures
    metric: S3 PutObject errors
    threshold: > 10 failures in 5 minutes
    action: PagerDuty critical alert

  - name: DynamoDBIndexLag
    metric: Kinesis → DynamoDB lag
    threshold: > 60 seconds
    action: SNS notification

  - name: IntegrityCheckFailures
    metric: Custom metric (hash mismatches)
    threshold: > 0
    action: PagerDuty critical alert + security team notification
```

### 7.2 Disaster Recovery

**Multi-Region Replication:**

```yaml
disaster_recovery:
  primary_region: us-east-1
  replica_region: us-west-2

  s3_replication:
    source: eemp-audit-trail-hot (us-east-1)
    destination: eemp-audit-trail-hot-replica (us-west-2)
    replication_time: < 15 minutes (S3 RTC)

  rto: 1 hour (switch to replica region)
  rpo: < 15 minutes (S3 replication lag)

  failover_procedure:
    1. Update Route53 to point API Gateway to us-west-2
    2. Update Lambda environment variables (S3 bucket names)
    3. Verify replica data integrity
    4. Resume operations
```

### 7.3 Data Retention and Lifecycle

**Automated Lifecycle Management:**

```python
# S3 Lifecycle Policy
lifecycle_policy = {
    'Rules': [
        {
            'Id': 'TransitionToWarmTier',
            'Status': 'Enabled',
            'Transitions': [
                {
                    'Days': 90,
                    'StorageClass': 'INTELLIGENT_TIERING'
                }
            ]
        },
        {
            'Id': 'TransitionToColdTier',
            'Status': 'Enabled',
            'Transitions': [
                {
                    'Days': 730,  # 2 years
                    'StorageClass': 'GLACIER_DEEP_ARCHIVE'
                }
            ]
        },
        {
            'Id': 'ExpireAfter7Years',
            'Status': 'Enabled',
            'Expiration': {
                'Days': 2555  # 7 years
            }
        }
    ]
}

s3_client.put_bucket_lifecycle_configuration(
    Bucket='eemp-audit-trail-hot',
    LifecycleConfiguration=lifecycle_policy
)
```

---

## 8. Architectural Decisions Record (ADR)

### ADR-001: S3-Based Audit Trail (No CloudTrail)

**Decision:** Build custom audit trail on S3 instead of using AWS CloudTrail.

**Context:**
- AWS CloudTrail is not permitted by firm policy (regulatory or compliance reasons)
- Financial services require 7+ year audit retention
- Need domain-specific enrichment beyond CloudTrail capabilities

**Rationale:**
1. **Compliance Requirement:** Firm policy prohibits CloudTrail
2. **Enrichment:** CloudTrail provides only AWS API calls, not business context
3. **Cost:** Custom solution cost-competitive ($15/month vs. $6/month) but far more capable
4. **Control:** Full control over data format, retention, and enrichment

**Consequences:**
- **Pros:** Full customization, domain enrichment, flexible retention, cost-effective at scale
- **Cons:** More operational complexity than managed CloudTrail, need to build query tools

---

### ADR-002: Multi-Tier Storage with S3 Object Lock

**Decision:** Use multi-tier S3 storage (Hot/Warm/Cold) with Object Lock for immutability.

**Rationale:**
1. **Cost Optimization:** Glacier Deep Archive is 96% cheaper than S3 Standard for long-term retention
2. **Immutability:** Object Lock (Compliance mode) provides tamper-proof audit trail for regulatory compliance
3. **Performance:** Hot tier (90 days) provides fast access for recent data

**Consequences:**
- **Pros:** Massive cost savings (7-year retention at $0.09/month vs. $1.80/month), regulatory compliance
- **Cons:** Retrieval latency for cold data (12-48 hours), cannot delete objects (acceptable for audit)

---

### ADR-003: Real-Time Enrichment via Kinesis + Lambda

**Decision:** Enrich audit events in real-time using Kinesis Data Stream + Lambda before writing to S3.

**Rationale:**
1. **Completeness:** Audit records include full business context at time of write
2. **Consistency:** All audit records have uniform enrichment
3. **Performance:** Asynchronous enrichment doesn't block event processing

**Consequences:**
- **Pros:** Complete audit records, no post-processing needed
- **Cons:** Enrichment failures could lose audit data (mitigated with DLQ and retries)

---

### ADR-004: DynamoDB Index for Recent Data

**Decision:** Maintain DynamoDB index for recent audit data (90 days) for fast queries.

**Rationale:**
1. **Performance:** Sub-second queries for recent data vs. seconds/minutes with Athena
2. **Use Case:** 90% of audit queries are for recent data (debugging, monitoring)
3. **Cost:** DynamoDB cheaper than Athena for frequent queries on small datasets

**Consequences:**
- **Pros:** Fast queries for operational use cases
- **Cons:** Additional infrastructure, TTL management, eventual consistency with S3

---

## 9. Summary

This custom audit trail architecture provides:

✓ **Regulatory Compliance:** Immutable, tamper-proof audit trail with 7+ year retention
✓ **Domain Enrichment:** Business context (user details, account info) included in every audit record
✓ **Multi-Pattern Discovery:** DynamoDB (fast, recent), Athena (SQL, historical), API (programmatic)
✓ **Cost-Effective:** ~$15/month for 10K events/day with 7-year retention
✓ **Scalable:** Proven S3 architecture scales to billions of events
✓ **Compliant:** Meets SOX, GDPR, PCI-DSS, and banking regulatory requirements

**Key Differentiators vs. CloudTrail:**
- Domain-specific enrichment (not just AWS API calls)
- Custom retention policies (7+ years with cost optimization)
- Flexible query interfaces (DynamoDB, Athena, API)
- Full data control (no AWS-managed service constraints)

---

**Document End**
