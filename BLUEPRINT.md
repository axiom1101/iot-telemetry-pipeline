# АРХИТЕКТУРНЫЙ ДОКУМЕНТ (ARCHITECTURE DECISION RECORD)
**Проект:** `iot-telemetry-pipeline`  
**Версия:** 1.0 (MVP)  
**Статус:** Accepted / In Development  
**Автор:** Олег Глушков  
**Дата:** 2024-05-24  

---

## 1. CONTEXT & PROBLEM STATEMENT (Контекст и Проблема)

**Цель:** Создать работающий энд-ту-энд прототип пайплайна телеметрии IoT для демонстрации навыков **Junior Backend/Platform Engineer** на собеседованиях.

**Бизнес-кейс (Симуляция):**  
Имеется парк датчиков (температура, влажность, вибрация) на удаленных объектах. Необходимо собирать телеметрию в реальном времени, хранить историю, визуализировать текущее состояние и автоматически детектировать аномалии (перегрев, заклинивание, утечка) для алертинга.

**Ограничения (Constraints):**
*   **Стек:** Python 3.11+, FastAPI, PostgreSQL/TimescaleDB, MQTT, Docker.
*   **Ресурсы:** Локальная машина / дешевый VPS (2 vCPU, 4GB RAM).
*   **Срок:** MVP за 2-3 вечера.
*   **Нет:** Kubernetes, Kafka, Airflow, облачных управляемых сервисов (AWS IoT Core, Timestream), сложных CI/CD.

---

## 2. ARCHITECTURE OVERVIEW (Архитектура: High Level)

```mermaid
flowchart LR
    subgraph Edge ["Edge / Device Layer (Simulated)"]
        Sim[Device Simulator\nPython/asyncio\nMQTT Client]
    end

    subgraph Network ["Transport"]
        MQTT[(MQTT Broker\nMosquitto 2.x\nPort 1883/8883)]
    end

    subgraph Platform ["Platform / Cloud Layer (Docker Compose)"]
        API[Ingestion API\nFastAPI + Pydantic\nPort 8000]
        DB[(TimescaleDB\nPostgreSQL 16 + TimescaleDB ext\nPort 5432)]
        ML[Anomaly Detector\nPython / scikit-learn\nBatch/Stream]
        Mon[Monitoring & Viz\nGrafana + Prometheus\nPort 3000 / 9090]
    end

    Sim -->|Telemetry JSON\nTopic: sensors/+| MQTT
    MQTT -->|Subscribes\nsensors/#| API
    API -->|INSERT\nHypertables| DB
    DB -->|SELECT\nTime-range| Mon
    DB -->|READ\nWindow| ML
    ML -->|Anomaly Events\nTopic: alerts| MQTT
    MQTT -->|Alerts| Mon
```

---

## 3. COMPONENT SPECIFICATIONS (Спецификация компонентов)

### 3.1. Device Simulator (`simulator/`)
*   **Роль:** Эмуляция N устройств (без железа).
*   **Стек:** `asyncio`, `aiomqtt` / `paho-mqtt`, `pydantic`.
*   **Логика:**
    *   Генерирует реалистичные временные ряды (синус + шум + тренд + случайные спайки).
    *   Публикует в топик: `sensors/{device_id}/telemetry` (QoS 1).
    *   Payload: JSON (`ts`, `device_id`, `metrics: {temp, hum, vib}`, `meta`).
    *   Конфигурация через `.env` / `config.yaml` (кол-во устройств, частота, профиль шума).

### 3.2. MQTT Broker (`mosquitto`)
*   **Образ:** `eclipse-mosquitto:2.0`
*   **Конфиг:** `mosquitto.conf` (persistence, websockets port 9001 для отладки, ACL анонимный доступ для MVP).
*   **Порты:** 1883 (MQTT), 9001 (WebSockets).

