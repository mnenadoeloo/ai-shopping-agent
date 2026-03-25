# Serving / Config

## Стек моделей

### LLM — Qwen3.5-9B

| Параметр | Значение |
|---|---|
| Модель | `Qwen/Qwen3.5-9B` |
| Формат | `bfloat16` |
| Контекстное окно | в PoC используем 8k для контроля latency |
| Function calling | нативная поддержка через `tools` параметр OpenAI-совместимого API |
| Runtime | vLLM ≥ 0.6.x |

### Embedding — BGE-M3

| Параметр | Значение |
|---|---|
| Модель | `BAAI/bge-m3` |
| Размерность | 1024-dim |
| Языки | мультиязычный (100+), включая русский — важно для запросов пользователей |
| Max sequence | 8192 токенов |
| Режим | dense (для PoC); поддерживает также sparse и colbert — можно активировать позже |
| Инференс | HuggingFace `FlagEmbedding` или `sentence-transformers` |

### Reranker — BGE-Reranker-v2-M3

| Параметр | Значение |
|---|---|
| Модель | `BAAI/bge-reranker-v2-m3` |
| Тип | cross-encoder |
| Языки | мультиязычный, совместим с BGE-M3 |
| Max sequence | 512 токенов (query + document) |
| Latency | ~150–300 мс на 20 кандидатов (CPU), ~50–100 мс (GPU) |
| Инференс | `FlagEmbedding` `FlagReranker` |

---

## Запуск

### vLLM (LLM inference server)

```bash
vllm serve Qwen/Qwen3.5-9B \
  --dtype bfloat16 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --port 8001 \
  --served-model-name qwen3.5-9b
```

### Embedding + Reranker (отдельный сервис)

```bash
# Через FlagEmbedding inference server или кастомный FastAPI wrapper
python -m FlagEmbedding.inference.embedder.encoder_only.m3 \
  --model_name_or_path BAAI/bge-m3 \
  --port 8002

# Reranker
python -m FlagEmbedding.inference.reranker.encoder_only.base \
  --model_name_or_path BAAI/bge-reranker-v2-m3 \
  --port 8003
```

### Основное приложение

```bash
# dev
uvicorn app.main:app --reload --port 8000

# prod
docker compose up --build
```

---

## Конфигурация (env vars)

| Переменная | Описание | Default |
|---|---|---|
| `LLM_BASE_URL` | URL vLLM сервера | `http://localhost:8001/v1` |
| `LLM_MODEL` | Имя модели | `qwen3.5-9b` |
| `LLM_TIMEOUT_MS` | Timeout LLM вызова | `10000` |
| `LLM_MAX_TOKENS_OUTPUT` | Лимит токенов в ответе | `1000` |
| `LLM_TEMPERATURE` | Temperature | `0.4` |
| `EMBED_BASE_URL` | URL embedding сервиса | `http://localhost:8002` |
| `RERANKER_BASE_URL` | URL reranker сервиса | `http://localhost:8003` |
| `VECTOR_DB` | `faiss` или `qdrant` | `faiss` |
| `QDRANT_URL` | URL Qdrant instance | — |
| `REDIS_URL` | URL Redis | `redis://localhost:6379` |
| `PRICE_API_URL` | URL Price API | — |
| `AVAILABILITY_API_URL` | URL Availability API | — |
| `PROMOTION_API_URL` | URL Promotion API | — |
| `API_TIMEOUT_MS` | Timeout внешних API | `1000` |
| `SESSION_TTL_SEC` | TTL сессии в Redis | `1800` |
| `MAX_TOKENS_INPUT` | Лимит входных токенов (context budget) | `6000` |
| `RETRIEVER_TOP_K` | top-k для retrieval | `20` |
| `RERANKER_TOP_N` | top-n после reranking | `5` |
| `RATE_LIMIT_RPM` | Запросов в минуту per session | `20` |
| `LOG_LEVEL` | Уровень логирования | `INFO` |

## Секреты

| Секрет | Способ передачи |
|---|---|
| `PRICE_API_KEY` | Env var / secrets manager |
| `AVAILABILITY_API_KEY` | Env var / secrets manager |
| `PROMOTION_API_KEY` | Env var / secrets manager |

## Версии моделей

| Роль | Модель | 
|---|---|
| LLM | `Qwen/Qwen3.5-9B` | 
| Embedding | `BAAI/bge-m3` | 
| Reranker | `BAAI/bge-reranker-v2-m3` | 