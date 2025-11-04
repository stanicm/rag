# RAG Stack Deployment Comparison

## Quick Decision Guide

| Your Situation | Recommended Stack | File to Use |
|----------------|------------------|-------------|
| **Most users** - Want best performance with reasonable VRAM | **Hybrid** | `docker-compose-hybrid-rag.yaml` |
| **Air-gapped** - No internet / Maximum privacy | **Full Local** | `docker-compose-full-rag-stack.yaml` |
| **Testing** - Just want to try things out | **Hybrid** | `docker-compose-hybrid-rag.yaml` |
| **Production** - High throughput requirements | **Full Local** | `docker-compose-full-rag-stack.yaml` |

---

## Detailed Comparison

### 📦 Hybrid Stack (Recommended)
**File**: `docker-compose-hybrid-rag.yaml`

#### Local Components (GPU Required)
```
✅ Nemotron 49B FP4 (~66 GB VRAM)
   - 65K context window
   - Superior reasoning
   - Memory optimized

✅ Document Extraction NIMs (~12 GB VRAM)
   - Page Elements
   - Graphic Elements  
   - Table Structure
   
✅ Infrastructure (~2 GB VRAM)
   - Milvus Vector DB
   - Redis, etcd, MinIO
   - RAG Server & Frontend
   
Total VRAM: ~80 GB (peak during init)
Steady State: ~70 GB
```

#### Cloud API Components (No GPU)
```
☁️ Embedding - nv-embedqa-e5-v5 (latest, better quality)
☁️ Reranking - nv-rerankqa-mistral-4b-v3 (same quality as local)
☁️ OCR - paddleocr (occasional use)

Requires: Internet connection + NGC API Key
Cost: Minimal (embedding/reranking calls)
```

#### Pros
- ✅ **15GB less VRAM** than full local
- ✅ **Better LLM** (49B vs 9B)
- ✅ **Latest embedding model** (e5-v5 vs 1B)
- ✅ **No maintenance** for embed/rerank
- ✅ **Lower cooling/power** requirements

#### Cons
- ❌ Requires internet connection
- ❌ API call latency (~200-500ms)
- ❌ Text snippets sent to NVIDIA for ranking
- ❌ Small ongoing cost (minimal)

#### Best For
- Development and testing
- Most production deployments
- Memory-constrained GPUs (70-90GB)
- Users who want best performance/cost ratio

---

### 🏠 Full Local Stack
**File**: `docker-compose-full-rag-stack.yaml`

#### All Local Components (GPU Required)
```
✅ Nemotron 9B (~65 GB VRAM)
   - 120K context window
   - Good performance
   - Memory optimized

✅ Embedding NIM (~8 GB VRAM)
   - llama-3.2-nv-embedqa-1b-v2
   - Local inference

✅ Reranking NIM (~6 GB VRAM)
   - nv-rerankqa-mistral-4b-v3
   - Same model as cloud

✅ PaddleOCR NIM (~2 GB VRAM)
   - Optical character recognition

✅ Document Extraction NIMs (~12 GB VRAM)
   - Page Elements
   - Graphic Elements
   - Table Structure

✅ Infrastructure (~2 GB VRAM)
   - Milvus Vector DB
   - Redis, etcd, MinIO
   - RAG Server & Frontend

Total VRAM: ~95 GB (peak during init)
Steady State: ~85 GB
```

#### Pros
- ✅ **Fully offline** (no internet required)
- ✅ **Maximum privacy** (nothing leaves your machine)
- ✅ **No API costs**
- ✅ **Longer context** (120K vs 65K)
- ✅ **Consistent latency** (no network calls)

#### Cons
- ❌ **15GB more VRAM** required
- ❌ **Smaller LLM** (9B vs 49B)
- ❌ **Older embedding model** (1B vs e5-v5)
- ❌ More services to maintain
- ❌ Higher power/cooling requirements

#### Best For
- Air-gapped environments
- Maximum privacy requirements
- High-throughput production
- GPUs with 95GB+ VRAM
- Users who need full offline capability

---

## Memory Breakdown

### Hybrid Stack (~70 GB steady state)

| Component | VRAM | Type | Replaceable? |
|-----------|------|------|--------------|
| Nemotron 49B | 66 GB | Local | ✅ Can use smaller LLM |
| Page Elements | 4 GB | Local | ❌ Unique to local |
| Graphic Elements | 4 GB | Local | ❌ Unique to local |
| Table Structure | 4 GB | Local | ❌ Unique to local |
| Infrastructure | 2 GB | Local | ❌ Required |
| **Embedding** | **0 GB** | **Cloud** | ✅ **Saves 8GB** |
| **Reranking** | **0 GB** | **Cloud** | ✅ **Saves 6GB** |
| **OCR** | **0 GB** | **Cloud** | ✅ **Saves 2GB** |

**Savings: 16GB from cloud APIs, but 49B uses +1GB = Net +15GB saved**

