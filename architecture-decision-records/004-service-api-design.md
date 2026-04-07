# ADR 004: Service API Design Pattern

**Status**: Accepted  
**Date**: 2024-03-15  
**Deciders**: Platform Architecture Team  
**Technical Story**: AI capabilities must be exposed as consumable services with clear contracts

---

## Context

The platform core provides AI capabilities that multiple consumers need to access:
- n8n workflows (orchestration layer)
- Web applications (direct user interaction)
- Other microservices (internal integrations)
- External systems (via API gateway)

**Design Question**: How should AI capabilities be exposed?

---

## Decision

**Selected**: REST APIs using FastAPI with domain-driven service boundaries

**Service Structure**:
- **Ingestion Service** (`/api/v1/ingest`): Document chunking, embedding, vector storage
- **Retrieval Service** (`/api/v1/retrieve`): Semantic search, context retrieval
- **Analysis Service** (`/api/v1/analyze`): Document classification, entity extraction, summarization
- **Generation Service** (`/api/v1/generate`): Grounded answer generation with source attribution

---

## Rationale

### Why REST APIs with Domain Boundaries

**1. Consumer-Agnostic Design**

REST APIs work for all consumer types:
- n8n workflows make HTTP requests
- Web frontends call via JavaScript
- Python microservices use `requests` library
- External systems integrate via standard HTTP clients

No consumer is special-cased. Everyone speaks HTTP.

**2. Technology Independence**

Consumers don't care that services are implemented in Python + FastAPI. They could be rewritten in Go, Rust, or Java without breaking integrations. The API contract is the interface.

**3. Independent Deployment**

Each service can be deployed, scaled, and versioned independently:
- Ingestion service updated to improve chunking → deploy without touching retrieval
- Retrieval service scaled 3x during query spike → ingestion unaffected
- Analysis service gets new entity extraction model → version bump, backward compatible

**4. Clear Boundaries Based on Domain**

Services map to distinct platform capabilities, not technical layers:

| Service | Responsibility | When Called |
|---------|---------------|-------------|
| Ingestion | Add knowledge to the platform | Document uploaded, batch import |
| Retrieval | Find relevant knowledge | User query, context lookup |
| Analysis | Extract insights from documents | Automated processing, reporting |
| Generation | Create grounded answers | Chat interface, Q&A API |

**Not**: "Database service", "LLM service", "Vector service" (technical partitioning)  
**Instead**: Services organized by business capability

---

## API Design Principles

### 1. Synchronous Request-Response (Default)

```http
POST /api/v1/retrieve
Content-Type: application/json

{
  "query": "What is the contract value?",
  "top_k": 5,
  "filters": {"document_type": "contract"}
}

→ Response (200 OK):
{
  "results": [
    {
      "chunk_id": "abc123",
      "text": "The total contract value is $180,000...",
      "score": 0.89,
      "metadata": {...}
    }
  ],
  "latency_ms": 45
}
```

**Why**: Most operations complete in <1 second. Synchronous is simpler than async.

### 2. Async for Long-Running Operations

```http
POST /api/v1/ingest/batch
{
  "document_ids": ["doc1", "doc2", ...],
  "source": "google_drive"
}

→ Response (202 Accepted):
{
  "job_id": "job_xyz",
  "status_url": "/api/v1/jobs/job_xyz"
}

GET /api/v1/jobs/job_xyz

→ Response:
{
  "job_id": "job_xyz",
  "status": "processing",
  "progress": "45/100 documents",
  "started_at": "2024-03-15T10:00:00Z"
}
```

**Why**: Batch ingestion of 1000+ documents takes minutes. Async prevents timeout.

### 3. Versioned Endpoints

All endpoints include `/v1/` in the path. Breaking changes get `/v2/`.

**Why**: Allows gradual migration when API contracts change.

### 4. OpenAPI Specification

All services publish OpenAPI 3.0 specs at `/openapi.json`.

**Why**: Auto-generated client libraries, API documentation, contract testing.

---

## Alternatives Considered

### GraphQL API
**Pros**:
- Clients request exactly the data they need (no over-fetching)
- Strong typing via schema
- Single endpoint for all operations

**Cons**:
- ❌ Adds complexity (GraphQL server, schema management, query optimization)
- ❌ Overkill for simple request-response patterns
- ❌ n8n's GraphQL support is less mature than REST
- ❌ Caching more complex than REST

