# iot-telemetry-pipeline
Pet-project 

iot-telemetry-pipeline/
├── docker-compose.yml          # Mosquitto, TimescaleDB, Grafana, FastAPI-app
├── simulator/
│   └── device_simulator.py     # asyncio, генерит JSON телеметрию -> MQTT (каждые 5 сек)
├── ingestion/
│   ├── main.py                 # FastAPI: POST /telemetry (валидация Pydantic) -> TimescaleDB
│   ├── models.py               # SQLAlchemy (Core) модели
│   └── database.py             # asyncpg connection pool
├── ml/
│   └── anomaly_detector.py     # Isolation Forest (sklearn) на последних N точках
├── grafana/
│   └── dashboards/             # JSON дашборды (импорт через provisioning)
└── README.md                   # Архитектура, как запустить, почему FastAPI/TimescaleDB, как масштабировать
