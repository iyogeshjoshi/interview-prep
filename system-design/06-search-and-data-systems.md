# System Design — Search & Data Systems

## Search Architecture

### Why Not Just SQL LIKE / ILIKE?
```sql
SELECT * FROM products WHERE name ILIKE '%wireless headphone%';
-- Full table scan every time
-- No relevance ranking
-- Can't handle typos ("wireles headphon")
-- No stemming ("running" ≠ "run")
-- Performance collapses at >1M rows
```

Full-text search engines solve: inverted indexes, relevance scoring, fuzzy matching, faceting, aggregations.

---

## Elasticsearch / OpenSearch Architecture

### Core Concepts
```
Index    → like a database table (but schema-flexible)
Document → a JSON record (like a row)
Field    → a property in the document
Shard    → a Lucene index; an index is split into N shards
Replica  → copy of a shard for HA + read scaling

Cluster
  └── Node (x3 minimum for production)
        └── Index
              ├── Primary Shard 0  + Replica Shard 0
              ├── Primary Shard 1  + Replica Shard 1
              └── Primary Shard 2  + Replica Shard 2
```

### Inverted Index (How Search Works)
```
Documents:
  doc1: "wireless noise-cancelling headphones"
  doc2: "wireless earbuds bluetooth"
  doc3: "over-ear headphones premium"

Inverted Index:
  "wireless"    → [doc1, doc2]
  "headphones"  → [doc1, doc3]
  "bluetooth"   → [doc2]
  "noise"       → [doc1]

Query: "wireless headphones"
  → intersect [doc1, doc2] AND [doc1, doc3] = doc1 (highest relevance)
  → union for OR queries
```

### Relevance Scoring (BM25)
- **TF** (Term Frequency): how often term appears in doc
- **IDF** (Inverse Doc Frequency): how rare the term is across all docs
- **Field length normalization**: shorter fields score higher for the same match
- BM25 improves on TF-IDF with saturation (prevents spam repetition from gaming score)

### Mapping (Schema)
```json
{
  "mappings": {
    "properties": {
      "name":        { "type": "text", "analyzer": "english" },
      "brand":       { "type": "keyword" },
      "price":       { "type": "float" },
      "category":    { "type": "keyword" },
      "description": { "type": "text" },
      "created_at":  { "type": "date" },
      "tags":        { "type": "keyword" }
    }
  }
}
```
- `text`: analyzed (tokenized, lowercased, stemmed) — for full-text search
- `keyword`: not analyzed — for exact match, filtering, sorting, aggregations
- Never use `text` for filtering/sorting; use `keyword` or add `.keyword` sub-field

### Query DSL
```json
// Bool query (most common — combine must/should/filter/must_not)
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "wireless headphones" } }
      ],
      "filter": [
        { "term":  { "category": "electronics" } },
        { "range": { "price": { "gte": 50, "lte": 300 } } }
      ],
      "should": [
        { "term": { "brand": "Sony" } }
      ]
    }
  },
  "sort": [{ "_score": "desc" }, { "created_at": "desc" }],
  "aggs": {
    "by_brand": { "terms": { "field": "brand" } },
    "price_range": { "histogram": { "field": "price", "interval": 50 } }
  }
}
```

`must` → must match + contributes to score
`filter` → must match + does NOT affect score (cached, faster)
`should` → boosts score if matches; not required (unless only clause)
`must_not` → must not match + no score contribution

### Fuzzy Search & Autocomplete
```json
// Fuzzy: handles typos ("wireles" → "wireless")
{ "match": { "name": { "query": "wireles", "fuzziness": "AUTO" } } }

// Autocomplete with edge_ngram analyzer
// Index "headphone" as: h, he, hea, head, headp, headph, ...
// Query prefix matches instantly

// Completion suggester: optimized for typeahead (stored in memory)
{ "suggest": { "product-suggest": { "prefix": "wire", "completion": { "field": "suggest" } } } }
```

---

## Keeping Search in Sync with Primary Database

### The Core Problem
Your source of truth is PostgreSQL/MongoDB. Elasticsearch is a derived view. How do you keep them in sync without losing writes?

### Option 1: Dual Write (Synchronous)
```typescript
async function createProduct(data: ProductDto) {
  const product = await db.create(data);          // write to DB
  await esClient.index({ index: 'products', document: product }); // write to ES
  return product;
}
// Problem: ES write fails → DB has it, ES doesn't → inconsistency
// Fix: accept eventual consistency; re-sync on failure via retry queue
```

### Option 2: Change Data Capture (CDC) — Preferred
```
PostgreSQL WAL
  → Debezium (Kafka connector)
  → Kafka topic: products.changes
  → ES Sink Connector (Kafka Connect)
  → Elasticsearch

Properties:
- Decoupled: app doesn't know about ES
- At-least-once delivery (idempotent ES writes by doc ID)
- Lag: ~1-3 seconds (acceptable for search)
- Full-reindex: replay Kafka topic from beginning
```

