# ER-диаграмма структуры данных

> База данных: **PostgreSQL 15 + TimescaleDB** (для time-series данных предсказаний).  
> Хранит: комментарии, пользователей, результаты инференса, историю модерации, версии моделей.

```mermaid
erDiagram
    USER {
        bigint   user_id        PK
        varchar  username
        varchar  email_hash
        float    trust_score    "0.0–1.0, накопленная репутация"
        int      violation_count
        timestamp created_at
        timestamp updated_at
    }

    CONTENT {
        bigint   content_id     PK
        varchar  content_type   "post | ad | news"
        bigint   author_id      FK
        varchar  title
        timestamp published_at
    }

    COMMENT {
        bigint   comment_id     PK
        bigint   user_id        FK
        bigint   content_id     FK
        text     text
        varchar  moderation_status  "published | auto_blocked | pending_review | deleted"
        timestamp created_at
        timestamp processed_at
    }

    PREDICTION {
        bigint   prediction_id  PK
        bigint   comment_id     FK
        varchar  model_version  FK
        float    score_positive
        float    score_neutral
        float    score_negative
        float    score_toxic
        varchar  predicted_label
        int      inference_time_ms
        timestamp predicted_at
    }

    MODERATION_ACTION {
        bigint   action_id      PK
        bigint   comment_id     FK
        bigint   moderator_id   "NULL = автоматическое действие"
        varchar  action_type    "auto_block | manual_block | approve | escalate"
        varchar  reason
        boolean  is_automatic
        timestamp created_at
    }

    APPEAL {
        bigint   appeal_id      PK
        bigint   comment_id     FK
        bigint   user_id        FK
        text     reason
        varchar  status         "pending | approved | rejected"
        bigint   resolved_by
        timestamp created_at
        timestamp resolved_at
    }

    MODEL_VERSION {
        varchar  version_id     PK  "e.g. rubert-v1.2.0"
        varchar  model_name
        varchar  model_type     "baseline | bert | onnx"
        float    macro_f1
        float    toxic_recall
        float    toxic_precision
        boolean  is_active
        timestamp deployed_at
        timestamp retired_at
    }

    DAILY_STATS {
        bigint   stat_id        PK
        date     stat_date
        bigint   content_id     FK
        int      total_comments
        int      positive_count
        int      neutral_count
        int      negative_count
        int      toxic_count
        float    avg_toxic_score
    }

    USER        ||--o{ COMMENT          : "пишет"
    CONTENT     ||--o{ COMMENT          : "содержит"
    COMMENT     ||--o{ PREDICTION       : "имеет предсказание"
    COMMENT     ||--o{ MODERATION_ACTION: "получает действие"
    COMMENT     ||--o{ APPEAL           : "оспаривается"
    USER        ||--o{ APPEAL           : "подаёт апелляцию"
    MODEL_VERSION ||--o{ PREDICTION     : "производит"
    CONTENT     ||--o{ DAILY_STATS      : "агрегируется в"
```

## Описание ключевых сущностей

### `COMMENT` — центральная сущность
Хранит текст и текущий статус каждого комментария. Поле `moderation_status` обновляется в ходе обработки.

### `PREDICTION` — результаты инференса
Каждый комментарий имеет ровно одно актуальное предсказание (при переинференсе создаётся новая запись). Хранение всех предсказаний с `model_version` обеспечивает аудит для регулятора.

### `MODEL_VERSION` — реестр моделей
Связан с MLflow Model Registry. Только одна версия имеет `is_active = true`. Откат — изменение флага без удаления истории.

### `DAILY_STATS` — агрегат для дашборда
Партиционируется по `stat_date` (TimescaleDB hypertable) для быстрой выборки аналитики без нагрузки на оперативные таблицы.

## Индексы

```sql
-- Быстрый поиск по статусу (очередь модерации)
CREATE INDEX idx_comment_status ON comment(moderation_status, created_at DESC);

-- Предсказания по времени (TimescaleDB)
SELECT create_hypertable('prediction', 'predicted_at');

-- Поиск предсказаний к конкретному комментарию
CREATE INDEX idx_prediction_comment ON prediction(comment_id);

-- Аналитика по контенту
CREATE INDEX idx_daily_stats_date ON daily_stats(stat_date DESC, content_id);
```
