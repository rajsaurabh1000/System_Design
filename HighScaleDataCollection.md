# High Scale Data Collection System - System Design

<img width="4635" height="10385" alt="High_Scale_Data_Collection" src="https://github.com/user-attachments/assets/9efb9744-83bf-41fd-96d9-db0cd1b52904" />


## 1. Functional Requirements (FR)

### Core Features
1. **Data Ingestion**
   - Accept data from millions of sources (IoT devices, servers, applications)
   - Support multiple protocols (HTTP, MQTT, gRPC, WebSocket)
   - Handle high-throughput data streams (millions of events/second)
   - Batch and real-time ingestion
   - Data validation and sanitization

2. **Data Processing**
   - Real-time data transformation
   - Aggregation and enrichment
   - Filtering and routing
   - Schema validation
   - Data deduplication

3. **Data Storage**
   - Store raw data for historical analysis
   - Store processed/aggregated data
   - Support both hot and cold storage
   - Data retention policies
   - Efficient querying

4. **Data Query & Analytics**
   - Real-time queries on recent data
   - Historical data analysis
   - Aggregated metrics and reports
   - Time-series analysis
   - Dashboard and visualization support

5. **Monitoring & Alerting**
   - System health monitoring
   - Data pipeline monitoring
   - Anomaly detection
   - Alert on thresholds
   - SLA monitoring

6. **Data Export**
   - Export to data warehouses
   - Export to analytics platforms
   - API access to data
   - Batch exports

### Out of Scope (for this discussion)
- Machine learning model training
- Complex ETL transformations
- Data governance and compliance (can be discussed as extension)

## 2. Non-Functional Requirements (NFR)

1. **Scalability**
   - Handle 10M+ data sources
   - Process 10M+ events per second
   - Store petabytes of data
   - Horizontal scaling (add more nodes)
   - Auto-scaling based on load

2. **Performance**
   - Ingestion latency: < 100ms (p99)
   - Query latency: < 1 second for recent data
   - Historical query: < 10 seconds
   - Real-time processing: < 500ms end-to-end

3. **Availability**
   - 99.99% uptime
   - No single point of failure
   - Multi-region deployment
   - Disaster recovery
   - Graceful degradation

4. **Durability**
   - No data loss
   - At-least-once delivery guarantee
   - Persistent storage with replication
   - Backup and recovery

5. **Consistency**
   - Eventual consistency acceptable for analytics
   - Strong consistency for critical alerts
   - Idempotent processing

6. **Cost Efficiency**
   - Optimize storage costs (hot vs cold)
   - Efficient compression
   - Resource utilization
   - Pay-per-use model

## 3. Core Entities

### 3.1 DataSource
```
DataSource {
  sourceId: UUID
  name: string
  type: enum (IOT_DEVICE, SERVER, APPLICATION, MOBILE_APP)
  metadata: JSON
  schema: Schema
  ingestionRate: int (events/second)
  status: enum (ACTIVE, INACTIVE, ERROR)
  lastSeenAt: timestamp
  createdAt: timestamp
}
```

### 3.2 Event
```
Event {
  eventId: UUID
  sourceId: UUID
  timestamp: long (epoch milliseconds)
  eventType: string
  payload: JSON
  metadata: {
    receivedAt: timestamp,
    processedAt: timestamp,
    version: string,
    tags: Map<string, string>
  }
}
```

### 3.3 Schema
```
Schema {
  schemaId: UUID
  version: string
  fields: List<Field>
  validations: List<Validation>
  createdAt: timestamp
}

Field {
  name: string
  type: enum (STRING, NUMBER, BOOLEAN, OBJECT, ARRAY)
  required: boolean
  defaultValue: any
}
```

### 3.4 Stream
```
Stream {
  streamId: UUID
  name: string
  sourceIds: List<UUID>
  filters: List<Filter>
  transformations: List<Transformation>
  destination: Destination
  status: enum (ACTIVE, PAUSED, ERROR)
}
```

