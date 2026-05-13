# Knowledge Graph

> Store: `memory-bank`
> Generated: May 14, 2026, 01:14 AM
> Version: 1

## Statistics

| Metric | Count |
|--------|-------|
| Entities | 8 |
| Observations | 0 |
| Relations | 6 |

**Entity Types:** component, concept, service

**Relation Types:** contains, implements, routes_to, shares_storage

## Recent Activity

*No recent activity.*

## Entities (8)

### component (4)

#### CircuitBreaker

- **Type:** component
- **ID:** `ent_Bdx3m268D--k`
- **Created:** just now
- **Attributes:**
  - trip_threshold: "3x 5xx"
  - eviction_seconds: 30
  - states: ["closed","open","half-open"]
  - metric: "infrallm_circuit_state"
  - file: "gateway/core/circuit_breaker.py"

**Incoming Relations:**
- ← `contains` ← **Gateway**

#### GuardrailsMiddleware

- **Type:** component
- **ID:** `ent_q0VF3r9I1h05`
- **Created:** just now
- **Attributes:**
  - library: "llm-guard"
  - pre_scanners: ["PromptInjection","Secrets","BanSubstrings","TokenLimit"]
  - post_scanners: ["Secrets"]
  - plan_b: "regex-only if image > 1GB"
  - file: "gateway/middleware/guardrails.py"

**Incoming Relations:**
- ← `contains` ← **Gateway**

#### Router

- **Type:** component
- **ID:** `ent_MTo9Ps9fkfEm`
- **Created:** just now
- **Attributes:**
  - strategies: ["round-robin","weighted","latency-EMA","priority"]
  - file: "gateway/core/router.py"

**Incoming Relations:**
- ← `contains` ← **Gateway**

#### SQLiteStorage

- **Type:** component
- **ID:** `ent_zbg_y2Pf3GCz`
- **Created:** just now
- **Attributes:**
  - mode: "WAL"
  - orm: "SQLModel"
  - tables: ["provider","auth_token","a2a_task"]
  - shared_via: "Docker volume bind ./data"

**Incoming Relations:**
- ← `shares_storage` ← **Gateway**

### concept (1)

#### A2AProtocol

- **Type:** concept
- **ID:** `ent_cXVbS8djdEHi`
- **Created:** just now
- **Attributes:**
  - version: "v0.1 MVP"
  - methods: ["tasks/send","tasks/get","tasks/cancel","tasks/sendSubscribe"]
  - excluded: ["push notifications","file artifacts","multi-turn streaming","sub-tasks"]

**Incoming Relations:**
- ← `implements` ← **Gateway**

### service (3)

#### Gateway

- **Type:** service
- **ID:** `ent_kg9Awj84bGpS`
- **Created:** just now
- **Attributes:**
  - port: 8000
  - role: "data-plane"
  - apis: ["OpenAI-compat /v1/chat/completions","A2A /a2a/jsonrpc","/.well-known/agent.json"]
  - file: "gateway/"

**Outgoing Relations:**
- → `contains` → **Router**
- → `contains` → **CircuitBreaker**
- → `contains` → **GuardrailsMiddleware**
- → `implements` → **A2AProtocol**
- → `routes_to` → **MockLLM**
- → `shares_storage` → **SQLiteStorage**

#### MockLLM

- **Type:** service
- **ID:** `ent_m6ZFeVhPmh5d`
- **Created:** just now
- **Attributes:**
  - ports: [9001,9002]
  - variants: ["mock-fast ~50ms","mock-slow ~500ms"]
  - file: "mock-llm/"

**Incoming Relations:**
- ← `routes_to` ← **Gateway**

#### Registry

- **Type:** service
- **ID:** `ent_okrOIUaQJXX_`
- **Created:** just now
- **Attributes:**
  - port: 8001
  - role: "control-plane"
  - apis: ["/admin/providers","/admin/a2a/agents"]
  - file: "registry/"

## Relations (6)

### contains (3)

```
Gateway --contains--> Router
Gateway --contains--> CircuitBreaker
Gateway --contains--> GuardrailsMiddleware
```

### implements (1)

```
Gateway --implements--> A2AProtocol
```

### routes_to (1)

```
Gateway --routes_to--> MockLLM
```

### shares_storage (1)

```
Gateway --shares_storage--> SQLiteStorage
```
