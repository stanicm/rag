# 🎉 RAG Pipeline Deployment - Success Summary

**Date:** November 4, 2025  
**Stack Type:** Hybrid Local + Cloud  
**Status:** ✅ Fully Operational

---

## 📊 What Was Deployed

### Core Components

1. **Nemotron 49B FP4** (LLM)
   - Port: `8999` (external) → `8000` (internal)
   - VRAM Usage: ~66GB (memory-optimized)
   - Model: `nvidia/llama-3.3-nemotron-super-49b-v1.5`
   - Configuration: `NIM_MAX_MODEL_LEN=65000`, `NIM_KVCACHE_PERCENT=0.5`, `NIM_LOW_MEMORY_MODE=1`

2. **Embedding NIM 1B** (Local)
   - Port: `9080`
   - VRAM Usage: ~3GB
   - Model: `nvidia/llama-3.2-nv-embedqa-1b-v2`
   - Purpose: Document and query embeddings

3. **Reranking NIM 1B** (Local)
   - Port: `9090`
   - VRAM Usage: ~3GB
   - Model: `nvidia/llama-3.2-nv-rerankqa-1b-v2`
   - Purpose: Rerank retrieved documents

4. **Document Extraction NIMs** (Local)
   - Page Elements NIM: `nvcr.io/nim/nvidia/nemoretriever-page-elements-v2`
   - Table Structure NIM: `nvcr.io/nim/nvidia/nemoretriever-table-structure-v1`
   - Graphic Elements NIM: `nvcr.io/nim/nvidia/nemoretriever-graphic-elements-v1`
   - Purpose: Extract text, tables, and charts from PDFs

5. **Vector Database**
   - Milvus Standalone: Port `19530`
   - MinIO (Object Storage): Port `9000`
   - etcd (Coordination): Port `2379`

6. **RAG Services**
   - RAG Server: Port `8081`
   - Ingestor Server: Port `8082`
   - RAG Frontend UI: Port `8090`
   - NV-Ingest Runtime: Port `7670`

### Cloud Services (NVIDIA API)
- PaddleOCR: `https://ai.api.nvidia.com/v1/cv/baidu/paddleocr`

---

## 🔧 Key Configuration Changes

### Issue #1: Embedding 404 Errors
**Problem:** `nv-ingest-ms-runtime` was trying to call NVIDIA API for embeddings but getting 404 errors.

**Solution:** Deployed local Embedding NIM (`llama-3.2-nv-embedqa-1b-v2`) and configured both `ingestor-server` and `rag-server` to use it:
```yaml
APP_EMBEDDINGS_SERVERURL: nemoretriever-embedding-ms:8000
APP_EMBEDDINGS_MODELNAME: nvidia/llama-3.2-nv-embedqa-1b-v2
APP_EMBEDDINGS_DIMENSIONS: 2048
```

### Issue #2: Reranking 404 Errors
**Problem:** `rag-server` was trying to call NVIDIA API for reranking but getting 404 errors.

**Solution:** Deployed local Reranking NIM (`llama-3.2-nv-rerankqa-1b-v2`) and configured `rag-server`:
```yaml
APP_RANKING_SERVERURL: nemoretriever-ranking-ms:8000
APP_RANKING_MODELNAME: nvidia/llama-3.2-nv-rerankqa-1b-v2
```

### Issue #3: Environment Variable Reloading
**Problem:** Docker `restart` command doesn't reload environment variables from `.env` file.

**Solution:** Used `docker compose stop` followed by `docker compose up -d` to ensure fresh environment loading.

### Issue #4: Milvus Flush Required
**Problem:** Documents were inserted but showed 0 entities when queried.

**Solution:** Milvus requires explicit `flush()` to persist data:
```python
collection.flush()
```

---

## 📈 Performance Metrics

### Document Ingestion
- **Documents Processed:** 3 SPH research papers
- **Total Entities:** 459 chunks stored in Milvus
- **Processing Time:** ~30-35 seconds per document
- **Extraction Success:** Text, tables, and charts extracted successfully

### RAG Query Performance
- **LLM Time to First Token (TTFT):** ~1344ms
- **RAG Time to First Token (TTFT):** ~2040ms
- **Knowledge Base Retrieval:** ✅ Working
- **Response Quality:** Context-aware responses based on ingested documents

### VRAM Usage Summary
| Component | VRAM Usage |
|-----------|------------|
| Nemotron 49B FP4 | ~66GB |
| Embedding NIM 1B | ~3GB |
| Reranking NIM 1B | ~3GB |
| **Total** | **~72GB** |

---

## 🌐 Access Points

### Web Interfaces
- **RAG Frontend UI:** http://localhost:8090
  - Interactive chat interface with knowledge base
  - Collection management
  - Document upload

