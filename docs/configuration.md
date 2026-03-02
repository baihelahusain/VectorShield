# Configuration Reference

This document describes all available configuration keys, their types, default values, and behavior.

---

## Configuration Table

| Key | Type | Default | Description |
|----|----|----|----|
| `cache_backend` | `str` | `"memory"` | Cache implementation: `memory`, `redis`, `none` |
| `cache_ttl` | `int` | `3600` | Cache TTL in seconds |
| `redis_host` | `str` | `"localhost"` | Redis hostname |
| `redis_port` | `int` | `6379` | Redis port |
| `redis_db` | `int` | `0` | Redis database index |
| `embedding_model` | `str` | `"BAAI/bge-small-en-v1.5"` | FastEmbed model name |
| `embedding_batch_size` | `int` | `32` | Batch size for `embed_batch` calls |
| `vectorstore_backend` | `str` | `"chroma"` | Vector store: `chroma` or `custom` |
| `vectorstore_path` | `str` | `"./vectorshield_data"` | ChromaDB persistence directory |
| `vectorstore_collection` | `str` | `"documents"` | ChromaDB collection name |
| `parallel_processing` | `bool` | `True` | Run PII detection and embedding in parallel |
| `max_batch_size` | `int` | `100` | Maximum documents per `process_batch` call |
| `pii_entities` | `list[str]` | See below | Presidio entity types to detect |
| `tracking_enabled` | `bool` | `False` | Enable experiment tracking |
| `tracking_backend` | `str` | `"none"` | Tracker: `mlflow` or `none` |
| `mlflow_tracking_uri` | `str` | `"http://localhost:5000"` | MLflow server URI |
| `mlflow_experiment_name` | `str` | `"vectorshield"` | MLflow experiment name |
| `timeout_seconds` | `int` | `30` | Per-document processing timeout |

---

## Default PII Entities

```python
[
    "PERSON",
    "PHONE_NUMBER",
    "EMAIL_ADDRESS",
    "CREDIT_CARD",
    "US_SSN",
    "LOCATION",
    "DATE_TIME",
    "MEDICAL_LICENSE",
    "US_PASSPORT"
]
```

---

## Example Configuration

```yaml
cache_backend: memory
cache_ttl: 3600
embedding_model: BAAI/bge-small-en-v1.5
vectorstore_backend: chroma
vectorstore_path: ./vectorshield_data
parallel_processing: true
tracking_enabled: false
timeout_seconds: 30
```