**Why Rejected**: GraphQL solves problems we don't have (complex nested queries, mobile bandwidth constraints)

---

### gRPC
**Pros**:
- High performance (binary protocol, HTTP/2)
- Strong typing via Protobuf
- Streaming support

**Cons**:
- ❌ Harder to debug (binary protocol)
- ❌ Not browser-friendly (requires gRPC-Web proxy)
- ❌ n8n doesn't support gRPC natively
- ❌ Adds operational complexity (Protobuf compilation, schema versioning)

**Why Rejected**: Performance gain not needed (services are I/O-bound, not CPU-bound). REST is simpler.

---

### Message Queue (Kafka, RabbitMQ)
**Pros**:
- Decouples producers from consumers
- Natural fit for event-driven architectures
- Handles high-volume async workloads

**Cons**:
- ❌ Adds infrastructure complexity (broker, topics, consumer groups)
- ❌ Synchronous request-response becomes awkward (publish + wait for response on reply queue)
- ❌ Debugging is harder (messages flow through broker, not direct service-to-service)

**Why Rejected**: REST handles our throughput requirements (<1000 req/sec). Message queue overhead not justified.

---

### Direct Function Calls (Monolith)
**Pros**:
- Lowest latency (no network)
- No API versioning complexity
- Simpler deployment (single service)

**Cons**:
- ❌ Services can't scale independently
- ❌ Failure in one service crashes entire platform
- ❌ Tight coupling (changing ingestion logic risks breaking retrieval)
- ❌ Can't rewrite services in different languages

**Why Rejected**: Violates separation of concerns principle from ADR 003

---

## Consequences

### Positive
✅ **Technology independence**: Services can be rewritten without breaking consumers  
✅ **Independent scaling**: Each service scales based on its own load profile  
✅ **Clear contracts**: OpenAPI specs document exact request/response formats  
✅ **Consumer-agnostic**: n8n, web apps, microservices all use same APIs  
✅ **Testable**: Services can be tested independently with mock consumers  

### Negative
❌ **Network latency**: Service-to-service calls add 5-10ms vs. in-process  
❌ **Distributed failure modes**: Network failures, timeouts, rate limits  
❌ **API versioning overhead**: Breaking changes require coordination  

### Neutral
⚠️ **REST is not the final choice**: If performance becomes a bottleneck (>10,000 req/sec), we can migrate to gRPC. The service boundaries remain unchanged.

---

## API Example: Retrieval Service

**OpenAPI Specification** (excerpt):

```yaml
openapi: 3.0.0
info:
  title: Retrieval Service API
  version: 1.0.0

paths:
  /api/v1/retrieve:
    post:
      summary: Retrieve relevant document chunks
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [query]
              properties:
                query:
                  type: string
                  description: User's search query
                top_k:
                  type: integer
                  default: 5
                  description: Number of results to return
                filters:
                  type: object
                  description: Metadata filters (document_type, date_range, etc.)
      responses:
        '200':
          description: Successfully retrieved results
          content:
            application/json:
              schema:
                type: object
                properties:
                  results:
                    type: array
                    items:
                      $ref: '#/components/schemas/ChunkResult'
                  latency_ms:
                    type: integer
```

**Why this matters**: Consumers know *exactly* what to send and what to expect. No ambiguity.

---

## Decision Drivers

Ranked by importance:

1. **Consumer-agnostic design** (n8n, web, APIs all use same interface)
2. **Independent deployability** (services evolve separately)
3. **Clear contracts** (OpenAPI specs prevent integration surprises)
4. **Technology independence** (can rewrite services without breaking integrations)
5. **Simplicity** (REST is well-understood, tooling is mature)

---

## Review Criteria

This decision should be revisited if:
- Request throughput exceeds 10,000 req/sec (consider gRPC for performance)
- Network latency becomes >20% of end-to-end response time (consider consolidation)
- API versioning overhead exceeds benefits of independent deployment
- WebSocket or streaming use cases emerge (REST doesn't handle these well)

---

## Related Decisions
- [ADR 003: Orchestration Layer Separation](003-orchestration-layer-separation.md) - Why n8n calls these APIs
- [ADR 005: Multi-Use-Case Reusability](005-multi-use-case-reusability.md) - How same APIs support different use cases

---

**Last Reviewed**: 2024-03-15  
**Next Review**: 2024-09-15
