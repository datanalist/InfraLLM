# InfraLLM — Product Requirements Document (PRD)

> BMad Workflow: `CP` — bmad-create-prd
> Дата: 2026-05-13
> Версия: 1.0 (Draft)
> Источники: [product-brief.md](./product-brief.md), [tr-a2a-and-guardrails.md](./tr-a2a-and-guardrails.md)

---

## 1. Обзор

**InfraLLM** — LLM Gateway с реестром A2A-агентов, умной маршрутизацией, наблюдаемостью и guardrails. Цель — ИТМО LLM Track coursework, демонстрация концепций LLM-инфраструктуры.

**Контракт на входе:** OpenAI-compatible REST (`/v1/chat/completions` с streaming) + **Google A2A v0.1 MVP** (JSON-RPC `/a2a/jsonrpc` + `/.well-known/agent.json`).
**Контракт на выходе к провайдерам:** через `litellm.acompletion()` (универсальный SDK).
**Хранилище:** SQLite (файл, persist через volume).

## 2. Стейкхолдеры

| Роль | Интерес |
|---|---|
| Студент-автор | Сдать проект, получить хорошую оценку |
| Преподаватель ИТМО LLM Track | Оценить понимание концепций, демо, документацию |
| (Гипотетически) Разработчик-интегратор | Использовать gateway как drop-in для multi-LLM |

## 3. Функциональные требования

### 3.1. Уровень 1 (L1) — Базовый прокси и мониторинг

| ID | Требование | Приёмка |
|---|---|---|
| FR-L1-1 | Gateway принимает `POST /v1/chat/completions` со стандартной OpenAI-схемой | Запрос `curl` с любым OpenAI-SDK работает |
| FR-L1-2 | Поддержка `stream: true` через Server-Sent Events | Чанки приходят в реальном времени, корректные `data: ...` строки, финальный `data: [DONE]` |
| FR-L1-3 | Маршрутизация по полю `model` к зарегистрированному провайдеру | `model: claude-3-haiku` → Anthropic; `model: mock-fast` → mock-fast container |
| FR-L1-4 | При нескольких репликах одного `model` — round-robin или weighted | Логи показывают чередование выбора |
| FR-L1-5 | Health-check endpoint `/health` на каждом сервисе | Возвращает 200 + JSON `{status, version, uptime}` |
| FR-L1-6 | OTel-инструментация HTTP-входа | Trace появляется в Tempo/Jaeger (или OTel collector) |
| FR-L1-7 | Prometheus-метрики на `/metrics` | RPS, latency histogram, статусы, активные коннекты |
| FR-L1-8 | 2 Grafana-дашборда: latency-overview и traffic-by-provider | JSON-дашборды в репозитории, автоинициализация |
| FR-L1-9 | 2 mock-провайдера с управляемыми задержками | mock-fast (~50ms), mock-slow (~500ms) — параметризуется ENV |
| FR-L1-10 | Реальный Anthropic-провайдер | Подключение через `litellm`, ключ из `.env` |

### 3.2. Уровень 2 (L2) — Реестры и умная маршрутизация

