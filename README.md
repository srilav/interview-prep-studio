# Interview Prep Studio

A self-study application for technical interviews. Look up a topic, get an explanation with a Java example, and discuss questions with other learners and an AI tutor (Claude Haiku).

## Status

Design phase. This repository currently holds:

| Path | What it is |
|---|---|
| [`docs/DESIGN.md`](docs/DESIGN.md) | Architecture: bounded contexts (DDD), domain events, gRPC contracts, WebSocket discussion, LLM integration |
| [`prototype/prep-studio.html`](prototype/prep-studio.html) | Clickable UI prototype of the content admin screen (taxonomy and keys). Open it in a browser; it runs on sample data and nothing is saved. |

## Planned stack

- **Frontend:** React + TypeScript (Vite), Connect-Web gRPC-Web client, STOMP over WebSocket
- **Backend:** Java 21, Spring Boot, gRPC (behind Envoy for gRPC-Web), modular monolith organised by bounded context
- **Events:** transactional outbox relayed to Kafka
- **Storage:** PostgreSQL, Redis
- **LLM:** Claude Haiku 5.5 (`claude-haiku-5-5`) via the Anthropic Java SDK, behind an anti-corruption layer

## Prototype

```bash
open prototype/prep-studio.html
```

The prototype shows the three-pane admin screen: Language/Platform › Category › Subcategory tree, key list with search and status filters, and a key editor with a learner preview.
