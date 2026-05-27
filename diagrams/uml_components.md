# UML-диаграмма компонентов

> Структурная диаграмма компонентов системы анализа тональности комментариев.  
> Показаны компоненты, их интерфейсы и зависимости.

```mermaid
flowchart LR
    subgraph ClientLayer["«layer»\nКлиентский уровень"]
        direction TB
        WEB["«component»\nWeb/Mobile App\n─────────────\n+ sendComment()\n+ viewFeed()"]
        ADM["«component»\nAdmin Dashboard\n─────────────\n+ reviewQueue()\n+ viewAnalytics()\n+ resolveAppeal()"]
    end

    subgraph GatewayLayer["«layer»\nAPI-шлюз"]
        GW["«component»\nAPI Gateway\n(nginx)\n─────────────\n+ route()\n+ rateLimit()\n+ authenticate()"]
    end

    subgraph CoreServices["«layer»\nОсновные сервисы"]
        direction TB
        CIS["«component»\nComment Ingestion Service\n─────────────\n+ receiveComment()\n+ validateInput()\n+ publishToKafka()"]

        SIS["«component»\nSentiment Inference Service\n─────────────\n+ consumeFromKafka()\n+ checkCache()\n+ runInference()\n+ savePrediction()\n+ publishResult()"]

        MODS["«component»\nModeration Service\n─────────────\n+ applyThreshold()\n+ updateStatus()\n+ assignToQueue()\n+ processAppeal()"]
    end

    subgraph MLLayer["«layer»\nML-платформа"]
        direction TB
        ONNX["«component»\nONNX Runtime\n─────────────\n+ loadModel()\n+ predict(text)\n+ getScores()"]

        MLFLOW["«component»\nMLflow\nModel Registry\n─────────────\n+ registerModel()\n+ getActiveVersion()\n+ transitionStage()"]

        TRAINER["«component»\nTraining Orchestrator\n(Airflow)\n─────────────\n+ scheduleTrain()\n+ evaluateModel()\n+ promoteModel()"]
    end

    subgraph InfraLayer["«layer»\nИнфраструктура"]
        direction TB
        KAFKA["«component»\nApache Kafka\n─────────────\nTopics:\n• comments.raw\n• comments.scored\n• moderation.actions"]

        PG["«component»\nPostgreSQL\n+ TimescaleDB\n─────────────\n• USER\n• COMMENT\n• PREDICTION\n• MODERATION_ACTION\n• MODEL_VERSION"]

        REDIS["«component»\nRedis Cache\n─────────────\n+ get(key)\n+ set(key, val, ttl)\n+ del(key)"]

        S3["«component»\nObject Storage\n(S3-compatible)\n─────────────\n+ uploadModel()\n+ downloadModel()\n+ listVersions()"]
    end

    subgraph MonitoringLayer["«layer»\nМониторинг"]
        direction TB
        PROM["«component»\nPrometheus\n─────────────\n+ scrapeMetrics()\n+ storeTimeSeries()"]

        GRAF["«component»\nGrafana\n─────────────\n+ renderDashboard()\n+ sendAlert()"]

        DRIFT["«component»\nEvidentlyAI\n─────────────\n+ calcPSI()\n+ detectDrift()\n+ generateReport()"]
    end

    %% Client → Gateway
    WEB -->|"HTTPS REST"| GW
    ADM -->|"HTTPS REST + JWT"| GW

    %% Gateway → Services
    GW -->|"REST POST /comments"| CIS
    GW -->|"REST GET/POST /moderation"| MODS

    %% Ingestion → Kafka
    CIS -->|"Produce: comments.raw"| KAFKA

    %% Kafka → Inference
    KAFKA -->|"Consume: comments.raw"| SIS

    %% Inference internals
    SIS -->|"predict()"| ONNX
    SIS -->|"cache lookup/store"| REDIS
    SIS -->|"INSERT"| PG
    SIS -->|"Produce: comments.scored"| KAFKA

    %% Kafka → Moderation
    KAFKA -->|"Consume: comments.scored"| MODS
    MODS -->|"UPDATE"| PG

    %% ML Platform
    MLFLOW -->|"load active model"| ONNX
    MLFLOW -->|"store artifacts"| S3
    TRAINER -->|"register model"| MLFLOW
    TRAINER -->|"read training data"| S3

    %% Monitoring
    SIS -->|"metrics"| PROM
    MODS -->|"metrics"| PROM
    CIS -->|"metrics"| PROM
    PG -->|"data samples"| DRIFT
    PROM --> GRAF
    DRIFT --> GRAF
```

## Описание интерфейсов

### Comment Ingestion Service
| Интерфейс | Тип | Описание |
|---|---|---|
| `POST /api/v1/comments` | REST (входящий) | Приём нового комментария от фронтенда |
| Kafka Producer | Message Bus (исходящий) | Публикация в `comments.raw` |

### Sentiment Inference Service
| Интерфейс | Тип | Описание |
|---|---|---|
| Kafka Consumer Group | Message Bus (входящий) | Подписка на `comments.raw` |
| `predict(text) → scores` | Internal | Вызов ONNX Runtime |
| `GET/SET cache` | Redis | Кэш по хэшу текста |
| `POST /predictions` | PostgreSQL | Запись результата |
| Kafka Producer | Message Bus (исходящий) | Публикация в `comments.scored` |

### Moderation Service
| Интерфейс | Тип | Описание |
|---|---|---|
| Kafka Consumer Group | Message Bus (входящий) | Подписка на `comments.scored` |
| `POST /comments/{id}/hide` | REST (исходящий) | Скрытие комментария в соцсети |
| `GET /queue` | REST (входящий от Dashboard) | Получение очереди на проверку |
| `POST /actions` | REST (входящий от Dashboard) | Действие модератора |
