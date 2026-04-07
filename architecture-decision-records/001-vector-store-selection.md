# ADR 001: Vector Store Selection (Qdrant)

**Status**: Accepted  
**Date**: 2024-03-15  
**Deciders**: Platform Architecture Team  
**Technical Story**: Platform requires persistent vector storage for document embeddings with semantic search capabilities

---

## Context

The platform needs a vector database to store and retrieve document embeddings. The vector store must support:
- Semantic similarity search across 10K+ documents initially, scaling to 1M+ documents
- Metadata filtering (document source, upload date, classification tags)
- Low-latency retrieval (sub-100ms p95) for interactive queries
- Integration with existing infrastructure (Docker, Kubernetes)
- Support for multiple collections (different use cases or tenants)

**Scale Requirements** (Year 1):
- 50,000 documents
- Average 5 chunks per document = 250,000 vectors
- 768-dimensional embeddings (sentence-transformers)
- 10-50 concurrent queries

---

## Decision

**Selected**: Qdrant (self-hosted, containerized deployment)

**Deployment Model**: Docker container on existing Kubernetes cluster, with persistent volume for index storage

---

## Rationale

### Why Qdrant Fits This Use Case

**1. Self-Hosted Control**
- Data stays within enterprise infrastructure (compliance requirement for regulated industries)
- No vendor lock-in on the storage layer
- Predictable cost structure (infrastructure cost, not per-query or per-vector pricing)

**2. Performance Profile**
- Benchmarked at 15-20ms p95 latency for our target scale (250K vectors)
- HNSW index algorithm provides good accuracy/speed trade-off
- Supports filtering during search (not post-filtering), reducing irrelevant retrievals

**3. Operational Fit**
- Docker-native, integrates cleanly with existing container orchestration
- REST API matches our FastAPI service layer design
- Python client library well-maintained and production-ready

**4. Feature Completeness**
- Payload/metadata filtering (critical for multi-tenant or multi-use-case deployment)
- Quantization support for index size reduction (useful at >1M vectors)
- Collection-based isolation (supports multiple use cases on same infrastructure)

---

## Alternatives Considered

### Pinecone (Managed SaaS)
**Pros**:
- Fully managed, no operational overhead
- Excellent performance at very large scale (>10M vectors)
- Strong developer experience

**Cons**:
- ❌ Data leaves enterprise infrastructure (compliance blocker for this deployment)
- ❌ Cost scales per query + storage (unpredictable at enterprise query volume)
- ❌ Vendor lock-in (migration would require full re-indexing)
- ❌ Limited control over instance sizing and scaling triggers

**Why Rejected**: Data residency requirements and cost unpredictability at enterprise scale

---

### Weaviate (Self-Hosted)
**Pros**:
- Self-hosted option available
- Strong hybrid search (vector + keyword) capabilities
- GraphQL API (useful for complex data relationships)

**Cons**:
- ❌ Higher resource requirements (RAM-intensive at scale)
- ❌ GraphQL adds complexity we don't need (simple REST is sufficient)
- ❌ Less mature Python client compared to Qdrant

**Why Rejected**: Higher operational complexity without corresponding feature benefit for this use case

---

### pgvector (PostgreSQL Extension)
**Pros**:
- Runs on existing PostgreSQL infrastructure
- Familiar operational model (we already run Postgres)
- Joins vector data with relational metadata in single query

**Cons**:
- ❌ Performance degrades faster at scale (not optimized for vector workloads)
- ❌ No native HNSW support in production-ready versions (slower queries)
- ❌ Index size grows faster than specialized vector DBs (disk cost at scale)

**Why Rejected**: Performance profile doesn't meet sub-100ms p95 latency requirement at target scale

---

### Elasticsearch with k-NN Plugin
**Pros**:
- We already run Elasticsearch for search
- Could consolidate infrastructure
- Strong ecosystem and tooling

**Cons**:
- ❌ k-NN plugin is bolt-on, not core to Elasticsearch design
- ❌ Resource-heavy (Elasticsearch overhead + vector search)
- ❌ More complex to tune for vector-specific performance

**Why Rejected**: Using a general-purpose search engine for specialized vector workload introduces unnecessary complexity

---

## Consequences

### Positive
✅ **Data residency**: All vectors stay on-premise or in enterprise VPC  
✅ **Predictable costs**: Infrastructure-based pricing, not usage-based  
✅ **Portability**: Can migrate to other vector stores without API changes (abstraction in FastAPI service layer)  
✅ **Performance**: Meets latency requirements at target scale  
✅ **Operational simplicity**: Docker-native deployment matches existing patterns  

### Negative
❌ **Self-hosting overhead**: We own backup, monitoring, and scaling operations  
❌ **No free tier at scale**: Once we exceed single-instance limits, we manage clustering ourselves  
❌ **Less mature ecosystem**: Qdrant has smaller community vs. Pinecone or Weaviate  

### Neutral
⚠️ **Migration path exists**: If requirements change (e.g., need for >10M vectors), we can swap to Pinecone or Weaviate. The FastAPI ingestion service abstracts the vector store, so the migration is isolated to that service.

---

## Decision Drivers

Ranked by importance for this deployment:

1. **Data residency requirements** (compliance mandate)
2. **Cost predictability** (enterprise budget planning)
3. **Performance at target scale** (user experience)
4. **Operational fit** (existing Kubernetes infrastructure)
5. **Vendor neutrality** (avoid lock-in)

---

## Review Criteria

This decision should be revisited if:
- Document count exceeds 1M vectors (need to benchmark Qdrant clustering vs. managed alternatives)
- Data residency requirements change (managed SaaS becomes viable)
- Operational overhead of self-hosting Qdrant exceeds 20% of platform engineering time
- Retrieval latency p95 exceeds 100ms consistently

---

## Related Decisions
- [ADR 002: Local vs. API Embeddings](002-local-vs-api-embeddings.md) - Embedding generation strategy
- [ADR 005: Multi-Use-Case Reusability](005-multi-use-case-reusability.md) - Collection isolation patterns

---

**Last Reviewed**: 2024-03-15  
**Next Review**: 2024-09-15 (6 months)