### 3.5 Metric
```
Metric {
  metricId: UUID
  name: string
  value: float
  dimensions: Map<string, string>
  timestamp: long
  aggregationType: enum (SUM, AVG, COUNT, MAX, MIN, P99)
}
```

### 3.6 Alert
```
Alert {
  alertId: UUID
  name: string
  condition: string (e.g., "cpu_usage > 80")
  threshold: float
  window: int (minutes)
  severity: enum (INFO, WARNING, CRITICAL)
  notificationChannels: List<Channel>
  status: enum (ACTIVE, RESOLVED, ACKNOWLEDGED)
  triggeredAt: timestamp
}
```

## 4. API Design

### 4.1 Data Ingestion APIs
```
POST /api/v1/events
Headers: { 
  X-API-Key: <api-key>,
  X-Source-Id: <source-id>
}
Request: {
  events: [
    {
      timestamp: long,
      eventType: string,
      payload: {...}
    }
  ]
}
Response: {
  accepted: int,
  rejected: int,
  errors: [...]
}

POST /api/v1/events/batch
Headers: { X-API-Key: <api-key> }
Request: gzip/snappy compressed batch
Response: { batchId: UUID, status: "accepted" }

// Real-time streaming
WS /api/v1/events/stream
Headers: { Authorization: Bearer <token> }
Client -> Server: { eventType, payload }
Server -> Client: { status: "ack", eventId: UUID }
```

### 4.2 Query APIs
```
POST /api/v1/query
Headers: { Authorization: Bearer <token> }
Request: {
  from: timestamp,
  to: timestamp,
  filters: [
    { field: "eventType", operator: "=", value: "page_view" },
    { field: "userId", operator: "in", value: [...] }
  ],
  aggregations: [
    {
      field: "duration",
      function: "AVG",
      groupBy: ["country"]
    }
  ],
  limit: 1000
}
Response: {
  results: [...],
  totalCount: int,
  executionTime: int (ms)
}

GET /api/v1/metrics/{metricName}
Headers: { Authorization: Bearer <token> }
Query: { 
  from: timestamp,
  to: timestamp,
  granularity: "1m",
  dimensions: "country,device_type"
}
Response: {
  datapoints: [
    { timestamp, value, dimensions: {...} }
  ]
}
```

### 4.3 Stream Management APIs
```
POST /api/v1/streams
Headers: { Authorization: Bearer <token> }
Request: {
  name: string,
  sourceIds: [UUID],
  filters: [...],
  transformations: [...],
  destination: {...}
}
Response: {
  streamId: UUID,
  status: "created"
}

PUT /api/v1/streams/{streamId}/pause
Headers: { Authorization: Bearer <token> }
Response: { status: "paused" }

PUT /api/v1/streams/{streamId}/resume
Headers: { Authorization: Bearer <token> }
Response: { status: "active" }
```

### 4.4 Monitoring APIs
```
GET /api/v1/monitoring/health
Response: {
  status: "healthy",
  components: {
    ingestion: { status: "healthy", throughput: 100000 },
    processing: { status: "healthy", lag: 100 },
    storage: { status: "healthy", used: "50%" }
  }
}

GET /api/v1/monitoring/metrics
Headers: { Authorization: Bearer <token> }
Query: { from, to, metrics: "ingestion_rate,processing_lag" }
Response: {
  metrics: {
    ingestion_rate: { current: 100000, avg: 80000, max: 150000 },
    processing_lag: { current: 100, avg: 50, max: 500 }
  }
}

GET /api/v1/monitoring/alerts
Headers: { Authorization: Bearer <token> }
Query: { status: "active", severity: "critical" }
Response: {
  alerts: [
    {
      alertId, name, condition, triggeredAt,
      currentValue, threshold
    }
  ]
}
```

