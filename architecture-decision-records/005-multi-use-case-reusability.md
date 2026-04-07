# ADR 005: Multi-Use-Case Reusability Pattern

**Status**: Accepted  
**Date**: 2024-03-15  
**Deciders**: Platform Architecture Team  
**Technical Story**: Platform must support multiple enterprise use cases without rebuilding core infrastructure

---

## Context

The initial implementation targets **contract intelligence**: analyzing vendor contracts, extracting key terms, and answering questions about contractual obligations.

However, other business units have similar needs:
- **Compliance team**: Query policy documents and regulatory requirements
- **Sales team**: Generate proposals using past successful proposals
- **Architecture team**: Build a knowledge base from past architecture decisions
- **HR team**: Answer questions about employee handbooks and policies

**Design Question**: Should we build separate systems for each use case, or design for reusability?

---

## Decision

**Selected**: Build a **reusable platform core** that supports multiple use cases through configuration, not code changes

**Platform Core** (reusable across all use cases):
- Ingestion service (chunking, embedding, vector storage)
- Retrieval service (semantic search)
- Generation service (grounded answer generation)
- Knowledge layer (Qdrant vector store)

**Use Case Layer** (varies per use case):
- Source system integrations (Google Drive folder, SharePoint site, Confluence space)
- Prompts and analysis templates (contract extraction vs. policy interpretation)
- Output channels (Google Docs reports vs. Slack bot vs. API responses)
- Metadata schemas (contract-specific fields vs. policy-specific fields)

---

## Rationale

### Why Reusability Matters

**1. Faster Time-to-Value for New Use Cases**

With a reusable platform:
- Contract intelligence → 8 weeks to build (includes platform)
- Policy compliance assistant → 2 weeks to configure (platform already exists)
- Proposal generator → 2 weeks to configure
- Architecture knowledge base → 2 weeks to configure

Without reusability:
- Each use case takes 6-8 weeks (rebuild ingestion, retrieval, storage every time)

**2. Consistent User Experience**

All use cases deliver:
- Grounded answers with source attribution
- Sub-second query response time
- Same quality of retrieval and generation

Users don't encounter "the compliance bot works but the proposal generator is slow" inconsistency.

**3. Centralized Platform Improvements**

When we improve the platform core (better chunking strategy, faster retrieval, improved prompts), **all use cases benefit automatically**.

Example: Upgrade to hybrid search (vector + keyword) → improves retrieval for contracts, policies, proposals, and architecture docs simultaneously.

**4. Reduced Operational Overhead**

One platform to monitor, secure, scale, and maintain.

**Not**: 4 separate vector stores, 4 separate ingestion pipelines, 4 separate deployment pipelines.

---

## Reusability Pattern

### What Stays the Same (Platform Core)

```
┌─────────────────────────────────────┐
│      Platform Core (Reusable)      │
├─────────────────────────────────────┤
│ • Ingestion Service                 │
│   - Chunking logic                  │
│   - Embedding generation            │
│   - Vector storage                  │
│                                     │
│ • Retrieval Service                 │
│   - Semantic search                 │
│   - Metadata filtering              │
│   - Reranking                       │
│                                     │
│ • Generation Service                │
│   - Grounded answer generation      │
│   - Source attribution              │
│                                     │
│ • Knowledge Layer                   │
│   - Qdrant vector database          │
│   - Metadata storage                │
└─────────────────────────────────────┘
```

### What Changes Per Use Case (Configuration Layer)

| Use Case | Source Integration | Metadata Schema | Prompts | Output Channel |
|----------|-------------------|-----------------|---------|----------------|
| **Contract Intelligence** | Legal Google Drive | contract_type, vendor, value, expiry_date | Extract: obligations, risks, financial terms | Google Docs report |
| **Policy Compliance** | Compliance SharePoint | policy_category, effective_date, applies_to | Interpret: policy requirements, exceptions | Chat interface |
| **Proposal Generation** | Sales repository | rfp_type, client_industry, win_rate | Generate: executive summary, solution approach | Draft in Google Docs |
| **Architecture KB** | Confluence + GitHub | tech_domain, decision_date, status | Retrieve: past decisions, design patterns | Slack bot |

**Key Insight**: Same platform services, different *configuration* of sources, prompts, and outputs.

---

## Implementation Approach

### Collection-Based Isolation

Qdrant supports **collections** — logically separate vector stores within the same instance.

```python
# Contract Intelligence Collection
collection_name = "contracts"
metadata_schema = {
    "document_type": "string",
    "vendor": "string",
    "contract_value": "integer",
    "expiry_date": "datetime"
}

# Policy Compliance Collection
collection_name = "policies"
metadata_schema = {
    "policy_category": "string",
    "effective_date": "datetime",
    "department": "string"
}
```

**Benefits**:
- Same Qdrant instance → single infrastructure to manage
- Collections are isolated → no cross-contamination of search results
- Per-collection metadata schemas → each use case gets custom filtering

