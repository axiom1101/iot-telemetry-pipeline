# iot-telemetry-pipeline
Pet-project 

## README.MD — ТОЧКА ВХОДА ДЛЯ СОБЕСЕДУЮЩЕГО

> **Обязательные секции:**
> 1.  **One-liner:** "End-to-end IoT Telemetry Pipeline: Simulator → MQTT → FastAPI → TimescaleDB → Grafana + ML Anomaly Detection".
> 2.  **Architecture Diagram** (Mermaid/SVG).
> 3.  **Quick Start:** `cp .env.example .env && docker compose up -d` → Grafana `localhost:3000` (admin/admin), API `localhost:8000/docs`.
> 4.  **Key Technical Decisions (ADR Summary)** — ссылки на ADR.
> 5.  **API Docs:** Ссылка на `/docs` (Swagger UI).
> 6.  **How to Test:** `pytest`, `locust -f load_test.py`.
> 7.  **What I Learned / Challenges:** (Например: "Батчинг в asyncpg дал x10 throughput", "Continuous Aggregates в TimescaleDB ускорили дашборды в 50 раз").

---