### 4.5 Admin APIs
```
POST /api/v1/admin/sources
Headers: { Authorization: Bearer <admin-token> }
Request: {
  name: string,
  type: enum,
  schema: {...}
}
Response: {
  sourceId: UUID,
  apiKey: string
}

DELETE /api/v1/admin/sources/{sourceId}
Headers: { Authorization: Bearer <admin-token> }
Response: { success: boolean }

POST /api/v1/admin/retention-policy
Headers: { Authorization: Bearer <admin-token> }
Request: {
  dataType: string,
  retentionDays: int,
  archiveAfterDays: int
}
Response: { policyId: UUID }
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA SOURCES                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   IoT    │  │  Servers │  │   Apps   │  │  Mobile  │       │
│  │ Devices  │  │   Logs   │  │  Events  │  │   Apps   │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │              │
│       └─────────────┼─────────────┼─────────────┘              │
└─────────────────────┼─────────────┼────────────────────────────┘
                      │             │
                      │  Internet   │
                      │             │
┌─────────────────────┼─────────────┼────────────────────────────┐
│              INGESTION LAYER                                    │
│          ┌───────────▼─────────────▼─────┐                     │
│          │   Global Load Balancer        │                     │
│          │   (GeoDNS / CloudFlare)       │                     │
│          └───────────┬───────────────────┘                     │
│                      │                                         │
│       ┌──────────────┼──────────────┐                          │
│       │              │              │                          │
│  ┌────▼────┐    ┌────▼────┐    ┌────▼────┐                    │
│  │   API   │    │   API   │    │   API   │                    │
│  │ Gateway │    │ Gateway │    │ Gateway │                    │
│  │ (Region │    │ (Region │    │ (Region │                    │
│  │   US)   │    │   EU)   │    │  APAC)  │                    │
│  └────┬────┘    └────┬────┘    └────┬────┘                    │
│       │              │              │                          │
│       └──────────────┼──────────────┘                          │
│                      │                                         │
│              ┌───────▼────────┐                                │
│              │  Message Queue │                                │
│              │  (Kafka/Kinesis│                                │
│              │   Pulsar)      │                                │
│              │  - Partitioned │                                │
│              │  - Replicated  │                                │
│              └───────┬────────┘                                │
└──────────────────────┼─────────────────────────────────────────┘
                       │
┌──────────────────────┼─────────────────────────────────────────┐
│              PROCESSING LAYER                                   │
│          ┌───────────▼─────────────┐                           │
│          │  Stream Processing      │                           │
│          │  (Flink / Spark         │                           │
│          │   Streaming / Storm)    │                           │
│          │                         │                           │
│          │  - Validation           │                           │
│          │  - Transformation       │                           │
│          │  - Aggregation          │                           │
│          │  - Enrichment           │                           │
│          └───────────┬─────────────┘                           │
│                      │                                         │
│       ┌──────────────┼──────────────┐                          │
│       │              │              │                          │
│  ┌────▼─────┐   ┌────▼────┐   ┌────▼────┐                     │
│  │Real-time │   │  Batch  │   │ Complex │                     │
│  │Analytics │   │Processing│  │  Event  │                     │
│  │          │   │ (Spark) │   │Processing│                    │
│  └────┬─────┘   └────┬────┘   └────┬────┘                     │
└───────┼──────────────┼─────────────┼─────────────────────────┘
        │              │             │
┌───────┼──────────────┼─────────────┼─────────────────────────┐
│              STORAGE LAYER          │                         │
│       │              │             │                          │
│  ┌────▼─────┐   ┌────▼────┐   ┌────▼────┐   ┌─────────────┐  │
│  │   Time   │   │ Object  │   │  OLAP   │   │   Search    │  │
│  │  Series  │   │ Storage │   │Database │   │  Engine     │  │
│  │    DB    │   │  (S3/   │   │(ClickH- │   │(Elastic-    │  │
│  │(InfluxDB)│   │  Blob)  │   │ouse)    │   │ search)     │  │
│  │          │   │         │   │         │   │             │  │
│  │  - Hot   │   │ - Raw   │   │- Columnar│  │- Full-text  │  │
│  │   data   │   │  data   │   │- Fast   │   │  search     │  │
│  │  (recent)│   │- Archive│   │ queries │   │- Logs       │  │
│  └──────────┘   └─────────┘   └─────────┘   └─────────────┘  │
│                                                               │
│  ┌─────────────┐   ┌─────────────┐                           │
│  │   Redis     │   │  PostgreSQL │                           │
│  │  (Cache &   │   │  (Metadata, │                           │
│  │   Counters) │   │   Config)   │                           │
│  └─────────────┘   └─────────────┘                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────────┐
│              QUERY & ANALYTICS LAYER                         │
│          ┌───────────▼─────────────┐                         │
│          │   Query Service         │                         │
│          │   (Presto / Druid)      │                         │
│          └───────────┬─────────────┘                         │
│                      │                                       │
│       ┌──────────────┼──────────────┐                        │
│       │              │              │                        │
│  ┌────▼─────┐   ┌────▼────┐   ┌────▼────┐                   │
│  │   API    │   │Dashboard│   │Alerting │                   │
│  │ Gateway  │   │ Service │   │ Service │                   │
│  └──────────┘   └─────────┘   └─────────┘                   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│              MONITORING & MANAGEMENT                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Prometheus  │  │   Grafana    │  │  PagerDuty   │       │
│  │  (Metrics)   │  │ (Dashboard)  │  │  (Alerting)  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Data Flow

```
1. Data Ingestion
   ┌─────────┐
   │ Source  │
   └────┬────┘
        │ HTTP POST / MQTT / gRPC
        ▼
   ┌─────────┐
   │   API   │ - Authentication
   │ Gateway │ - Rate limiting
   └────┬────┘ - Validation
        │
        ▼
   ┌─────────┐
   │  Kafka  │ - Buffering
   │ Topic   │ - Partitioning
   └────┬────┘ - Replication
        │
        
