# Hybrid RAG Stack - Local + Cloud NIMs

This Docker Compose configuration deploys a **memory-optimized hybrid RAG pipeline** that uses:
- **Local NIMs** for unique models and the powerful 49B LLM
- **NVIDIA Cloud APIs** for embedding and reranking (saving ~15GB VRAM)

## 🎯 Architecture Overview

### Local NIMs (Running on GPU)
- **Nemotron 49B FP4** - Main LLM with 65K context (~66GB VRAM)
- **Page Elements** - Document structure extraction
- **Graphic Elements** - Image/chart extraction  
- **Table Structure** - Table detection and extraction
- **Vector DB Stack** - Milvus, etcd, MinIO

### Cloud APIs (NVIDIA Hosted)
- **Embedding** - `nv-embedqa-e5-v5` (latest, better than local)
- **Reranking** - `nv-rerankqa-mistral-4b-v3`
- **OCR** - `paddleocr` (via API)

## 💾 Memory Savings

| Component | Previous | Hybrid | Saved |
|-----------|----------|--------|-------|
| LLM (9B → 49B) | 65 GB | 66 GB | -1 GB |
| Embedding NIM | 8 GB | 0 GB | +8 GB |
| Reranking NIM | 6 GB | 0 GB | +6 GB |
| PaddleOCR NIM | 2 GB | 0 GB | +2 GB |
| **Net Change** | **81 GB** | **66 GB** | **+15 GB** |

**Result**: Better model (49B vs 9B) with 15GB less VRAM! 🎉

## 🚀 Quick Start

### Prerequisites

