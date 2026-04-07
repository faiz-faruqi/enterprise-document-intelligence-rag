# Cost Modeling & TCO Analysis

Total Cost of Ownership (TCO) analysis for the Enterprise RAG Platform across different deployment scales.

---

## Assumptions

**Volume**: 50,000 documents (Year 1), 250,000 vectors  
**Query Rate**: 500 queries/day average  
**Geographic Region**: US East (AWS us-east-1, Azure East US, GCP us-east1)

---

## Component Cost Breakdown

### 1. Compute Infrastructure

#### Docker Compose Deployment (Current)

| Component | Instance Type | Monthly Cost |
|-----------|--------------|--------------|
| n8n + FastAPI + Qdrant | t3.xlarge (4 vCPU, 16 GB) | ~$120 |
| **Total** | | **$120/month** |

#### Kubernetes Deployment (Future Scale)

| Component | Instance Type | Quantity | Monthly Cost |
|-----------|--------------|----------|--------------|
| Control Plane | Managed K8s (EKS/AKS/GKE) | 1 | $73 |
| Worker Nodes | t3.large (2 vCPU, 8 GB) | 3 | $225 |
| **Total** | | | **$298/month** |

---

### 2. Storage Costs

#### Vector Storage (Qdrant)

| Metric | Size | Cost/Month |
|--------|------|------------|
| 250K vectors @ 384-dim | ~360 MB | $0.08 (EBS gp3) |
| 1M vectors @ 384-dim | ~1.4 GB | $0.32 |
| 10M vectors @ 768-dim | ~57 GB | $12.80 |

**Note**: Larger embeddings (768-dim vs 384-dim) cost 2x in storage.

#### Document Storage (S3/Blob)

| Metric | Size | Cost/Month |
|--------|------|------------|
| 50K documents (avg 100KB) | 5 GB | $0.12 |
| 100K documents | 10 GB | $0.23 |

#### Backup & Snapshots

| Backup Frequency | Retention | Cost/Month |
|-----------------|-----------|------------|
| Daily snapshots | 7 days | ~$5 |
| Weekly snapshots | 4 weeks | ~$2 |

---

### 3. LLM Inference Costs

#### Query Generation (Answer Generation)

**Assumptions**: 
- 500 queries/day
- Average 5 retrieved chunks per query = 2500 tokens context
- Average response = 200 tokens
- Model: GPT-4 (via OpenRouter)

| Metric | Volume | Rate | Cost |
|--------|--------|------|------|
| Input tokens/month | 37.5M (500 × 2500 × 30) | $30/1M tokens | $1,125 |
| Output tokens/month | 3M (500 × 200 × 30) | $60/1M tokens | $180 |
| **Total LLM Cost** | | | **$1,305/month** |

**Cost Optimization**:
- Use Claude 3 Haiku: ~70% cheaper → **$390/month**
- Use GPT-3.5-turbo: ~90% cheaper → **$130/month**
- Cache frequent queries: 20-30% reduction

#### Document Analysis (Ingestion)

**Assumptions**:
- 50K documents/year = 4,200/month
- 2000 tokens/document
- Model: GPT-4 for extraction

| Metric | Volume | Rate | Cost |
|--------|--------|------|------|
| Input tokens/month | 8.4M (4200 × 2000) | $30/1M | $252 |
| Output tokens/month | 420K (4200 × 100) | $60/1M | $25 |
| **Total Analysis Cost** | | | **$277/month** |

**Total LLM Cost** (Query + Analysis): **$1,582/month**  
**Optimized** (using Claude Haiku for queries): **$667/month**

---

### 4. Embedding Costs

#### Option A: Local Embeddings (Chosen)

| Cost Component | Amount |
|---------------|--------|
| Model download | One-time, 90 MB |
| Compute overhead | $0 marginal (already paying for CPU) |
| GPU acceleration (if needed) | $200-400/month (p3.2xlarge spot) |
| **Total** | **$0-400/month** |

#### Option B: API Embeddings (OpenAI)

| Metric | Volume | Rate | Cost |
|--------|--------|------|------|
| Initial ingestion | 100M tokens | $0.10/1M | $10 |
| Ongoing queries | 7.5M tokens/month (500 queries/day) | $0.10/1M | $0.75/month |
| **Total Year 1** | | | **~$19/year** |

**Verdict**: API embeddings are actually cheaper at this scale, BUT data locality requirements justify local approach.

---

### 5. Data Transfer

| Type | Volume/Month | Cost |
|------|--------------|------|
| Ingress (upload docs) | 5 GB | Free |
| Egress (download reports) | 2 GB | $0.18 |
| Inter-service (within VPC) | 50 GB | Free |
| **Total** | | **$0.18/month** |

---

### 6. Monitoring & Logging

| Service | Cost/Month |
|---------|-----------|
| CloudWatch / Azure Monitor | $15 |
| Log retention (30 days) | $5 |
| **Total** | **$20/month** |

---

## Total Cost of Ownership (TCO)

### Year 1: Docker Compose Deployment

