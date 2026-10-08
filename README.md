# shiptracka
# ShiTrackA: Supply-Chain Operations Copilot (Spring AI + MCP + Kafka)

> An agentic copilot that answers operational questions such as *"Why is shipment SH-1042 delayed, and what should we do?"* by calling secured, audited tools exposed over the **Model Context Protocol (MCP)**, backed by event-driven Spring Boot microservices.

![build](https://img.shields.io/badge/build-passing-brightgreen) ![java](https://img.shields.io/badge/Java-21-blue) ![spring](https://img.shields.io/badge/Spring%20Boot-4.x-green) ![spring-ai](https://img.shields.io/badge/Spring%20AI-2.0-brightgreen)

**Demo video (2-3 min):** `[add link]`  |  **Architecture decisions:** [`docs/adr`](docs/adr)  |  **Author:** Prabhakar Vasudevan, Solution Architect

---

## Why this project exists

Most "AI + Java" demos stop at a chatbot. This project explores what it takes to let an LLM safely operate on enterprise systems:

- **Tool access through MCP**, not hard-coded prompts or ad-hoc REST wrappers
- **Least-privilege security**: OAuth2 scopes per tool, token propagation, no shared secrets
- **Human approval** for any action that changes state, with a full audit trail
- **Designed for failure**: LLM calls are slow and non-deterministic, so the system uses timeouts, circuit breakers and fallbacks
- **Observable by default**: traces, token usage and cost per request

The domain (shipment tracking, warehouse docks, delivery exceptions) mirrors real retail and logistics platforms I have architected.

## Architecture

```mermaid
flowchart LR
    U[Operator / Chat UI] --> A[copilot-agent<br/>Spring AI ChatClient]
    A -- MCP client --> M[logistics-mcp-server<br/>@McpTool]
    A --> L[(LLM provider<br/>Azure OpenAI / Claude / Ollama)]
    M -- REST + OAuth2 --> S[shipment-service]
    M -- REST + OAuth2 --> W[warehouse-service]
    SIM[event-simulator] -- events --> K[(Kafka)]
    K --> S
    K --> W
    S --> PG[(Postgres)]
    W --> PG
    KC[Keycloak<br/>OAuth2 / OIDC] -. tokens .-> A
    KC -. tokens .-> M
    A --> AUD[(Audit log)]
```

## Services

| Service | Responsibility | Key tech |
|---|---|---|
| `event-simulator` | Generates realistic shipment and dock events (scan, delay, exception) | Spring Boot, Kafka producer |
| `shipment-service` | Shipment state, read model built from Kafka events, REST API | Spring Boot, Spring Data JPA, Kafka consumer, Postgres |
| `warehouse-service` | Dock schedule, inventory, slot availability | Spring Boot, JPA, Kafka consumer |
| `logistics-mcp-server` | Exposes business capabilities as MCP tools, enforces per-tool scopes | Spring AI MCP server, Spring Security (resource server) |
| `copilot-agent` | Conversation, tool orchestration, memory, approval workflow, audit | Spring AI ChatClient, advisors, MCP client |
| `keycloak` | Identity provider (local dev) | Keycloak, OIDC |

## MCP tools

| Tool | Type | Required scope | Approval |
|---|---|---|---|
| `getShipmentStatus` | read | `shipments:read` | no |
| `listDelayedShipments` | read | `shipments:read` | no |
| `getDockSchedule` | read | `warehouse:read` | no |
| `getInventoryLevel` | read | `warehouse:read` | no |
| `rescheduleDelivery` | **write** | `shipments:write` | **yes (human)** |
| `escalateToCarrier` | **write** | `shipments:write` | **yes (human)** |

## Quick start

```bash
# prerequisites: Java 21, Docker, Maven
git clone <repo-url> && cd shipsense
docker compose -f deploy/compose/docker-compose.yml up -d   # Kafka, Postgres, Keycloak, Grafana
./mvnw clean package -DskipTests
./mvnw -pl services/copilot-agent spring-boot:run
```

Then open `http://localhost:8080` and ask: *"Which shipments are delayed more than 4 hours, and which docks are free tomorrow?"*

Configure the model provider in `services/copilot-agent/src/main/resources/application.yml`. Switching between Azure OpenAI, Claude and a local Ollama model requires only configuration changes.

## Security model

- OAuth2 / OIDC via Keycloak; the MCP server is an OAuth2 resource server
- Each tool requires a specific scope; the agent can only call what the user's token allows
- Tool outputs are treated as **untrusted input** (prompt-injection mitigation)
- PII is redacted before text is sent to the model
- Every tool call is written to an immutable audit log (who, what, when, approved by)

See [ADR-006](docs/adr/006-mcp-security-and-scopes.md) and [ADR-007](docs/adr/007-human-approval-for-write-tools.md).

## Observability

- Micrometer metrics: request latency, tool-call count and failures, **token usage and estimated cost per request**
- OpenTelemetry traces spanning agent, LLM call, MCP server and downstream services
- Grafana dashboards in `deploy/grafana`
- Screenshot: `[add dashboard image]`

## Quality and evaluation

- Unit and contract tests with mocked model responses
- **Golden-question evaluation:** `[N]` questions with the expected tool calls; current accuracy `[X]%` (`eval/` folder)
- Load test (JMeter): `[X]` concurrent users, p95 latency `[X]` ms, error rate `[X]%` (`perf/` folder)

## Non-functional targets

| Quality attribute | Target | Measured |
|---|---|---|
| Read-only query p95 latency (excl. LLM) | < `[X]` ms | `[ ]` |
| End-to-end answer p95 latency | < `[X]` s | `[ ]` |
| Availability when LLM provider is down | read APIs unaffected, graceful fallback message | `[ ]` |
| Cost per conversation | < `$[X]` | `[ ]` |

## Resilience

Timeouts and retries with backoff on LLM and MCP calls, a circuit breaker with a fallback response, idempotent Kafka consumers, and dead-letter topics for poison events.

## Architecture decisions

| # | Decision |
|---|---|
| 001 | Monorepo with Maven multi-module build |
| 002 | Kafka event design: topics, keys, schema evolution |
| 003 | MCP for tool exposure versus direct REST tool calling |
| 004 | Tool granularity: business-intent tools, not CRUD wrappers |
| 005 | Model-provider abstraction and portability |
| 006 | MCP security: OAuth2 scopes and token propagation |
| 007 | Human approval for state-changing tools |
| 008 | Conversation memory strategy and retention |
| 009 | Observability and LLM cost tracking |
| 010 | Evaluation approach for non-deterministic behaviour |
| 011 | Deployment target: AKS versus Container Apps |

## Deployment

Infrastructure as Code in `deploy/terraform` (or `deploy/bicep`). CI/CD with GitHub Actions: build, test, evaluation gate, container image, deploy.

## Roadmap

- [ ] Multi-agent split (investigator and planner agents)
- [ ] Per-tenant data isolation
- [ ] Streaming responses to the UI
- [ ] Policy-as-code for approval rules

## What I learned

`[Add 3-4 honest lessons after building: for example, tool descriptions matter more than prompts; evaluations catch regressions that unit tests miss; approval UX is harder than the API.]`

## License

None
