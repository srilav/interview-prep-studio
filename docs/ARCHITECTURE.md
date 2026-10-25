# Interview Prep Studio: Architecture

This page describes the system as it is built and runs today: each part, its role, and how a request moves through them. [DESIGN.md](DESIGN.md) is the full design, including parts that are planned but not built yet (see [Built vs planned](#built-vs-planned)).

## System overview

```
                         Browser
┌──────────────── React SPA (Vite + TypeScript) · :5173 ─────────────────┐
│   Search (/)  │  Key page (/keys/:id)  │  Prep Studio (/studio)  │  Progress (/progress)  │
└──────┬─────────────────────────────────────────────────┬───────────────┘
       │ gRPC-Web (unary + server streaming)               │ STOMP over WebSocket
       ▼                                                   │
┌──────────────┐                                           │
│ Envoy · :8081│  gRPC-Web → gRPC, CORS                    │
└──────┬───────┘                                           │
       │ gRPC                                              │
       ▼                                                   ▼
┌───────────── Prep Studio API: Spring Boot modular monolith · gRPC :9090 · WS :8080 ─────────────┐
│  catalog            learning             discussion             progress                       │
│  taxonomy, keys,    AI explanations,     threads, messages,     study-activity                 │
│  search, drafts     cached per level     AI tutor               read model                     │
│  ─────────── shared-kernel: domain events → outbox (same transaction) → in-process listeners ──│
│  ai-gateway: LlmGateway port → Claude adapter (or offline fake)                                │
└──────┬───────────────────────────────┬──────────────────────────────────┬──────────────────────┘
       │ JDBC                          │ HTTPS                            │ Kafka (optional)
       ▼                               ▼                                  ▼
┌────────────────────┐        ┌──────────────────────┐          ┌──────────────────────────┐
│ PostgreSQL · :5432 │        │ Claude API           │          │ Redpanda · :19092        │
│ schema public      │        │ claude-haiku-5-5     │          │ catalog.events,          │
│ schema jobs        │◄──┐    └──────────▲───────────┘          │ learning.events,         │
└────────────────────┘   │               │                      │ discussion.events        │
                         │ JDBC          │ HTTPS                └──────────────────────────┘
                  ┌──────┴───────────────┴──────┐
                  │ Prep Studio Jobs · gRPC :9091│── gRPC SearchKeys ──► API :9090
                  │ job link → skills → plan     │
                  └──────────────────────────────┘
```

## Parts and their roles

| Part | Repo | Role | Talks to |
|---|---|---|---|
| **Web SPA** | interview-prep-studio-web | Everything the user sees: search, key page with streamed explanation and live discussion, the Prep Studio editor, progress. Holds no business rules. Asks for a display name once and keeps it in `localStorage` (there is no login yet). | Envoy (gRPC-Web), API WebSocket (STOMP) |
| **Envoy** | api: `infra/envoy` | Protocol translator. Browsers cannot speak native gRPC, so Envoy turns gRPC-Web into gRPC and handles CORS. It holds no logic and no state. | API gRPC :9090 |
| **Prep Studio API** | interview-prep-studio-api | The backend and the owner of the data and the `.proto` contracts. One Spring Boot process, split into modules by bounded context (below). | Postgres, Claude API, Redpanda |
| **Prep Studio Jobs** | interview-prep-studio-jobs (local only, not on GitHub yet) | A separate service. It reads a job posting from a link, extracts the role and skills with Claude, checks which skills the catalog already covers, and streams a study plan. It never applies for jobs. The web app does not call it yet; use a gRPC client. | Postgres (`jobs` schema), Claude API, API `CatalogService.SearchKeys` |
| **PostgreSQL** | api: `docker-compose.yml` | The system of record. The API's tables live in `public` (Flyway V1–V5), and Jobs has its own `jobs` schema. Each service owns its schema; neither reads the other's tables. | |
| **Claude API** | external | The LLM: explanations, AI tutor replies, key drafts for editors, job extraction and plans. Reached only through the `LlmGateway` port, so the domain never sees SDK types. `LLM_PROVIDER=fake` swaps in canned text for offline work. | |
| **Redpanda (Kafka)** | api: compose profile `kafka` | Optional. With `KAFKA_ENABLED=true`, outbox rows are relayed to topics for consumers outside the API. Nothing inside the app depends on it. | |

## Inside the API

### Bounded contexts (Maven modules)

| Module | Role | Aggregates and tables | gRPC service | Publishes |
|---|---|---|---|---|
| `catalog` | The content: Platform › Category › Subcategory taxonomy and the keys (terms and interview questions). Search, editing, publishing, and AI drafts of new keys for editors. | `Platform`, `Key` · `platform`, `category`, `subcategory`, `catalog_key` | `CatalogService` (SearchKeys, GetKey, ListKeys), `TaxonomyAdminService` (ListTree, Create*, UpsertKey, MoveKey, DeleteKey, GenerateKeyDraft) | `PlatformAdded`, `CategoryAdded`, `SubcategoryAdded`, `KeyCreated`, `KeyUpdated`, `KeyMoved`, `KeyPublished`, `KeyDeleted` |
| `learning` | AI explanations of a key at a difficulty level. It streams text while Claude writes it, then stores the result, so the next request for the same key and level is served from the database. | `Explanation` · `explanation` | `LearningService.StreamExplanation` (server streaming) | `ExplanationRequested`, `ExplanationGenerated`, `ExplanationGenerationFailed`, `ExplanationViewed` |
| `discussion` | Live threads per key. Messages arrive over STOMP and are stored and broadcast. Mentioning `@tutor` makes the AI tutor reply, streamed into the thread (5 requests per learner per minute). | `DiscussionThread` · `discussion_thread`, `discussion_message` | `DiscussionService` (OpenThread, ListThreads, ListMessages), STOMP `/ws` | `ThreadOpened`, `MessagePosted`, `AiReplyRequested` |
| `progress` | A read model of study activity, built only from other contexts' events. It never calls them. | `study_activity` | `ProgressService.GetSummary` | |

### Supporting modules

| Module | Role |
|---|---|
| `proto` | The API contract: `.proto` files, the single source of truth. The web app generates its TypeScript client from here, and Jobs vendors `catalog.proto`. |
| `api-contracts` | Java stubs generated from `proto/`. |
| `shared-kernel` | `AggregateRoot`, `DomainEvent`, `DomainEventPublisher`, the outbox table and the relays. |
| `ai-gateway` | `LlmGateway` port, `ClaudeLlmGateway` (Anthropic Java SDK), `FakeLlmGateway`. |
| `app` | The Spring Boot entry point. Wires the modules, runs the Flyway migrations, and maps domain errors to gRPC status codes. |

### Layers and rules

- Each context is layered `api → application → domain`. `api` adapts gRPC and STOMP, `application` runs use cases in transactions, and `domain` holds the aggregates and invariants with no framework types.
- Contexts never touch each other's aggregates or tables. Others read the catalog only through `CatalogQueries`, a published snapshot API, and react to changes through events.
- That separation is what lets a module become its own service later; Jobs already is one.

### Domain events and the outbox

1. An application service changes an aggregate. The aggregate records events.
2. In the same database transaction, `DomainEventPublisher` writes each event to `outbox_event` and publishes it as a Spring event.
3. After commit, in-process listeners run: `ThreadBroadcaster` (pushes to WebSocket subscribers), `AiTutor` (answers `@tutor`) and `ProgressProjection` (updates `study_activity` from `ExplanationViewed`, `ExplanationGenerated` and `MessagePosted`).
4. Every 500 ms a relay reads unpublished outbox rows. `KafkaOutboxRelay` sends them to Kafka, keyed by aggregate id; with Kafka off, `InProcessOnlyOutboxRelay` just marks them handled.

Because the event is stored with the change, a crash cannot lose an event or publish one for a change that rolled back.

## Request flows

**Search a topic** (`/`)
SPA → Envoy → `CatalogService.SearchKeys` → catalog reads published keys from Postgres → results. `ListTree` fills the platform filter.

**Read a key and its explanation** (`/keys/:id`)
1. `CatalogService.GetKey` returns the definition, detail and Java example.
2. `LearningService.StreamExplanation` (server streaming):
   - If the explanation for that key and level is already stored, it is returned in one message.
   - If not, learning builds a prompt from the catalog snapshot, streams Claude's text to the browser chunk by chunk, stores the finished explanation and publishes `ExplanationGenerated`.
3. Viewing publishes `ExplanationViewed`, and progress counts it.

**Discuss with the AI tutor**
1. The SPA subscribes to `/topic/threads/{id}` and sends to `/app/threads/{id}/messages`.
2. discussion stores the message and publishes `MessagePosted`, and `ThreadBroadcaster` pushes it to every subscriber.
3. If the message mentions `@tutor`, `AiTutor` streams Claude's reply as `ai_chunk` frames, then posts the final message.

**Edit content** (`/studio`)
`TaxonomyAdminService` creates platforms, categories and subcategories and upserts, moves, publishes or deletes keys. `GenerateKeyDraft` asks Claude for a first draft of a key that the editor reviews. Bulk content, such as the interview question sets, ships as Flyway migrations (V2–V5).

**Turn a job link into a prep plan** (Jobs)
`AddJobPostings(url)`, then:
1. Fetch the page (jsoup, behind an SSRF guard).
2. Claude extracts the role and skills as structured output.
3. Each skill is looked up with `SearchKeys` on the API and marked covered, partial or missing.
4. `StreamPrepPlan` streams a study plan.

Pages behind a sign-in become `NEEDS_TEXT`, and the learner pastes the text instead.

## Running locally

| Process | Port | Start |
|---|---|---|
| Postgres, Envoy | 5432, 8081 (Envoy admin 9901) | `docker compose up -d` in the API repo |
| Redpanda (optional) | 19092 | `docker compose --profile kafka up -d` |
| API | gRPC 9090, HTTP/WebSocket 8080 | `mvn -pl app -am package -DskipTests`, then `java -jar app/target/app-0.1.0-SNAPSHOT.jar` (JDK 21) |
| Jobs (optional) | gRPC 9091 | see the Jobs README |
| Web | 5173 | `npm run dev` in the web repo |

Configuration that changes behaviour: `LLM_PROVIDER` (`claude` or `fake`), `ANTHROPIC_API_KEY`, `KAFKA_ENABLED`, `DATABASE_URL`, `ALLOWED_ORIGINS` (WebSocket origins).

### What runs in Docker and what runs on the host

```
┌──────────────────────────── your machine ────────────────────────────┐
│  host processes                       Docker (API repo's compose)     │
│  ┌──────────────┐                     ┌───────────────────────────┐   │
│  │ Vite :5173   │── gRPC-Web ───────► │ envoy        :8081, :9901 │   │
│  └──────┬───────┘                     └─────────────┬─────────────┘   │
│         │ WebSocket :8080                           │ host.docker.    │
│         ▼                                           │ internal:9090   │
│  ┌──────────────┐ ◄─────────────────── gRPC ────────┘                 │
│  │ API (JVM)    │── JDBC localhost:5432 ─► ┌───────────────────────┐  │
│  │ :9090, :8080 │                          │ postgres :5432        │  │
│  └──────────────┘                          │ (volume pgdata)       │  │
│  ┌──────────────┐── JDBC ────────────────► └───────────────────────┘  │
│  │ Jobs :9091   │                          ┌───────────────────────┐  │
│  └──────────────┘                          │ redpanda :19092       │  │
│                                            │ (profile kafka only)  │  │
│                                            └───────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

| Container (`docker compose ps`) | Image | Published ports | Notes |
|---|---|---|---|
| `interview-prep-studio-api-postgres-1` | `postgres:16.4` | 5432 | Database `prepstudio`, user and password `prepstudio`. Data survives restarts in the `pgdata` volume; `docker compose down -v` wipes it. |
| `interview-prep-studio-api-envoy-1` | `envoyproxy/envoy:v1.31.2` | 8081, 9901 | Config mounted read-only from `infra/envoy/envoy.yaml`. The image also declares 10000, which is not published or used. |
| `…-redpanda-1` | `redpandadata/redpanda:v24.2.7` | 19092 | Only with `--profile kafka`. |

The API, Jobs and the web dev server run on the host, not in Docker. Envoy reaches the API through `host.docker.internal`, which compose maps to the host with `extra_hosts: host-gateway`.

### How the web app reaches the API

The web app does not configure Envoy; it only knows two URLs:

| Variable | Default | Used for |
|---|---|---|
| `VITE_GRPC_WEB_URL` | `http://localhost:8081` (Envoy) | `src/lib/transport.ts` builds one `createGrpcWebTransport`, and `src/lib/clients.ts` creates a client per service on it: Catalog, TaxonomyAdmin, Learning, Discussion, Progress |
| `VITE_WS_URL` | `ws://localhost:8080/ws` (the API itself) | STOMP discussion. It bypasses Envoy, because gRPC-Web cannot do bidirectional streaming. |

Envoy is configured in the API repo, in `infra/envoy/envoy.yaml`:

| Setting | Why |
|---|---|
| Listener `0.0.0.0:8081` | The browser's gRPC-Web endpoint |
| `envoy.filters.http.grpc_web` | Translates gRPC-Web from the browser into gRPC over HTTP/2 |
| `envoy.filters.http.cors`, origins starting `http://localhost` and `http://127.0.0.1` | Lets the page on :5173 call :8081. It allows the gRPC and Connect request headers and exposes `grpc-status`, `grpc-message` and `grpc-status-details-bin`, so the browser can read errors. |
| Route `/` → cluster `prep_studio_api` at `host.docker.internal:9090`, HTTP/2 | Sends every service to the API |
| `timeout: 0s`, `stream_idle_timeout: 0s`, `grpc_timeout_header_max: 0s` | Turns off the time limits so long explanation streams are not cut off |
| Admin `:9901` | Envoy stats, clusters and config dump, for debugging |

For production, the CORS origins, the upstream address (an API service name instead of `host.docker.internal`) and TLS all need changing.

## Built vs planned

| From DESIGN.md | Status |
|---|---|
| Catalog, Learning, Discussion, Progress contexts | Built |
| Outbox → Kafka | Built (Kafka optional) |
| Envoy for gRPC-Web | Built |
| Job-link prep | Built as the separate Jobs service; not in the web app yet |
| Identity (users, login via Keycloak/Auth0) | Not built: learners are identified by a display name |
| Redis for WebSocket fan-out and caching | Not built: the in-memory STOMP broker works for one API instance only |
| Separate discussion WebSocket gateway | Not built: the API serves STOMP itself |
| `TopicViewed`, `AiReplyPosted`, `KeyDraftGenerated`, bookmarks and confidence in Progress | Not built |

Scaling past one API instance needs Redis (or a STOMP broker relay) first, because subscribers on one instance would not see messages posted on another.
