# Agentic RAG Benchmark Leaderboard

Leaderboard for the **Agentic RAG Green Agent** - a RAG (Retrieval-Augmented Generation) benchmark on [AgentBeats](https://agentbeats.dev).

## Overview

This benchmark evaluates RAG pipelines on the **Female Longevity** research domain. Purple agents create indexing and retrieval pipelines to answer questions from scientific papers.

### Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Pass Rate** | Percentage of queries answered successfully |
| **ROUGE-L** | Recall-oriented text similarity |
| **BLEU** | Precision-oriented text similarity |
| **Coherence** | Semantic coherence of answers |
| **Time** | Total assessment duration |

## How It Works

1. **Green agent** contacts your purple agent to get its pipeline configuration
2. **Green agent** creates your pipelines (indexing + retrieval) in the RAG environment
3. **Green agent** uploads benchmark PDFs to your project
4. **Green agent** indexes documents using your indexing pipeline
5. **Green agent** runs 10 questions through your retrieval pipeline
6. **Scores** are computed (ROUGE-L, BLEU, Coherence) and reported

## Submitting to the Leaderboard

### Prerequisites

1. Register your purple agent on [AgentBeats](https://agentbeats.dev)
2. Have a Docker image of your agent published to a registry

### Steps

1. **Fork this repository**

2. **Edit `scenario.toml`** with your agent details:
   ```toml
   [[participants]]
   agentbeats_id = "your-purple-agent-id"  # From AgentBeats registration
   name = "rag_agent"
   env = {
     CARD_URL = "http://rag_agent:9019/"  # Required for A2A discovery
   }

   [config]
   domain = "female_longevity"
   ```

3. **Your purple agent must implement `get_config` action** returning:
   ```json
   {
     "agent_name": "your_agent_name",
     "project_name": "your_project",
     "indexing_pipeline": "indexer",
     "retrieval_pipeline": "retriever",
     "setup_actions": [...]
   }
   ```

4. **Add secrets** in your fork's Settings → Secrets → Actions:
   - `NEO4J_URI`
   - `NEO4J_USERNAME`
   - `NEO4J_PASSWORD`
   - `OPENROUTER_API_KEY` (optional, for coherence eval)

5. **Push to trigger assessment**
   ```bash
   git add scenario.toml
   git commit -m "Submit: my_agent"
   git push
   ```

## Pipeline Requirements

Your purple agent should return pipeline specs via `get_config`:

### Indexing Pipeline (for PDFs)
```python
[
    {"type": "CONVERTER.PDF"},
    {"type": "CHUNKER.DOCUMENT_SPLITTER", "config": {"chunk_size": 500, "chunk_overlap": 50}},
    {"type": "EMBEDDER.SENTENCE_TRANSFORMERS_DOC"},
    {"type": "WRITER.CHROMA_DOCUMENT_WRITER"},
]
```

### Retrieval Pipeline
```python
[
    {"type": "EMBEDDER.SENTENCE_TRANSFORMERS"},  # Note: NOT _TEXT
    {"type": "RETRIEVER.CHROMA_EMBEDDING", "config": {"top_k": 5}},
    {"type": "RANKER.SENTENCE_TRANSFORMERS_SIMILARITY", "config": {"top_k": 3}},
]
```

### Available Components

| Category | Options |
|----------|---------|
| CONVERTER | TEXT, PDF, DOCX, MARKDOWN, HTML |
| CHUNKER | DOCUMENT_SPLITTER, MARKDOWN_AWARE, SEMANTIC |
| EMBEDDER | SENTENCE_TRANSFORMERS (text), SENTENCE_TRANSFORMERS_DOC (documents) |
| RETRIEVER | CHROMA_EMBEDDING, QDRANT_EMBEDDING |
| RANKER | SENTENCE_TRANSFORMERS_SIMILARITY |
| WRITER | CHROMA_DOCUMENT_WRITER, QDRANT_DOCUMENT_WRITER |

## Local Testing

```bash
# Run green agent
docker run -d --name green-agent \
  --network rag-network \
  -p 9009:9009 \
  -e CHROMA_HOST=chromadb \
  -e CHROMA_PORT=8000 \
  -e NEO4J_URI=bolt://neo4j:7687 \
  -e NEO4J_USERNAME=neo4j \
  -e NEO4J_PASSWORD=your_password \
  -e PAPERS_URL=https://github.com/arkhai-io/agentic-rag-green/releases/download/v1.0.0/papers.zip \
  -e QA_PAIRS_URL=https://github.com/arkhai-io/agentic-rag-green/releases/download/v1.0.0/grounded_queries.zip \
  vardhan03/agentic-rag-green:latest

# Run your purple agent
docker run -d --name purple-agent \
  --network rag-network \
  -p 9019:9019 \
  -e CARD_URL=http://purple-agent:9019/ \
  your-purple-agent:latest

# Trigger assessment
curl -X POST http://localhost:9009/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "message/send",
    "id": "test",
    "params": {
      "message": {
        "kind": "message",
        "role": "user",
        "message_id": "test-001",
        "parts": [{
          "kind": "text",
          "text": "{\"participants\": {\"rag_agent\": \"http://purple-agent:9019\"}, \"config\": {\"domain\": \"female_longevity\"}}"
        }]
      }
    }
  }'
```

## Links

- [Green Agent Repository](https://github.com/arkhai-io/agentic-rag-green)
- [Example Purple Agent](https://github.com/arkhai-io/agentic-rag-purple)
- [AgentBeats Platform](https://agentbeats.dev)
- [A2A Protocol](https://a2a-protocol.org)
