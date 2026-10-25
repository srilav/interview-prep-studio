# Interview Prep Studio: Design

**Stack:** React (TypeScript) frontend, Java 21 + Spring Boot + gRPC backend, WebSocket for live discussion, Claude Haiku 5.5 (`claude-haiku-5-5`) as the LLM, Domain-Driven Design with published domain events.

## 1. Architecture

```
┌──────────────────────── React SPA (Vite + TS) ────────────────────────┐
│  Topic Search │ Explanation Viewer │ Discussion │ Admin: Prep Studio  │
└───────┬──────────────────────────────┬───────────────────────────┬────┘
        │ gRPC-Web (unary + server-stream)                          │ WebSocket (STOMP)
        ▼                                                           ▼
   ┌─────────┐                                            ┌──────────────────┐
   │  Envoy  │  gRPC-Web → gRPC                           │ Discussion WS GW │
   └────┬────┘                                            └────────┬─────────┘
        ▼                                                          │
┌──────────────────── Spring Boot (modular monolith) ──────────────┴──────┐
│  catalog        │  learning (AI)      │  discussion      │  progress    │
│  (taxonomy,keys)│  (explanations)     │  (threads/msgs)  │  (study log) │
│  ────────────── domain events via Outbox → Kafka ───────────────────────│
└───────┬────────────────┬──────────────────────┬──────────────────────────┘
        ▼                ▼                      ▼
   PostgreSQL       Claude API (Haiku)      Redis (WS fan-out, cache)
```

- Browsers cannot speak native gRPC. Envoy translates gRPC-Web. gRPC-Web supports unary and **server streaming** (enough to stream LLM output) but not client or bidirectional streaming, so discussion uses WebSocket.
- Start as a **modular monolith**: one module per bounded context with its own package and tables. Modules communicate only through application services and events, so they can be split into services later.
- **Transactional outbox**: events are written in the same transaction as the aggregate change and relayed to Kafka, so no event is lost.

## 2. Bounded contexts

| Context | Responsibility | Aggregates | Events published |
|---|---|---|---|
| **Catalog** | Taxonomy (Language/Platform › Category › Subcategory), keys and definitions, search | `Platform` (→ `Category` → `Subcategory`), `Key` | `PlatformAdded`, `CategoryAdded`, `SubcategoryAdded`, `KeyCreated`, `KeyUpdated`, `KeyMoved`, `KeyPublished`, `KeyDeleted`, `TopicViewed` |
| **Learning** | AI-generated explanations and Java examples, caching, versioning | `Explanation` (→ `CodeExample`, `KeyPoint`) | `ExplanationRequested`, `ExplanationGenerated`, `ExplanationGenerationFailed`, `KeyDraftGenerated` |
| **Discussion** | Threads per topic or question, messages, AI tutor replies | `DiscussionThread` (→ `Message`) | `ThreadOpened`, `MessagePosted`, `AiReplyRequested`, `AiReplyPosted` |
| **Progress** | Studied topics, bookmarks, confidence | `StudyPlan`, `StudySession` | `TopicMarkedLearned`, `BookmarkAdded` |
| **Identity** | Users and auth (delegate to Keycloak/Auth0) | `User` | `UserRegistered` |

Context map: Learning conforms to Catalog (reads `KeyId` / `SubcategoryId`). Progress subscribes to events from the other contexts. The Claude API sits behind an anti-corruption layer (`LlmGateway` port) so the domain never sees SDK types.

### Ubiquitous language

- **Platform**: a language, framework, platform or tool (Java, Spring Boot, Apache Kafka, AWS).
- **Category / Subcategory**: grouping under a platform (Java › Collections › Map).
- **Key**: an interview term or concept with a definition (HashMap, `volatile`, Consumer group).
- **Explanation**: structured AI content for a key: summary, key points, complexity, pitfalls, Java example.
- **Difficulty**: `EASY`, `MEDIUM`, `HARD`.
- **Thread**: a discussion about a key or a specific interview question.
- **AI Tutor**: Haiku replying in threads.

## 3. Catalog: taxonomy and keys

### Aggregates and invariants

- `Platform` is the aggregate root for its `Category` and `Subcategory` children (fixed depth of three).
  - Names are unique among siblings (case-insensitive).
  - `kind` is one of `LANGUAGE`, `FRAMEWORK`, `PLATFORM`, `TOOL`.
