# Orchestrator

## Роль

LLM Orchestrator — ядро агента. Управляет всем execution flow: от входящего запроса до финального ответа. Использует LLM для intent analysis, выбора инструментов и генерации ответа.

---

## Шаги выполнения

```
1. receive_input(user_msg, session_id)
2. sanitize(user_msg) → clean_msg
3. intent = analyze_intent(clean_msg, history)  # LLM call #1
4. if intent.confidence < threshold:
       return clarification_question(intent)     # stop, wait for user
5. candidates = retrieve(intent.query)           # retriever
6. ranked = rerank(candidates, intent.query)     # reranker
7. if not ranked:
       return fallback_no_results()              # stop
8. enriched = enrich(ranked)                     # API layer (async)
9. context = build_context(session, enriched, clean_msg)
10. response = llm_generate(context)             # LLM call #2
11. validated = validate(response, ranked)       # ID check
12. update_memory(session, user_msg, response)
13. return validated
```

---

## Правила переходов

| Состояние | Условие | Переход |
|---|---|---|
| intent analysis | confidence < 0.6 | → clarification |
| clarification | получен ответ пользователя | → intent analysis (повторно) |
| retrieval | результатов 0 | → fallback_no_results |
| LLM call | 5xx / timeout (retry 1) | → повторить |
| LLM call | 5xx / timeout (retry 2) | → fallback provider (OSS LLM) |
| LLM call | fallback тоже упал | → return generic error message |
| validation | ID не в каталоге | → удалить из ответа, продолжить |
| validation | 0 валидных ID | → return fallback_no_results |

---

## Stop Conditions

| Условие | Действие |
|---|---|
| Получен валидный ответ с ≥1 товаром | Вернуть ответ пользователю |
| clarification_pending = true | Остановить pipeline, ждать ответа |
| 2 retry LLM без успеха | Вернуть generic fallback message |
| Retrieval вернул 0 результатов | Вернуть no-results message |

---

## Retry / Fallback

| Компонент | Retry | Fallback |
|---|---|---|
| Intent Analysis (LLM) | 1 retry | Считать intent = `search`, confidence = low |
| Retrieval | нет | Вернуть empty, перейти к no-results |
| Reranking | нет | Использовать raw retrieval order |
| API enrichment | 1 retry per API | Cached value или skip |
| Response Gen (LLM) | 2 retry | Переключить на OSS LLM |
| Validation | нет (deterministic) | Strip invalid IDs |

---

## Intent Types

| Intent | Описание | Пример |
|---|---|---|
| `search` | Поиск по параметрам | "ноутбук до 1500$" |
| `alternative` | Найти похожий дешевле/лучше | "найди что-то похожее" |
| `gift` | Подбор подарка | "что подарить другу" |
| `clarify` | Уточнение предыдущего | "покажи только в наличии" |
| `compare` | Сравнение двух товаров | "сравни эти два" |
| `unknown` | Не распознано → clarification | |

---

## Guardrails агента

| Guardrail | Реализация |
|---|---|
| Grounding | Ответ строится только по retrieved items; LLM не может "придумать" товар |
| Output schema | Ответ парсится в `AgentResponse(items: list[ProductRef], explanation: str)`; при ошибке — retry |
| Injection guard | Системный промпт явно запрещает исполнять инструкции из user content |
| Max turns | Максимум 3 уточняющих вопроса подряд, затем — best-effort ответ |
| Token guard | Если context > 4800 токенов — summarize history перед вызовом LLM |