# UML-диаграмма последовательности

> Поведенческая диаграмма: полный путь комментария от публикации до вынесения решения по модерации.

## Сценарий 1 — Пограничный случай: комментарий помещается в очередь (score=0.90)

```mermaid
sequenceDiagram
    actor User as 👤 Пользователь
    participant FE as Web App
    participant GW as API Gateway
    participant CIS as Comment Ingestion Service
    participant KR as Kafka (comments.raw)
    participant SIS as Sentiment Inference Service
    participant RD as Redis Cache
    participant ONNX as ONNX Runtime
    participant PG as PostgreSQL
    participant KS as Kafka (comments.scored)
    participant MODS as Moderation Service
    participant SN as Social Network API

    User->>FE: Публикует комментарий
    FE->>GW: POST /api/v1/comments\n{user_id, content_id, text}
    GW->>CIS: Маршрутизация запроса
    CIS->>CIS: Валидация входных данных
    CIS->>PG: INSERT comment\n(status='processing')
    CIS->>KR: Produce {comment_id, text, created_at}
    CIS-->>GW: 202 Accepted {comment_id}
    GW-->>FE: 202 Accepted
    FE-->>User: Комментарий отправлен ✓

    Note over KR,SIS: Асинхронная обработка (~50–100 мс)

    KR->>SIS: Consume message {comment_id, text}
    SIS->>RD: GET cache(sha256(text))
    RD-->>SIS: MISS (нет в кэше)
    SIS->>ONNX: predict(tokenized_text)
    ONNX-->>SIS: scores {positive:0.02, neutral:0.03,\nnegative:0.05, toxic:0.90}
    SIS->>RD: SET cache(sha256(text), scores, TTL=1h)
    SIS->>PG: INSERT prediction\n{comment_id, label='toxic',\nscore_toxic=0.90, model_version}
    SIS->>KS: Produce {comment_id, label='toxic',\nscore_toxic=0.90}

    Note over KS,MODS: Применение бизнес-правил

    KS->>MODS: Consume scored {comment_id, score_toxic=0.90}
    MODS->>MODS: 0.90 >= 0.92? НЕТ (не автоблок)\n0.90 >= 0.70? ДА → pending_review

    Note over MODS: Пограничный случай: score ниже порога автоблока

    MODS->>PG: UPDATE comment SET status='pending_review'
    MODS->>PG: INSERT moderation_action\n(type='auto_queue', is_automatic=true)

    Note over User,SN: Комментарий ждёт ручной проверки модератором
```

## Сценарий 2 — Явно токсичный (score ≥ 0.92), автоблок

```mermaid
sequenceDiagram
    participant KS as Kafka (comments.scored)
    participant MODS as Moderation Service
    participant PG as PostgreSQL
    participant SN as Social Network API
    participant DASH as Admin Dashboard

    KS->>MODS: {comment_id, score_toxic=0.96}
    MODS->>MODS: 0.96 ≥ 0.92 → АВТОБЛОК
    MODS->>SN: POST /comments/{id}/hide
    SN-->>MODS: 200 OK
    MODS->>PG: UPDATE comment SET status='auto_blocked'
    MODS->>PG: INSERT moderation_action\n(type='auto_block', is_automatic=true)
    MODS->>DASH: WebSocket push: новый авто-заблокированный комментарий

    Note over DASH: Модератор видит в дашборде (для аудита/апелляций)
```

## Сценарий 3 — Ручная проверка модератором

```mermaid
sequenceDiagram
    actor Mod as 🧑‍💼 Модератор
    participant DASH as Admin Dashboard
    participant GW as API Gateway
    participant MODS as Moderation Service
    participant PG as PostgreSQL
    participant SN as Social Network API

    Mod->>DASH: Открывает очередь pending_review
    DASH->>GW: GET /moderation/queue?status=pending_review
    GW->>MODS: Запрос очереди
    MODS->>PG: SELECT comments WHERE status='pending_review' ORDER BY score_toxic DESC
    PG-->>MODS: [list of comments with predictions]
    MODS-->>DASH: JSON: список комментариев + метки + scores

    Mod->>DASH: Принимает решение по комментарию: БЛОКИРОВАТЬ
    DASH->>GW: POST /moderation/actions\n{comment_id, action='block', reason}
    GW->>MODS: Обработка действия
    MODS->>SN: POST /comments/{id}/hide
    SN-->>MODS: 200 OK
    MODS->>PG: UPDATE comment SET status='deleted'
    MODS->>PG: INSERT moderation_action\n(moderator_id, type='manual_block', is_automatic=false)
    MODS-->>DASH: 200 OK: Действие выполнено
    DASH-->>Mod: ✓ Комментарий заблокирован
```

## Сценарий 4 — Кэш-хит (повторный одинаковый текст)

```mermaid
sequenceDiagram
    participant KR as Kafka (comments.raw)
    participant SIS as Sentiment Inference Service
    participant RD as Redis Cache
    participant PG as PostgreSQL
    participant KS as Kafka (comments.scored)

    KR->>SIS: Consume {comment_id=9002, text="[spam текст]"}
    SIS->>RD: GET cache(sha256("[spam текст]"))
    RD-->>SIS: HIT {label='toxic', score_toxic=0.96}
    Note over SIS: ONNX НЕ вызывается → экономия ~50 мс
    SIS->>PG: INSERT prediction (из кэша, inference_time_ms=2)
    SIS->>KS: Produce to comments.scored
```
