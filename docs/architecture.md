# Architecture

This document describes the internal design of the ingestion system, its data flow, and the reasoning behind important implementation decisions.

---

## High Level Overview

The system processes raw documents, removes sensitive information, generates embeddings, and stores searchable vectors — while preserving retrieval quality and maintaining privacy guarantees.

Core goals:

- Prevent storage of raw PII
- Preserve semantic search quality
- Maintain low latency
- Allow pluggable infrastructure components
- Support observability and reproducibility

---

## End‑to‑End Pipeline Flow

```
Document Input
     │
     ▼
Stage 1: Cache Lookup (hash of original text → Redis/Memory)
     │                    │
  Cache HIT           Cache MISS
     │                    │
     │          ┌─────────┴─────────┐
     │          ▼                   ▼
     │    PII Detection       Embedding Gen
     │    (Presidio)          (FastEmbed)
     │          └─────────┬─────────┘
     │                    ▼
     │              Redaction Applied
     │                    │
     │              Cache Result Stored
     │                    │
     └──────────────┬──────┘
                    ▼
           Stage 4: VectorStore.add_vectors()
           (stores sanitised text + embedding)
                    │
                    ▼
           Stage 5: Metrics logged (Tracker)
```

---

## Processing Stages

### 1. Document Intake

The pipeline accepts a `Document` object:

- `id` → unique identifier
- `text` → raw input (may contain sensitive data)
- `metadata` → optional attributes for filtering/search

The original text is **never persisted** beyond processing scope.

---

### 2. Cache Lookup

A keyed hash (HMAC) of the original text is generated and used as the cache key.

Cache stores:

- Redacted text
- Detected entity types

Cache never stores:

- Raw text
- Embeddings

Design reasoning:

Caching embeddings risks stale vector representations after model upgrades. Regenerating embeddings guarantees consistency.

---

### 3. Parallel Processing (Miss Path)

On cache miss, two independent operations execute concurrently:

```python
# executed simultaneously
results, embedding = await asyncio.gather(
    pii_detector.detect(text),
    embedder.embed_text(text)
)
```

Benefits:

- Reduces latency by ~30–50%
- Fully utilizes CPU + model inference time
- Prevents PII detection from becoming a bottleneck

---

### 4. PII Detection & Redaction

PII entities are detected using Presidio.

Example transformation:

```
Input:  "Call John at 9876543210 tomorrow"
Output: "Call <PERSON> at <PHONE_NUMBER> tomorrow"
```

Stored output:

- Sanitised text
- Entity types list

Original text is discarded after embedding generation.

---

### 5. Shadow Mode Embedding

Embeddings are generated from the **original text (before redaction)**.

Why this matters:

Redacted text loses semantic meaning:

```
"Call <PERSON> at <PHONE_NUMBER>"
```

Embedding quality drops significantly because placeholders carry no contextual information.

Security note:

Embeddings are high‑dimensional float vectors. Recovering original PII from them is computationally impractical with current techniques. This trade‑off should still be disclosed in compliance documentation.

---

### 6. Vector Storage

The vector database stores:

- Embedding vector
- Sanitised text
- Metadata

It never stores raw PII.

This enables semantic search without exposing sensitive information.

---

### 7. Observability & Metrics

The tracker logs:

- Processing time
- Cache hit rate
- Detection counts
- Batch throughput

This allows performance tuning and monitoring in production environments.

---

## Component Responsibilities

| Component | Responsibility |
|--------|----|
| Cache | Avoid repeated PII detection |
| PII Detector | Identify sensitive entities |
| Redactor | Replace entities with tokens |
| Embedder | Convert text → vector representation |
| Vector Store | Persist searchable embeddings |
| Tracker | Metrics & experiment logging |

---

## Failure Handling

| Failure | Behavior |
|------|------|
| PII detection fails | Document marked failed |
| Embedding fails | Document skipped |
| Vector DB failure | Retry or batch partial success |
| Cache failure | Continue without cache |

The system favors availability over strict atomicity.

---

## Scaling Strategy

Horizontal scaling characteristics:

- Stateless workers
- Shared vector DB
- Shared cache layer

Recommended deployment:

- Multiple ingestion workers
- Central Redis cache
- Managed vector database

---

## Security Guarantees

The system ensures:

- Raw PII never stored in vector DB
- Cache stores only redacted data
- Embeddings generated transiently
- Hash‑based cache keys prevent reconstruction

---

## Design Trade‑offs

| Decision | Benefit | Cost |
|------|------|------|
| Embed original text | High search quality | Embedding contains indirect information |
| Do not cache embeddings | Consistency | Extra compute cost |
| Parallel execution | Low latency | Slight memory overhead |

---

## Summary

The architecture prioritizes privacy‑preserving semantic search. By separating redaction storage from embedding generation and executing expensive tasks in parallel, the system achieves strong retrieval performance while preventing sensitive data persistence.