1. **NVIDIA GPU** with 70GB+ VRAM
2. **Docker** with NVIDIA Container Toolkit
3. **NGC API Key** from [https://ngc.nvidia.com](https://ngc.nvidia.com)
4. **~100GB disk space** for model downloads

### Setup

1. **Set environment variables**:
```bash
cd /home/milos/eragbpv3/deploy/compose

# Export NGC API key
export NGC_API_KEY="your-api-key-here"
export NVIDIA_API_KEY="$NGC_API_KEY"

# Set model cache directory
export MODEL_DIRECTORY=~/.cache/model-cache
mkdir -p "$MODEL_DIRECTORY"

# Set user ID
export USERID=$(id -u)

# Set Nemotron 49B profile
export NIM_MODEL_PROFILE='496a3bcf32f7c7e81e59b1c17395d49b6c412dcb9e94d1bd4675c7ab61ed4b8c'
export NIM_MANIFEST_ALLOW_UNSAFE=1
```

2. **Login to NGC registry**:
```bash
echo $NGC_API_KEY | docker login nvcr.io --username '$oauthtoken' --password-stdin
```

3. **Launch the hybrid stack**:
```bash
docker compose -f docker-compose-hybrid-rag.yaml up -d
```

4. **Monitor startup**:
```bash
# Watch logs
docker compose -f docker-compose-hybrid-rag.yaml logs -f

# Check service health
docker compose -f docker-compose-hybrid-rag.yaml ps
```

⏱️ **First Launch**: ~20-30 minutes for Nemotron 49B download and initialization

### Verify Services

Once all services show `(healthy)` status:

1. **RAG Frontend UI**: http://localhost:8090
2. **RAG Server API**: http://localhost:8081/docs
3. **Ingestor API**: http://localhost:8082/docs
4. **LLM Health**: http://localhost:8999/v1/health/ready

## 📊 Service Ports

| Service | Port | Type | Notes |
|---------|------|------|-------|
| **RAG Frontend** | 8090 | Web UI | Main interface |
| **RAG Server** | 8081 | API | RAG queries |
| **Ingestor** | 8082 | API | Document upload |
| **Nemotron 49B** | 8999 | API | LLM endpoint |
| **Page Elements** | 8012-8014 | Internal | Extraction NIM |
| **Graphic Elements** | 8015-8017 | Internal | Extraction NIM |
| **Table Structure** | 8018-8020 | Internal | Extraction NIM |
| **Milvus** | 19530 | Internal | Vector DB |
| **MinIO** | 9010-9011 | Internal | Object storage |
| **Redis** | 6379 | Internal | Task queue |

## 🎨 What Makes This Hybrid?

### Local NIMs (GPU Required)
**Why Local?**
- Not available via cloud API
- Privacy (documents stay local)
- Speed (no network latency)
- Advanced multimodal extraction

**Services:**
1. **Nemotron 49B** - Superior performance to 9B
2. **Extraction NIMs** - Unique document processing capabilities

### Cloud APIs (No GPU Usage)
**Why Cloud?**
- Save VRAM (~15GB!)
- Latest model versions
- No maintenance
- Cost-effective for these services

**Services:**
1. **Embedding** - `nv-embedqa-e5-v5` (better than local 1B model)
2. **Reranking** - `nv-rerankqa-mistral-4b-v3` (same quality as local)
3. **OCR** - `paddleocr` (occasional use, saves 2GB)

## 🔧 Configuration

### Switch Back to Fully Local

If you want to run everything locally (e.g., for air-gapped environments):

1. Use the full stack: `docker-compose-full-rag-stack.yaml`
2. Requires ~81GB VRAM

### Use Different LLM

To switch to a different local LLM, modify in `docker-compose-hybrid-rag.yaml`:

```yaml
nim-llm:
  image: nvcr.io/nim/nvidia/<your-model>:latest
  environment:
    # Adjust memory params as needed
```

And update in `rag-server` environment:
```yaml
APP_LLM_MODELNAME: nvidia/<your-model-name>
```

### Adjust Memory Settings

For GPUs with different VRAM:

**More VRAM available** (increase context):
```yaml
NIM_MAX_MODEL_LEN: 120000  # 120K context
NIM_KVCACHE_PERCENT: 0.7
```

**Less VRAM available** (reduce context):
```yaml
NIM_MAX_MODEL_LEN: 32000   # 32K context
NIM_KVCACHE_PERCENT: 0.4
```

## 🧪 Testing

### 1. Test LLM Endpoint
```bash
curl -X POST http://localhost:8999/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/llama-3.3-nemotron-super-49b-v1.5",
    "messages": [{"role":"user", "content":"Hello!"}],
    "max_tokens": 100
  }'
```

### 2. Upload a Document
```bash
curl -X POST http://localhost:8082/v1/documents \
  -F "documents=@your-file.pdf" \
  -F 'data={"collection_name": "test_collection"}'
```

### 3. Query RAG
```bash
curl -X POST http://localhost:8081/v1/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is in the document?",
    "collection_name": "test_collection"
  }'
```

## 🐛 Troubleshooting

### Issue: Out of Memory
**Symptom**: Nemotron 49B crashes with CUDA OOM

**Solutions**:
1. Reduce context window: `NIM_MAX_MODEL_LEN: 32000`
2. Lower KV cache: `NIM_KVCACHE_PERCENT: 0.4`
3. Check GPU usage: `nvidia-smi`

### Issue: Cloud API Errors
**Symptom**: Embedding or reranking failures

**Solutions**:
1. Verify NGC_API_KEY is set correctly
2. Check network connectivity
3. View logs: `docker logs rag-server`

### Issue: Extraction NIMs Slow to Start
**Symptom**: Services take >10 minutes to start

**Solutions**:
1. First launch downloads models (~15GB each)
2. Check disk space: `df -h`
3. Monitor logs: `docker logs -f page-elements`

### Issue: Frontend Can't Connect
**Symptom**: UI loads but can't query

**Solutions**:
1. Check all services are healthy: `docker compose ps`
2. Verify RAG server: `curl http://localhost:8081/docs`
3. Check browser console for errors

## 📈 Performance Characteristics

### Latency
- **LLM Generation**: ~2-5 seconds (49B model)
- **Embedding** (Cloud): ~200-500ms
- **Reranking** (Cloud): ~300-600ms
- **Document Extraction**: ~5-30 seconds depending on complexity

### Throughput
- **Single request mode**: Optimized for quality
- **Batch size**: 1 (memory optimized)
- **Best for**: Interactive use, not high-throughput production

## 🔐 Security Notes

- Documents are processed **locally** (never sent to cloud)
- Only **embeddings and reranking** use cloud APIs
- Text snippets sent to cloud APIs for ranking
- API key stored in environment (use secrets management for production)

## 📚 Integration with Mercury

Mercury Agent can connect to this RAG stack:

**Mercury config** (`mercury_agent/configs/config.yml`):
```yaml
functions:
  nvbp_rag:
    base_url: "http://localhost:8081/v1"
    collection_name: "SPH"

llms:
  nim_llm:
    model_name: nvidia/llama-3.3-nemotron-super-49b-v1.5
    base_url: "http://localhost:8999/v1"
```

## 🆚 Comparison: Hybrid vs Full Local

| Aspect | Hybrid Stack | Full Local Stack |
|--------|--------------|------------------|
| **VRAM Usage** | ~66 GB | ~81 GB |
| **LLM** | Nemotron 49B (better) | Nemotron 9B |
| **Context** | 65K tokens | 120K tokens |
| **Embedding** | Cloud (e5-v5, latest) | Local (1B model) |
| **Reranking** | Cloud (same quality) | Local (same model) |
| **Internet Required** | Yes (for embed/rank) | No (fully offline) |
| **Cost** | API calls | None |
| **Privacy** | Documents stay local | Fully local |
| **Best For** | Most users | Air-gapped/high privacy |

## 📖 Resources

- **Nemotron 49B**: https://build.nvidia.com/nvidia/llama-3.3-nemotron-super-49b-v1.5
- **Embedding Model**: https://build.nvidia.com/nvidia/nv-embedqa-e5-v5
- **Reranking Model**: https://build.nvidia.com/nvidia/nv-rerankqa-mistral-4b-v3
- **RAG Blueprint**: https://github.com/NVIDIA/GenerativeAIExamples
- **Mercury Documentation**: [../../README.md](../../README.md)

## 📝 Version History

| Version | Date | Changes |
|---------|------|---------|
| v1.0 | Nov 4, 2025 | Initial hybrid deployment with Nemotron 49B + cloud APIs |

---

**💡 Tip**: This hybrid approach gives you the **best of both worlds** - powerful local 49B model for generation, with efficient cloud APIs for utility functions, all while saving 15GB of VRAM!

