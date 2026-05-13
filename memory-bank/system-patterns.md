# System Patterns

## Структура репозитория

```
infrallm/
├── gateway/          # Data plane — FastAPI :8000
├── registry/         # Control plane — FastAPI :8001
├── mock-llm/         # Mock провайдеры (fast :9001, slow :9002)
├── shared/           # Общие Pydantic-модели, утилиты
├── infra/            # prometheus.yml, grafana provisioning
├── docs/
│   ├── architecture/
│   └── planning/
├── tests/
│   └── load/         # locust files
├── scripts/          # seed, demo, helpers
├── docker-compose.yml
└── pyproject.toml    # uv workspace root
```

## Паттерны кода

### Async generator для SSE streaming
Все upstream-обращения и SSE-ответы используют async generators + `StreamingResponse`.
Backpressure через AnyIO. Cancellation — через `request.is_disconnected()`.

### OpenAI-compat SSE формат
```
data: {"id":"...","choices":[{"delta":{"content":"chunk"}}]}
data: [DONE]
```

### A2A SSE формат (tasks/sendSubscribe)
```
data: {"id":"...","status":{"state":"working"}}
data: {"id":"...","artifact":{"parts":[{"text":"chunk"}]}}
data: {"id":"...","status":{"state":"completed"},"final":true}
```

### Response middleware — единая точка observability
После каждого ответа (stream или non-stream) middleware эмитирует:
- Prometheus метрики (RPS, latency, TTFT, TPOT, tokens, cost)
- MLflow run (через background task, не блокирует ответ)
- OTel span close

### FastAPI Dependencies для auth + guardrails
```python
# auth
async def require_token(authorization: str = Header(...)) -> TokenRecord: ...

# pre-guardrail
async def pre_guardrail(request: CompletionRequest) -> CompletionRequest: ...
```

### Circuit Breaker
Состояния: `closed → open → half-open → closed/open`
Trip: 3 подряд 5xx или timeout. Recovery: 30 сек, probe-запрос.
Метрика: `infrallm_circuit_state{provider=...}`

### EMA Latency (LatencyStrategy)
```python
ema = alpha * latest_latency + (1 - alpha) * prev_ema
# alpha = 0.3, обновляется в post-response hook
```

## Именование

- HTTP endpoints: kebab-case (`/admin/a2a/agents`)
- Python: snake_case (PEP 8)
- Pydantic models: PascalCase
- Prometheus metrics: `infrallm_<noun>_<unit>{labels}` (underscore, snake_case)
- SQLite tables: snake_case singular (`provider`, `auth_token`, `a2a_task`)

## Конфигурация

- `.env` файл (не коммитится) + `.env.example` в репо
- Pydantic Settings с `model_config = SettingsConfigDict(env_file='.env')`
- `providers.yaml` для начальной статической конфигурации провайдеров
- ENV переменные: `ANTHROPIC_API_KEY`, `MOCK_FAST_DELAY_MS`, `MOCK_SLOW_DELAY_MS`, `DB_PATH`, `LOG_LEVEL`

## Docker Compose паттерны

- Shared SQLite volume: `type: bind, source: ./data, target: /data`
- Health checks через `curl -f http://localhost:PORT/health`
- Сервисы зависят от gateway через `depends_on: {gateway: {condition: service_healthy}}`
- Grafana provisioning через `grafana/provisioning/` в compose volume
