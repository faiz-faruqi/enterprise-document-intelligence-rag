# ADR 003: Orchestration Layer Separation

**Status**: Accepted  
**Date**: 2024-03-15  
**Deciders**: Platform Architecture Team  
**Technical Story**: Platform requires workflow orchestration for document ingestion, processing, and output delivery

---

## Context

The platform must coordinate multiple steps when processing documents:
1. Detect new documents in source systems (Google Drive, SharePoint)
2. Download and extract text
3. Send to AI services for analysis (summarization, entity extraction)
4. Send to ingestion service for vector storage
5. Generate business reports
6. Deliver outputs to stakeholders (Google Docs, email, APIs)
7. Handle errors and retries

**Two architectural approaches exist:**

**Option A**: Embed all orchestration logic inside the AI service layer (FastAPI)  
**Option B**: Separate orchestration into a dedicated workflow engine (n8n, Airflow, Prefect)

---

## Decision

**Selected**: Separate orchestration layer using **n8n** as the workflow engine

**Division of Responsibility**:
- **n8n (Orchestration Layer)**: Event detection, routing, lifecycle management, output delivery, error handling
- **FastAPI (AI Service Layer)**: AI capabilities exposed as stateless REST APIs

---

## Rationale

### Why Separation Matters

**1. Separation of Concerns**

Orchestration logic and AI logic have fundamentally different characteristics:

| Orchestration Logic | AI Service Logic |
|---------------------|------------------|
| Stateful workflows | Stateless operations |
| Long-running (minutes to hours) | Fast (seconds) |
| Error handling and retries | Pure computation |
| Integration with external systems | Self-contained processing |
| Business rule routing | Technical transformation |

Mixing these in a single service creates operational complexity and tight coupling.

**2. Independent Scalability**

- Orchestration engine scales based on **concurrent workflows** (10-100 active flows)
- AI service scales based on **request throughput** (100-1000 req/sec at peak)

These have different resource profiles. Separating them allows independent horizontal scaling.

**3. Reusability of AI Services**

FastAPI services can be called by:
- n8n workflows (scheduled batch processing)
- Web applications (real-time user requests)
- Other microservices (internal APIs)
- External systems (via API gateway)

If orchestration logic lived inside FastAPI, the service would be tightly coupled to n8n's specific workflow patterns.

**4. Workflow Visibility and Debugging**

n8n provides:
- Visual workflow designer (non-developers can understand and modify flows)
- Execution history with step-by-step logs
- Error inspection and manual retry
- A/B testing of workflow variations

Embedding orchestration in Python code loses this visibility. Debugging becomes "read logs and trace through code."

**5. Operational Agility**

Business users can modify workflows without touching code:
- Change routing rules (send legal docs to one folder, HR docs to another)
- Add notification steps (email stakeholders when high-risk contracts are detected)
- Adjust retry policies (retry 3 times with exponential backoff)
- Connect new source systems (add SharePoint alongside Google Drive)

This agility is impossible when orchestration logic is compiled into the service.

---

## Alternatives Considered

### Embedded Orchestration in FastAPI (Celery + Redis)
**Pros**:
- Single technology stack (all Python)
- Lower infrastructure complexity (fewer services to manage)
- Tight integration between orchestration and AI logic

