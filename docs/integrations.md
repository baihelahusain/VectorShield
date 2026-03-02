# Custom Integrations

This guide explains how to plug external services and custom implementations into the ingestion pipeline.

---

## Custom Embedder (OpenAI)

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

---

## Custom Cache (Memcached)

```python
import memcache, json
mc = memcache.Client(['127.0.0.1:11211'])

pipeline = IngestionPipeline(
    cache_get_func=lambda k: json.loads(mc.get(k)) if mc.get(k) else None,
    cache_set_func=lambda k, v, ttl: mc.set(k, json.dumps(v), time=ttl)
)
```

---

## Custom Vector Store (Pinecone)

```python
from pinecone import Pinecone
pc = Pinecone(api_key="...")
index = pc.Index("my-index")

pipeline = IngestionPipeline(
    add_vectors_func=lambda ids, embs, docs, meta: index.upsert(
        vectors=[
            (id_, emb, {"text": doc, **(m or {})})
            for id_, emb, doc, m in zip(ids, embs, docs, meta or [{}]*len(ids))
        ]
    ),
    search_func=lambda emb, k, filters: [
        {"id": m["id"], "document": m["metadata"]["text"], "score": m["score"]}
        for m in index.query(vector=emb, top_k=k, filter=filters).matches
    ]
)
```

---

## Direct Instance Injection

```python
from vectorshield.implementations import MemoryCache
from my_project import MyCustomEmbedder, MyCustomVectorStore

pipeline = IngestionPipeline(
    cache=MemoryCache(),
    embedder=MyCustomEmbedder(),
    vectorstore=MyCustomVectorStore()
)
```

