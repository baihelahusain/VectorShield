# VectorShield 🛡️

**Privacy-Preserving RAG Middleware — Stop PII from reaching your vector database.**

[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PyPI version](https://img.shields.io/pypi/v/vectorshield.svg)](https://pypi.org/project/vectorshield/)

---

## What is VectorShield?

VectorShield is a backend middleware layer that sits between your application and your vector database. It automatically detects and redacts Personally Identifiable Information (PII) and Protected Health Information (PHI) **before** embeddings and documents are stored — ensuring that raw sensitive data never reaches persistent storage.

Privacy is enforced at the **data ingestion layer**, not at the LLM output layer.

```
Your App  →  VectorShield  →  ChromaDB / Pinecone / Weaviate
                  ↓
         PII detected & redacted
         Embeddings generated from original (semantic quality preserved)
         Only sanitised text stored
```

### The Core Problem It Solves

RAG systems retrieve and store documents in vector databases. If those documents contain names, phone numbers, SSNs, emails, or medical data, that information becomes **permanently embedded** in your vector store — queryable, retrievable, and at risk. VectorShield intercepts this data before it ever lands.

---

## Key Features

- **Parallel PII detection + embedding** — Both run concurrently using `asyncio.gather` to minimise latency overhead
- **Semantic quality preserved** — Embeddings are generated from the *original* text before redaction; only sanitised text is stored
- **Redis-backed caching** — PII detection results are cached to eliminate redundant processing of repeated text
- **Pluggable architecture** — Swap out any component (embedder, vector store, cache, tracker) with your own implementation
- **Zero vendor lock-in** — Works with OpenAI, Cohere, Sentence-Transformers, Pinecone, Weaviate, Qdrant, Memcached, and more
- **9 PII entity types** out of the box (PERSON, PHONE, EMAIL, SSN, CREDIT_CARD, LOCATION, DATE_TIME, MEDICAL_LICENSE, PASSPORT)
- **Metrics & experiment tracking** — Optional MLflow integration for latency, cache hit rate, and PII entity counts
- **Async-first** — Built for FastAPI and high-throughput batch ingestion

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                 IngestionPipeline                │
│                                                 │
│  Document → [Cache Check]                       │
│               ├─ HIT  → reuse clean_text        │
│               └─ MISS → parallel processing:    │
│                          ├─ PII Detection       │
│                          └─ Embedding           │
│             ↓                                   │
│          Redaction → clean_text                 │
│             ↓                                   │
│          VectorStore.add_vectors(               │
│              embedding=from original,           │  ← semantic quality
│              document=clean_text                │  ← privacy safe
│          )                                      │
└─────────────────────────────────────────────────┘
```

**Privacy guarantee:** The embedding captures the full semantic meaning of the original text. The stored document contains only the redacted version. The raw PII is never written to disk.

---

## Installation

```bash
pip install vectorshield
```

**With optional extras:**

```bash
# Redis caching support
pip install vectorshield[redis]

# MLflow experiment tracking
pip install vectorshield[mlflow]

# All optional dependencies
pip install vectorshield[all]
```

**System requirements:**
- Python 3.9+
- FastEmbed downloads BAAI/bge-small-en-v1.5 (~130MB) on first run
- ChromaDB requires SQLite 3.35+ (standard on Ubuntu 22.04+, macOS 12+)

---

## Quickstart

```python
import asyncio
from vectorshield import IngestionPipeline, Document

async def main():
    # Initialise with defaults (memory cache + ChromaDB + FastEmbed)
    pipeline = IngestionPipeline()
    await pipeline.initialize()

    # Process a document containing PII
    doc = Document(
        id="doc_001",
        text="Please call John Smith at 555-867-5309 or email john@example.com",
        metadata={"source": "hr_system", "category": "employee"}
    )

    result = await pipeline.process_document(doc)

    print(result.clean_text)
    # → "Please call <PERSON> at <PHONE_NUMBER> or email <EMAIL_ADDRESS>"

    print(result.pii_entities)
    # → ["PERSON", "PHONE_NUMBER", "EMAIL_ADDRESS"]

    print(result.embedding_preview)
    # → [0.023, -0.014, 0.091, ...]  (384-dim from original text)

    await pipeline.shutdown()

asyncio.run(main())
```

---

## Batch Processing

```python
async def batch_example():
    pipeline = IngestionPipeline()
    await pipeline.initialize()

    documents = [
        Document(id="1", text="Patient Jane Doe, SSN: 123-45-6789, admitted 12 Jan 2024"),
        Document(id="2", text="Invoice for Robert Johnson, card ending 4532 1234 5678 9012"),
        Document(id="3", text="Meeting notes from Q4 planning — no PII here"),
    ]

    result = await pipeline.process_batch(documents)

    print(f"Processed: {result['total_processed']}")
    print(f"Time: {result['total_time_seconds']}s")

    for r in result["results"]:
        print(f"[{r['id']}] PII found: {r['pii_found']}")

    await pipeline.shutdown()
```

---

## Configuration

VectorShield ships with safe defaults. Override only what you need:

```python
config = {
    # Cache
    "cache_backend": "redis",         # "memory" | "redis" | "none"
    "redis_host": "localhost",
    "redis_port": 6379,
    "cache_ttl": 3600,

    # Embeddings
    "embedding_model": "BAAI/bge-small-en-v1.5",

    # Vector store
    "vectorstore_backend": "chroma",
    "vectorstore_path": "./my_data",
    "vectorstore_collection": "documents",

    # Processing
    "parallel_processing": True,
    "max_batch_size": 100,

    # Tracking (optional)
    "tracking_enabled": True,
    "tracking_backend": "mlflow",
    "mlflow_tracking_uri": "http://localhost:5000",
}

pipeline = IngestionPipeline(config=config)
```

Or load from a YAML file:

```python
pipeline = IngestionPipeline()
# In CLI:
# vectorshield process -i docs.json -c config.yaml
```

---

## Custom Components

VectorShield's pluggable architecture lets you integrate any backend.

### Custom Embedder (OpenAI)

```python
from openai import OpenAI

client = OpenAI(api_key="sk-...")

pipeline = IngestionPipeline(
    embed_func=lambda text: client.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    ).data[0].embedding
)
```

### Custom Cache (Existing Redis Client)

```python
import redis, json

