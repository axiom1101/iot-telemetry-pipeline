# BLUEPRINT: iot-telemetry-pipeline (MVP в 3 вечера)

## Цель
Рабочий `docker compose up -d` → Данные текут: Simulator → MQTT → FastAPI → TimescaleDB → Grafana + ML Alerts.
Репозиторий готов к показу на собесе (README, CI, ADR).

---

## Вечер 1: Фундамент + Данные (Инфраструктура + Simulator + Ingestion Skeleton)
**Фокус:** Поднять стек, заставить течь данные Simulator → MQTT → API → TimescaleDB.

- [ ] **Репо:** `git init`, `.gitignore` (Python, Docker, .env, __pycache__, *.db, *.log), `README.md` (заглушка).
- [ ] **Структура папок:** Создать по `ARCHITECTURE.md` (simulator, ingestion, ml, timescaledb, mosquitto, grafana, prometheus).
- [ ] **`docker-compose.yml`**: Написать скелет всех 6 сервисов (build, ports, depends_on, env_file, healthchecks).
- [ ] **`.env.example`**: `DATABASE_URL`, `MQTT_BROKER`, `LOG_LEVEL`, `SECRET_KEY`.
- [ ] **`timescaledb/init.sql`**: 
    - [ ] `CREATE EXTENSION timescaledb;`
    - [ ] `CREATE TABLE telemetry ...` (гипертаблица, JSONB, индексы).
    - [ ] Continuous Aggregate `telemetry_1m`.
    - [ ] Retention Policy (30 дней).
- [ ] **`mosquitto/config/mosquitto.conf`**: listener 1883, persistence true, allow_anonymous true.
- [ ] **`simulator/`**:
    - [ ] `Dockerfile` (python:3.11-slim, deps: paho-mqtt/paho-mqtt[asyncio], pydantic, pyyaml).
    - [ ] `config.yaml` (devices: 5, interval: 5s, profiles: normal/spike).
    - [ ] `simulator.py`: async main, MQTT connect, loop publish JSON в `sensors/{id}/telemetry`.
- [ ] **`ingestion/` (Скелет)**:
    - [ ] `Dockerfile` (python:3.11-slim, deps: fastapi, uvicorn, asyncpg, sqlalchemy, pydantic, paho-mqtt, prometheus-client, python-dotenv).
    - [ ] `app/config.py` (Pydantic Settings из .env).
    - [ ] `app/models/telemetry.py` (Pydantic модели: TelemetryPoint, TelemetryBatch).
    - [ ] `app/core/database.py` (asyncpg pool, init_db).
    - [ ] `app/core/mqtt_client.py` (Async subscriber на `sensors/#` -> парсинг -> батч -> INSERT).
    - [ ] `app/api/v1/endpoints/telemetry.py` (POST /telemetry, GET /telemetry).
    - [ ] `app/main.py` (FastAPI app, lifespan: startup DB pool + MQTT sub, shutdown close).
- [ ] **Запуск:** `docker compose up -d --build`.
- [ ] **Проверка:** `docker compose logs -f simulator ingestion-api` → видно "Published", "Received", "Inserted batch". `psql -h localhost -U admin -d telemetry -c "SELECT * FROM telemetry LIMIT 5;"` → данные есть.

**Критерий готовности Вече готовности Вечера 1:** `docker compose up` → данные в БД. Swagger UI на `:8000/docs` открывается.

---

## Вечер 2: API Polish + ML + Observability (Мозги + Глаза)
**Фокус:** Production-ready API, ML-детектор, Графики в Grafana.

- [ ] **`ingestion/` (Допиливаем API):**
    - [ ] Pydantic валидация (кастомные валидаторы для metrics).
    - [ ] Обработка ошибок (422, 500), логирование (`structlog` JSON).
    - [ ] `GET /telemetry` с пагинацией/фильтрами (from, to, device_id, limit).
    - [ ] `/health` (liveness/readiness: проверка PG pool, MQTT connection).
    - [ ] `/metrics` (Prometheus: `prometheus-fastapi-instrumentator` + кастомные: `msg_received_total`, `db_insert_latency_seconds`).
    - [ ] Батчинг в subscriber'е (накопление 100 шт или 1 сек -> `executemany`).
