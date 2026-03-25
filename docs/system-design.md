# System Design — AI Shopping Agent

## 1. Ключевые архитектурные решения

| Решение | Обоснование |
|---|---|
| LLM как оркестратор с tool use | Агент выбирает инструменты динамически; логика не захардкожена в коде |
| RAG (retrieval-augmented generation) | Ответ строится только на retrieved товарах — снижает hallucination rate |
| Reranking cross-encoder поверх dense retrieval | Повышает precision без потери recall; приемлемо по латентности |
| Stateful memory per session | Персонализация и диалоговый контекст без повторных вопросов |
| Grounding + post-process validation | Каждый товар в ответе верифицируется против каталога перед отправкой |
| Fallback-цепочка для LLM и API | Отказ внешнего провайдера не блокирует всю систему |

---

## 2. Модули и их роли

| Модуль | Роль |
|---|---|
| **User Interface** | Чат-интерфейс; отправляет сообщения, отображает ответы и карточки товаров |
| **LLM Orchestrator** | Ядро агента: intent analysis, выбор инструментов, grounding, генерация ответа |
| **Product Retriever** | Dense embedding search (FAISS/Qdrant), возвращает top-k кандидатов |
| **Reranker** | Cross-encoder переранжирует кандидатов, возвращает top-n релевантных |
| **External API Layer** | Интеграции с price/availability/promotion API; применяет таймауты и retry |
| **Memory Store** | Хранит историю диалога, предпочтения, предыдущие рекомендации (Redis) |
| **Validator** | Post-process проверка: ID товаров из ответа LLM существуют в каталоге |
| **Observability** | OTel Collector → Prometheus → Grafana; MLflow для eval-метрик |

---

## 3. Основной workflow

```
1.  User Input           → санитизация → LLM Orchestrator
2.  Intent Analysis      → intent type, entities (бюджет, категория, constraints)
3.  Low confidence?      → clarification question → User (stop, ждать ответ)
4.  Product Retrieval    → dense search, top-k=20
5.  Reranking            → cross-encoder, top-n=5
6.  External APIs        → price, availability, promotions (параллельно, timeout=1s)
7.  Context Assembly     → retrieved items + API data + memory → prompt context
8.  LLM Response Gen     → генерация ответа с объяснением
9.  Validation           → все упомянутые product ID существуют в каталоге
10. Memory Update        → сохранить диалог и предпочтения
11. Response Delivery    → User Interface
```

---

## 4. State / Memory / Context Handling

### Session State

- Идентификатор: анонимный UUID per session
- Хранение: Redis (in-memory, TTL=30 min)
- Содержит: `dialog_history`, `user_preferences`, `last_recommendations`, `clarification_pending`

### Memory Policy

| Тип данных | TTL | Действие при истечении |
|---|---|---|
| Dialog history | 30 min | Удаление; новая сессия начинается с чистого листа |
| User preferences | 30 min | Удаление; можно расширить до persistent для production |
| Last recommendations | 30 min | Удаление |

### Context Budget (LLM)

Ограничения на размер контекста, передаваемого в LLM:

| Слот | Лимит |
|---|---|
| System prompt | ~800 токенов (фиксированный) |
| Dialog history | последние 10 сообщений или ≤2000 токенов |
| Retrieved products | top-5, краткие карточки ≤1500 токенов |
| User message | ≤500 токенов (входная санитизация) |
| **Итого контекст** | **≤4800 токенов** |

При превышении: history summarization через отдельный LLM-вызов.

---

## 5. Retrieval-контур

### Pipeline

```
User Query → Query Encoder (HuggingFace bi-encoder)
           → Dense Vector (768-dim)
           → Vector DB Search (FAISS/Qdrant, cosine similarity)
           → top-k=20 candidates
           → Cross-Encoder Reranker (relevance score)
           → top-n=5 final candidates
```

### Параметры

| Параметр | Значение |
|---|---|
| Embedding model | sentence-transformers/all-MiniLM-L6-v2 (PoC) |
| Index type | FAISS IVF_FLAT (PoC) / Qdrant HNSW (scale) |
| top-k retrieval | 20 |
| top-n после reranking | 5 |
| Reranker | cross-encoder/ms-marco-MiniLM-L-6-v2 |
| Index update | offline batch (PoC); streaming updates — future |

