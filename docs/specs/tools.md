# Tools / APIs 

## Архитектура Tool Layer

Все инструменты вызываются через единый `ToolDispatcher`, который управляет параллельным выполнением, таймаутами, retry и fallback.

```python
results = await asyncio.gather(
    price_api.get(product_ids),
    availability_api.get(product_ids),
    promotion_api.get(product_ids),
    return_exceptions=True
)
```

---

## Price API

| Параметр | Значение |
|---|---|
| Контракт | `GET /price?product_id={id}` |
| Response | `{product_id, price, currency, updated_at}` |
| Timeout | 1000 мс |
| Retry | 1 раз при сетевом таймауте; нет при 4xx |
| Cache TTL | 5 мин |
| Fallback | cached price с меткой `"price_stale": true` |
| Side effects | нет |
| Auth | API key в `X-API-Key` header |
| Errors | 404 → товар не найден; 429 → rate limit, ждать retry-after; 5xx → использовать cache |

---

## Availability API

| Параметр | Значение |
|---|---|
| Контракт | `GET /availability?product_id={id}` |
| Response | `{product_id, in_stock, quantity, eta_days}` |
| Timeout | 1000 мс |
| Retry | 1 раз при таймауте |
| Cache TTL | 5 мин |
| Fallback | `{in_stock: unknown}` с меткой в ответе пользователю |
| Side effects | нет |
| Auth | API key в `X-API-Key` header |

---

## Promotion API

| Параметр | Значение |
|---|---|
| Контракт | `GET /promotions?product_id={id}` |
| Response | `{product_id, discount_pct, promo_label, expires_at}` |
| Timeout | 800 мс |
| Retry | нет (некритичное; graceful degradation) |
| Cache TTL | 2 мин (акции меняются чаще) |
| Fallback | не показывать скидку; не блокировать ответ |
| Side effects | нет |
| Auth | API key в `X-API-Key` header |

---

## Защита Tool Layer

| Мера | Реализация |
|---|---|
| Rate limiting | ≤ 3 API calls per recommendation request |
| Circuit breaker | После 5 последовательных ошибок — disable + использовать cache on всех запросах |
| Timeout chain | Суммарный timeout tool layer = 1200 мс (= max individual + buffer) |
| Secret management | API keys в env vars; не в коде; ротация через secrets manager |
| No side effects | Все вызовы — GET only; никаких записей или транзакций |

---

## LLM Tool (Orchestrator call)

| Параметр | Значение |
|---|---|
| Provider | OpenAI GPT-4o (primary), open-source LLM (fallback) |
| Timeout | 8000 мс |
| Retry | 2 раза при 5xx или таймауте |
| Circuit breaker | После 3 ошибок — переключить на fallback provider |
| Max tokens (input) | 4800 токенов |
| Max tokens (output) | 1000 токенов |
| Temperature | 0.3 (детерминированные рекомендации) |
| Fallback | open-source LLM локально (Ollama / vLLM) |