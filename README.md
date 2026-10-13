# ADRA AI Orchestrator — `adra-ai-orchestrator`

Spring Boot service that owns the AI chat conversation. It authenticates the caller from a JWT,
classifies each turn, routes it to the agent that should answer, persists the thread, and scores
how well the answer was grounded in its sources.

Extracted from `orchestrator-service` (`com.altametrics.orchestrator`) into the `com.sud.ai`
base package, keeping only the orchestration path. Left behind: the `spf` platform framework
(menus, JPQL runner, OTP, S3, CSV import, SSO, notifications), admin user management, feedback
triage, dashboard analytics, AI model config and PDF generation.

## Documentation

| Document | Read it for |
| --- | --- |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Components, a chat turn end to end, the supervisor graph, clients, ticket flow, security, design decisions, known limitations |
| [docs/API.md](docs/API.md) | Every REST endpoint with request/response schemas, curl examples, error mapping and enums |
| [docs/DATA-MODEL.md](docs/DATA-MODEL.md) | JPA entities, tables, columns, relationships and repository queries |
| [docs/CONFIGURATION.md](docs/CONFIGURATION.md) | Every property and environment variable, defaults, minimal local setup, production checklist |
| [docs/RUNBOOK.md](docs/RUNBOOK.md) | Build and deploy, health checks, kill switches, troubleshooting by symptom, correlation ids, audit log |
| [docs/DEVELOPER-GUIDE.md](docs/DEVELOPER-GUIDE.md) | Onboarding, conventions, how to add an agent, a local intent handler or an endpoint |
| [docs/TESTING.md](docs/TESTING.md) | Test strategy and conventions, how to run tests, coverage map, known gaps |

All classes carry Javadoc; the docs cite class and method names so you can jump straight to the code.

## Quick start

Prerequisites: JDK 21, PostgreSQL with a database named `adra-ai-orchestrator`, Lombok support in
your IDE. Maven is provided by the wrapper.

```bash
export JWT_SECRET=<base64 key, >= 256 bits, shared with the token issuer>
export DB_PASSWORD=...
export OPENAI_API_KEY=...        # optional; without it all knowledge turns go to RAG
./mvnw spring-boot:run           # http://localhost:8085/orchestrator
curl http://localhost:8085/orchestrator/v1/health
```

Then send a turn with a bearer token signed with the same `JWT_SECRET`:

```bash
curl -X POST http://localhost:8085/orchestrator/v1/chat/sessions/messages \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"content":"How do I close the day in Plum?"}'
```

See [docs/API.md](docs/API.md) for the full request and response shapes.

## What answers what

| Agent | Answers | Transport |
| --- | --- | --- |
| KW-RAG knowledge agent | Product and troubleshooting questions | HTTP JSON, or SSE aggregated server-side |
| altametrics-agenthub | "Raise a ticket" escalations (VTiger) | HTTP JSON |
| billing-service | The user's invoices, payments, subscription | HTTP JSON (`/api/billing/chat`) |
| server-health-service | Zabbix server health | HTTP JSON (`/api/v1/chat`) |
| OpenAI via Spring AI | Domain classification and groundedness scoring only | Spring AI `ChatClient` / raw HTTP |

## A chat turn

```
POST /v1/chat/sessions/{id}/messages   (Authorization: Bearer <jwt>)
  ChatService  tx #1  persist user message (tx released before the AI call)
  ChatTurnRouter
    RuleBasedIntentClassifier (keyword rules, no LLM)
      greeting / small talk / persona ...  -> local handlers, no network
      KNOWLEDGE -> SupervisorGraphRunner (LangGraph4j)
                   classifyDomain (hints -> Caffeine cache -> gpt-4o-mini)
                     KNOWLEDGE      -> knowledgeAgent -> KW-RAG
                     SUPPORT_TICKET -> ticketIntake (offer only)
                     BILLING        -> billing -> billing-service
                     SERVER_HEALTH  -> serverHealth -> server-health-service
                     NON_ANSWERABLE -> fallback
  ChatService  tx #2  persist reply + citations; afterCommit -> async groundedness score
```

## Package layout