2. Stream Processing
        │
        ▼
   ┌─────────┐
   │  Flink  │ - Parse & validate
   │ Worker  │ - Transform
   └────┬────┘ - Enrich
        │       - Aggregate
        │
        ├──────────────┬─────────────┐
        │              │             │
        ▼              ▼             ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │InfluxDB │   │   S3    │   │ClickHouse│
   │(recent) │   │ (raw)   │   │(analytics)│
   └─────────┘   └─────────┘   └─────────┘

3. Querying
        │
        ▼
   ┌─────────┐
   │  Query  │ - Join multiple sources
   │ Service │ - Aggregate
   └────┬────┘ - Filter
        │
        ▼
   ┌─────────┐
   │   API   │
   │ Response│
   └─────────┘
```

### 5.3 Partitioning Strategy

**Kafka Topics**:
```
Partition by sourceId or eventType

Topic: events-raw
├─ Partition 0 (sourceId % 100 == 0-9)
├─ Partition 1 (sourceId % 100 == 10-19)
├─ ...
└─ Partition 99 (sourceId % 100 == 90-99)

Benefits:
- Even distribution of load
- Parallel processing
- Fault tolerance
```

**Time-Series DB**:
```
Partition by time (sharding by day/hour)

Shard: 2025-11-18
├─ events_2025_11_18_00
├─ events_2025_11_18_01
├─ ...
└─ events_2025_11_18_23