| Component | Cost/Month | Cost/Year |
|-----------|-----------|-----------|
| Compute | $120 | $1,440 |
| Storage | $7 | $84 |
| LLM Inference (Optimized) | $667 | $8,004 |
| Embeddings (Local) | $0 | $0 |
| Monitoring | $20 | $240 |
| **Total TCO** | **$814/month** | **$9,768/year** |

### Year 1: If Using K8s + Full GPT-4

| Component | Cost/Month | Cost/Year |
|-----------|-----------|-----------|
| Compute (K8s) | $298 | $3,576 |
| Storage | $7 | $84 |
| LLM Inference (GPT-4) | $1,582 | $18,984 |
| Embeddings (Local) | $0 | $0 |
| Monitoring | $20 | $240 |
| **Total TCO** | **$1,907/month** | **$22,884/year** |

---

## Cost Drivers & Optimization Strategies

### Primary Cost Driver: LLM Inference (82% of total cost)

**Optimization Strategies**:

1. **Model Selection**
   - GPT-4 → Claude Haiku: Save $915/month (58% reduction)
   - Implement model routing (complex queries → GPT-4, simple → GPT-3.5)
   
2. **Caching**
   - Cache embeddings for frequently asked questions
   - Cache LLM responses for common queries
   - Expected savings: 20-30% reduction

3. **Prompt Optimization**
   - Reduce context window size (retrieve 3 chunks vs. 5)
   - Use prompt compression techniques
   - Expected savings: 15-25% reduction

4. **Query Batching**
   - Batch multiple questions in single LLM call
   - Reduce per-request overhead
   - Expected savings: 10-15% reduction

**Combined Optimization Potential**: 50-60% cost reduction on LLM → **~$300-400/month savings**

---

## Break-Even Analysis: Local vs. API Embeddings

**Break-even point** where API embeddings become cheaper than GPU:

| Metric | Local (GPU) | API (OpenAI) |
|--------|-------------|--------------|
| Fixed cost/month | $400 (GPU instance) | $0 |
| Variable cost/query | $0 | $0.0015 (500 tokens @ $0.10/1M) |

Break-even = $400 ÷ $0.0015 = **267,000 queries/month**

**At current scale** (15K queries/month): Local wins  
**At 10x scale** (150K queries/month): Local still wins  
**At 18x scale** (270K queries/month): API becomes competitive

**Conclusion**: Local embeddings justified for foreseeable scale.

---

## Cost by Use Case

### Contract Intelligence (Initial Use Case)

| Metric | Value |
|--------|-------|
| Documents/month | 4,200 |
| Queries/month | 15,000 |
| Monthly cost | $814 |
| **Cost per document** | **$0.19** |
| **Cost per query** | **$0.05** |

### Adding Policy Compliance (Use Case #2)

**Incremental Cost**:
- Same infrastructure (no added compute cost)
- +2000 documents/month → +$95/month (LLM analysis)
- +10,000 queries/month → +$500/month (LLM generation)
- **Total incremental**: ~$595/month

**Multi-tenancy benefit**: Platform core amortizes across use cases.

---

## Scaling Projections

### 10x Scale (500K documents, 5M queries/year)

| Component | Cost/Month |
|-----------|-----------|
| Compute (K8s, 6 nodes) | $600 |
| Storage (10M vectors) | $25 |
| LLM Inference | $6,670 (optimized with caching) |
| Monitoring | $50 |
| **Total** | **$7,345/month** |

**Per-query cost**: $0.049 (economies of scale)

---

## Vendor Comparison: Managed vs. Self-Hosted

### Self-Hosted (Current Approach)

**Year 1 TCO**: $9,768  
**Per-document**: $0.19  
**Per-query**: $0.05  

### Fully Managed (Pinecone + OpenAI + Zapier)

| Component | Cost/Month |
|-----------|-----------|
| Pinecone (250K vectors, serverless) | $70 |
| OpenAI (embeddings + generation) | $1,600 |
| Zapier (workflow automation) | $50 |
| **Total** | **$1,720/month** = **$20,640/year** |

**TCO Premium**: +$10,872/year (+111% more expensive)

**Trade-off**: Pay 2x more, get zero operational overhead.

---

## Recommendations

### For Budget-Conscious Deployments
1. Use Docker Compose (not K8s) until scale demands it
2. Use Claude Haiku or GPT-3.5-turbo for generation
3. Implement aggressive caching (20-30% savings)
4. Local embeddings (save ~$400/month vs. GPU)

**Target TCO**: ~$400-500/month

### For Performance-Optimized Deployments
1. Kubernetes for auto-scaling
2. GPT-4 for highest quality responses
3. GPU-accelerated embeddings for <10ms latency
4. Multi-region deployment for <100ms global latency

**Target TCO**: ~$2,000-2,500/month

### For Hybrid (Recommended)
1. Docker Compose initially → K8s at 50K+ documents
2. Claude Haiku for routine queries, GPT-4 for complex
3. CPU embeddings → GPU only if latency becomes issue
4. Aggressive caching + prompt optimization

**Target TCO**: ~$600-800/month

---

*Cost estimates updated: March 2024. Cloud pricing changes frequently — revalidate quarterly.*