```
com.sud.ai
├── AdraAiOrchestratorApplication   entry point (standalone or WAR)
├── chat/          controllers, ChatService, history, tickets, groundedness, intent, local handlers
├── supervisor/    LangGraph4j graph: config, runner, state, domain classifier, nodes
├── client/        HTTP clients for KW-RAG, agenthub, billing-service, server-health-service
├── billing/       billing identity + invoice PDF proxy
├── health/        liveness + runtime config summary
├── user/          AppUser entity, CurrentUserService, JWT-claim provisioning, role resolver
├── security/      JwtService, JwtAuthFilter, SecurityConfig, 401/403 writer
├── audit/         outbound agent-call audit log
└── common/        ApiResponse, PageResponse, BaseEntity, exceptions, PII masking
```

## Authentication

Every endpoint except health and API docs needs `Authorization: Bearer <jwt>`, HS256-signed with
`JWT_SECRET` (Base64, at least 256 bits). The old `X-Api-Key` / `X-Request-Signature` headers are
gone.

| Claim | Use |
| --- | --- |
| `sub` | Username (usually an email); the `AppUser` key |
| `email`, `phoneNumber` | Ticket reporter and billing identity |
| `firstName`, `lastName` | Greeting and ticket reporter name |
| `roleId` | Forwarded to server-health-service |
| `admin` | `true` unlocks `/v1/chat-maintenance/**` |

With `app.security.auto-provision-users=true` (default) an unknown subject gets an `AppUser` row
on first request and its profile is refreshed from later tokens. Set it `false` to accept only
pre-provisioned users.

## API

All paths are under `/orchestrator`. Responses use the `ApiResponse<T>` envelope.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/v1/chat/sessions` | List threads (paged, `includeArchived`) |
| GET | `/v1/chat/sessions/{id}` | Thread with messages and citations |
| PATCH / POST | `/v1/chat/sessions/{id}` | Rename or archive |
| DELETE | `/v1/chat/sessions/{id}` | Hard delete |
| POST | `/v1/chat/sessions/messages` | First message of a new thread |
| POST | `/v1/chat/sessions/{id}/messages` | Send a message |
| PATCH / POST | `/v1/chat/messages/{id}/feedback` | Rate an answer |
| POST | `/v1/chat/messages/{id}/regenerate` | Regenerate the latest answer |
| GET | `/v1/chat/messages/{id}/ticket-draft` | Ticket dialog prefill |
| POST | `/v1/chat/messages/{id}/ticket` | Raise the offered ticket |
| POST | `/v1/chat/messages/{id}/ticket/dismiss` | Decline the offer |
| GET | `/v1/billing/invoices/{n}/pdf`, `/v1/billing/invoices/latest/pdf` | Invoice PDF proxy |
| POST | `/v1/chat-maintenance/backfill-user-prompt` | Admin backfill |
| GET | `/v1/health`, `/v1/health/info` | Liveness, config summary (public) |

Swagger UI: `http://localhost:8085/orchestrator/swagger-ui.html`. Full reference with examples:
[docs/API.md](docs/API.md).

## Build, run and test

```bash
./mvnw spring-boot:run           # run locally
./mvnw test                      # unit tests (no database or network needed)
./mvnw clean package             # target/adra-ai-orchestrator.war (runs standalone or on Tomcat)
```

Every secret comes from the environment; nothing sensitive is committed. The main variables:

| Variable | Default |
| --- | --- |
| `JWT_SECRET` | none (startup fails) |
| `DB_URL` / `DB_USERNAME` / `DB_PASSWORD` | `jdbc:postgresql://localhost:5432/adra-ai-orchestrator` / `postgres` / empty |
| `JPA_DDL_AUTO` | `update` |
| `KW_RAG_AGENT_URL` / `KW_RAG_AGENT_API_KEY` | `https://localhost:9020/plumai/v1` / empty |
| `SUPPORT_TICKET_AGENT_URL` / `SUPPORT_TICKET_AGENT_API_KEY` | `http://localhost:9010/agenthub` / empty |
| `BILLING_SERVICE_URL` / `BILLING_SERVICE_API_KEY` | `http://localhost:8081` / empty |
| `SERVER_HEALTH_SERVICE_URL` / `SERVER_HEALTH_SERVICE_API_KEY` | `http://localhost:8082` / empty |
| `OPENAI_API_KEY` | empty |
| `APP_CORS_ALLOWED_ORIGINS` | `http://localhost:3000,http://localhost:4200` |
| `SERVER_PORT` | `8085` |

