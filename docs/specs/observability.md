# Observability / Evals

## Метрики (Prometheus)

| Метрика | Тип | Описание |
|---|---|---|
| `request_latency_ms` | Histogram | End-to-end latency per request |
| `retrieval_latency_ms` | Histogram | Время retrieval + reranking |
| `llm_latency_ms` | Histogram | Время LLM вызова |
| `llm_tokens_used` | Counter | Токены per request |
| `api_call_latency_ms` | Histogram | Latency внешних API по endpoint |
| `api_call_errors_total` | Counter | Ошибки API по типу |
| `fallback_triggered_total` | Counter | Срабатывания fallback по компоненту |
| `hallucination_detected_total` | Counter | Случаи, когда validator отбросил ID |
| `clarification_requested_total` | Counter | Уточняющие вопросы |
| `session_count_active` | Gauge | Активные сессии |

## Трейсы (OpenTelemetry)

Каждый запрос трейсируется как span-дерево:

```
request (root span)
├── sanitize
├── intent_analysis (LLM span: model, tokens, latency)
├── retrieval (span: query, top_k, latency)
├── reranking (span: top_n, latency)
├── api_enrichment
│   ├── price_api (span: product_ids, latency, status)
│   ├── availability_api
│   └── promotion_api
├── llm_generate (span: model, tokens, latency)
└── validate (span: valid_count, stripped_count)
```

Атрибуты span: `session_id` (anonymized), `intent_type`, `latency_ms`, `model_version`.

## Логи

Формат: JSON, structured.

| Событие | Поля |
|---|---|
| Request received | `timestamp, session_id, intent_type, msg_hash` |
| Retrieval result | `session_id, top_k_count, latency_ms` |
| LLM call | `session_id, model, input_tokens, output_tokens, latency_ms` |
| Validation | `session_id, valid_count, stripped_count` |
| Fallback triggered | `session_id, component, reason` |
| Error | `session_id, component, error_type, message` |

## Offline Evals (MLflow)

| Метрика | Метод |
|---|---|
| `precision@5` | Labeled test set (ручная разметка 200 запросов) |
| `recall@10` | То же |
| `nDCG@5` | То же |
| `hallucination_rate` | Validator hits / total requests |
| `clarification_rate` | Доля запросов с clarification |
| `intent_accuracy` | Accuracy против labeled intent test set |

Запускаются при каждом изменении моделей (retriever, reranker, LLM). Результаты логируются в MLflow `:5050`.

## Дашборды (Grafana)

| Дашборд | Содержание |
|---|---|
| System Health | p50/p95 latency, error rate, fallback rate |
| LLM Performance | tokens/request, latency distribution, fallback triggers |
| Quality | hallucination_rate, clarification_rate, retrieval metrics |
| API Layer | per-endpoint latency, error rate, cache hit ratio |