### 3.3. Ingestion API (`ingestion/`) — **Core Backend**
*   **Фреймворк:** **FastAPI** (async, OpenAPI, Pydantic validation).
*   **Эндпоинты:**
    *   `POST /api/v1/telemetry` — Прием батча телеметрии (от MQTT subscriber'а или HTTP устройств).
    *   `GET /api/v1/devices/{id}/telemetry` — Чтение последних N точек (для Grafana/проверки).
    *   `GET /health` — Liveness/Readiness probes.
*   **Внутренняя логика:**
    *   **MQTT Subscriber:** Фоновая задача (`asyncio.create_task`) на старте приложения (`lifespan`), слушает `sensors/#`, парсит, валидирует (Pydantic), батчит (накопление 100 шт или 1 сек) → `INSERT` в TimescaleDB.
    *   **БД:** `asyncpg` (пул коннектов) + `SQLAlchemy Core` (без ORM оверхеда) или чистый SQL.
    *   **Схема БД:** Гипертаблица `telemetry` (time, device_id, metrics JSONB, tags JSONB).
    *   **Обработка ошибок:** Dead Letter Queue (логирование в файл/БД невалидных сообщений), ретраи с backoff.

### 3.4. TimescaleDB (`timescaledb`)
*   **Образ:** `timescale/timescaledb:latest-pg16`
*   **Схема:**
    ```sql
    CREATE TABLE telemetry (
        time        TIMESTAMPTZ       NOT NULL,
        device_id   TEXT              NOT NULL,
        metrics     JSONB             NOT NULL, -- {temp: 23.4, hum: 55, vib: 0.1}
        tags        JSONB             DEFAULT '{}'::jsonb, -- {location: "zone_a", firmware: "1.0.2"}
        meta        JSONB             DEFAULT '{}'::jsonb  -- raw payload backup
    );
    SELECT create_hypertable('telemetry', 'time', chunk_time_interval => INTERVAL '1 day');
    CREATE INDEX idx_telemetry_device_time ON telemetry (device_id, time DESC);
    -- Continuous Aggregates для дашбордов (1m, 1h buckets)
    CREATE MATERIALIZED VIEW telemetry_1m
    WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 minute', time) AS bucket, device_id,
           avg((metrics->>'temp')::float) AS avg_temp,
           max((metrics->>'temp')::float) AS max_temp,
           count(*) as cnt
    FROM telemetry GROUP BY bucket, device_id;
    ```
*   **Retention Policy:** `add_retention_policy('telemetry', INTERVAL '30 days')` (для демо).

### 3.5. Anomaly Detector (`ml/`) — **AI Hook**
*   **Подход:** Batch-обработка (cron/APScheduler каждые 5 мин) или Stream (читает из MQTT `sensors/#`).
*   **Алгоритм (MVP):** **Isolation Forest** (`sklearn.ensemble.IsolationForest`) или простой **Z-score / Rolling Z-score** на окне 100 точек.
*   **Фичи:** `temp`, `hum`, `vib` (последние N точек на устройство).
*   **Выход:** Публикация алерта в MQTT `alerts/{device_id}`:
    ```json
    {"ts": "...", "device_id": "sensor-1", "metric": "temp", "value": 85.2, "score": -0.8, "type": "spike"}
    ```
*   **Стек:** `scikit-learn`, `pandas`, `joblib` (сохранение модели).

### 3.5. Monitoring & Visualization
*   **Prometheus:** Скрапит `/metrics` с FastAPI (`prometheus-fastapi-instrumentator`) + `node-exporter` + `postgres-exporter`.
*   **Grafana:** Дашборды (JSON provisioning):
    1.  **Real-time Telemetry:** Time series (temp/hum/vib) на устройство.
    2.  **System Health:** API latency (p50/p95/p99), Error rate, Throughput (msg/sec), DB size, MQTT connected clients.
    3.  **Anomalies:** Таблица/график алертов из топика `alerts/+`.

---

## 4. DATA FLOW & CONTRACTS (Поток данных и Контракты)

### 4.1. MQTT Topic Structure
| Topic | Direction | Payload Schema (JSON) |
| :--- | :--- | :--- |
| `sensors/{device_id}/telemetry` | Device → Platform | `{"ts": "ISO8601", "device_id": "str", "metrics": {"temp": float, "hum": float, "vib": float}, "tags": {"loc": "str"}}` |
| `sensors/{device_id}/command` | Platform → Device | `{"cmd": "reboot|calibrate", "params": {...}, "req_id": "uuid"}` |
| `alerts/{device_id}` | ML → Platform | `{"ts": "...", "device_id": "...", "metric": "temp", "value": 99.9, "anomaly_score": -0.9, "rule": "isolation_forest"}` |
| `platform/status` | Platform → All | `{"service": "ingestion-api", "status": "healthy", "uptime": 3600, "msg_processed": 12345}` |

### 4.2. API Contract (OpenAPI 3.0 — генерируется FastAPI автоматически)
*   `POST /api/v1/telemetry` — Body: `TelemetryBatch` (List[TelemetryPoint]).
*   `GET /api/v1/devices/{device_id}/telemetry?from=...&to=...&limit=1000` — Response: `List[TelemetryPoint]`.

---

## 5. INFRASTRUCTURE & DEPLOYMENT (Docker Compose)

```yaml
# docker-compose.yml (Structure)
services:
  mosquitto:
    image: eclipse-mosquitto:2
    volumes: ["./mosquitto/config:/mosquitto/config", "./mosquitto/data:/mosquitto/data", "./mosquitto/log:/mosquitto/log"]
    ports: ["1883:1883", "9001:9001"]
    healthcheck: ["mosquicmping", "localhost"]

  timescaledb:
    image: timescale/timescaledb:latest-pg16
    environment: [POSTGRES_DB=telemetry, POSTGRES_USER=admin, POSTGRES_PASSWORD=secret]
    volumes: ["./timescaledb/data:/var/lib/postgresql/data", "./timescaledb/init.sql:/docker-entrypoint-initdb.d/init.sql"]
    ports: ["5432:5432"]
    healthcheck: ["pg_isready", "-U", "admin"]

  ingestion-api:
    build: ./ingestion
    environment: [DATABASE_URL=postgresql+asyncpg://admin:secret@timescaledb:5432/telemetry, MQTT_BROKER=mosquitto, LOG_LEVEL=INFO]
    ports: ["8000:8000"]
    depends_on: [timescaledb, mosquitto]
    command: ["uvicorn", "main:app", "--host", "0.0.0.0", "--workers", "2"]

  simulator:
    build: ./simulator
    environment: [MQTT_BROKER=mosquitto, DEVICE_COUNT=10, INTERVAL_SEC=5]
    depends_on: [mosquitto]

  anomaly-detector:
    build: ./ml
    environment: [DATABASE_URL=..., MQTT_BROKER=mosquitto, MODEL_PATH=/models/iso_forest.joblib]
    depends_on: [timescaledb, mosquitto]
    volumes: ["./ml/models:/models"]

  grafana:
    image: grafana/grafana:latest
    volumes: ["./grafana/provisioning:/etc/grafana/provisioning", "./grafana/dashboards:/var/lib/grafana/dashboards"]
    ports: ["3000:3000"]
    depends_on: [timescaledb, prometheus]

  prometheus:
    image: prom/prometheus:latest
    volumes: ["./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml"]
    ports: ["9090:9090"]
```

---

## 6. NON-FUNCTIONAL REQUIREMENTS (NFR) & QUALITY ATTRIBUTES

| Атрибут | Требование (MVP) | Реализация |
| :--- | :--- | :--- |
| **Reliability** | Не потерять сообщение при рестарте API | MQTT QoS 1 + Persistent Session + DB Transaction (Commit после батча) |
| **Scalability (Vertical)** | 10k msg/sec на 2 vCPU | Async IO (FastAPI/asyncpg), батчинг INSERT (executemany), Connection Pooling |
| **Observability** | "Что происходит?" за 10 сек | Structured Logging (JSON, `structlog`), Prometheus Metrics (Counter, Histogram), Health Checks |
| **Security (Basic)** | Нет открытых паролей в коде | `.env` + `docker secrets` / `env_file`, TLS для MQTT (self-signed для MVP), Basic Auth для Grafana/API |
| **Maintainability** | Читаемость кода | Clean Architecture (слои: `api`, `services`, `repositories`, `models`), Type Hints (mypy strict), Ruff/Black |

---

## 7. DEVELOPMENT ROADMAP (План реализации за 3 вечера)

| Этап | Задачи | Definition of Done |
| :--- | :--- | :--- |
| **Вечер 1: Инфраструктура + Данные** | 1. `docker-compose.yml` (все сервисы).<br>2. `timescaledb/init.sql` (гипертаблица, индексы, CA).<br>3. `mosquitto.conf`.<br>4. `simulator` (минимальный: 1 датчик, JSON в MQTT).<br>4. `ingestion-api`: FastAPI skeleton + lifespan (MQTT sub) + `POST /telemetry` + `INSERT` batch. | `docker compose up -d` → данные текут: Симулятор → MQTT → API → TimescaleDB. `SELECT * FROM telemetry LIMIT 10` возвращает строки. |
| **Вечер 2: API + ML + Observability** | 1. `ingestion-api`: Pydantic модели, валидация, ошибки, `GET /telemetry`, `/health`, `/metrics` (Prometheus).<br>2. `ml/anomaly_detector`: Загрузка данных из БД → обучение IsolationForest → сохранение `joblib` → инференс каждые 5 мин → публикация алертов в MQTT.<br>3. `prometheus.yml` + `grafana/provisioning` (datasources + dashboards JSON). | Grafana показывает Real-time графики. Prometheus собирает метрики API. Алерты появляются в MQTT топике `alerts/+` и на дашборде. |
| **Вечер 3: Polish & Docs** | 1. `simulator`: Конфиг (N устройств, профили шума, аномалий).<br>2. `README.md`: Архитектура, запуск, архитектурные решения (ADR), как проверить.<br>3. `.github/workflows/ci.yml`: Lint (Ruff), Type Check (Mypy), Test (Pytest), Build Docker.<br>3. Настройка `nginx` reverse proxy (SSL self-signed) для API/Grafana (опционально). | `docker compose up -d` работает "из коробки". CI зеленый. README позволяет любому запустить за 1 команду. Репозиторий готов к показу на собесе. |

---

## 8. ADR (Architecture Decision Records) — Ключевые решения для собеседования

| ID | Decision | Rationale | Trade-offs |
| :--- | :--- | :--- | :--- |
| **ADR-001** | **TimescaleDB вместо InfluxDB / PostgreSQL** | PostgreSQL-совместимость (SQL, JOIN, ORM), нативные гипертаблицы, Continuous Aggregates, Retention Policies, надежность PG. | Сложнее setup, чем InfluxDB; чуть выше накладные расходы на запись vs чистый TSDB. |
| **ADR-002** | **MQTT (Mosquitto) как шина данных** | Стандарт IoT, QoS 1/2, Persistent Sessions, легковесный, Pub/Sub pattern, разделение Edge/Cloud. | Нет встроенного реплея (как Kafka), нет схем (Schema Registry). Для MVP — идеально. |
| **ADR-003** | **FastAPI + asyncpg (async)** | Высокая производительность на I/O (сеть, БД), автогенерация OpenAPI, Pydantic валидация, современный стандарт Python. | Требует понимания async/await, сложнее дебажить блокирующий код. |
| **ADR-004** | **Isolation Forest для аномалий** | Unsupervised (не нужны размеченные данные), работает с многомерностью, быстрый инференс, встроен в sklearn. | Требует обучения на "нормальных" данных, чувствителен к contamination parameter, "черный ящик". |
| **ADR-005** | **Docker Compose для оркестрации** | Простота воспроизведения среды (Dev/Prod parity), декларативность, нативная поддержка сети/томов. | Не подходит для продакшн-масштаба (нет self-healing, rolling updates, service mesh). Для пет-проекта/демо — стандарт. |
| **ADR-006** | **JSONB в PostgreSQL для метрик/тегов** | Схема-независимость (разные датчики = разные поля), GIN-индексы для поиска, гибкость. | Нет строгой валидации на уровне БД (делегировано приложению), больше места vs колонки. |

---

## 9. TESTING STRATEGY

| Уровень | Инструменты | Покрытие |
| :--- | :--- | :--- |
| **Unit** | `pytest`, `pytest-asyncio`, `pytest-mock` | Сервисы (Ingestion logic, Anomaly scoring), Pydantic модели, Утилиты. |
| **Integration** | `pytest`, `testcontainers` (Postgres, MQTT) | API → DB, MQTT Subscriber → API → DB, ML Inference → MQTT Publish. |
| **Contract** | `schemathesis` / `pytest-openapi` | Соответствие API OpenAPI схеме. |
| **Load (Smoke)** | `locust` / `hey` | 1000 req/s на `/telemetry` (проверка батчинга и пула коннектов). |

---

## 10. REPOSITORY STRUCTURE (Финальная)

```text
iot-telemetry-pipeline/
├── .github/workflows/ci.yml          # Lint, TypeCheck, Test, Build
├── docker-compose.yml                # Main orchestration
├── docker-compose.override.yml.example # For local dev (ports, volumes)
├── .env.example                      # Template for secrets
├── README.md                         # **Главная страница проекта**
├── ARCHITECTURE.md                   # This file
├── simulator/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── config.yaml                   # Devices, profiles, intervals
│   └── simulator.py                  # Async MQTT Publisher
├── ingestion/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── pyproject.toml                # Mypy, Ruff config
│   ├── alembic/                      # Migrations (optional for MVP)
│   └── app/
│       ├── main.py                   # FastAPI entrypoint, lifespan
│       ├── config.py                 # Pydantic Settings
│       ├── api/v1/endpoints/telemetry.py
│       ├── core/
│       │   ├── mqtt_client.py        # Async Subscriber/Publisher
│       │   ├── database.py           # AsyncPG Pool, SQLAlchemy Core
│       │   └── security.py           # API Key / Basic Auth (optional)
│       ├── models/                   # Pydantic Models (Request/Response/DB)
│       ├── repositories/             # Data Access Layer (SQL)
│       └── services/                 # Business Logic (Batching, Validation)
├── ml/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── train.py                      # Offline training script
│   ├── inference.py                  # Scheduled inference job
│   ├── models/                       # Saved .joblib models
│   └── features.py                   # Feature engineering helpers
├── timescaledb/
│   ├── init.sql                      # Schema, Hypertable, CA, Policies
│   └── queries.sql                   # Useful queries for Grafana/API
├── mosquitto/
│   ├── config/mosquitto.conf
│   └── aclfile                       # ACL for production
├── grafana/
│   ├── provisioning/datasources/datasource.yml
│   ├── provisioning/dashboards/dashboard.yml
│   └── dashboards/telemetry_overview.json
├── prometheus/
│   └── prometheus.yml
└── nginx/                            # Optional Reverse Proxy
    └── nginx.conf
```

---

## 11. README.MD — ТОЧКА ВХОДА ДЛЯ СОБЕСЕДУЮЩЕГО

> **Обязательные секции:**
> 1.  **One-liner:** "End-to-end IoT Telemetry Pipeline: Simulator → MQTT → FastAPI → TimescaleDB → Grafana + ML Anomaly Detection".
> 2.  **Architecture Diagram** (Mermaid/SVG).
> 3.  **Quick Start:** `cp .env.example .env && docker compose up -d` → Grafana `localhost:3000` (admin/admin), API `localhost:8000/docs`.
> 4.  **Key Technical Decisions (ADR Summary)** — ссылки на ADR.
> 5.  **API Docs:** Ссылка на `/docs` (Swagger UI).
> 6.  **How to Test:** `pytest`, `locust -f load_test.py`.
> 7.  **What I Learned / Challenges:** (Например: "Батчинг в asyncpg дал x10 throughput", "Continuous Aggregates в TimescaleDB ускорили дашборды в 50 раз").

---

### 🎯 ЧЕКЛИСТ ПЕРЕД КОММИТОМ В MAIN
- [ ] `docker compose up -d` поднимает **все 6 сервисов** без ошибок (`healthy`).
- [ ] Данные текут: Симулятор → MQTT → API → TimescaleDB → Grafana (графики растут в реальном времени).
- [ ] Алерты появляются в MQTT `alerts/+` и на Grafana дашборде.
- [ ] `pytest -v` проходит (Unit + Integration).
- [ ] `mypy --strict` проходит (нет `Any`, все типы аннотированы).
- [ ] `ruff check .` чисто.
- [ ] `README.md` позволяет запустить проект **любому разработчику за 1 минуту**.
- [ ] В GitHub Actions зеленая галочка на последнем коммите.

---

**Этот документ — твоя шпаргалка на собесе.** Печатай, изучай, будь готов ответить на любой пункт "Почему так?", "А как бы ты масштабировал?", "Что если упадет MQTT?".
