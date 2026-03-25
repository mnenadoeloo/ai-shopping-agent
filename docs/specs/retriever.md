# Retriever

## Назначение

Семантический поиск по каталогу товаров на основе dense embeddings. Возвращает top-k кандидатов для последующего reranking.

---

## Источники данных

| Источник | Описание |
|---|---|
| Product Catalog | ~10k товаров: название, описание, категория, цена, рейтинг, stock, popularity, discount |
| Vector Index | Предвычисленные эмбеддинги всех товаров, индексированы в FAISS (PoC) или Qdrant (scale) |

---

## Индекс

| Параметр | Значение |
|---|---|
| Embedding model | `BAAI/bge-m3` (1024-dim, multilingual) |
| Index type | FAISS `IVFFlat` (PoC), Qdrant HNSW (production) |
| Similarity metric | Cosine similarity |
| Indexed fields | title + description (конкатенация) |
| Index update policy | Offline batch при изменении каталога (PoC) |
| Index size | ~10k vectors, ~40 MB (PoC; 1024-dim vs 384-dim у MiniLM) |

---

## Search

```python
def retrieve(query: str, top_k: int = 20) -> list[Product]:
    query_vector = encoder.encode(query)
    results = index.search(query_vector, top_k)
    return hydrate(results)  # обогатить метаданными из каталога
```

**Параметры поиска:**

| Параметр | Значение |
|---|---|
| top_k | 20 |
| Min similarity threshold | 0.3 (ниже — не возвращать) |
| Pre-filter | по категории или бюджету, если явно указаны в intent |

---

## Reranking

| Параметр | Значение |
|---|---|
| Model | `BAAI/bge-reranker-v2-m3` |
| Input | query + (title + description) пары |
| Output | relevance score per candidate |
| top_n | 5 |
| Latency | ~80–150 мс на 20 кандидатов (GPU); ~250–400 мс (CPU) |
| Max sequence | 512 токенов (query + document); длинные описания обрезаются |
| Fallback | при exception — вернуть raw retrieval results без reranking |

---

## Ограничения

- Семантический поиск хуже работает при коротких (1–2 слова) запросах без контекста
- Bi-encoder не учитывает синтаксис запроса — только семантику
- Переиндексация занимает ~5–10 мин для 10k товаров (BGE-M3 медленнее MiniLM из-за 1024-dim)
- При масштабировании до 1M товаров — необходим Qdrant HNSW + distributed deployment
- Max sequence reranker = 512 токенов: длинные описания товаров обрезаются; рекомендуется заранее нормализовать описания до ~300 токенов при индексации