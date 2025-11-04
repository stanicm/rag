# 🚀 Quick Start Guide - Full RAG Stack

## One-Command Launch

\`\`\`bash
cd /home/milos/eragbpv3/deploy/compose

# Set environment
export NGC_API_KEY="your-ngc-api-key-here"
export MODEL_DIRECTORY=~/.cache/model-cache
export USERID=$(id -u)
export DOCKER_VOLUME_DIRECTORY=./volumes

# Launch everything
docker compose -f docker-compose-full-rag-stack.yaml up -d

# Monitor startup (Ctrl+C to exit, services keep running)
docker compose -f docker-compose-full-rag-stack.yaml logs -f
\`\`\`

## Access Points

Once all services show \`(healthy)\`:

- **RAG Chat UI**: http://localhost:8090
- **RAG API Docs**: http://localhost:8081/docs  
- **Ingestor API**: http://localhost:8082/docs

## Stop Everything

\`\`\`bash
docker compose -f docker-compose-full-rag-stack.yaml down
\`\`\`

## Key Features

✅ **Nemotron Nano 9B** with 120K context window
✅ **Memory optimized** (~65GB VRAM)
✅ **Full document extraction** (text, tables, images)
✅ **Vector search** with Milvus
✅ **Production-ready** with health checks
✅ **Sequential startup** - no race conditions

See \`FULL_STACK_README.md\` for detailed documentation.