### Ограничения

- Семантический поиск работает хуже при коротких (1–2 слова) или слишком общих запросах
- Reranking добавляет ~200–400 мс к латентности
- При изменении каталога требуется переиндексация

---

## 6. Tool / API интеграции

### Price API

| Параметр | Значение |
|---|---|
| Контракт | `GET /price?product_id={id}` → `{price, currency, updated_at}` |
| Timeout | 1000 мс |
| Retry | 1 раз при timeout, нет при 4xx |
| Fallback | использовать cached price с меткой "цена могла измениться" |
| Side effects | нет |

### Availability API

| Параметр | Значение |
|---|---|
| Контракт | `GET /availability?product_id={id}` → `{in_stock, quantity, eta}` |
| Timeout | 1000 мс |
| Retry | 1 раз при timeout |
| Fallback | показать товар с пометкой "наличие уточните" |
| Side effects | нет |

### Promotion API

| Параметр | Значение |
|---|---|
| Контракт | `GET /promotions?product_id={id}` → `{discount_pct, promo_label, expires_at}` |
| Timeout | 800 мс |
| Retry | нет (некритичное) |
| Fallback | не показывать скидку (graceful degradation) |
| Side effects | нет |

Все три вызываются параллельно (asyncio.gather), суммарный timeout = max(1000, 1000, 800) = 1000 мс.

---

## 7. Failure Modes, Fallbacks и Guardrails

### Failure Modes и Fallbacks

| Компонент | Failure | Fallback |
|---|---|---|
| LLM API (primary) | timeout / 5xx | Переключение на open-source LLM (local), circuit breaker после 3 ошибок |
| Product Retrieval | пустой результат | Clarification question пользователю; не генерировать товары из головы |
| Reranker | exception | Пропустить reranking, использовать raw retrieval results |
| Price/Availability API | timeout | Cached значения с явной меткой актуальности |
| Memory Store (Redis) | недоступен | Stateless режим: контекст только из текущего запроса |
| Validator | не прошёл | Удалить несуществующий product ID из ответа, не отправлять пользователю |

### Guardrails

| Guardrail | Реализация |
|---|---|
| Prompt injection protection | Санитизация input; user content в role=user, не в system |
| Hallucination grounding | Ответ только по retrieved items; post-process валидация ID |
| Context overflow | Обрезка history + summarization при превышении бюджета |
| Rate limiting | ≤20 req/min per session; ≤2000 токенов per LLM call |
| Output schema validation | Ответ LLM парсится в ожидаемую структуру; при ошибке — retry |

### Stop Conditions агента

- Получен ответ с валидными product ID → завершить
- 2 неудачных retry LLM → вернуть fallback-сообщение пользователю
- Clarification pending → остановить retrieval до ответа пользователя
- Budget exceeded → summarize history, продолжить

---

## 8. Технические и операционные ограничения

### Latency Budget (p95 = 5 с)

| Шаг | Ожидаемая латентность |
|---|---|
| Intent Analysis (LLM) | ~500 мс |
| Product Retrieval | ~100–200 мс |
| Reranking | ~200–400 мс |
| External APIs (параллельно) | ~500–1000 мс |
| Response Generation (LLM) | ~1000–2000 мс |
| **Итого p50** | **~2.5 с** |
| **Итого p95** | **≤5 с** |

### Cost Ограничения (PoC)

| Ресурс | Ограничение |
|---|---|
| LLM токены | ≤2000 токенов per request; бюджет мониторится |
| External API calls | ≤3 per recommendation request |
| Vector DB | FAISS (local, бесплатно); Qdrant cloud — при масштабировании |

### Reliability

| Параметр | Цель |
|---|---|
| Uptime (PoC) | ≥95% |
| LLM API failover | ≤2 с на переключение на fallback LLM |
| Data freshness (price/stock) | обновление через API при каждом запросе; cache TTL = 5 мин |