- `Key` is its own aggregate, referencing `SubcategoryId`, so editing a key never locks the tree.
  - `term` is unique within a subcategory.
  - `shortDefinition` is required and at most 280 characters.
  - Status flow: `DRAFT → IN_REVIEW → PUBLISHED`. Publishing requires a short definition.
  - An AI draft always sets status to `DRAFT` and fills only empty fields.

```java
public class Key extends AggregateRoot<KeyId> {
    private SubcategoryId subcategoryId;
    private String term;
    private List<String> aliases;
    private String shortDefinition;     // ≤ 280 chars
    private String detail;              // markdown
    private String javaExample;
    private Difficulty difficulty;
    private KeyStatus status;
    private Set<String> tags;
    private Set<String> relatedTerms;

    public void moveTo(SubcategoryId target) {
        if (target.equals(subcategoryId)) return;
        var from = subcategoryId;
        subcategoryId = target;
        registerEvent(new KeyMoved(id(), from, target, Instant.now()));
    }

    public void publish() {
        if (shortDefinition == null || shortDefinition.isBlank())
            throw new DomainException("A key needs a short definition before it can be published");
        status = KeyStatus.PUBLISHED;
        registerEvent(new KeyPublished(id(), subcategoryId, Instant.now()));
    }
}
```

Term uniqueness across `Key` aggregates is enforced in the application service plus a DB unique index on `(subcategory_id, lower(term))`.

### Admin UI

See [`prototype/prep-studio.html`](../prototype/prep-studio.html): taxonomy tree with inline add at each level, key list with search and status filters, key editor with placement (move), learner preview, and "Draft with Haiku".

## 4. Learning: explanation aggregate

```java
public class Explanation extends AggregateRoot<ExplanationId> {
    private final KeyId keyId;
    private final Difficulty level;
    private ExplanationStatus status;          // REQUESTED, GENERATING, READY, FAILED
    private String summary;
    private List<KeyPoint> keyPoints = new ArrayList<>();
    private List<CodeExample> codeExamples = new ArrayList<>();
    private int version;

    public static Explanation request(KeyId keyId, Difficulty level, UserId by) {
        var e = new Explanation(ExplanationId.newId(), keyId, level);
        e.status = ExplanationStatus.REQUESTED;
        e.registerEvent(new ExplanationRequested(e.id(), keyId, level, by, Instant.now()));
        return e;
    }

    public void complete(GeneratedContent content) {
        if (content.codeExamples().stream().noneMatch(c -> c.language() == Language.JAVA))
            throw new DomainException("Explanation must include a Java example");
        this.summary = content.summary();
        this.keyPoints = content.keyPoints();
        this.codeExamples = content.codeExamples();
        this.status = ExplanationStatus.READY;
        this.version++;
        registerEvent(new ExplanationGenerated(id(), keyId, version, Instant.now()));
    }
}
```

### Shared kernel

```java
public abstract class AggregateRoot<ID> {
    private final List<DomainEvent> events = new ArrayList<>();
    protected void registerEvent(DomainEvent e) { events.add(e); }
    public List<DomainEvent> pullEvents() { var c = List.copyOf(events); events.clear(); return c; }
}
```

### Outbox

```java
@Transactional
public void save(Explanation agg) {
    jpaRepo.save(mapper.toEntity(agg));
    agg.pullEvents().forEach(e -> outboxRepo.save(OutboxRecord.of(e)));   // same transaction
}

@Scheduled(fixedDelay = 500)
void relay() {
    outboxRepo.findUnpublished(100).forEach(r -> {
        kafka.send(r.topic(), r.aggregateId(), r.payload());
        r.markPublished();
    });
}
```

## 5. Package layout

```
backend/
├─ proto/                 shared .proto files (also generate TS clients)
├─ shared-kernel/         AggregateRoot, DomainEvent, IDs, Outbox
├─ catalog/
│  ├─ domain/             Platform, Category, Subcategory, Key, repositories (ports)
│  ├─ application/        CreatePlatform, UpsertKey, MoveKey, SearchKeys use cases
│  ├─ infrastructure/     JPA repositories, Postgres full-text search
│  └─ api/grpc/           CatalogGrpcService, TaxonomyAdminGrpcService
├─ learning/              Explanation, LlmGateway (port), ClaudeLlmGateway (adapter)
├─ discussion/            DiscussionThread, Message, STOMP controller
└─ progress/
frontend/
├─ src/gen/               generated by buf (connect-es)
└─ src/features/          search, explanation, discussion, progress, admin
```

