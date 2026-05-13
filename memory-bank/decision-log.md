# Decision Log

## ADR-001: API Contract — Hybrid OpenAI-compat + A2A

**Дата:** 2026-05-13  
**Контекст:** Нужен API для LLM вызовов и для agent-to-agent взаимодействия.  
**Решение:** Hybrid — OpenAI-compatible REST для LLM calls + Google A2A v0.1 MVP-subset для A2A.  
**Альтернативы:** Custom REST (отклонено: потеря совместимости + больше кода), чистый A2A (отклонено: нет поддержки в OpenAI SDK).  
**Последствия:** Два SSE-формата в одном gateway (раздельные encoders), более сложная роутинг-логика.

---

## ADR-002: Proxy Stack — FastAPI + httpx + AnyIO + litellm (только как SDK)

**Дата:** 2026-05-13  
**Контекст:** Technical Research сравнивал Go, LiteLLM-Router, FastAPI.  
**Решение:** FastAPI + httpx + AnyIO; litellm используется только как provider-adapter SDK (NOT как Router).  
**Альтернативы:** Go (отклонено: больше бойлерплейта, меньше Python-экосистемы для guardrails); LiteLLM-Router (отклонено: скрывает роутинг-логику — теряется образовательная ценность).  
**Последствия:** Собственный routing/balancing/metrics/guardrails код; litellm берёт на себя 5 provider-клиентов.

---

## ADR-003: Storage — SQLite + SQLModel

**Дата:** 2026-05-13  
**Контекст:** Нужно хранить registry, auth tokens, A2A task state. Timeline не позволяет PostgreSQL.  
**Решение:** SQLite в WAL-режиме, shared через Docker volume; SQLModel ORM.  
**Альтернативы:** PostgreSQL (overhead на compose + операции), Redis (нет персистентности без AOF).  
**Последствия:** WAL позволяет concurrent reads из gateway и registry; одна точка отказа (файл).

---

## ADR-004: Mid-stream Failover — Fail-Fast

**Дата:** 2026-05-13  
**Контекст:** При начале стрима провайдер может упасть.  
**Решение:** Fail-fast — после первого чанка нет retry. Retry только до первого чанка.  
**Альтернативы:** Mid-stream switch (отклонено: сложно, не требуется по brief).  
**Последствия:** Упрощает upstream.py и router; клиент получает error event при обрыве стрима.

---

## ADR-005: A2A MVP Scope

**Дата:** 2026-05-13  
**Контекст:** Полный A2A spec очень большой (push notifications, file artifacts, multi-turn, sub-tasks).  
**Решение:** MVP-подмножество: только text/data message parts, 4 JSON-RPC метода (tasks/send, tasks/get, tasks/cancel, tasks/sendSubscribe), без push notifications.  
**Последствия:** ~1.5-2 дня работы vs полный spec; достаточно для demo и образовательной цели.

---

## ADR-006: Guardrails — llm-guard + Plan B regex

**Дата:** 2026-05-13  
**Контекст:** llm-guard тянет ML-модели (~500 MB+), может сделать image > 1 GB.  
**Решение:** Pre-pull моделей в Dockerfile; если image > 1 GB — fallback на regex-only scanners.  
**Альтернативы:** Только regex (быстро, но не показывает ML-guardrails концепцию).  
**Последствия:** День 11 — решение принимается после build Dockerfile; документируется в README.