- [ ] **`ml/` (Anomaly Detector):**
    - [ ] `Dockerfile` (python:3.11-slim, deps: scikit-learn, pandas, joblib, psycopg2/asyncpg, paho-mqtt).
    - [ ] `train.py`: Читает из БД последние 10к точек -> обучает `IsolationForest(contamination=0.01)` -> сохраняет `models/iso_forest.joblib`.
    - [ ] `inference.py`: APScheduler каждые 5 мин -> читает окно 100 точек/устройство -> `predict` -> если аномалия -> публикует в MQTT `alerts/{device_id}`.
    - [ ] `features.py`: Генерация фичей (rolling mean, std, diff, z-score).
- [ ] **`prometheus/prometheus.yml`**: Скрапит `ingestion-api:8000/metrics`, `node-exporter`, `postgres-exporter`.
- [ ] **`grafana/provisioning/`**:
    - [ ] `datasources/datasource.yml` (Prometheus, Loki, Postgres/TimescaleDB).
    - [ ] `dashboards/dashboard.yml` (путь к JSON).
    - [ ] `dashboards/telemetry_overview.json` (Импорт: Time series temp/hum/vib, System Health: CPU/Mem/DB/MsgRate, Anomalies Table).
- [ ] **Запуск:** `docker compose up -d --build ml grafana prometheus`.
- [ ] **Проверка:** Grafana `:3000` (admin/admin) → Дашборд "Telemetry Overview" показывает живые графики. Prometheus `:9090/targets` — все UP. MQTT `alerts/+` — появляются сообщения.

**Критерий готовности Вечера 2:** Живой дашборд в Grafana, алерты в MQTT, Prometheus собирает метрики API.

---

## Вечер 3: Polish + CI/CD + Docs (Продукт для собеседования)
**Фокус:** "Можно показать любому Senior'у — он скажет: 'Ок, молодец'."

- [ ] **`simulator/` (Конфиг):** `config.yaml` профили устройств (normal, noisy, failing), флаг `inject_anomaly: true`.
- [ ] **`README.md` (Главная витрина):**
    - [ ] One-liner + Architecture Diagram (Mermaid).
    - [ ] **Quick Start:** `cp .env.example .env && docker compose up -d`.
    - [ ] **Access Points:** Grafana (3000), API Docs (8000/docs), Prometheus (9090), MQTT (1883).
    - [ ] **ADR Summary** (ссылки на `ARCHITECTURE.md`).
    - [ ] **How to Test:** `pytest`, `locust -f load_test.py`.
    - [ ] **What I Learned / Challenges** (честно: батчинг, Continuous Aggregates, async lifespan).
- [ ] **`ARCHITECTURE.md`**: Финальная версия (тот, что я прислал).
- [ ] **`.github/workflows/ci.yml`:**
    - [ ] `lint` (ruff), `typecheck` (mypy --strict), `test` (pytest -v), `build` (docker buildx).
    - [ ] Запуск на `push` и `pull_request`.
- [ ] **Тесты (`pytest`):**
    - [ ] Unit: `test_models.py` (валидация Pydantic), `test_features.py` (ML фичи).
    - [ ] Integration (с `testcontainers` или моками): `test_ingestion_flow.py` (POST -> DB -> GET).
- [ ] **Load Test (Smoke):** `locust -f load_test.py --headless -u 50 -r 10 -t 30s` (цель: 500 req/s без 500 ошибок).
- [ ] **Финальный прогон:** `docker compose down -v && docker compose up -d --build` → всё работает с чистого листа.
- [ ] **GitHub:** Push, проверка Actions (зеленая галочка), Release v0.1.0.

**Критерий готовности Вечера 3:** Репозиторий публичный, CI зеленый, `README` позволяет запустить за 1 минуту, на собесе можно открыть Grafana и показать живой пайплайн.

---

## Backlog (После MVP / На собесе как "Next Steps")
- [ ] TLS для MQTT (self-signed certs + CA).
- [ ] Auth для API (API Key / JWT).
- [ ] Kubernetes Manifests (Helm Chart) для деплоя на VPS.
- [ ] Реальный датчик (ESP32) вместо симулятора.
- [ ] Миграция на Kafka / Redis Streams вместо MQTT для реплея.
- [ ] Полноценный CI/CD (ArgoCD / Flux) на кластер.