Dependencies point inward: `api → application → domain ← infrastructure`. Enforce with ArchUnit.

## 6. gRPC contracts

```proto
// proto/catalog/v1/taxonomy_admin.proto
syntax = "proto3";
package catalog.v1;

service TaxonomyAdminService {
  rpc ListTree(ListTreeRequest) returns (Tree);
  rpc CreatePlatform(CreatePlatformRequest) returns (Platform);
  rpc CreateCategory(CreateCategoryRequest) returns (Category);
  rpc CreateSubcategory(CreateSubcategoryRequest) returns (Subcategory);
  rpc UpsertKey(UpsertKeyRequest) returns (Key);
  rpc MoveKey(MoveKeyRequest) returns (Key);
  rpc DeleteKey(DeleteKeyRequest) returns (DeleteKeyResponse);
  rpc GenerateKeyDraft(GenerateKeyDraftRequest) returns (stream KeyDraftChunk);
}

enum PlatformKind { PLATFORM_KIND_UNSPECIFIED = 0; LANGUAGE = 1; FRAMEWORK = 2; PLATFORM = 3; TOOL = 4; }
enum Difficulty { DIFFICULTY_UNSPECIFIED = 0; EASY = 1; MEDIUM = 2; HARD = 3; }
enum KeyStatus { KEY_STATUS_UNSPECIFIED = 0; DRAFT = 1; IN_REVIEW = 2; PUBLISHED = 3; }

message Platform { string id = 1; string name = 2; PlatformKind kind = 3; repeated Category categories = 4; }
message Category { string id = 1; string name = 2; repeated Subcategory subcategories = 3; }
message Subcategory { string id = 1; string name = 2; int32 key_count = 3; }
message Tree { repeated Platform platforms = 1; }

message Key {
  string id = 1; string subcategory_id = 2; string term = 3; repeated string aliases = 4;
  string short_definition = 5; string detail = 6; string java_example = 7;
  Difficulty difficulty = 8; KeyStatus status = 9; repeated string tags = 10; repeated string related_terms = 11;
}

message ListTreeRequest {}
message CreatePlatformRequest { string name = 1; PlatformKind kind = 2; }
message CreateCategoryRequest { string platform_id = 1; string name = 2; }
message CreateSubcategoryRequest { string platform_id = 1; string category_id = 2; string name = 3; }
message UpsertKeyRequest { Key key = 1; }
message MoveKeyRequest { string key_id = 1; string target_subcategory_id = 2; }
message DeleteKeyRequest { string key_id = 1; }
message DeleteKeyResponse {}
message GenerateKeyDraftRequest { string subcategory_id = 1; string term = 2; Difficulty difficulty = 3; }
message KeyDraftChunk { oneof payload { string markdown_delta = 1; Key completed = 2; string error = 3; } }
```

```proto
// proto/learning/v1/learning.proto
service LearningService {
  rpc GetExplanation(GetExplanationRequest) returns (Explanation);
  rpc StreamExplanation(GetExplanationRequest) returns (stream ExplanationChunk);
  rpc AskFollowUp(FollowUpRequest) returns (stream ExplanationChunk);
}
message GetExplanationRequest { string key_id = 1; catalog.v1.Difficulty level = 2; bool regenerate = 3; }
message ExplanationChunk { oneof payload { string markdown_delta = 1; Explanation completed = 2; string error = 3; } }
message Explanation {
  string id = 1; string key_id = 2; string summary = 3; repeated string key_points = 4;
  repeated CodeExample code_examples = 5; string time_complexity = 6;
  repeated string common_pitfalls = 7; repeated string interview_questions = 8;
}
message CodeExample { string title = 1; string language = 2; string code = 3; string explanation = 4; }
```

Java stubs: `protoc-gen-grpc-java`. TypeScript: `buf` + `@connectrpc/connect-web` (gRPC-Web transport to Envoy).

## 7. LLM integration (anti-corruption layer)

