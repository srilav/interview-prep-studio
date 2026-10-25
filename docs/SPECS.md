# Interview Prep Studio: Specification

| | |
|---|---|
| Version | 1.0 (2026-10-25) |
| Status | Baseline: describes the system as built, with gaps called out |
| Covers | interview-prep-studio-api, interview-prep-studio-web, interview-prep-studio-jobs |
| Related | [ARCHITECTURE.md](ARCHITECTURE.md) (parts and flows), [DESIGN.md](DESIGN.md) (full design, including planned work) |

This is the source of truth for **what** the system must do. Change it first, then the design, tests and code (see [§13 Change process](#13-change-process)). Requirements are numbered (`FR-`, `NFR-`), and every requirement has acceptance criteria (`AC-`) that name the test which verifies them, or are marked **Gap** when no test does yet.

---

## 1. Overview

A self-study application for technical interviews. A learner looks up an interview topic or question, reads its definition and Java example, gets a longer explanation written by Claude Haiku, discusses it live with other learners and an AI tutor, and sees their study activity. Editors maintain the content in Prep Studio. A separate Jobs service turns a job-posting link into a study plan based on the catalog.

### Goals
- G1: A learner finds a topic in under three keystrokes of search, from a curated catalog.
- G2: Every topic has a trustworthy short answer, and an AI explanation that always includes Java code.
- G3: Learners can ask follow-up questions in context, with an AI tutor, without leaving the topic.
- G4: Editors can grow the catalog without a developer, with AI help for first drafts.
- G5: A learner can see what to study for a specific job.

### Non-goals (this version)
- User accounts, login, roles (Identity is planned; see [§12](#12-open-questions-and-planned-work)).
- Spaced repetition, quizzes or scoring.
- Applying for jobs on the learner's behalf: the Jobs service only reads postings.
- Mobile apps; multi-instance deployment of the API.
- Languages other than English for content.

---

## 2. Glossary

Use these terms exactly, in code, API, UI and conversation.

| Term | Meaning |
|---|---|
| **Platform** | A language, framework, platform or tool at the top of the taxonomy (Java, Spring Boot, Apache Kafka, Databases). Has a **kind**: `LANGUAGE`, `FRAMEWORK`, `PLATFORM`, `TOOL`. |
| **Category** | A group under a platform (Java › Collections). |
| **Subcategory** | A group under a category (Java › Collections › Map). Keys live in subcategories. |
| **Taxonomy** | The Platform › Category › Subcategory tree. |
| **Key** | One interview term, concept or question, with its content. The unit of study. |
| **Question key** | A key whose term is an interview question and whose short definition is the short answer. Same entity as Key. |
| **Term** | The key's name or question, as shown in search. |
| **Short definition** | The key's one-paragraph answer (≤ 280 characters). Required. |
| **Detail** | The key's longer plain-text explanation, written by an editor. |
| **Java example** | Code shown with the key. |
| **Alias** | Another name a learner might search for (`Comparator.comparing` for Comparator). |
| **Key path** | Platform › Category › Subcategory of a key. |
| **Key status** | `DRAFT` → `IN_REVIEW` → `PUBLISHED`. Only published keys are visible to learners. |
| **Difficulty / level** | `EASY`, `MEDIUM`, `HARD`. On a key, how hard the topic is; on an explanation, how deep it goes. |
| **Explanation** | AI-generated markdown about a key at one level, stored after generation. |
| **Revision** | The number of times an explanation has been generated successfully. |
| **Regenerate** | Ask for a new explanation even though one is stored. |
| **Key draft** | AI-suggested content for a new key, which an editor reviews before saving. |
| **Thread** | A discussion attached to a key, with a title. |
| **Message** | One post in a thread, by a learner or the AI tutor. |
| **AI tutor** | Claude replying in a thread when a message mentions `@tutor`. Its author name is "AI Tutor", reserved. |
| **Learner** | A person studying, identified by a display name (no account yet). |
| **Editor** | A person maintaining content in Prep Studio (same app, no role check yet). |
| **Study activity** | A record that a learner viewed an explanation or posted a message. |
| **Job posting** | A job ad a learner adds by link or pasted text, in the Jobs service. |
| **Skill match** | One skill from a posting with its **coverage** in the catalog: `COVERED`, `PARTIAL`, `MISSING`, `UNKNOWN`. |
| **Prep plan** | A study plan for one posting, streamed by Claude and stored. |
| **Domain event** | A fact that happened (`KeyPublished`), recorded with the change that caused it. |
| **Outbox** | The table where domain events are stored in the same transaction as the change, before delivery. |

---

## 3. Domain entities

```
Platform 1──* Category 1──* Subcategory 1──* Key 1──* Explanation (one per level)
                                              │
                                              └──* DiscussionThread 1──* Message
Learner ─(by name)─ StudyActivity  ◄── projected from events
Learner ─(by name)─ JobPosting 1──* SkillMatch ──(SearchKeys)──► Key
                         └──0..1 PrepPlan
OutboxEvent ◄── every aggregate change that raises an event
```

| Entity | Context | Identity | Key attributes | Invariants | Table |
|---|---|---|---|---|---|
| Platform (aggregate) | Catalog | id | name, kind, categories | name 1–120 chars; kind required; names unique ignoring case | `platform` |
| Category | Catalog | id | name, position, subcategories | name 1–120; unique among siblings ignoring case | `category` |
| Subcategory | Catalog | id | name, position | name 1–120; unique among siblings ignoring case | `subcategory` |
| Key (aggregate) | Catalog | id | term, aliases, short definition, detail, Java example, difficulty, status, tags, related terms | term 1–160 chars; short definition 1–280; term unique in its subcategory ignoring case; lists trimmed, blank and duplicate entries removed; difficulty defaults to `MEDIUM`; new keys start as `DRAFT` | `catalog_key` |
| Explanation (aggregate) | Learning | id; unique (key, level) | status, markdown, revision, failure reason | content must contain a ```` ```java ```` block; one generation at a time per (key, level); a failed regeneration keeps the previous content | `explanation` |
| DiscussionThread (aggregate) | Discussion | id | key, title, opened by, closed | title 1–200 chars; no posts when closed | `discussion_thread` |
| Message | Discussion | id | author, from AI tutor, content, posted at | content 1–4000 chars; author name required and not "AI Tutor" for learners | `discussion_message` |
| StudyActivity | Progress | id | learner, key, kind, at | written only from events | `study_activity` |
| OutboxEvent | Shared kernel | id | type, aggregate id, payload, occurred at, published at | written in the same transaction as the change | `outbox_event` |
| JobPosting (aggregate) | Jobs | id; unique (learner, URL hash) | source URL, status, application status, text, profile, matches, plan | text 200–60,000 chars; learner name 1–80; state machine in [FR-J2](#jobs-service) | `jobs.*` |

---

## 4. Assumptions and constraints

- A1: One API instance. WebSocket fan-out and the tutor rate limit are in memory.
- A2: No authentication. The display name a learner types is trusted.
- A3: Claude (`claude-haiku-5-5`) is reachable over HTTPS; `LLM_PROVIDER=fake` replaces it for offline work and tests.
- C1: Java 21, Spring Boot 3.3, PostgreSQL 16, gRPC with `.proto` files as the contract.
- C2: Browsers reach gRPC through Envoy (gRPC-Web), which supports unary and server streaming only. Bidirectional discussion uses STOMP over WebSocket.
- C3: Content shipped with the product is delivered as Flyway migrations, so every environment gets the same catalog.

---

## 5. Functional requirements

### Catalog
- **FR-1 Taxonomy.** Editors can create platforms (with a kind), categories and subcategories, and view the full tree with key counts per subcategory.
- **FR-2 Key editing.** Editors can create and edit a key in a subcategory, move it to another subcategory, change its status, and delete it.
- **FR-3 Publishing.** A key becomes visible to learners only when its status is `PUBLISHED`.
- **FR-4 Search.** Learners can search published keys by term or alias, case-insensitively, by substring, optionally within one platform. Results are ordered by term; 20 by default, at most 100.
- **FR-5 Key page.** Learners can open a key and see its path, term, short definition, detail and Java example.
- **FR-6 AI key draft.** Editors can ask for an AI draft of a key's short definition, detail and Java example, which they review before saving.

### Learning
- **FR-7 Streamed explanation.** When a learner opens a key, the system streams an explanation at the chosen level (default `MEDIUM`) as it is generated, then stores it.
- **FR-8 Reuse.** Later requests for the same key and level return the stored explanation without calling the model, unless the learner asks to regenerate.
- **FR-9 Explanation quality.** An explanation is accepted only if it includes Java code.

### Discussion
- **FR-10 Threads.** Learners can open a titled thread on a key, list a key's threads and read a thread's messages.
- **FR-11 Live messages.** A posted message is stored and delivered to everyone subscribed to the thread, without a page reload.
- **FR-12 AI tutor.** When a message mentions `@tutor`, the AI tutor replies in the same thread, streamed to subscribers, at most 5 requests per learner per minute.

### Progress
- **FR-13 Activity summary.** A learner can see how many explanations they viewed, how many messages they posted, and their recent activity.

### Platform behaviour
- **FR-14 Domain events.** Every catalog, learning and discussion change listed in [ARCHITECTURE.md](ARCHITECTURE.md#bounded-contexts-maven-modules) raises a domain event stored in the outbox with the change; with Kafka enabled, events are relayed to `catalog.events`, `learning.events` and `discussion.events`, keyed by aggregate id.
- **FR-15 Errors.** Business-rule violations reach the client as gRPC status codes with a readable message: invalid input `INVALID_ARGUMENT`, duplicates `ALREADY_EXISTS`, missing items `NOT_FOUND`, concurrent edits `ABORTED`, model unavailable `UNAVAILABLE`.

### Jobs service
- **FR-J1 Add postings.** A learner can add job postings by link or by pasted text. The same link twice for the same learner is one posting (tracking parameters, fragment and host case are ignored).
- **FR-J2 Pipeline.** A posting moves `FETCHING` → `EXTRACTING` → `READY`. A page that needs sign-in or JavaScript moves to `NEEDS_TEXT` until the learner pastes the text; errors move to `FAILED`, and the learner can retry.
- **FR-J3 Skill matching.** For a ready posting, each extracted skill (at most 30) is matched to the catalog as `COVERED`, `PARTIAL`, `MISSING`, or `UNKNOWN` when the catalog is unreachable.
- **FR-J4 Prep plan.** A learner can stream a study plan for a ready posting; it is stored and served from storage afterwards.
- **FR-J5 Application status.** A learner can set `TO_APPLY`, `APPLIED`, `INTERVIEWING` or `CLOSED` on a posting. The system never submits applications.

## 6. Non-functional requirements

- **NFR-1 Streaming latency.** The first explanation or tutor chunk reaches the browser within 3 s of the request at p95 (model time included).
- **NFR-2 Read latency.** Search and GetKey respond within 300 ms at p95 with the shipped catalog.
- **NFR-3 No lost events.** A committed change always has its event in the outbox; a rolled-back change never does.
- **NFR-4 No lost updates.** Concurrent edits to the same aggregate never silently overwrite each other (optimistic locking with `lock_version`).
- **NFR-5 Cost control.** The model is called at most once per (key, level) unless regeneration is requested; tutor requests are rate-limited (FR-12).
- **NFR-6 Offline development.** With `LLM_PROVIDER=fake` and no Kafka, the whole system runs and all tests pass without network access to external services.
- **NFR-7 Security (Jobs).** The Jobs fetcher only reads public `http`/`https` URLs on standard ports, never private or local network addresses, and rejects URLs containing credentials.
- **NFR-8 Security (web).** Envoy CORS and WebSocket origins accept only configured origins (`http://localhost*` and `ALLOWED_ORIGINS` locally).
- **NFR-9 Contract compatibility.** `.proto` changes are backward compatible: fields are added, never renumbered or removed.
- **NFR-10 Module boundaries.** Contexts never read each other's tables or aggregates; they use `CatalogQueries` and events.

---

## 7. Acceptance criteria

Status: **Test** = verified by the named automated test · **Gap** = no automated test yet · **Manual** = checked by hand.

Test locations: `api-it` = `app/.../PrepStudioApiTest`, `KeyTest`, `PlatformTest`, `KeyDraftServiceTest` (catalog), `ExplanationTest` (learning), `DiscussionThreadTest`, `TutorRateLimiterTest` (discussion); `jobs-it` = `JobsGrpcServiceIntegrationTest`, plus the Jobs unit tests.

### Catalog

| ID | Req | Given / When / Then | Status · verified by |
|---|---|---|---|
| AC-01 | FR-1 | **Given** a fresh database **When** the tree is listed **Then** Java, Spring Boot and Apache Kafka exist and Java › Collections › Map has 3 keys | Test · `api-it.sampleTaxonomyIsLoaded` |
| AC-02 | FR-1 | **Given** an editor **When** they create a platform, category and subcategory **Then** each step raises its event (`PlatformAdded`, `CategoryAdded`, `SubcategoryAdded`) | Test · `PlatformTest.buildsTheTaxonomyAndRaisesAnEventPerStep` |
| AC-03 | FR-1 | **Given** a sibling named "Collections" **When** an editor adds "collections" **Then** it is rejected; through the API, a platform named "java" returns `ALREADY_EXISTS` | Test · `PlatformTest.rejectsDuplicateSiblingNamesIgnoringCase`, `api-it.businessRuleViolationsBecomeGrpcStatusCodes` |
| AC-04 | FR-1 | **When** a platform has no name or no kind **Then** it is rejected | Test · `PlatformTest.requiresNameAndKind` |
| AC-05 | FR-2 | **When** a key is created **Then** it starts as `DRAFT`, its text is trimmed and its lists drop blank and duplicate entries | Test · `KeyTest.createsADraftAndNormalisesContent` |
| AC-06 | FR-2 | **When** the short definition is empty or longer than 280 characters **Then** the key is rejected with how much to shorten it; through the API, `INVALID_ARGUMENT` | Test · `KeyTest.enforcesTheShortDefinitionRules`, `api-it.businessRuleViolationsBecomeGrpcStatusCodes` |
| AC-07 | FR-2 | **Given** a key "HashMap" in a subcategory **When** another key "hashmap" is created there **Then** `ALREADY_EXISTS` | **Gap** |
| AC-08 | FR-2 | **When** a key moves to another subcategory **Then** `KeyMoved` is raised; moving to the same subcategory does nothing | Test · `KeyTest.movingToAnotherSubcategoryRaisesKeyMoved` |
| AC-09 | FR-2 | **When** a key is deleted **Then** it no longer appears in search or the tree count and `KeyDeleted` is raised | **Gap** |
| AC-10 | FR-3 | **When** a key is published twice **Then** `KeyPublished` is raised only the first time | Test · `KeyTest.publishingRaisesKeyPublishedOnlyOnce` |
| AC-11 | FR-3, FR-4 | **Given** a `DRAFT` key **When** a learner searches its exact term **Then** it is not returned | **Gap** |
| AC-12 | FR-4 | **Given** a published key "\_\_eq\_\_" under Python **When** a learner searches "DUNDER" within Python **Then** exactly that key is returned (case-insensitive alias match) | Test · `api-it.editorBuildsTaxonomyAndPublishesAKeyThatLearnersCanFind` |
| AC-13 | FR-4 | **When** a search asks for 0 results or 500 **Then** 20 and 100 results are returned at most | **Gap** |
| AC-14 | FR-5 | **When** a key id does not exist **Then** `GetKey` returns `NOT_FOUND` | Test · `api-it.businessRuleViolationsBecomeGrpcStatusCodes` |
| AC-15 | FR-6 | **Given** a model reply with the three sections **When** a draft is requested **Then** short definition, detail and example are filled; an unformatted reply goes into detail; an overlong short definition is trimmed to 280 | Test · `KeyDraftServiceTest.*` (3 tests) |

### Learning

| ID | Req | Given / When / Then | Status · verified by |
|---|---|---|---|
| AC-16 | FR-7 | **When** a learner opens a key with no stored explanation **Then** more than two chunks stream, the final one contains ```` ```java ```` and revision 1 | Test · `api-it.explanationIsStreamedOnceThenServedFromStoreAndCountsTowardProgress` |
| AC-17 | FR-8, NFR-5 | **When** the same key and level are requested again **Then** one message returns the stored explanation with the same id, and the model is not called | Test · same test as AC-16 |
| AC-18 | FR-8 | **Given** a stored explanation **When** regeneration fails **Then** the previous content is still served | Test · `ExplanationTest.aFailedRegenerationKeepsServingThePreviousRevision` |
| AC-19 | FR-9 | **When** the model returns text without a Java code block **Then** the explanation is rejected and not stored as ready | Test · `ExplanationTest.rejectsContentWithoutJava`, `completesWhenTheContentHasAJavaExample` |
| AC-20 | FR-7, NFR-5 | **Given** a generation in progress **When** a second request for the same key and level arrives **Then** it is refused instead of calling the model twice | Test · `ExplanationTest.refusesASecondConcurrentGeneration` |
| AC-21 | FR-7 | **Given** a generation stuck for more than 5 minutes **When** a new request arrives **Then** it takes over | **Gap** |
| AC-22 | FR-15 | **Given** the model is unreachable **When** an explanation is requested **Then** the client gets `UNAVAILABLE` with "The AI model is unavailable right now" | **Gap** |

### Discussion

| ID | Req | Given / When / Then | Status · verified by |
|---|---|---|---|
| AC-23 | FR-10 | **When** a message is over 4000 characters, the thread is closed, or a learner posts as "AI Tutor" **Then** the post is rejected. (Empty messages and the 1–200 character title rule are not tested yet.) | Test (partial) · `DiscussionThreadTest.enforcesPostingRules` |
| AC-24 | FR-12 | **When** a message contains `@tutor` **Then** an AI reply is requested; without the mention or without quota, none is | Test · `DiscussionThreadTest.mentioningTheTutorRequestsAnAiReply`, `noAiRequestWithoutAMentionOrWithoutQuota` |
| AC-25 | FR-11, FR-12 | **Given** a learner subscribed to a thread **When** they post "@tutor why…" **Then** their message and then the AI tutor's reply appear in the thread | Test · `api-it.tutorRepliesInTheThreadWhenMentioned` |
| AC-26 | FR-12 | **Given** a limit of N per minute (5 by default, `prepstudio.discussion.tutor-requests-per-minute`) **When** a learner makes request N+1 within a minute **Then** it is refused; names compare ignoring case, another learner is unaffected, and requests are allowed again after the minute | Test · `TutorRateLimiterTest.allowsNPerMinutePerLearner` |
| AC-27 | FR-11 | **Given** two browsers on the same thread **When** one posts **Then** the other sees the message without reloading | Manual |

### Progress and platform

| ID | Req | Given / When / Then | Status · verified by |
|---|---|---|---|
| AC-28 | FR-13 | **When** a learner views two explanations **Then** their summary shows 2 explanations viewed | Test · `api-it.explanationIsStreamedOnceThenServedFromStoreAndCountsTowardProgress`, `ExplanationTest.viewingAReadyExplanationIsRecorded` |
| AC-29 | FR-13 | **When** a learner posts a message **Then** their summary's messages-posted count goes up by one | **Gap** |
| AC-30 | FR-14, NFR-3 | **When** an editor creates a platform, category, subcategory and published key **Then** the outbox contains `PlatformAdded`, `CategoryAdded`, `SubcategoryAdded`, `KeyCreated`, `KeyPublished` | Test · `api-it.editorBuildsTaxonomyAndPublishesAKeyThatLearnersCanFind` |
| AC-31 | FR-14 | **Given** `KAFKA_ENABLED=true` **When** a key is published **Then** a record keyed by the key id arrives on `catalog.events` | **Gap** |
| AC-32 | FR-15, NFR-4 | **Given** two editors load the same key **When** both save **Then** the second gets `ABORTED` ("Someone else changed this…") | **Gap** |
| AC-33 | NFR-6 | **Given** `LLM_PROVIDER=fake` and Kafka off **When** `mvn test` runs **Then** all API tests pass | Test · the API suite (the test profile uses the fake LLM with Kafka off). There is no CI pipeline yet. |
| AC-34 | NFR-8 | **Given** an origin not in `ALLOWED_ORIGINS` **When** it opens the WebSocket **Then** the handshake is refused | **Gap** |
| AC-35 | NFR-10 | **When** a module imports another context's `domain` package or repository **Then** the build fails | **Gap** (DESIGN.md plans ArchUnit) |
| AC-36 | C3 | **Given** an empty Postgres **When** the API starts **Then** Flyway applies V1–V5 and the shipped keys are searchable | Test · every `api-it` run on H2; Manual on Postgres |

### Jobs service

| ID | Req | Given / When / Then | Status · verified by |
|---|---|---|---|
| AC-37 | FR-J1 | **When** the same job link is added with different tracking parameters, fragment or host case **Then** it normalizes to the same URL and hash | Test · `SourceUrlTest.removesTrackingParametersFragmentAndHostCase`, `sameJobGivesSameHash` |
| AC-38 | FR-J1, NFR-7 | **When** a link is not http(s) or contains a user name or password **Then** it is rejected | Test · `SourceUrlTest.rejectsNonHttpLinks`, `rejectsCredentialsInLink` |
| AC-39 | NFR-7 | **When** a link points to another scheme, a non-standard port, or a private/local address **Then** the fetcher refuses it; including `169.254.169.254`, `10/8`, `192.168/16`, `::1` and IPv4-mapped loopback; public addresses are allowed | Test · `UrlGuardTest.blocksPrivateAndLocalAddresses`, `blocksOtherSchemesAndPorts`, `allowsPublicAddresses` |
| AC-40 | FR-J2 | **When** a readable link is added **Then** it is fetched, extracted and matched, reaching `READY` | Test · `jobs-it.linkIsFetchedExtractedAndMatched`, `JobPostingTest.happyPathReachesReady` |
| AC-41 | FR-J2 | **When** a page is unreadable **Then** the posting waits in `NEEDS_TEXT`, and pasting text continues the pipeline | Test · `jobs-it.unreadablePageWaitsForPastedText`, `JobPostingTest.unreadablePageAsksForTextThenContinues` |
| AC-42 | FR-J2 | **When** a failed posting is retried **Then** it resumes from extraction if text exists, otherwise refetches | Test · `JobPostingTest.retryResumesFromExtractionWhenTextExists`, `retryRefetchesWhenNoText` |
| AC-43 | FR-J2 | **When** text is under 200 characters or the learner is blank **Then** it is rejected; text over 60,000 is capped | Test · `JobPostingTest.rejectsShortTextAndBlankLearner`, `capsVeryLongText` |
| AC-44 | FR-J3 | **When** skills are matched **Then** they are classified covered/partial/missing, capped at 30, and all `UNKNOWN` after the catalog fails once | Test · `CatalogMatcherTest.*` |
| AC-45 | FR-J4 | **When** a plan is requested for a ready posting **Then** it streams once and is served from storage afterwards; for a posting that is not ready, it is refused | Test · `jobs-it.planIsStreamedOnceThenServedFromStorage`, `JobPostingTest.planNeedsReadyPosting` |
| AC-46 | FR-J5 | **When** a learner sets an application status **Then** it is stored, and nothing is sent to the employer | **Gap** (status change has no dedicated test) |

---

## 8. Traceability

| Requirement | Acceptance criteria | Gaps |
|---|---|---|
| FR-1 Taxonomy | AC-01..04 | |
| FR-2 Key editing | AC-05..09 | AC-07, AC-09 |
| FR-3 Publishing | AC-10, AC-11 | AC-11 |
| FR-4 Search | AC-11..13 | AC-11, AC-13 |
| FR-5 Key page | AC-14 | |
| FR-6 AI key draft | AC-15 | |
| FR-7 Streamed explanation | AC-16, AC-20, AC-21 | AC-21 |
| FR-8 Reuse | AC-17, AC-18 | |
| FR-9 Explanation quality | AC-19 | |
| FR-10 Threads | AC-23 | |
| FR-11 Live messages | AC-25, AC-27 | AC-27 manual |
| FR-12 AI tutor | AC-24..26 | |
| FR-13 Activity summary | AC-28, AC-29 | AC-29 |
| FR-14 Domain events | AC-30, AC-31 | AC-31 |
| FR-15 Errors | AC-03, AC-06, AC-14, AC-22, AC-32 | AC-22, AC-32 |
| FR-J1..J5 | AC-37..46 | AC-46 |
| NFR-1, NFR-2 latency | none yet | needs a load test (§10) |
| NFR-3 No lost events | AC-30 | crash test not automated |
| NFR-4 No lost updates | AC-32 | AC-32 |
| NFR-5 Cost control | AC-17, AC-20, AC-26 | |
| NFR-6 Offline development | AC-33 | |
| NFR-7 Jobs URL safety | AC-38, AC-39 | |
| NFR-8 Origins | AC-34 | AC-34 |
| NFR-9 Contract compatibility | none | add `buf breaking` to CI |
| NFR-10 Module boundaries | AC-35 | AC-35 |

**Coverage:** 46 criteria · 33 automated (AC-23 only partly; AC-36 also checked by hand on Postgres) · 1 manual (AC-27) · **12 gaps**: AC-07, 09, 11, 13, 21, 22, 29, 31, 32, 34, 35, 46. NFR-1, NFR-2 and NFR-9 have no criteria yet.

---

## 9. Interfaces

The `.proto` files are the contract; this section lists them, it does not repeat them.

| Service | RPCs | Owner |
|---|---|---|
| `prepstudio.catalog.v1.CatalogService` | SearchKeys, GetKey, ListKeys | API |
| `prepstudio.catalog.v1.TaxonomyAdminService` | ListTree, CreatePlatform, CreateCategory, CreateSubcategory, UpsertKey, MoveKey, DeleteKey, GenerateKeyDraft | API |
| `prepstudio.learning.v1.LearningService` | StreamExplanation (server streaming; `level`, `regenerate`) | API |
| `prepstudio.discussion.v1.DiscussionService` | OpenThread, ListThreads, ListMessages | API |
| `prepstudio.progress.v1.ProgressService` | GetSummary | API |
| STOMP `/ws` | send `/app/threads/{id}/messages`; subscribe `/topic/threads/{id}` (frames `message`, `ai_chunk`, `ai_error`) | API |
| `jobs.v1` | AddJobPostings, AddJobPostingText, SubmitPostingText, RetryPosting, ListPostings, GetPosting, SetApplicationStatus, StreamPrepPlan (server streaming) | Jobs |

---

## 10. Test strategy

| Level | What | Where |
|---|---|---|
| Domain unit | Aggregate invariants and events, no Spring | `KeyTest`, `PlatformTest`, `ExplanationTest`, `DiscussionThreadTest`, `JobPostingTest`, `SourceUrlTest` |
| Application unit | Services with fakes: draft parsing, rate limit, matching, extraction | `KeyDraftServiceTest`, `TutorRateLimiterTest`, `CatalogMatcherTest`, `PostingExtractorTest`, `SkillNormalizerTest`, `HtmlTextExtractorTest`, `UrlGuardTest` |
| Integration | Full Spring context, H2 in PostgreSQL mode, in-process gRPC, fake LLM | `PrepStudioApiTest`, `JobsGrpcServiceIntegrationTest` |
| Web | Type-check only (`npm run typecheck`) | **Gap**: no component or end-to-end tests |
| CI | **Gap**: no pipeline; tests run locally only | |
| To add | CI running both suites and the web type-check, ArchUnit boundaries (AC-35), Testcontainers Postgres + Kafka (AC-31, AC-36), WebSocket origin test (AC-34), optimistic-lock test (AC-32), Playwright for AC-27, `buf breaking` (NFR-9), a k6 load test for NFR-1/2 | |

Rule: a requirement is **done** when every acceptance criterion it traces to is automated or explicitly marked Manual with a reason.

---

## 11. Known limitations

1. One API instance only: in-memory STOMP broker and in-memory tutor rate limit (A1).
2. No authentication: anyone can edit content and post under any name (A2).
3. Search is substring `LIKE` over term and aliases, not full-text ranking (DESIGN.md plans `tsvector`).
4. The key page shows `detail` as plain text, so markdown in editor content is not rendered (explanations are rendered as markdown).
5. Progress counts only explanation views and messages; no bookmarks or confidence yet.
6. Jobs is not wired into the web app and has no GitHub remote.
7. `./mvnw` fails on machines whose `wget` is broken, and `spring-boot:run` with `-am` fails on the parent POM; the API README's run instructions need updating.

## 12. Open questions and planned work

| # | Question / work | Affects |
|---|---|---|
| Q1 | Identity: Keycloak or Auth0? Do editors need a separate role from learners? | A2, new FR |
| Q2 | Redis pub/sub or a STOMP broker relay for multi-instance fan-out? | A1, FR-11 |
| Q3 | Should regeneration be limited per learner, like the tutor? | NFR-5 |
| Q4 | Should search move to Postgres full-text with ranking, and what latency target then? | FR-4, NFR-2 |
| Q5 | Should the web app render key `detail` as markdown? | FR-5 |
| P1 | Close the 12 gaps in §8, starting with AC-11 (drafts hidden) and AC-32 (lost updates) | §7 |
| P2 | Add the Jobs pages to the web app | FR-J1..J5 |

## 13. Change process

1. Propose the change as an edit to this file: new or changed `FR`/`NFR`, with acceptance criteria.
2. Review and bump the version in the header and the revision history.
3. Update the design (`DESIGN.md`, `ARCHITECTURE.md`) and the `.proto` contracts if they change.
4. Write the failing tests named in the new acceptance criteria, then implement.
5. In the pull request, list the requirement and acceptance-criterion IDs it delivers; update the status column in §7.

If code and spec disagree, the code has a bug or the spec is stale; fix whichever is wrong in the same pull request.

## 14. Revision history

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-10-25 | Baseline spec of the system as built: glossary, entities, FR-1..15, FR-J1..J5, NFR-1..10, AC-01..46 mapped to tests, 12 gaps |