r = redis.Redis(host="localhost")

pipeline = IngestionPipeline(
    cache_get_func=lambda k: json.loads(r.get(k)) if r.get(k) else None,
    cache_set_func=lambda k, v, ttl: r.setex(k, ttl or 3600, json.dumps(v)),
)
```

### Custom Vector Store (Pinecone)

```python
from pinecone import Pinecone

pc = Pinecone(api_key="...")
index = pc.Index("my-index")

pipeline = IngestionPipeline(
    add_vectors_func=lambda ids, embs, docs, meta: index.upsert(
        vectors=[(id_, emb, {"text": doc, **(m or {})})
                 for id_, emb, doc, m in zip(ids, embs, docs, meta or [{}]*len(ids))]
    ),
    search_func=lambda emb, k, filters: [
        {"id": m["id"], "document": m["metadata"]["text"], "score": m["score"]}
        for m in index.query(vector=emb, top_k=k, filter=filters).matches
    ]
)
```

---

## CLI Usage

```bash
# Process documents from a JSON file
vectorshield process -i documents.json -o results.json

# With config and stats
vectorshield process -i documents.json -c config.yaml --stats

# Read from stdin
echo '{"documents": [{"id": "1", "text": "Call me at 555-0100"}]}' | vectorshield process

# Check version
vectorshield version
```

**Input JSON format:**

```json
{
  "documents": [
    {"id": "doc_1", "text": "Contact Alice Brown at alice@corp.com", "metadata": {"source": "email"}},
    {"id": "doc_2", "text": "Quarterly revenue report — no PII", "metadata": {"source": "finance"}}
  ]
}
```

---

## Metrics & Monitoring

```python
# Get metrics after processing
metrics = pipeline.get_metrics()

print(metrics["overview"])
# → {"total_documents": 50, "successful": 49, "failed": 1, "success_rate": "98.00%"}

print(metrics["privacy"])
# → {"total_pii_entities_redacted": 87, "unique_pii_types": ["PERSON", "EMAIL_ADDRESS", ...]}

print(metrics["performance"])
# → {"avg_latency_ms": "142.35", "min_latency_ms": "98.12", "max_latency_ms": "312.44"}