```java
public interface LlmGateway {
    Flux<String> streamExplanation(KeySnapshot key, Difficulty level);
    Flux<String> streamKeyDraft(String term, TaxonomyPath path, Difficulty level);
    Flux<String> streamTutorReply(ThreadContext ctx, String question);
}
```

```java
@Component
class ClaudeLlmGateway implements LlmGateway {
    private static final String MODEL = "claude-haiku-5-5";
    private final AnthropicClient client;   // AnthropicOkHttpClient.fromEnv()

    @Override
    public Flux<String> streamExplanation(KeySnapshot key, Difficulty level) {
        var params = MessageCreateParams.builder()
            .model(MODEL)
            .maxTokens(16000L)
            .system(SYSTEM_PROMPT)
            .addUserMessage("Key: %s\nPath: %s\nLevel: %s".formatted(key.term(), key.path(), level))
            .build();

        return Flux.using(
            () -> client.messages().createStreaming(params),
            resp -> Flux.fromStream(resp.stream()
                .flatMap(ev -> ev.contentBlockDelta().stream())
                .flatMap(d -> d.delta().text().stream())
                .map(t -> t.text())),
            StreamResponse::close
        ).subscribeOn(Schedulers.boundedElastic());
    }
}
```

- Persist generated explanations; regenerate only on request.
- Cache the static system prompt (prompt caching).
- Resilience4j circuit breaker and timeout around the gateway; per-user rate limit on generation.

## 8. Discussion over WebSocket

STOMP over WebSocket (Spring `@MessageMapping`), Redis broker relay for horizontal scale.

| Direction | Destination | Payload |
|---|---|---|
| Client → server | `/app/threads/{threadId}/messages` | `{ content, mentionsAi }` |
| Server → clients | `/topic/threads/{threadId}` | `MessagePosted`, `AiReplyChunk`, `AiReplyPosted` |
| Server → clients | `/topic/threads/{threadId}/presence` | typing, joins |

Flow for `@tutor`: `PostMessageUseCase` → `DiscussionThread.post()` emits `MessagePosted` and `AiReplyRequested` → outbox → `AiTutorReplyHandler` builds context (key, explanation summary, last N messages) → streams chunks to `/topic/threads/{id}` → `thread.postAiReply()` emits `AiReplyPosted`.

Invariants: message ≤ 4000 characters, closed threads reject posts, AI replies rate-limited (5 per user per minute), only the AI tutor authors AI replies.

## 9. Event envelope

```json
{ "eventId": "uuid", "type": "catalog.KeyPublished", "version": 1,
  "aggregateId": "key-123", "occurredAt": "2026-10-14T10:00:00Z",
  "payload": { "subcategoryId": "s-map" } }
```

Kafka topics per context (`catalog.events`, `learning.events`, `discussion.events`), partitioned by aggregate ID to keep per-aggregate ordering.

## 10. Cross-cutting

- Local dev: docker-compose with Postgres, Envoy and Redpanda (Redpanda only with `--profile kafka`). Redis and Keycloak join it when WebSocket fan-out and Identity are built. See [ARCHITECTURE.md](ARCHITECTURE.md#what-runs-in-docker-and-what-runs-on-the-host).
- Auth: JWT validated by a gRPC `ServerInterceptor`; STOMP `CONNECT` carries the token. Admin RPCs require an `editor` role.
- Search: Postgres full-text (`tsvector`) on term, aliases and short definition.
- Observability: OpenTelemetry across gRPC, events and LLM calls; log token usage.
- Tests: pure unit tests for aggregates, ArchUnit for module boundaries, Testcontainers for Postgres and Kafka, a fake `LlmGateway`.

## 11. Build order

1. Catalog: taxonomy and keys admin (matches the prototype), search.
2. Learning: streaming explanations through Envoy, with persistence.
3. Outbox and events, Progress projections.
4. Discussion over WebSocket, then the AI tutor.
5. Auth, rate limits, observability.

## 12. Open questions

1. Is a fixed three-level taxonomy enough, or is unlimited nesting needed?
2. Can a key belong to more than one subcategory, or is "related keys" enough?
3. Keep the Draft → In review → Published workflow, or only Draft and Published?
4. One Java example per key, or several?
5. Where should rename, reorder and delete live in the admin UI?