| ID | Требование | Приёмка |
|---|---|---|
| FR-L2-1 | CRUD реестра LLM-провайдеров через `/admin/providers` | Добавление провайдера на лету подхватывается без рестарта |
| FR-L2-2 | Provider record содержит: `name`, `base_url`, `model_aliases`, `price_input`, `price_output`, `priority`, `rps_limit` | Pydantic-схема, SQLite persist |
| FR-L2-3 | A2A discovery — `GET /.well-known/agent.json` отдаёт Agent Card самого gateway по A2A v0.1 schema | JSON валиден по A2A spec, `curl` показывает name/description/capabilities/endpoints |
| FR-L2-3a | A2A JSON-RPC endpoint `POST /a2a/jsonrpc` поддерживает методы: `tasks/send`, `tasks/get`, `tasks/cancel`, `tasks/sendSubscribe` | Все 4 метода проходят корректные/невалидные запросы, ошибки в формате JSON-RPC error |
| FR-L2-3b | Task state machine: submitted → working → completed/failed/canceled, состояния в SQLite | Lifecycle переходы корректны, видны через `tasks/get` |
| FR-L2-3c | `tasks/sendSubscribe` стримит `TaskStatusUpdateEvent` и `TaskArtifactUpdateEvent` через SSE | Клиент получает события в реальном времени, финальное состояние видно |
| FR-L2-3d | Регистрация downstream-агентов через `POST /admin/a2a/agents` (внутренний API) | Gateway маршрутизирует `tasks/send` к downstream по capability |
| FR-L2-4 | Latency-based routing — выбор провайдера с минимальным EMA-latency | При искусственном замедлении mock-fast трафик переключается на mock-slow |
| FR-L2-5 | Health-aware routing — circuit breaker | 3 подряд 5xx → провайдер исключён на 30 сек, потом probe |
| FR-L2-6 | Стратегия маршрутизации настраивается per-model | `round-robin`, `weighted`, `latency`, `priority` |
| FR-L2-7 | Метрики: TTFT, TPOT, input_tokens, output_tokens, cost_usd | Каждый запрос имеет полную запись в Prometheus |
| FR-L2-8 | MLflow интеграция — трасса каждого запроса | MLflow UI показывает run с тегами `provider`, `model`, `request_id` |

### 3.3. Уровень 3 (L3) — Guardrails, auth, нагрузка

| ID | Требование | Приёмка |
|---|---|---|
| FR-L3-1 | Pre-prompt guardrails: PromptInjection, Secrets, BanSubstrings, TokenLimit | 5+ типовых примеров блокируются с 400 + reason |
| FR-L3-2 | Post-response guardrails: Secrets scan | На моке-промпте, провоцирующем "leak", ответ блокируется с 502 |
| FR-L3-3 | Bearer-token auth для всех `/v1/*` и `/a2a/*` | Без токена → 401; с валидным → 200 |
| FR-L3-4 | Auth-токены хранятся в SQLite, hashed (bcrypt/argon2) | CLI/seed-script создаёт начальный токен |
| FR-L3-5 | Нагрузочный тест с locust: 3 сценария (steady, spike, provider-failure) | Отчёт `docs/load-test-report.md` с цифрами и графиками |
| FR-L3-6 | При отказе провайдера под нагрузкой система продолжает обслуживать ≥80% RPS | Видно в отчёте |

## 4. Нефункциональные требования

| ID | НФТ | Целевое значение |
|---|---|---|
| NFR-1 | Время старта `docker-compose up` до готовности | < 60 сек (без pull) |
| NFR-2 | Прокси overhead (без учёта upstream latency) | p50 < 10 мс, p95 < 30 мс |
| NFR-3 | Streaming — первый чанк клиенту от первого чанка провайдера | < 50 мс (overhead прокси) |
| NFR-4 | Стабильная работа при 100 RPS на одном `gateway` контейнере | без 5xx от прокси-уровня |
| NFR-5 | Failover в стриме | **Fail-fast** — после первого чанка не переключаемся (mid-stream switch вне scope) |
| NFR-6 | Container image size | < 500 MB на сервис |
| NFR-7 | Все коды и API задокументированы | OpenAPI 3 на `/docs` для каждого сервиса |
| NFR-8 | Логи — структурированные JSON | stdout, читаются Promtail/Loki если будет добавлено |
| NFR-9 | Конфигурация через ENV + опциональный YAML | `.env.example` в репозитории |

## 5. User Stories (высокий уровень — детализация в `CE`)