### Workflow-Based Configuration

n8n workflows configure per-use-case behavior:

**Contract Intelligence Workflow**:
```
[Google Drive Monitor] 
  → Download Doc
  → POST /api/v1/analyze {"analysis_type": "contract"}
  → POST /api/v1/ingest {"collection": "contracts"}
  → Generate Contract Summary
  → Write to Google Docs
```

**Policy Compliance Workflow**:
```
[SharePoint Monitor]
  → Download Doc
  → POST /api/v1/analyze {"analysis_type": "policy"}
  → POST /api/v1/ingest {"collection": "policies"}
  → Index for Search
  → Notify Compliance Team
```

**Key Point**: Same API endpoints (`/analyze`, `/ingest`), different parameters and downstream actions.

---

## Alternatives Considered

### Separate Systems Per Use Case
**Pros**:
- Complete independence (no shared failure modes)
- Optimized specifically for each use case

**Cons**:
- ❌ 4x infrastructure cost (4 vector stores, 4 ingestion services)
- ❌ 4x development time (rebuild everything for each use case)
- ❌ 4x operational overhead (monitoring, security, scaling)
- ❌ Improvements don't transfer (better chunking for contracts doesn't help policies)

**Why Rejected**: Cost and time inefficiency

---

### Single Shared Collection (No Isolation)
**Pros**:
- Simplest possible design
- No collection management needed

**Cons**:
- ❌ Search results leak across use cases (policy query returns contract chunks)
- ❌ Metadata schema conflicts (contracts need "value", policies need "department")
- ❌ Can't tune per-use-case (chunk size ideal for contracts might not work for policies)

**Why Rejected**: Cross-contamination risk and lack of customization

---

### Separate Platform Instances Per Use Case
**Pros**:
- Complete isolation (no shared state)
- Each instance can run different platform versions

**Cons**:
- ❌ Higher infrastructure cost (4 instances of everything)
- ❌ Platform improvements must be replicated 4 times
- ❌ Harder to consolidate if use cases converge

**Why Rejected**: Infrastructure overhead without corresponding benefit

---

## Consequences

### Positive
✅ **Faster use case deployment**: 2 weeks vs. 8 weeks for subsequent use cases  
✅ **Consistent experience**: All use cases get platform improvements automatically  
✅ **Lower TCO**: Shared infrastructure, single operational surface  
✅ **Centralized learning**: Insights from one use case improve all others  

### Negative
❌ **Shared failure mode**: Platform core outage affects all use cases  
❌ **Coordination overhead**: Breaking changes to platform APIs require coordination across use case teams  
❌ **Noisy neighbor risk**: One use case's query spike affects others  

### Mitigations
- **Rate limiting per collection**: Prevent one use case from monopolizing resources
- **Circuit breakers**: Degrade gracefully if platform core is unhealthy
- **Collection-level monitoring**: Track resource usage per use case

---

## Scaling the Pattern

As use cases grow, the platform can evolve:

**Phase 1** (Current): Single platform instance, collection-based isolation  
**Phase 2** (6+ use cases): Add read replicas for retrieval service, single write path  
**Phase 3** (10+ use cases): Shard by use case category (high-query vs. high-ingestion)  
**Phase 4** (20+ use cases): Consider multi-tenant platform with resource quotas  

The architecture supports this evolution without breaking existing use cases.

---

## Use Case Comparison

### Contract Intelligence vs. Policy Compliance

| Dimension | Contract Intelligence | Policy Compliance |
|-----------|----------------------|-------------------|
| **Source** | Legal Google Drive | Compliance SharePoint |
| **Volume** | 5,000 contracts | 2,000 policies |
| **Update Frequency** | Weekly (new contracts) | Monthly (policy updates) |
| **Query Pattern** | Batch + ad-hoc | Real-time only |
| **Output** | Reports | Chat responses |
| **Platform Services Used** | Ingest, Retrieve, Analyze, Generate | Ingest, Retrieve, Generate |

**Same platform core, different operational profile.** The architecture handles both.

---

## Decision Drivers

Ranked by importance:

1. **Time-to-value for new use cases** (business agility)
2. **Total cost of ownership** (infrastructure efficiency)
3. **Consistent user experience** (quality across use cases)
4. **Centralized improvements** (platform benefits all use cases)

---

## Review Criteria

This decision should be revisited if:
- Number of use cases exceeds 20 (may need multi-tenant architecture)
- Use cases diverge significantly (95% of code is customization, 5% is platform)
- Shared platform becomes a bottleneck (coordination overhead > efficiency gains)

---

## Related Decisions
- [ADR 001: Vector Store Selection](001-vector-store-selection.md) - Collection-based isolation pattern
- [ADR 003: Orchestration Layer Separation](003-orchestration-layer-separation.md) - Workflow-based configuration

---

**Last Reviewed**: 2024-03-15  
**Next Review**: 2024-12-15 (after 3 use cases are live)
