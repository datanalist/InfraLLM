# Active Context

## Текущие задачи

- [ ] День 1: Repo bootstrap — `uv init` на 3 сервиса + shared, Compose skeleton, mock-llm, health checks
- [ ] День 2: Gateway streaming pipeline — httpx forwarding + SSE StreamingResponse
- [ ] День 3: Routing + Prometheus metrics
- [ ] День 4: Anthropic integration + OTel + Grafana → CP-1
- [ ] День 5: SQLite schema + Registry CRUD
- [ ] День 6: ProviderCache + LatencyStrategy (EMA)
- [ ] День 7: Circuit breaker + buffer
- [ ] День 8: TTFT/TPOT/cost metrics + MLflow
- [ ] День 9: A2A Agent Card + tasks/send → CP-2
- [ ] День 10: tasks/sendSubscribe + tasks/get/cancel
- [ ] День 11: Bearer auth + pre-guardrails
- [ ] День 12: Post-guardrails + locust setup → CP-3
- [ ] День 13: Load test scenarios + analysis
- [ ] День 14: Documentation + defense prep

## Известные проблемы

- llm-guard тянет тяжёлые ML-модели (~500 MB+). Plan B: regex-only guardrails если image > 1 GB
- A2A SSE и OpenAI SSE — разные форматы в одном gateway; нужны раздельные encoders
- litellm совместимость со streaming мок-провайдера (мок должен говорить OpenAI-compat SSE)

## Следующие шаги

1. Запустить `uv init` и создать структуру проекта (gateway, registry, mock-llm, shared)
2. Написать Compose skeleton (docker-compose.yml)
3. Создать Pydantic shared-модели (CompletionRequest/Response, Message)
4. Реализовать mock-llm с SSE-генератором и `/health`

## Заметки сессии

- 2026-05-14: Memory bank инициализирован. Проект в самом начале — только pyproject.toml и main.py-заглушка. Всё планирование (PRD, architecture, sprint-plan, epics) уже задокументировано в docs/.
- Sprint start: День 1 ещё не начат (нет ни одного сервиса)
- Дедлайн сдачи: День 14 от старта спринта (дату начала выбирает студент)


## Current Session Notes

- [22:14:43] [vallo] Memory Bank инициализирован: Созданы все 5 core-файлов (product-context, active-context, progress, decision-log, system-patterns) и knowledge graph с 8 сущностями (Gateway, Registry, MockLLM, Router, CircuitBreaker, A2AProtocol, SQLiteStorage, GuardrailsMiddleware) + 7 связями. Источник: docs/planning/prd.md, architecture.md, sprint-plan.md, project_infrallm.md из памяти.
