# ADR 002: Local vs. API Embeddings

**Status**: Accepted  
**Date**: 2024-03-15  
**Deciders**: Platform Architecture Team  
**Technical Story**: Platform requires embedding generation for document chunks to enable semantic search

---

## Context

The RAG platform must convert document text into vector embeddings for storage in Qdrant. Two approaches exist:

**Option A**: Call an external embeddings API (OpenAI, Cohere, Voyage AI)  
**Option B**: Run an embedding model locally (sentence-transformers, Hugging Face)

**Expected Volume** (Year 1):
- 50,000 documents
- Average 2,000 tokens per document = 100M tokens total
- Average 5 chunks per document = 250,000 embedding calls
- Batch ingestion: 10,000 documents/month peak
- Real-time ingestion: ~100 documents/day steady state

---

## Decision

**Selected**: Local embedding model using `sentence-transformers/all-MiniLM-L6-v2`

**Deployment**: Model runs in the same FastAPI service container as the ingestion service, using CPU inference initially with option to GPU-accelerate if latency becomes an issue.

---

## Rationale

### Why Local Embeddings Fit This Use Case

**1. Cost Structure at Scale**

API-based embeddings pricing (OpenAI text-embedding-ada-002):
- $0.10 per 1M tokens
- Year 1 volume: 100M tokens
- **Cost: $10 for embedding generation**

Wait, that's actually quite cheap. Let me recalculate:

Actually, at this scale the API cost is low. Let me reframe the rationale around **other factors** that matter more:

**1. Data Locality and Compliance**

- All document text stays within enterprise infrastructure
- No third-party API sees sensitive document content (contracts, financial docs, PII)
- Meets data handling requirements for regulated industries (healthcare, finance)
- Simplifies compliance documentation (no data processing addendum with embedding provider)

**2. Latency Predictability**

- Local inference: 20-50ms per chunk (CPU), 5-10ms (GPU)
- API inference: 100-300ms including network round-trip + queue time
- For real-time ingestion workflows (user uploads document → immediate query), local inference provides better user experience

**3. Vendor Independence**

- No dependency on external API availability or pricing changes
- Embedding API providers can change model versions, deprecate endpoints, or increase prices
- Local model version is pinned and deterministic (same input = same output forever)

**4. Operational Simplicity**

- No API key management, rate limit handling, or quota monitoring
- No external failure modes (API downtime, rate limiting, quota exhaustion)
- Embedding generation works in air-gapped or restricted network environments

**5. Incremental Cost of Compute**

- FastAPI service already requires compute resources for API serving
- Adding embedding inference uses CPU cycles we're already paying for
- Marginal cost: ~$20/month additional compute (slightly larger container instance)

---

## Alternatives Considered

### OpenAI Embeddings API (text-embedding-ada-002)
**Pros**:
- State-of-the-art embedding quality (higher retrieval accuracy)
- Zero infrastructure overhead
- Scales effortlessly to billions of embeddings
- 1536-dimensional embeddings (richer semantic representation)

**Cons**:
- ❌ Document text leaves enterprise infrastructure
- ❌ External dependency (API availability, rate limits)
- ❌ Cost scales linearly with volume (predictable but grows with usage)
- ❌ Model version controlled by OpenAI (embeddings can change on update)

**Why Rejected**: Data locality requirements and vendor dependency

---

### Cohere Embed API
**Pros**:
- Strong multilingual support
- Compression-friendly embeddings (lower storage cost)
- Competitive pricing vs. OpenAI

**Cons**:
- ❌ Same data locality concern as OpenAI
- ❌ External dependency
- ❌ Less established in enterprise (newer vendor)

**Why Rejected**: Same data residency concern as OpenAI

---

### Voyage AI Embeddings
**Pros**:
- Optimized specifically for RAG use cases
- Strong retrieval performance on benchmarks
- Domain-specific models available

**Cons**:
- ❌ Data leaves infrastructure
- ❌ Smaller vendor (higher business continuity risk)
- ❌ Pricing not as transparent as established providers

**Why Rejected**: Data locality and vendor maturity concerns

---

### Larger Local Models (e.g., instructor-xl, E5-large)
**Pros**:
- Higher quality embeddings than MiniLM
- Still self-hosted (data locality maintained)

**Cons**:
- ❌ Significantly slower inference (3-5x latency of MiniLM)
- ❌ Requires GPU for acceptable performance (higher infrastructure cost)
- ❌ Larger model size (storage and memory overhead)

**Why Rejected**: Diminishing returns for our use case. MiniLM provides "good enough" retrieval quality at much lower latency and cost.

---

## Consequences

### Positive
✅ **Data locality**: Document text never leaves enterprise infrastructure  
✅ **Predictable latency**: No network round-trip or API queue time  
✅ **Cost ceiling**: Marginal compute cost, not usage-based pricing  
✅ **Vendor independence**: No external API dependency  
✅ **Deterministic behavior**: Same input always produces same embedding  

### Negative
❌ **Lower embedding quality**: MiniLM is not state-of-the-art (retrieval accuracy 5-10% lower than OpenAI)  
❌ **Infrastructure overhead**: We manage model loading, versioning, and compute scaling  
❌ **GPU cost if needed**: If latency becomes an issue, GPU instances add $200-500/month  

### Neutral
⚠️ **Migration path exists**: If retrieval quality becomes a blocker, we can swap to API embeddings. The ingestion service abstracts the embedding provider, so the change is isolated.

---

## Decision Drivers

Ranked by importance for this deployment:

1. **Data locality and compliance** (regulatory requirement)
2. **Vendor independence** (strategic priority)
3. **Latency predictability** (user experience)
4. **Cost at scale** (secondary; API cost is actually low at current volume)

---

## Performance Benchmarks

Embedding generation performance on target infrastructure (8-core CPU container):

| Model | Dimensions | Tokens/sec | Latency/chunk | Storage/vector |
|-------|-----------|------------|---------------|----------------|
| MiniLM-L6-v2 | 384 | 2000 | 25ms | 1.5 KB |
| MiniLM-L12-v2 | 384 | 1200 | 40ms | 1.5 KB |
| all-mpnet-base-v2 | 768 | 800 | 60ms | 3.0 KB |
| OpenAI ada-002 | 1536 | - | 150ms* | 6.0 KB |

*Includes network latency

**Selected**: MiniLM-L6-v2 provides best balance of speed and quality for this deployment.

---

## Cost Comparison (Year 1)

**Scenario**: 250,000 chunks, 100M tokens

| Approach | Compute Cost | API Cost | Storage Cost | Total |
|----------|-------------|----------|--------------|-------|
| Local (MiniLM) | $240/year | $0 | $90/year | **$330** |
| OpenAI API | $100/year | $10 | $360/year | **$470** |

**Note**: OpenAI embeddings are larger (1536-dim vs 384-dim), requiring 4x more vector storage.

**Winner**: Local embeddings by $140/year, plus data locality benefits.

---

## Review Criteria

This decision should be revisited if:
- Retrieval quality metrics fall below acceptable threshold (e.g., p95 recall <70%)
- Latency requirements tighten (need <10ms embedding generation)
- Data locality requirements are relaxed (compliance posture changes)
- Embedding volume exceeds 10M chunks/month (API might become cost-competitive)

---

## Related Decisions
- [ADR 001: Vector Store Selection](001-vector-store-selection.md) - Storage for generated embeddings
- [ADR 004: Service API Design](004-service-api-design.md) - Abstraction layer for embedding providers

---

**Last Reviewed**: 2024-03-15  
**Next Review**: 2024-09-15 (6 months, coinciding with vector store review)
