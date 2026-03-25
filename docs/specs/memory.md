# Memory / Context

## Session State

Каждая сессия идентифицируется анонимным UUID (`session_id`). Состояние хранится в Redis.

```python
SessionState = {
    "session_id": str,           # UUID, анонимный
    "dialog_history": list[msg], # последние N сообщений
    "user_preferences": dict,    # бюджет, категории, фильтры
    "last_recommendations": list, # ID последних рекомендованных товаров
    "clarification_pending": bool # ждём ответа на уточняющий вопрос
}
```

---

## Memory Policy

| Тип данных | TTL | При истечении |
|---|---|---|
| Dialog history | 30 мин | Удаление; сессия завершена |
| User preferences | 30 мин | Удаление (production: может быть persistent) |
| Last recommendations | 30 мин | Удаление |
| clarification_pending | 30 мин | Сброс в false; новый запрос начинается чисто |

---

## Context Budget

Бюджет токенов, передаваемых в LLM за один вызов:

| Слот | Лимит | Приоритет при переполнении |
|---|---|---|
| System prompt | 800 токенов (фиксированный) | Не обрезается |
| Dialog history | ≤ 2000 токенов | Обрезается с начала (oldest first) |
| Retrieved items | ≤ 1500 токенов (5 карточек) | Обрезается по количеству |
| User message | ≤ 500 токенов | Обрезается с конца |
| **Итого** | **≤ 4800 токенов** | |

**Summarization:** если история превышает 2000 токенов — вызвать LLM для краткого резюме (отдельный call, ≤ 200 токенов), заменить историю резюме.

---

## Context Assembly

```python
def build_context(session: SessionState, items: list, user_msg: str) -> Prompt:
    history = truncate(session.dialog_history, max_tokens=2000)
    item_cards = format_items(items[:5], max_tokens=1500)
    msg = truncate(user_msg, max_tokens=500)

    return Prompt(
        system=SYSTEM_PROMPT,
        messages=history + [{"role": "user", "content": item_cards + "\n" + msg}]
    )
```

---

## Ограничения

- Нет персистентной памяти между сессиями в PoC (каждая сессия начинается чисто)
- Redis — single point of failure; при недоступности — stateless режим
- Summarization добавляет ~500–1000 мс latency при срабатывании
- Preferences хранятся как flat dict; сложные предпочтения (иерархические) — outside scope PoC