- **Mercury Interface:** http://localhost:3001
  - Multi-model chat interface
  - Audio transcription & TTS
  - Nemotron 49B integration

### API Endpoints

#### RAG Server (`http://localhost:8081`)
```bash
# Query with knowledge base
curl -X POST http://localhost:8081/generate \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is SPH?"}],
    "use_knowledge_base": true,
    "collection_name": "SPH"
  }'
```

#### Ingestor Server (`http://localhost:8082`)
```bash
# Ingest documents
curl -X POST http://localhost:8082/documents \
  -F "documents=@document.pdf" \
  -F 'data={"collection_name":"SPH"}'
```

#### Mercury Agent (via Interface)
- Chat endpoint handles routing to Nemotron 49B
- Configured in `mercury_agent/configs/config.yml`

---

## 📁 Important Files

### Docker Compose
- **Main Compose File:** `/home/milos/eragbpv3/deploy/compose/docker-compose-hybrid-rag.yaml`
- **Environment File:** `/home/milos/eragbpv3/deploy/compose/.env`

### Mercury Configuration
- **Agent Config:** `/home/milos/mercury-assistant/mercury_agent/configs/config.yml`
- **Interface Server:** `/home/milos/mercury-assistant/mercury_interface/server.js`

### Documentation
- **Nemotron Deployment:** `/home/milos/mercury-assistant/documentation/NEMOTRON_DEPLOYMENT.md`
- **Hybrid vs Full Comparison:** `/home/milos/eragbpv3/deploy/compose/HYBRID_VS_FULL_COMPARISON.md`
- **Configuration Gaps:** `/home/milos/mercury-assistant/CONFIGURATION_GAPS.md`

---

## 🧪 Verification Commands

### Check All Services
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | \
  grep -E "NAME|nemotron|embed|ranking|rag|ingestor|milvus|redis"
```

### Check Milvus Collections
```python
from pymilvus import connections, utility, Collection

connections.connect("default", host="localhost", port="19530")
print("Collections:", utility.list_collections())

collection = Collection("SPH")
collection.flush()
print(f"SPH Entities: {collection.num_entities}")
```

### Test RAG Query
```bash
curl -X POST http://localhost:8081/generate \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is SPH used for?"}],
    "use_knowledge_base": true,
    "collection_name": "SPH"
  }' | python3 -m json.tool
```

---

## 🔄 Restart Commands

### Restart Individual Services
```bash
cd /home/milos/eragbpv3/deploy/compose
export USERID=$(id -u)
export NGC_API_KEY="nvapi-..."

# Restart RAG server
docker compose -f docker-compose-hybrid-rag.yaml restart rag-server

# Restart ingestor
docker compose -f docker-compose-hybrid-rag.yaml restart ingestor-server
```

### Full Stack Restart
```bash
cd /home/milos/eragbpv3/deploy/compose
export USERID=$(id -u)
export NGC_API_KEY="nvapi-..."

docker compose -f docker-compose-hybrid-rag.yaml down
docker compose -f docker-compose-hybrid-rag.yaml up -d
```

---

## 🎯 What's Working

✅ Document ingestion with PDF extraction (text, tables, charts)  
✅ Local embedding generation (no API calls)  
✅ Local reranking (no API calls)  
✅ Vector database storage and retrieval  
✅ RAG query processing with Nemotron 49B  
✅ Web UI for chat and document management  
✅ Mercury Interface integration  
✅ End-to-end RAG pipeline functionality  

---

## 🚀 Next Steps

1. **Test More Documents:** Ingest additional PDFs into different collections
2. **Try RAG Frontend UI:** Access http://localhost:8090 for web-based interaction
3. **Mercury Chat:** Test queries through Mercury Interface at http://localhost:3001
4. **Monitor Performance:** Check VRAM usage with `nvidia-smi` during heavy queries
5. **Explore Advanced Features:** Hybrid search, citation extraction, metadata filtering

---

## 📝 Lessons Learned

1. **Environment Variables:** Docker `restart` doesn't reload `.env` - use `stop` + `up -d`
2. **Milvus Persistence:** Always call `flush()` after inserting data
3. **Embedding Configuration:** Both ingestor and rag-server need proper embedding endpoints
4. **Local vs Cloud:** For the best experience, embedding and reranking should be local
5. **Model Size:** 49B model works great with memory optimization parameters

---

## 🙏 Credits

- **NVIDIA RAG Blueprint:** https://github.com/stanicm/rag
- **Mercury Assistant:** Custom agent system with Haystack orchestration
- **Nemotron Models:** NVIDIA's advanced LLM family
- **NeMo Retriever:** NVIDIA's retrieval and extraction microservices

---

**Deployment Completed Successfully!** 🎉

