# ADR 006: Deployment Topology

**Status**: Accepted  
**Date**: 2024-03-15  
**Deciders**: Platform Architecture Team  
**Technical Story**: Platform must be deployed in a way that balances operational simplicity, scalability, and cost

---

## Context

The platform consists of multiple services that must run somewhere:
- n8n (orchestration engine)
- FastAPI services (ingestion, retrieval, analysis, generation)
- Qdrant (vector database)
- Supporting infrastructure (databases, caches, monitoring)

**Decision Required**: What deployment model should we use?

---

## Decision

**Selected**: **Containerized deployment using Docker Compose** for initial rollout, with a clear migration path to Kubernetes

**Phase 1** (Current — Portfolio/Reference Implementation):
- Docker Compose orchestration
- Single host deployment (sufficient for <10,000 documents, <100 queries/day)
- Local development mirrors production

**Phase 2** (Production — Enterprise Scale):
- Kubernetes deployment (EKS, AKS, or GKE)
- Multi-node cluster with auto-scaling
- Managed services where appropriate (managed Postgres, managed observability)

---

## Rationale

### Why Docker Compose First?

**1. Operational Simplicity**

Docker Compose provides just enough orchestration:
- Single `docker-compose.yml` defines entire stack
- `docker-compose up` starts everything
- `docker-compose down` tears it down
- No cluster management, no control plane, no networking complexity

**For a portfolio implementation or early validation**, this is the right level of complexity.

**2. Cost Efficiency at Small Scale**

A single host running Docker Compose:
- **AWS**: t3.xlarge (4 vCPU, 16 GB RAM) = ~$150/month
- **Kubernetes cluster**: 3-node minimum = ~$450/month (control plane + worker nodes)

At <10K documents and <100 queries/day, the additional $300/month buys nothing useful.

**3. Local Development Parity**

Developers run the exact same `docker-compose.yml` locally.

**Not**: "It works on my machine" vs. "Kubernetes config is different."

**4. Faster Iteration**

No cluster provisioning, no YAML debugging, no ingress controllers. Deploy changes in seconds.

**5. Clear Migration Path**

Docker Compose → Kubernetes is a well-trodden path. When scale demands it, we:
1. Convert `docker-compose.yml` to Kubernetes manifests (tools exist: Kompose)
2. Add horizontal pod autoscaling
3. Add Ingress and LoadBalancer
4. Migrate to managed services (RDS, managed Qdrant)

The *service boundaries* don't change. Only the deployment substrate changes.

---

## Docker Compose Stack

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  fastapi:
    build: ./fastapi
    ports:
      - "8000:8000"
    environment:
      - QDRANT_URL=http://qdrant:6333
      - OPENROUTER_API_KEY=${OPENROUTER_API_KEY}
    depends_on:
      - qdrant

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_storage:/qdrant/storage

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  qdrant_storage:
  postgres_data:
```

**Key Points**:
- All services on a single network (service discovery via hostnames)
- Persistent volumes for state
- Environment variables for configuration
- Health checks and dependencies managed by Compose

---

## Phase 2: Kubernetes Migration

When scale requires it (>50K documents, >1000 queries/day), migrate to Kubernetes:

### Kubernetes Deployment Pattern

```
┌──────────────────────────────────────┐
│        Ingress Controller            │
│    (NGINX / ALB / API Gateway)       │
└────────┬──────────────────┬──────────┘
         │                  │
    ┌────▼─────┐      ┌────▼─────┐
    │   n8n    │      │ FastAPI  │
    │ (Stateful│      │ (Stateless)
    │   Set)   │      │  Deployment │
    └────┬─────┘      └────┬─────┘
         │                  │
    ┌────▼──────────────────▼─────┐
    │       Qdrant Cluster         │
    │   (StatefulSet, 3 replicas)  │
    └──────────────────────────────┘