Benefits:
- Fast recent data queries
- Easy data lifecycle management
- Efficient compression per shard
```

## 6. Optimizations & Approaches

### 6.1 Batching for Throughput

**Problem**: Individual event processing is expensive
**Solution**: Micro-batching

# Throughput improvement: 100x
# Individual writes: 1000 events/sec
# Batch writes: 100,000 events/sec

### 6.2 Compression

**Problem**: Large data volume
**Solution**: Compression at multiple levels



### 6.3 Hot/Warm/Cold Storage Tiering

**Strategy**:

### 6.4 Sampling for Analytics

**Problem**: Too much data to process for real-time analytics
**Solution**: Adaptive sampling

### 6.5 Approximate Algorithms

**HyperLogLog for Unique Counts**:

**Count-Min Sketch for Frequency**:

### 6.6 Indexing Strategy

**ClickHouse Indexing**:
```sql
-- Primary index: sparse index on (timestamp, source_id)
CREATE TABLE events (
    timestamp DateTime,
    source_id UInt64,
    event_type String,
    payload String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (timestamp, source_id)
SETTINGS index_granularity = 8192;

-- Skip index for fast filtering
ALTER TABLE events ADD INDEX idx_event_type event_type 
TYPE bloom_filter GRANULARITY 4;

-- Query optimization:
-- Range scan: 1B rows/sec
-- Indexed lookup: 100M rows/sec
```

### 6.7 Pre-aggregation

**Problem**: Repeated aggregation queries are expensive
**Solution**: Materialized views

```sql
-- Create hourly aggregation
CREATE MATERIALIZED VIEW events_hourly_agg
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (timestamp, event_type)
AS SELECT
    toStartOfHour(timestamp) as timestamp,
    event_type,
    count() as event_count,
    uniqHLL(user_id) as unique_users
FROM events
GROUP BY timestamp, event_type;

-- Query speedup: 1000x
-- Raw query: 10 seconds
-- Pre-aggregated query: 10ms
```

### 6.8 Circuit Breaker Pattern

**Problem**: Cascading failures when downstream service fails
**Solution**: Circuit breaker

### 6.9 Back-pressure Handling

**Problem**: Producer faster than consumer
**Solution**: Back-pressure mechanism

### 6.10 Data Validation Pipeline

**Multi-stage validation**:

## 7. Low-Level Design (LLD)


<img width="3303" height="762" alt="image" src="https://github.com/user-attachments/assets/412a2e4b-0153-45b4-adc2-6ecb8d4a98a3" />


### 7.2 Scalability Calculations

**Ingestion**:
- Target: 10M events/sec
- Event size: 1KB average
- Throughput: 10GB/sec = 36TB/hour
- Kafka partitions: 1000 (10K events/sec per partition)
- API servers: 100 (100K events/sec per server)

**Processing**:
- Flink workers: 200 (50K events/sec per worker)
- CPU: 2 cores per worker = 400 cores total
- Memory: 8GB per worker = 1.6TB total

**Storage**:
- Daily ingestion: 36TB/hour × 24 = 864TB/day
- Compressed: 864TB ÷ 10 = 86TB/day
- Yearly: 86TB × 365 = 31PB/year
- Hot (7 days): 86TB × 7 = 602TB (SSD)
- Warm (90 days): 86TB × 83 = 7PB (HDD)
- Cold (rest): 23PB (S3 Glacier)

**Cost Estimation**:
- Hot storage: 602TB × $0.10/GB/month = $60K/month
- Warm storage: 7PB × $0.02/GB/month = $140K/month
- Cold storage: 23PB × $0.004/GB/month = $92K/month
- Total storage: ~$300K/month
- Compute: ~$100K/month
- Network: ~$50K/month
- **Total: ~$450K/month for 10M events/sec**

---

## Interview Discussion Points

### Clarifying Questions:
1. **Data characteristics**: Event size? Schema? Structured vs unstructured?
2. **Scale**: Events per second? Number of sources? Data retention?
3. **Latency**: Real-time or batch? Acceptable delay?
4. **Use cases**: What queries are most common? Analytics type?
5. **Budget**: Cost constraints?
6. **Compliance**: GDPR? Data residency requirements?

### Trade-offs:
1. **Latency vs Throughput**: Real-time processing vs batch
2. **Cost vs Performance**: Hot vs cold storage
3. **Accuracy vs Speed**: Exact vs approximate algorithms
4. **Consistency vs Availability**: Strong vs eventual
5. **Complexity vs Features**: Simple pipeline vs advanced processing

### Extensions:
1. **Machine Learning**: Anomaly detection, predictive analytics
2. **Data Quality**: Validation, cleansing, deduplication
3. **Compliance**: Encryption, access control, audit logs
4. **Multi-tenancy**: Isolation, quotas, billing
5. **Schema Evolution**: Backward compatibility, versioning

This comprehensive design covers all aspects of a high-scale data collection system capable of handling millions of events per second with proper monitoring, alerting, and cost optimization!

