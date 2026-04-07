# Platform Architecture Overview

## Executive Summary

The Enterprise Document Intelligence RAG Platform is a **layered architecture** that transforms unstructured documents into queryable enterprise knowledge. The platform combines workflow orchestration, AI-driven analysis, semantic retrieval, and grounded generation to deliver document intelligence capabilities across multiple business use cases.

**Key Design Principles**:
- **Separation of concerns**: Orchestration, intelligence, storage, and delivery are independent layers
- **Vendor neutrality**: Every component can be replaced without rewriting the platform
- **Reusability**: Same platform core supports multiple use cases through configuration
- **Operational agility**: Business workflows can change without code deployment

---

## Architecture Layers

The platform uses a six-layer architecture:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 6: Experience & Delivery                             │
│  (Reports, Chat Interfaces, APIs, Enterprise Integrations)  │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Layer 5: Model Layer                                       │
│  (LLM Generation, Embedding Models)                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Layer 4: Knowledge & Memory                                │
│  (Vector Database, Metadata Storage)                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Layer 3: GenAI Platform Core                               │
│  (Ingestion, Retrieval, Analysis, Generation Services)      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Layer 2: Workflow Orchestration                            │
│  (Event Handling, Routing, Lifecycle Management)            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  Layer 1: Enterprise Data Sources                           │
│  (Google Drive, SharePoint, Email, APIs)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer Details

### Layer 1: Enterprise Data Sources

**Responsibility**: Document intake from enterprise systems

**Components**:
- Google Drive integration (file monitoring, download)
- SharePoint connector (site monitoring, content extraction)
- Email processing (attachment extraction)
- REST API endpoints (programmatic document submission)

**Key Characteristics**:
- Event-driven (triggers on new document upload)
- Pull-based for batch processing (scheduled imports)
- Source-agnostic (abstraction layer for different systems)

---

### Layer 2: Workflow Orchestration (n8n)

**Responsibility**: Business workflow automation and lifecycle management

**Capabilities**:
- **Event detection**: Monitor source systems for new documents
- **Routing logic**: Send different document types to different processing paths
- **Error handling**: Retry failed operations, escalate persistent failures
- **Output delivery**: Write reports, send notifications, trigger downstream systems
- **Workflow versioning**: A/B test different processing approaches

**Technology**: n8n (visual workflow designer)

**Why n8n**:
- Visual workflows editable by business users
- Strong integration ecosystem (400+ connectors)
- Self-hosted option (data stays on-premise)
- API-first design (programmatic workflow management)

**Alternatives**: Apache Airflow (batch-oriented), Prefect (Python-native), Azure Logic Apps (cloud-only)

---

### Layer 3: GenAI Platform Core (FastAPI)

**Responsibility**: Reusable AI capabilities exposed as REST APIs

**Services**:

#### Ingestion Service (`/api/v1/ingest`)
- Document chunking (semantic splitting)
- Embedding generation (sentence-transformers)
- Vector storage (Qdrant)
- Metadata indexing

#### Retrieval Service (`/api/v1/retrieve`)
- Semantic search (vector similarity)
- Metadata filtering (document type, date range, source)
- Result reranking (relevance scoring)
- Context assembly (prepare chunks for LLM)

#### Analysis Service (`/api/v1/analyze`)
- Document classification (contract, policy, proposal, etc.)
- Entity extraction (dates, monetary values, parties, obligations)
- Summarization (executive summaries, key points)
- Risk flagging (contractual risks, compliance issues)

#### Generation Service (`/api/v1/generate`)
- Grounded answer generation (RAG pattern)
- Source attribution (chunk IDs, document references)
- Multi-document synthesis
- Report generation

**Technology**: FastAPI (Python), Pydantic (data validation), OpenAPI 3.0 (API specs)

**Design Principles**:
- Stateless services (horizontal scaling)
- Domain-driven boundaries (services map to capabilities, not tech layers)
- Versioned APIs (/v1/, /v2/ for breaking changes)
- OpenAPI specs for contract-first development

---

### Layer 4: Knowledge & Memory

**Responsibility**: Persistent storage of document intelligence

**Components**:

#### Vector Database (Qdrant)
- Stores document chunk embeddings (384-dim or 768-dim)
- Supports semantic similarity search (HNSW algorithm)
- Collection-based isolation (separate vector spaces per use case)
- Metadata filtering during search (not post-search filtering)

#### Metadata Storage
- Document provenance (source, upload date, processing status)
- Chunk relationships (which chunks belong to which document)
- Processing audit trail (who ingested, when, what version)

**Technology**: Qdrant (self-hosted vector DB), PostgreSQL (metadata)

**Scaling Characteristics**:
- Qdrant scales to 1M+ vectors per collection
- Sub-100ms p95 query latency at target scale
- Horizontal scaling via Qdrant clustering (when needed)

---

### Layer 5: Model Layer

**Responsibility**: LLM inference and embedding generation

**Components**:

#### LLM (Generation)
- Provider: OpenRouter-compatible endpoint (Azure OpenAI, AWS Bedrock, Anthropic Claude)
- Use cases: Document analysis, answer generation, summarization
- Model selection: GPT-4, Claude 3, or domain-specific fine-tuned models

#### Embedding Model
- Provider: Local sentence-transformers (self-hosted)
- Model: `all-MiniLM-L6-v2` (384-dim, fast) or `all-mpnet-base-v2` (768-dim, higher quality)
- Deployment: Runs in FastAPI service container (CPU inference initially, GPU-accelerated if needed)