**Cons**:
- ❌ Celery workflows defined in Python code (no visual designer)
- ❌ Workflow changes require code deployment
- ❌ No built-in UI for workflow monitoring or retry
- ❌ FastAPI service becomes stateful (workflow state storage)
- ❌ Scaling becomes coupled (can't scale orchestration separately from AI inference)

**Why Rejected**: Loses operational agility and workflow visibility

---

### Apache Airflow
**Pros**:
- Industry-standard workflow orchestration
- Strong scheduling and DAG management
- Rich ecosystem of integrations
- Excellent for complex data pipelines

**Cons**:
- ❌ Overkill for our use case (designed for batch data pipelines, not event-driven workflows)
- ❌ Higher operational overhead (requires PostgreSQL, scheduler, workers, web server)
- ❌ DAGs defined in Python code (loses low-code advantage)
- ❌ Not optimized for real-time event triggers (designed for scheduled jobs)

**Why Rejected**: Too heavy for the workflow patterns we need. Airflow excels at scheduled data pipelines; we need event-driven document processing.

---

### Prefect
**Pros**:
- Modern alternative to Airflow
- Better real-time support than Airflow
- Pythonic API (familiar for developers)

**Cons**:
- ❌ Still requires code for workflow definition (no visual designer)
- ❌ Less mature integration ecosystem than Airflow
- ❌ Cloud-first design (self-hosted version more complex)

**Why Rejected**: Lacks visual workflow designer; business users can't modify flows

---

### AWS Step Functions / Azure Logic Apps
**Pros**:
- Fully managed (no infrastructure to maintain)
- Native cloud integration
- Visual workflow designer

**Cons**:
- ❌ Vendor lock-in (AWS-only or Azure-only)
- ❌ JSON/YAML workflow definitions (harder to version and test)
- ❌ Limited local development (must deploy to cloud to test)
- ❌ Cost model based on state transitions (can get expensive at scale)

**Why Rejected**: Platform goal is vendor neutrality; cloud-specific orchestration creates lock-in

---

## Consequences

### Positive
✅ **Clear separation**: Orchestration and AI logic evolve independently  
✅ **Visual workflows**: Business users can understand and modify flows  
✅ **Independent scaling**: Scale orchestration and AI services separately  
✅ **Operational agility**: Change workflows without code deployment  
✅ **Reusable services**: FastAPI can be called from anywhere, not just n8n  

### Negative
❌ **Additional infrastructure**: Must deploy and manage n8n alongside FastAPI  
❌ **Network latency**: Orchestration → Service calls add milliseconds vs. in-process  
❌ **Two failure modes**: Both n8n and FastAPI must be healthy  

### Neutral
⚠️ **n8n is replaceable**: The separation pattern matters more than the specific tool. n8n can be swapped for Airflow, Prefect, or Temporal without changing the FastAPI services.

---

## Workflow Integration Pattern

### Event-Driven Ingestion Flow

```
[Google Drive] 
    │ (new file event)
    ↓
[n8n Workflow]
    │
    ├─→ Download document
    │
    ├─→ POST /api/analyze (FastAPI)
    │   └─→ Returns: {summary, entities, risks}
    │
    ├─→ POST /api/ingest (FastAPI)
    │   └─→ Chunks, embeds, stores in Qdrant
    │
    ├─→ Generate Google Doc report
    │
    └─→ Send email notification
```

**Key Point**: n8n orchestrates the *workflow*. FastAPI provides the *capabilities*.

---

## API Contract Example

**FastAPI exposes stateless services:**

```http
POST /api/v1/analyze
Content-Type: application/json

{
  "document_text": "...",
  "analysis_type": "contract_intelligence"
}

→ Response:
{
  "summary": "...",
  "key_obligations": [...],
  "financial_terms": {...},
  "risk_flags": [...]
}
```

**n8n consumes the API** and handles:
- Where the document came from
- What to do with the analysis result
- Error handling and retries
- Output delivery

---

## Decision Drivers

Ranked by importance:

1. **Operational agility** (business users can modify workflows)
2. **Separation of concerns** (orchestration vs. AI logic)
3. **Workflow visibility** (debugging and monitoring)
4. **Independent scalability** (orchestration ≠ AI service load profile)
5. **Reusability** (FastAPI services callable from anywhere)

---

## Review Criteria

This decision should be revisited if:
- Workflow complexity exceeds n8n's capabilities (need DAG versioning, complex branching)
- Cost of operating separate orchestration layer exceeds 20% of total platform cost
- Business users stop modifying workflows (low-code benefit unrealized)
- Real-time latency requirements tighten to <50ms end-to-end (network hop becomes bottleneck)

---

## Related Decisions
- [ADR 004: Service API Design](004-service-api-design.md) - How FastAPI exposes capabilities
- [ADR 005: Multi-Use-Case Reusability](005-multi-use-case-reusability.md) - Reusing services across workflows

---

**Last Reviewed**: 2024-03-15  
**Next Review**: 2024-09-15 (after 6 months of production operation)