### Full Local Stack (~85 GB steady state)

| Component | VRAM | Type |
|-----------|------|------|
| Nemotron 9B | 65 GB | Local |
| Embedding NIM | 8 GB | Local |
| Reranking NIM | 6 GB | Local |
| PaddleOCR NIM | 2 GB | Local |
| Page Elements | 4 GB | Local |
| Graphic Elements | 4 GB | Local |
| Table Structure | 4 GB | Local |
| Infrastructure | 2 GB | Local |

---

## Performance Comparison

### Query Latency

| Operation | Hybrid | Full Local |
|-----------|--------|------------|
| **LLM Generation** | 2-5s (49B) | 1-3s (9B) |
| **Embedding** | 0.3s (cloud) | 0.1s (local) |
| **Reranking** | 0.5s (cloud) | 0.2s (local) |
| **Extraction** | 10-30s | 10-30s |
| **Total Query** | ~3-6s | ~2-4s |

**Winner**: Full Local (slightly faster, but Hybrid has better quality)

### Quality

| Aspect | Hybrid | Full Local |
|--------|--------|------------|
| **LLM Quality** | ⭐⭐⭐⭐⭐ (49B) | ⭐⭐⭐⭐ (9B) |
| **Embedding Quality** | ⭐⭐⭐⭐⭐ (e5-v5) | ⭐⭐⭐⭐ (1B) |
| **Reranking** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ (same) |
| **Extraction** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ (same) |

**Winner**: Hybrid (better LLM and embeddings)

---

## Cost Analysis

### Hybrid Stack

**Setup Costs:**
- GPU: Already owned
- Storage: ~100GB for models
- NGC API Key: Free

**Ongoing Costs:**
- Power: ~300W GPU usage
- Cooling: Included
- API Calls: ~$0.01-0.10 per 1000 queries (estimate)

**Annual Cost (10K queries/month)**: ~$12-120/year API + power

### Full Local Stack

**Setup Costs:**
- GPU: Already owned (needs 15GB more VRAM)
- Storage: ~150GB for models
- NGC API Key: Free

**Ongoing Costs:**
- Power: ~350W GPU usage (+50W for extra NIMs)
- Cooling: Included
- API Calls: $0

**Annual Cost**: $0 API, but higher power (~$50-100/year extra)

**Winner**: Depends on usage - Hybrid cheaper for light use, Full Local for heavy use

---

## Use Case Recommendations

### Choose Hybrid If:
✅ You want the **best quality** results
✅ You have **70-95GB VRAM** available
✅ You have **reliable internet**
✅ You're **developing or testing**
✅ You prefer **less maintenance**
✅ You process **<1M queries/month**

### Choose Full Local If:
✅ You need **air-gapped deployment**
✅ You have **95GB+ VRAM** available
✅ You need **maximum privacy** (nothing leaves your machine)
✅ You need **consistent latency** (no network variance)
✅ You're in **production with high throughput**
✅ You process **>1M queries/month**

---

## Migration Path

### From Full Local → Hybrid

1. Stop full stack: `docker compose -f docker-compose-full-rag-stack.yaml down`
2. Export NGC_API_KEY
3. Start hybrid: `docker compose -f docker-compose-hybrid-rag.yaml up -d`
4. Test: Verify embeddings work with cloud API
5. Monitor: Check API usage and costs

**Data Migration**: Milvus data persists in volumes (no data loss)

### From Hybrid → Full Local

1. Stop hybrid: `docker compose -f docker-compose-hybrid-rag.yaml down`
2. Free up 15GB VRAM (stop other services if needed)
3. Start full: `docker compose -f docker-compose-full-rag-stack.yaml up -d`
4. Wait: Initial download of embedding/reranking NIMs (~20GB)

**Data Migration**: Milvus data persists in volumes (no data loss)

---

## Summary

| Metric | Hybrid | Full Local | Best |
|--------|--------|------------|------|
| **VRAM Required** | ~70 GB | ~85 GB | Hybrid |
| **Quality (LLM)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Hybrid |
| **Quality (Embed)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Hybrid |
| **Speed** | Good | Excellent | Full Local |
| **Privacy** | High | Maximum | Full Local |
| **Cost (light use)** | Low | Medium | Hybrid |
| **Cost (heavy use)** | Medium | Low | Full Local |
| **Maintenance** | Low | Medium | Hybrid |
| **Internet Required** | Yes | No | Full Local |
| **Best For** | Most Users | Air-gapped | - |

## 🏆 Recommendation

**For 90% of users**: Use the **Hybrid Stack**

- Better quality (49B LLM + e5-v5 embeddings)
- Lower VRAM (fits more GPUs)
- Less maintenance
- Cost-effective for typical usage

**Switch to Full Local** only if you have specific requirements:
- Air-gapped environment
- Maximum privacy needs
- Very high query volume (>1M/month)
- Consistent low latency critical

---

*Last Updated: November 4, 2025*