**Design Decisions**:
- **Why local embeddings?** Data locality, cost predictability, vendor independence (see ADR 002)
- **Why OpenRouter-compatible?** Vendor neutrality — can swap from Azure → Bedrock → Anthropic without changing code

---

### Layer 6: Experience & Delivery

**Responsibility**: Surface intelligence through user-facing channels

**Output Channels**:

#### Structured Reports (Google Docs)
- Automated document summaries
- Formatted analysis results (tables, bullet points, risk matrices)
- Stakeholder distribution via email or Slack

#### Chat Interfaces
- Open WebUI (self-hosted chat frontend)
- Custom web application (React + FastAPI backend)
- Slack bot or Teams bot integration

#### REST APIs
- `/api/v1/query` - Conversational Q&A over document corpus
- `/api/v1/documents` - Document listing and metadata
- `/api/v1/reports` - Programmatic report generation

#### Enterprise System Integrations
- ITSM ticket enrichment (automatically attach relevant policy docs)
- CRM integration (attach past proposals to sales opportunities)
- Email responses (auto-reply with relevant document references)

---

## Data Flow: End-to-End

### Ingestion Flow

```
1. Document uploaded to Google Drive
   ↓
2. n8n workflow detects new file event
   ↓
3. n8n downloads document
   ↓
4. POST /api/v1/analyze
   → Returns: {summary, entities, risks}
   ↓
5. n8n generates Google Docs report
   ↓
6. POST /api/v1/ingest (parallel to step 4)
   → Chunks document
   → Generates embeddings (local model)
   → Stores vectors in Qdrant
   ↓
7. n8n sends notification
```

**Latency**: 10-30 seconds for typical document (2000 tokens)

### Query Flow

```
1. User asks: "What is the contract value?"
   ↓
2. POST /api/v1/retrieve
   → Embeds question (local model, 25ms)
   → Queries Qdrant (semantic search, 40ms)
   → Returns top 5 chunks
   ↓
3. POST /api/v1/generate
   → Sends chunks + question to LLM
   → LLM generates grounded answer
   → Returns answer + source attribution
   ↓
4. Response: "$180,000 USD [Source: contracts/acme.pdf, p.4]"
```

**Latency**: 300-800ms end-to-end (embedding + retrieval + generation)

---

## Scalability & Performance

### Current Deployment (Docker Compose)

| Metric | Capacity |
|--------|----------|
| Documents | 50,000 |
| Vectors | 250,000 (5 chunks/doc) |
| Concurrent queries | 10-20 |
| Query latency (p95) | <1 second |
| Ingestion throughput | 1000 docs/hour |

**Infrastructure**: Single host (4 vCPU, 16 GB RAM)

### Future Deployment (Kubernetes)

| Metric | Capacity |
|--------|----------|
| Documents | 1,000,000+ |
| Vectors | 5,000,000+ |
| Concurrent queries | 100+ |
| Query latency (p95) | <500ms |
| Ingestion throughput | 10,000 docs/hour |

**Infrastructure**: Multi-node cluster, auto-scaling FastAPI pods, Qdrant cluster

---

## Security & Compliance

### Data Residency
- All document text stays within enterprise infrastructure
- Vector embeddings stored on-premise or in enterprise VPC
- No document content sent to external APIs (except LLM generation)

### Access Control
- API authentication via JWT tokens
- Collection-level access control (Qdrant supports authentication)
- Workflow-level permissions (n8n RBAC)

### Audit Trail
- All document ingestion events logged
- Query history tracked (who asked what, when)
- Processing lineage (which version of model/chunking was used)

---

## Monitoring & Observability

### Key Metrics

**Platform Health**:
- Service uptime (n8n, FastAPI, Qdrant)
- Request latency (p50, p95, p99)
- Error rates (4xx, 5xx responses)

**Business Metrics**:
- Documents ingested per day
- Queries per day
- Query satisfaction (thumbs up/down feedback)
- Retrieval accuracy (top-5 recall)

**Cost Metrics**:
- LLM token usage
- Infrastructure cost per document
- Cost per query

### Tooling
- Prometheus (metrics collection)
- Grafana (dashboards)
- Sentry (error tracking)
- Structured logging (JSON logs, ELK stack)

---

## Evolution Roadmap

### Phase 1: Foundation (Current)
✅ Core platform services operational  
✅ Contract intelligence use case live  
✅ Docker Compose deployment  

### Phase 2: Expansion (Q3 2024)
🔄 Policy compliance use case  
🔄 Proposal generation use case  
🔄 Kubernetes migration for scale  

### Phase 3: Maturity (Q4 2024)
📋 Advanced retrieval (hybrid search, reranking)  
📋 Multi-tenant architecture  
📋 Observability improvements (distributed tracing)  

### Phase 4: Enterprise-Grade (2025)
📋 High availability (multi-region)  
📋 Advanced security (PII detection, content filtering)  
📋 Performance optimization (GPU acceleration, caching)  

---

## Related Documentation

- [Architecture Decision Records](../architecture-decision-records/) - Key design decisions and trade-offs
- [Trade-Off Analysis](../docs/trade-off-analysis.md) - Detailed comparison of architectural choices
- [Deployment Patterns](../deployment/deployment-patterns.md) - Cloud deployment topologies
- [Workflow Patterns](../workflows/workflow-patterns.md) - Common n8n workflow designs

---

*For questions about the architecture, see the ADRs or [open an issue](../../issues).*