### Option 3: Event-Driven (Publish on Write)
```
App writes to DB → publishes "product.updated" to SQS/EventBridge
  → Lambda consumer → indexes to ES
Simpler than CDC; requires app to publish events
```

### Full Re-index Strategy (Zero Downtime)
```
1. Create new index: products_v2 with updated mapping
2. Write to both products_v1 and products_v2 (dual write during migration)
3. Backfill products_v2 from DB in background
4. Verify document counts match
5. Switch alias: products → products_v2 (atomic in ES)
6. Stop writing to products_v1; delete after verification
```
Always use index aliases in code, never the versioned name directly.

---

## Search at Scale: Operational Concerns

### Index Design Decisions
```
Shard sizing: aim for 10-50GB per shard
  Too few shards: can't parallelize queries
  Too many shards: overhead per shard (memory, coordination)

Shard count is fixed at index creation — plan ahead!
(Use rollover for time-based indices: logs-2024-01, logs-2024-02)

Replica count: at least 1 for production (HA + read throughput)
```

### Hot-Warm-Cold Architecture (for time-series)
```
Hot tier (SSD, high IOPS):   recent data (last 7 days)
Warm tier (SSD, lower IOPS): older data (7d - 90d)
Cold tier (HDD / S3):        archive (90d+, searchable but slow)
Frozen tier (S3 only):       on-demand restore for compliance
```

---

## Data Pipeline Architecture

### Lambda Architecture
```
Batch Layer:
  All data → S3 data lake → Spark/EMR batch jobs → precomputed views
  High latency (hours), high accuracy

Speed Layer:
  Real-time stream → Kafka → Stream processing (Flink/Lambda) → low-latency views
  Low latency, less complete

Serving Layer:
  Merge batch views + speed views → query
```

### Kappa Architecture (Simpler)
```
All data → Kafka (long retention) → Stream processor → Views

Re-process historical: replay Kafka topic from beginning
Simpler: one processing path instead of two
Works when stream processing is powerful enough (Flink)
```

### Data Lake on AWS
```
Sources: RDS (CDC via DMS/Debezium), App events (Kinesis), SaaS webhooks
  ↓
S3 Raw Zone (landing, immutable, original format)
  ↓ (AWS Glue ETL / Spark)
S3 Curated Zone (Parquet/ORC, partitioned by date/entity, compressed)
  ↓
Glue Data Catalog (schema registry for all datasets)
  ↓
Query engines:
  Athena     → ad-hoc SQL (pay per TB scanned)
  Redshift   → BI dashboards, complex aggregations
  EMR/Spark  → large-scale transformations
  QuickSight → dashboards

Cost optimization:
  Parquet + Snappy compression → 5-10x smaller than CSV/JSON
  Partition by date: WHERE date='2024-01' → scan only that partition
  Glue bookmark: track processed S3 objects (incremental ETL)
```

### Streaming Pipeline on AWS
```
Producers → Kinesis Data Streams
  (or MSK / Managed Kafka for higher throughput / replay)
  ↓
Lambda / Kinesis Data Analytics (Flink) → real-time processing
  ↓
Kinesis Firehose → S3 (batched, compressed delivery)
              → Elasticsearch / OpenSearch
              → Redshift
```

---

## OpenSearch on AWS

- AWS-managed fork of Elasticsearch (open source; avoid ES 7.10+ licensing issues)
- **Domains**: Managed cluster; specify instance types + count
- **UltraWarm**: S3-backed warm storage (10x cheaper than hot)
- **Fine-grained access control**: Index/field level security
- **OpenSearch Dashboards**: Kibana equivalent (included)
- **Ingestion**: Logstash, Kinesis Firehose, Lambda, direct API

### Common Production Pattern
```
App → structured logs → CloudWatch Logs
CloudWatch Logs subscription filter → Kinesis Firehose → OpenSearch
OpenSearch Dashboards → ops team
```

---

## Common Interview Design Questions

**"Design a search feature for an e-commerce site with 10M products"**
```
Indexing strategy:
  - product_id (keyword), name (text + keyword), brand (keyword),
    category (keyword), price (float), attributes (nested), stock (integer)
  - Custom analyzer for product names (synonyms: "TV" = "television")
  - Suggest field for autocomplete

Query flow:
  User types → Autocomplete (completion suggester, <50ms)
  User submits → Bool query: must=full-text + filter=category/price/brand
  Results: scored by BM25 + recency boost + in-stock boost

Sync: CDC from PostgreSQL → Kafka → ES consumer
Re-index: zero-downtime alias swap
```

**"How do you handle search ranking for personalization?"**
```
Two-phase approach:
  1. Retrieve: ES gets top-1000 candidates (fast, recall-oriented)
  2. Re-rank: ML model scores by user history, context, business rules
     (can be a Lambda; separate from ES)
     
Business boosts: promoted products, margin-weighted, inventory priority
Feedback loop: click-through data → retrain ranking model
```
