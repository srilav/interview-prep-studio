# Interview Prep Studio

A self-study application for technical interviews. Look up a topic, get an explanation with a Java example, and discuss questions with other learners and an AI tutor (Claude Haiku).

## Repositories

| Repo | What it is |
|---|---|
| [interview-prep-studio-api](https://github.com/srilav/interview-prep-studio-api) | Backend: Java 21, Spring Boot, gRPC, DDD bounded contexts, domain events (outbox → Kafka), WebSocket discussion, Claude Haiku. Owns the `.proto` contracts. |
| [interview-prep-studio-web](https://github.com/srilav/interview-prep-studio-web) | Frontend: React + TypeScript, gRPC-Web client generated from the API's protos, live discussion over STOMP. |
| interview-prep-studio-jobs (local only, not on GitHub yet) | Job-link prep: reads a job posting, extracts skills with Claude Haiku, matches them to the catalog and streams a study plan. Separate Spring Boot gRPC service on port 9091. |
| this repo | Architecture, design and the original UI prototype |

| Path | What it is |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | The system as built: each part and its role, the API's modules, the event flow, request flows, ports, and what is still only planned |
| [`docs/DESIGN.md`](docs/DESIGN.md) | Architecture: bounded contexts (DDD), domain events, gRPC contracts, WebSocket discussion, LLM integration |
| [`prototype/prep-studio.html`](prototype/prep-studio.html) | Clickable UI prototype of the content admin screen. Open it in a browser; it runs on sample data and nothing is saved. |

## Stack

- **Frontend:** React + TypeScript (Vite), Connect-Web gRPC-Web client, STOMP over WebSocket
- **Backend:** Java 21, Spring Boot, gRPC (behind Envoy for gRPC-Web), modular monolith organised by bounded context
- **Events:** transactional outbox relayed to Kafka
- **Storage:** PostgreSQL (Redis planned for scaling WebSocket fan-out)
- **LLM:** Claude Haiku 5.5 (`claude-haiku-5-5`) via the Anthropic Java SDK, behind an anti-corruption layer

## Prototype

```bash
open prototype/prep-studio.html
```

The prototype shows the three-pane admin screen: Language/Platform › Category › Subcategory tree, key list with search and status filters, and a key editor with a learner preview.