### Epic 1 — Базовая платформа (L1)
- **US-L1-1:** Как клиент, я отправляю OpenAI-compat запрос и получаю ответ.
- **US-L1-2:** Как клиент, я получаю стриминговый ответ через SSE.
- **US-L1-3:** Как admin, я вижу метрики в Grafana.
- **US-L1-4:** Как admin, я вижу live-трейсы в OTel.
- **US-L1-5:** Как клиент, при падении одного провайдера мой запрос обслуживает другой (round-robin для одинаковых моделей).

### Epic 2 — Реестр и умная маршрутизация (L2)
- **US-L2-1:** Как admin, я регистрирую LLM-провайдера через API без рестарта.
- **US-L2-2:** Как внешний клиент, я делаю `GET /.well-known/agent.json` и получаю Agent Card gateway по A2A v0.1 спеке.
- **US-L2-3:** Как A2A-клиент, я отправляю `tasks/send` через JSON-RPC и получаю результат.
- **US-L2-4:** Как A2A-клиент, я подписываюсь на `tasks/sendSubscribe` и получаю SSE поток TaskStatusUpdateEvent.
- **US-L2-5:** Как admin, я регистрирую downstream-агента и его capabilities появляются в агрегированной Agent Card gateway.
- **US-L2-6:** Как клиент, мои запросы автоматически идут к быстрейшему здоровому провайдеру.
- **US-L2-7:** Как клиент, при 5xx-серии провайдер исключается из пула.
- **US-L2-8:** Как разработчик, я вижу полную трассу запроса в MLflow с метриками TTFT/TPOT/cost.

### Epic 3 — Безопасность и нагрузка (L3)
- **US-L3-1:** Как security-инженер, prompt-injection попытки блокируются на входе.
- **US-L3-2:** Как security-инженер, утечка секретов в ответе блокируется.
- **US-L3-3:** Как admin, доступ к API защищён bearer-токеном.
- **US-L3-4:** Как разработчик, я запускаю нагрузочный тест и получаю отчёт.

## 6. Открытые риски

| Риск | Вероятность | Митигация |
|---|---|---|
| Не успеть L3 за timeline | Высокая | MVP-уровень L3, готовые библиотеки (llm-guard, locust); 2 нагрузочных сценария вместо 3 |
| A2A spec — нюансы JSON-RPC и task lifecycle | Средняя | MVP-подмножество (4 метода), strict adherence к A2A v0.1, тесты по примерам из spec |
| Конфликт двух SSE-форматов в одном gateway | Средняя | Раздельные encoder'ы, общий transport; интеграционные тесты на оба endpoint'а до интеграции guardrails |
| `litellm` несовместим со streaming мок-провайдера | Средняя | Спецификация мока: говорит OpenAI-compat SSE напрямую |
| MLflow heavy для контейнера | Низкая | Use `mlflow server` + sqlite backend, без artifacts |
| `llm-guard` тянет тяжёлые ML-модели | Средняя | Pre-pull в Dockerfile, или fallback на regex-only |
| GraFana datasource не подцепляется | Низкая | Provisioning YAML, тестируем заранее |

## 7. Out of Scope (явно)

- ❌ Real OpenAI API
- ❌ Web UI для управления
- ❌ Multi-tenancy и биллинг
- ❌ K8s / Helm
- ❌ Distributed cache / Redis
- ❌ Cost-based routing
- ❌ Long-term trace storage
- ❌ Fine-grained RBAC
- ❌ Mid-stream provider failover

## 8. Acceptance Criteria (общие)

Финальная сдача включает:
- [ ] Git-репозиторий с тегом `v1.0`
- [ ] `docker-compose.yaml` поднимает всё одной командой
- [ ] `README.md` с архитектурным обзором, инструкцией запуска и demo-сценарием
- [ ] `docs/architecture.md` с диаграммами C4 (Context, Container, Component) и sequence-диаграммами
- [ ] `docs/load-test-report.md` с результатами 3 сценариев
- [ ] Grafana и MLflow доступны на `localhost:<port>` после старта
- [ ] Тесты: pytest для unit + integration по 1-2 примера на эпик
