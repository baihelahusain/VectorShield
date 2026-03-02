# API Reference

---

## Document

```python
Document(id: str, text: str, metadata: Optional[Dict] = None)
```

Input document for ingestion.

| Attribute | Type | Description |
|--------|----|----|
| `id` | `str` | Unique document identifier |
| `text` | `str` | Raw document text (may contain PII) |
| `metadata` | `dict` | Arbitrary metadata stored alongside embedding |

---

## ProcessedDocument

Returned by `process_document`. Call `.to_dict()` for a JSON‑serialisable form.

| Attribute | Type | Description |
|--------|----|----|
| `id` | `str` | Document ID |
| `original_preview` | `str` | First 100 chars of original text |
| `clean_text` | `str` | Sanitised text with PII replaced by tokens |
| `pii_entities` | `List[str]` | Entity types found (e.g. `["PERSON", "EMAIL_ADDRESS"]`) |
| `embedding_preview` | `List[float]` | First 5 values of the embedding vector |
| `metadata` | `dict` | Original metadata |
| `status` | `str` | `"success"` or `"failed"` |

---

## IngestionPipeline

```python
IngestionPipeline(
    config: Optional[Dict] = None,
    cache: Optional[CacheBackend] = None,
    embedder: Optional[Embedder] = None,
    vectorstore: Optional[VectorStore] = None,
    tracker: Optional[Tracker] = None,
    pii_policy: Optional[PIIPolicy] = None,
    # Custom function overrides:
    embed_func=None, embed_batch_func=None,
    cache_get_func=None, cache_set_func=None,
    add_vectors_func=None, search_func=None
)
```

---

### Methods

#### `await pipeline.initialize()`
Connect all components. Must be called before processing.

#### `await pipeline.shutdown()`
Disconnect all components and flush pending data.

#### `await pipeline.process_document(doc: Document) -> ProcessedDocument`
Process a single document through the full pipeline.

#### `await pipeline.process_batch(docs: List[Document]) -> Dict`
Process multiple documents concurrently.

Example response:

```json
{
  "batch_id": "uuid",
  "total_processed": 10,
  "total_time_seconds": 1.23,
  "results": [{ ...ProcessedDocument.to_dict()... }]
}
```

#### `pipeline.get_metrics() -> Dict`
Return aggregate metrics summary.

#### `pipeline.reset_metrics()`
Reset metrics counters.

---

## Interfaces

All interfaces are abstract base classes that custom implementations must satisfy.

---

### CacheBackend

```python
async def connect() -> None
async def close() -> None
async def get(key: str) -> Optional[Dict]
async def set(key: str, value: Dict, ttl: int = 3600) -> None
async def delete(key: str) -> None
```

---

### Embedder

```python
async def embed_text(text: str) -> List[float]
async def embed_batch(texts: List[str]) -> List[List[float]]
def get_embedding_dimension() -> int
```

---

### VectorStore

```python
async def connect() -> None
async def close() -> None
async def add_vectors(ids, embeddings, documents, metadata) -> None
async def query(query_embedding, top_k, filter_metadata) -> List[Dict]
async def delete(ids) -> None
```

---

### Tracker

```python
async def connect() -> None
async def close() -> None
async def log_metric(key, value, step) -> None
async def log_metrics(metrics: Dict, step) -> None
async def log_param(key, value) -> None
async def log_artifact(file_path, artifact_path) -> None
```

