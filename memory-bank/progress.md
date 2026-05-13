# Progress Log

## Статус по Milestone

| Milestone | Статус | День |
|-----------|--------|------|
| Planning & Architecture docs | ✅ Готово | pre-sprint |
| L1: Mock LLM + health | ⬜ Не начато | День 1 |
| L1: Gateway streaming | ⬜ Не начато | День 2 |
| L1: Routing + Prometheus | ⬜ Не начато | День 3 |
| **CP-1** L1 demo-able | ⬜ Не начато | День 4 |
| L2: SQLite + Registry | ⬜ Не начато | День 5 |
| L2: ProviderCache + LatencyRouter | ⬜ Не начато | День 6 |
| L2: Circuit Breaker | ⬜ Не начато | День 7 |
| L2: Metrics + MLflow | ⬜ Не начато | День 8 |
| **CP-2** L2 core done | ⬜ Не начато | День 9 |
| L2: A2A streaming + methods | ⬜ Не начато | День 10 |
| L3: Auth + Pre-guardrails | ⬜ Не начато | День 11 |
| **CP-3** L3 minimum | ⬜ Не начато | День 12 |
| L3: Load tests | ⬜ Не начато | День 13 |
| Docs + Defense | ⬜ Не начато | День 14 |

## История изменений

### 2026-05-14 — Memory Bank инициализирован
- Создан memory-bank со всем контекстом проекта из docs/planning/
- Статус: код ещё не написан, только planning artifacts в репо
- Состав репо: pyproject.toml, main.py (заглушка), docs/ (PRD + arch + sprint-plan + epics), _bmad/


## Update History

- [2026-05-13 22:14:43] [vallo] - Memory Bank инициализирован: Созданы все 5 core-файлов (product-context, active-context, progress, decision-log, system-patterns) и knowledge graph с 8 сущностями (Gateway, Registry, MockLLM, Router, CircuitBreaker, A2AProtocol, SQLiteStorage, GuardrailsMiddleware) + 7 связями. Источник: docs/planning/prd.md, architecture.md, sprint-plan.md, project_infrallm.md из памяти.