print(metrics["caching"])
# → {"cache_hits": 23, "cache_misses": 27, "cache_hit_rate": "46.00%"}
```

---

## PII Entity Types

| Entity | Example Input | Redacted Output |
|--------|--------------|-----------------|
| PERSON | `John Smith` | `<PERSON>` |
| PHONE_NUMBER | `555-867-5309` | `<PHONE_NUMBER>` |
| EMAIL_ADDRESS | `john@example.com` | `<EMAIL_ADDRESS>` |
| CREDIT_CARD | `4532 1234 5678 9012` | `<CREDIT_CARD>` |
| US_SSN | `123-45-6789` | `<US_SSN>` |
| LOCATION | `London, UK` | `<LOCATION>` |
| DATE_TIME | `12 January 2024` | `<DATE_TIME>` |
| MEDICAL_LICENSE | `MD-12345` | `<MEDICAL_LICENSE>` |
| US_PASSPORT | `A12345678` | `<US_PASSPORT>` |

PII detection is powered by [Microsoft Presidio](https://microsoft.github.io/presidio/).

---

## Limitations & Honest Caveats

VectorShield is designed to **reduce** PII exposure in vector databases, not to provide absolute guarantees. Be aware of the following:

- **Detection is probabilistic.** Presidio uses NLP models — novel PII formats, obfuscated data, or non-English text may not be caught. No detection system is 100% accurate.
- **Embeddings encode semantics.** Vector embeddings generated from PII-containing text may carry some semantic information about that PII, even if the stored text is redacted. This is a known limitation of the "embed-then-redact" pattern.
- **Metadata is not redacted.** VectorShield does not inspect or sanitise the `metadata` dict you attach to documents. Ensure PII is not passed in metadata fields.
- **Not a compliance tool.** VectorShield is a technical privacy control, not a legal compliance framework. It does not constitute GDPR, HIPAA, or CCPA compliance on its own.
- **English-centric.** The default Presidio configuration performs best on English-language text.

---

## Comparison with Alternative Approaches

| Approach | VectorShield | On-device Anonymisation | Local LLM | Access Control Only |
|----------|-------------|------------------------|-----------|---------------------|
| PII stored in vector DB | ❌ No | ❌ No | ⚠️ Depends | ✅ Yes |
| Semantic quality preserved | ✅ Yes | ⚠️ Partial | ✅ Yes | ✅ Yes |
| Requires local GPU | ❌ No | ❌ No | ✅ Yes | ❌ No |
| Scales to cloud RAG | ✅ Yes | ⚠️ Limited | ❌ Hard | ✅ Yes |
| Pluggable / vendor-agnostic | ✅ Yes | ❌ No | ❌ No | ✅ Yes |

---

## Project Structure

```
vectorshield/
├── __init__.py                  # Public API
├── ingest.py                    # Core IngestionPipeline
├── interfaces/
│   ├── cache.py                 # CacheBackend ABC
│   ├── embedder.py              # Embedder ABC
│   ├── vectorstore.py           # VectorStore ABC
│   └── tracker.py               # Tracker ABC
├── implementations/
│   ├── cache/
│   │   ├── memory.py            # MemoryCache
│   │   ├── redis.py             # RedisCache + NoOpCache
│   │   └── generic.py           # GenericCache (custom functions)
│   ├── embedder/
│   │   ├── fastembed.py         # FastEmbedEmbedder (default)
│   │   └── generic.py           # GenericEmbedder (custom functions)
│   ├── vectorstore/
│   │   ├── chroma.py            # ChromaVectorStore (default)
│   │   └── generic.py           # GenericVectorStore (custom functions)
│   └── tracker/
│       └── mlflow.py            # MLflowTracker + NoOpTracker
├── config/
│   ├── defaults.py              # DEFAULT_CONFIG
│   └── loader.py                # load_config(), YAML support
├── stats.py                     # MetricsCollector, IngestionMetrics
└── cli.py                       # vectorshield CLI
```

---

## Development Setup

```bash
git clone https://github.com/yourusername/vectorshield.git
cd vectorshield

python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

pip install -e ".[dev]"

# Run tests
pytest tests/ -v

# Run linter
ruff check vectorshield/
```

---

## Contributing

Contributions are welcome. Please open an issue first to discuss significant changes.

1. Fork the repo
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit changes (`git commit -m "Add your feature"`)
4. Push to branch (`git push origin feature/your-feature`)
5. Open a Pull Request

---

## Citation

If you use VectorShield in academic work, please cite:

```bibtex
@software{vectorshield2025,
  title  = {VectorShield: Privacy-Preserving Middleware for RAG Vector Databases},
  author = {Your Name},
  year   = {2025},
  url    = {https://github.com/yourusername/vectorshield}
}
```

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

## Acknowledgements

- [Microsoft Presidio](https://microsoft.github.io/presidio/) — PII detection and anonymisation engine
- [FastEmbed](https://github.com/qdrant/fastembed) — Lightweight CPU-friendly embedding library
- [ChromaDB](https://www.trychroma.com/) — Persistent local vector database