Every other key is in [docs/CONFIGURATION.md](docs/CONFIGURATION.md).

Kill switches: `orchestrator.supervisor.enabled=false` sends every knowledge turn to RAG and
disables ticket offers; `orchestrator.supervisor.classifier.enabled=false` keeps the graph but
skips the LLM; `orchestrator.chat.local-handlers.enabled=false` sends greetings to RAG too. When
to use them: [docs/RUNBOOK.md](docs/RUNBOOK.md#kill-switches).

## Tests

80 test classes (~600 test methods) of plain JUnit 5 + Mockito unit tests. None starts a Spring
context, a database or a real downstream agent; `GroundednessServiceTest` binds a local HTTP
server on 127.0.0.1 in place of OpenAI. Conventions, the coverage map and known gaps (graph
wiring, serializer, repositories, no integration tests yet) are in
[docs/TESTING.md](docs/TESTING.md).

## Known limitations

A short list; details and code references are in
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md#12-known-limitations-and-possible-bugs).

- Downstream outages surface as HTTP 404 (`NotExist`), and unexpected request errors as 500.
- `AgentChatReply.parse` rejects a reply containing `"offerTicket": null`, so a valid billing or
  server-health answer is treated as the agent being unavailable.
- `GreetingHandler` has templates for `GETTING_STARTED` / `TAKE_TOUR` but does not claim those
  intents, so they go to the knowledge path.
- The code default of `orchestrator.supervisor.enabled` is `false`; only `application.properties`
  turns it on.

## Importing into Eclipse

The project is a standard Maven project (`pom.xml` + Maven wrapper) and ships with Eclipse/m2e
metadata (`.project`, `.classpath`, `.settings/`), so either import works:

- **File → Import → Existing Projects into Workspace**, root directory
  `D:\2022\hubworks\adra-ai-orchestrator`, or
- **File → Import → Maven → Existing Maven Projects**, same folder.

Then **right-click the project → Maven → Update Project** (Alt+F5) so m2e downloads the
dependencies. Requirements: a JDK 21 registered under *Window → Preferences → Java → Installed
JREs*, and the Lombok agent installed in Eclipse (same as `orchestrator-service`).

Run it from Eclipse with **Run As → Java Application** on `AdraAiOrchestratorApplication`, with
`JWT_SECRET` (and `DB_PASSWORD`, `OPENAI_API_KEY` as needed) set on the run configuration's
*Environment* tab. Tests: **Run As → JUnit Test** on `src/test/java`.

## Changes from `orchestrator-service`

- Base package `com.altametrics.orchestrator.app.*` → `com.sud.ai.*`.
- Fixed: a failed chat turn no longer leaves an unanswered user message behind (it is discarded in a
  compensating transaction), and message `seq` positions are reserved under a row lock so a retry or a
  concurrent send never reuses one.
- Fixed: a failed or empty domain-classifier call is no longer cached as `KNOWLEDGE` for the cache TTL.
- Java package later renamed `com.adra.ai.*` → `com.sud.ai.*` (Maven `groupId` too); the artifact
  name, `AdraAiOrchestratorApplication`, context path and database name are unchanged.
- `EOUser` + `AppUser` + `EORole` merged into one lean `AppUser` (username, names, email, phone,
  roleId, admin, active). `ChatSession.eoUser` is now `user` (column `userID`).
- JWT-only auth; signature headers, SSO, OTP and the global API-key filter removed.
- Secrets moved to environment variables; Actuator limited to `health,info,metrics`.
- `spring.main.allow-circular-references` removed (the context boots without it).
- Fixed: `orchestrator.chat.local-handlers.enabled` is now the key `ChatTurnRouter` reads (it
  read `aibuddy.*` before).
- Removed an accidental Netty `StringUtil` import in `SupportTicketAgentClient`.
- Schema is created by Hibernate (`ddl-auto=update`) for a fresh database; the legacy migration
  scripts were not carried over.