```

**Benefits of Kubernetes**:
- Horizontal pod autoscaling (FastAPI scales to 10+ replicas on query spike)
- StatefulSets for Qdrant (persistent storage per pod)
- Rolling updates with zero downtime
- Resource limits and requests (prevent resource contention)
- Namespace isolation (dev / staging / prod)

**Migration Effort**: 2-3 weeks for initial Kubernetes deployment, then incremental optimization.

---

## Alternatives Considered

### Serverless (AWS Lambda / Cloud Functions)
**Pros**:
- Zero infrastructure management
- Automatic scaling
- Pay-per-invocation pricing

**Cons**:
- ❌ Cold start latency (300-1000ms) unacceptable for interactive queries
- ❌ Execution time limits (15 min Lambda max) problematic for batch ingestion
- ❌ Qdrant can't run serverless (stateful vector store)
- ❌ n8n not designed for serverless execution model

**Why Rejected**: Stateful components (Qdrant, n8n) don't fit serverless model

---

### Virtual Machines (EC2 / VMs)
**Pros**:
- Full control over OS and networking
- Familiar operational model

**Cons**:
- ❌ Slower deployment (provision VM, install dependencies, configure)
- ❌ Manual orchestration (systemd scripts, process management)
- ❌ Less portable (VM images are provider-specific)
- ❌ Harder to scale (manual VM provisioning)

**Why Rejected**: Containers provide better portability and faster deployment

---

### Platform-as-a-Service (Heroku, Render, Railway)
**Pros**:
- Extremely simple deployment (git push)
- Managed databases and add-ons

**Cons**:
- ❌ Limited control over infrastructure
- ❌ Higher cost per resource vs. raw compute
- ❌ Vendor lock-in (platform-specific config)
- ❌ Not all services fit PaaS model (Qdrant requires custom setup)

**Why Rejected**: Loss of infrastructure control without corresponding simplicity benefit

---

### Docker Swarm
**Pros**:
- Built into Docker (no additional tooling)
- Simpler than Kubernetes

**Cons**:
- ❌ Smaller ecosystem than Kubernetes (fewer tools, less community support)
- ❌ Less mature autoscaling
- ❌ Unclear long-term viability (Kubernetes won the orchestration war)

**Why Rejected**: If we're going to learn orchestration, learn the industry standard (Kubernetes)

---

## Consequences

### Positive
✅ **Operational simplicity**: Docker Compose is easy to understand and debug  
✅ **Fast iteration**: Deploy changes in seconds  
✅ **Cost efficiency**: Single host sufficient for initial scale  
✅ **Local dev parity**: Same stack runs on laptop and production  
✅ **Clear migration path**: Kubernetes when needed, without architectural changes  

### Negative
❌ **Single point of failure**: One host down = entire platform down  
❌ **Manual scaling**: Can't auto-scale beyond single host resources  
❌ **Limited high availability**: No built-in redundancy  

### Mitigations
- **Backups**: Daily snapshots of Qdrant and n8n volumes
- **Monitoring**: Uptime checks, alert on service failure
- **Documented runbooks**: Recovery procedures for common failures

---

## Scaling Triggers

**When to migrate to Kubernetes**:
- Document count exceeds 50,000 (single-host resource limits)
- Query rate exceeds 1,000/day (need horizontal scaling)
- Uptime SLA requirement exceeds 99% (need redundancy)
- Multi-region deployment needed (regulatory or performance)

**When to adopt managed services**:
- Operational overhead exceeds 20% of platform engineering time
- Qdrant Cloud offers compelling features (e.g., managed backups, auto-scaling)
- Team lacks Kubernetes expertise (managed Kubernetes preferred)

---

## Deployment Environments

### Development
- Docker Compose on developer laptops
- Local file mounts for code changes (hot reload)
- Stubbed external integrations (no real Google Drive access)

### Staging
- Docker Compose on cloud VM (AWS EC2, Azure VM)
- Production-equivalent config
- Real integrations, test data

### Production
- **Phase 1**: Docker Compose on cloud VM with automated backups
- **Phase 2**: Kubernetes cluster with multi-node redundancy

---

## Decision Drivers

Ranked by importance:

1. **Operational simplicity** (easy to understand and maintain)
2. **Cost efficiency at small scale** (don't overpay for unused capacity)
3. **Fast iteration** (developer productivity)
4. **Clear scaling path** (can grow when needed)

---

## Review Criteria

This decision should be revisited when:
- Document count exceeds 50,000
- Query rate exceeds 1,000/day consistently
- Uptime requirements tighten (need HA)
- Operational overhead of Docker Compose exceeds benefits

**Expected timeline**: Re-evaluate in 6 months or when platform serves 3+ use cases.

---

## Related Decisions
- [ADR 001: Vector Store Selection](001-vector-store-selection.md) - Qdrant deployment considerations
- [ADR 003: Orchestration Layer Separation](003-orchestration-layer-separation.md) - n8n deployment model

---

**Last Reviewed**: 2024-03-15  
**Next Review**: 2024-09-15 (coinciding with scale triggers review)
