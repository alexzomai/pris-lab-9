# Диаграмма архитектуры системы

> Архитектура распределённой ML-системы анализа тональности комментариев.  
> Показано: как данные проходят путь от публикации комментария до вынесения решения по модерации.

## Общая схема (C4 Level 2 — Container Diagram)

```mermaid
flowchart TB
    subgraph External["🌐 Внешние пользователи"]
        U([👤 Пользователь\nсоциальной сети])
        M([🧑‍💼 Модератор])
        A([👨‍💻 ML-инженер])
    end

    subgraph Frontend["Фронтенд"]
        FE[Social Network\nWeb/Mobile App]
        DASH[Admin Dashboard\nReact SPA]
    end

    subgraph APILayer["API Gateway Layer"]
        GW[API Gateway\nnginx + rate limiting]
    end

    subgraph Services["Микросервисы (Kubernetes)"]
        CIS[Comment Ingestion\nService\nGo]
        SIS[Sentiment Inference\nService\nFastAPI + ONNX Runtime]
        MODS[Moderation\nService\nPython + FastAPI]
        TRNS[Training\nOrchestrator\nAirflow DAGs]
    end

    subgraph MessageBus["Шина событий"]
        KAFKA[Apache Kafka 3.x\n─────────────────\ntopics:\n• comments.raw\n• comments.scored\n• moderation.actions\n• model.events]
    end

    subgraph Storage["Хранилища данных (распределённые)"]
        PG[(PostgreSQL 15\n+ TimescaleDB\n─────────────────\nКомментарии\nПредсказания\nИстория модерации)]
        REDIS[(Redis 7\n─────────────────\nКэш предсказаний\nTTL: 1 час)]
        S3[(Object Storage\nS3-совместимый\n─────────────────\nONNX-модели\nОбучающие датасеты\nEDA-артефакты)]
    end

    subgraph MLPlatform["ML-платформа"]
        MLFLOW[MLflow\n─────────────────\nModel Registry\nExperiment Tracking]
        JUPYTER[JupyterHub\nEDA / Research]
    end

    subgraph Monitoring["Мониторинг и алёртинг"]
        PROM[Prometheus\nМетрики системы]
        GRAF[Grafana\nДашборды + Алёрты]
        DM[Data Drift\nMonitor\nEvidentlyAI]
    end

    %% Пользовательский поток
    U --> FE --> GW --> CIS
    CIS -->|"JSON: {comment_id, text, ...}"| KAFKA
    KAFKA -->|"topics: comments.raw"| SIS
    SIS -->|"Lookup / Write"| REDIS
    SIS -->|"topics: comments.scored\n{label, scores, model_version}"| KAFKA
    KAFKA -->|"topics: comments.scored"| MODS
    MODS -->|"UPDATE status"| PG
    MODS -->|"REST: скрыть / опубликовать"| GW

    %% Модератор
    M --> DASH --> GW --> MODS
    MODS -->|"Очередь pending_review"| PG

    %% Хранение предсказаний
    SIS -->|"INSERT prediction"| PG

    %% ML-платформа
    TRNS -->|"Загрузка обучающих данных"| S3
    TRNS -->|"Логирование метрик"| MLFLOW
    MLFLOW -->|"Регистрация ONNX-модели"| S3
    MLFLOW -->|"Загрузка активной модели"| SIS
    A --> JUPYTER --> MLFLOW

    %% Мониторинг
    SIS --> PROM
    MODS --> PROM
    CIS --> PROM
    PG --> DM
    PROM --> GRAF
    DM --> GRAF

    style External fill:#f8f9fa,stroke:#6c757d
    style Frontend fill:#e8f4f8,stroke:#0077b6
    style APILayer fill:#fff3cd,stroke:#ffc107
    style Services fill:#d4edda,stroke:#28a745
    style MessageBus fill:#fce4ec,stroke:#c62828
    style Storage fill:#e8eaf6,stroke:#3949ab
    style MLPlatform fill:#f3e5f5,stroke:#7b1fa2
    style Monitoring fill:#e0f2f1,stroke:#00695c
```

## Почему данные хранятся распределённо

| Хранилище | Тип данных | Причина распределения |
|---|---|---|
| **Kafka** | Горячие события (TTL 24 ч) | Буферизация пиков нагрузки; декаплинг producer/consumer; replay при сбоях |
| **Redis** | Кэш предсказаний (TTL 1 ч) | Дедупликация одинаковых текстов (спам); снижение latency при повторах |
| **PostgreSQL** | Структурированные данные | Долгосрочное хранение; сложные SQL-запросы; аудит для регулятора |
| **Object Storage (S3)** | Артефакты моделей, датасеты | Дешёвое хранение больших бинарных файлов; версионирование моделей |

> Распределённый характер хранения обусловлен разными требованиями к **скорости доступа**, **объёму**, **стоимости** и **retention policy** у разных типов данных.

## Потоки данных

```mermaid
sequenceDiagram
    participant K as Kafka
    participant SIS as Sentiment\nInference Service
    participant R as Redis
    participant PG as PostgreSQL

    K->>SIS: comment (comment_id, text)
    SIS->>R: GET cache(hash(text))
    alt cache hit
        R-->>SIS: cached prediction
    else cache miss
        SIS->>SIS: ONNX inference (~50ms)
        SIS->>R: SET cache(hash(text), prediction, TTL=1h)
        SIS->>PG: INSERT INTO prediction(...)
    end
    SIS->>K: publish to comments.scored
```
