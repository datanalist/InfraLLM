# InfraLLM — Product Context

## Обзор проекта

InfraLLM — LLM Gateway с реестром A2A-агентов, умной маршрутизацией, наблюдаемостью и guardrails. Цель — ИТМО LLM Track coursework, демонстрация концепций LLM-инфраструктуры.

## Технологический стек

- **Runtime:** Python 3.13, FastAPI, httpx, AnyIO
- **LLM SDK:** litellm (только как provider-adapter, НЕ как Router)
- **Storage:** SQLite + SQLModel (WAL-режим)
- **Observability:** OpenTelemetry → Prometheus → Grafana, MLflow
- **Guardrails:** llm-guard + FastAPI middleware
- **Auth:** Bearer токен + argon2id hashing
- **Load testing:** locust
- **Infra:** Docker Compose (7 сервисов)
- **Package manager:** uv

## API Контракты

- **Входящий от клиентов:** OpenAI-compatible (`/v1/chat/completions` со streaming SSE)
- **A2A Protocol:** Google A2A v0.1 MVP — Agent Card на `/.well-known/agent.json`, JSON-RPC на `/a2a/jsonrpc` (методы: tasks/send, tasks/get, tasks/cancel, tasks/sendSubscribe)
- **Исходящий к провайдерам:** через `litellm.acompletion()`
- **Admin API:** `/admin/providers` (CRUD), `/admin/a2a/agents`

## Архитектура (7 Docker Compose сервисов)

| Сервис | Порт | Роль |
|--------|------|------|
| gateway | :8000 | Data plane — OpenAI-compat + A2A |
| registry | :8001 | Control plane — admin API |
| mock-fast | :9001 | Mock LLM ~50ms |
| mock-slow | :9002 | Mock LLM ~500ms |
| prometheus | :9090 | Метрики |
| grafana | :3000 | Дашборды |
| mlflow | :5000 | Трассировка |

## Уровни (тиры)

### L1 — Базовый прокси и мониторинг
- OpenAI-compat proxy со streaming SSE
- Маршрутизация по полю `model`, round-robin/weighted для реплик
- 2 mock-провайдера + Anthropic через litellm
- OTel → Prometheus → Grafana (2 дашборда)
- Health checks на всех сервисах

### L2 — Реестр и умная маршрутизация  
- Dynamic provider registry через SQLite + CRUD API
- A2A Agent Card discovery + JSON-RPC task lifecycle
- Latency-based routing (EMA), circuit breaker (3x 5xx → 30s eviction)
- TTFT/TPOT/tokens/cost метрики, MLflow трасса каждого запроса

### L3 — Guardrails, Auth, Load
- Bearer token auth (argon2id, SQLite, 60s cache)
- Pre-guardrails: PromptInjection, Secrets, BanSubstrings, TokenLimit
- Post-guardrails: Secrets scan (non-stream + stream-aware)
- Locust load tests: steady/spike/provider-failure сценарии

## Ключевые архитектурные принципы

- **AP-1:** Data plane (gateway) и control plane (registry) — разные процессы, общий SQLite
- **AP-3:** Streaming — first-class citizen, все handlers — async generators
- **AP-4:** Fail-fast в стриме (после первого чанка — no retry)
- **AP-5:** Метрики/трасса/cost эмитятся в одной точке (response middleware)
- **AP-6:** Guardrails, auth — отдельные FastAPI Dependencies, легко выключаются

## Ограничения проекта

- Timeline: 14 дней соло (~90 ч)
- Реальный провайдер: только Anthropic API (нет OpenAI ключа)
- Out of scope: Web UI, K8s, Redis, multi-tenancy, mid-stream failover, cost-based routing
- A2A MVP исключает: push notifications, file artifacts, multi-turn streaming dialog, sub-tasks

## Acceptance Criteria финальной сдачи

- `docker compose up -d --build` → 7 сервисов за < 90 сек
- `bash scripts/demo.sh` проходит все 14 шагов без ошибок
- Git tag `v1.0.0` с релиз-notes
- docs/: brief, prd, tr, architecture, epics-and-stories, sprint-plan, load-test-report, README
