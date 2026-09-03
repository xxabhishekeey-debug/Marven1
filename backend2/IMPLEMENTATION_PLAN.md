# SIH26094 Backend Implementation Plan

## Repository findings

- The `main` branch contains the authoritative PostgreSQL schema, views, realistic seed data, and a synthetic dataset, but no Java source, build file, API contract, frontend, Docker configuration, or CI workflow.
- `backend/src/main/resources/schema.sql` is byte-for-byte identical to `database/schema.sql`; `backend/src/main/resources/data.sql` is byte-for-byte identical to `database/seed_synthetic.sql`.
- The separate `ai-intelligence-layer` branch contains a FastAPI service with `POST /ai/v1/score` and `POST /ai/v1/transcribe`. It is not merged into `main` and will remain a separate service.
- No PRD or System Architecture file is present in any fetched branch. The database documentation states that its schema was derived from those documents, so the checked-in schema and views remain the implementation source of truth.
- No existing Java package or build convention exists. The new base package will be `in.gov.mosje.sih26094`.

## Existing database mapping

| Table | JPA entity | Important mapping details |
|---|---|---|
| `users` | `User` | UUID, PostgreSQL `user_role`, active flag, BCrypt hash |
| `victims` | `Victim` | UUID, assigned counsellor, jurisdiction fields, current PostgreSQL `risk_tier` |
| `consent` | `Consent` | Append-only grant/withdrawal history; latest entry determines active consent |
| `checkins` | `CheckIn` | Multi-channel PostgreSQL enum, JSONB metadata, missed-check-in signal |
| `scores` | `Score` | Unique 1:1 check-in relation, JSONB signals/factors, validated DDS |
| `alerts` | `Alert` | Human-owned alert workflow and outcome notes |
| `audit_log` | `AuditLog` | Append-only sensitive-operation events with JSONB details |

Hibernate will use `ddl-auto=validate`; it will not create, update, truncate, or recreate the schema. The existing PostgreSQL trigger remains responsible for synchronizing `victims.current_risk_tier` after score insertion. Existing analytical views will be retained, with JPA/native projections used where they provide stable query shapes.

One additive database hardening change is required: a unique index on `alerts.score_id` to make score-to-alert creation idempotent under concurrent Redis retries. It does not rename or remove existing schema objects.

## API contract

- `POST /api/v1/auth/login` — active-user credential verification and JWT issuance.
- `POST /api/v1/consent` — append a grant or withdrawal event and audit it.
- `POST /api/v1/checkins` — validate access and current consent, persist immediately, and publish an asynchronous scoring job.
- `GET /api/v1/victims/{id}/trend` — authorized longitudinal DDS/risk/sentiment data.
- `GET /api/v1/alerts?status=open&page=0&size=20` — paginated, role/jurisdiction filtered alerts.
- `POST /api/v1/alerts/{id}/ack` — acknowledge/escalate/resolve/mark false positive with human notes and audit.
- `GET /api/v1/dashboard/summary` — aggregate-only role-scoped dashboard metrics.
- `/ws/alerts` — authenticated STOMP/WebSocket alert transport using user-specific destinations.
- `/v3/api-docs` and `/swagger-ui.html` — OpenAPI documentation.

All HTTP controllers will use request/response DTOs rather than exposing entities.

## Proposed package structure

```text
in.gov.mosje.sih26094
├── audit
├── config
├── controller
├── dto.request
├── dto.response
├── entity
├── exception
├── integration.ai
├── notification
├── queue
├── repository
├── scheduler
├── security
└── service
```

The implementation will keep layers small: controllers handle HTTP, services enforce business and access rules, repositories own persistence queries, and the AI/queue packages isolate external integration.

## Dependencies

- Java 21
- Spring Boot 3.5.x (latest stable 3.x line compatible with the requested stack)
- Spring Web, Validation, Data JPA, Security, OAuth2 Resource Server/JOSE, Data Redis, WebSocket, Actuator
- PostgreSQL JDBC driver
- springdoc OpenAPI WebMVC UI
- Spring Boot Test, Security Test, Testcontainers PostgreSQL, Awaitility, and MockWebServer for verification

No ML, messaging broker beyond Redis, Lombok, or cloud-notification SDK will be added.

## Environment variables

| Variable | Purpose |
|---|---|
| `SERVER_PORT`, `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `DB_POOL_SIZE` | HTTP and PostgreSQL connection/pool |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `REDIS_SSL`, `REDIS_TIMEOUT` | Redis connection |
| `JWT_SECRET`, `JWT_EXPIRATION` | HMAC JWT signing and lifetime |
| `AI_SERVICE_URL`, `AI_CONNECT_TIMEOUT`, `AI_READ_TIMEOUT`, `AI_MOCK_ENABLED` | Real/mock AI integration |
| `FRONTEND_URL` | Configurable CORS and WebSocket origin |
| `SCORING_QUEUE`, `SCORING_RETRY_QUEUE`, `SCORING_DEAD_LETTER` | Redis queue keys |
| `SCORING_MAX_ATTEMPTS`, `SCORING_BACKOFF`, `SCORING_STATUS_TTL`, `SCORING_POLL_DELAY`, `SCORING_RECOVERY_DELAY` | Queue retry/recovery policy |
| `ALERT_HIGH_DDS`, `ALERT_CRITICAL_DDS` | Centralized alert thresholds |
| `SCHEDULING_ENABLED`, `SCHEDULE_LOW_INTERVAL`, `SCHEDULE_MODERATE_INTERVAL`, `SCHEDULE_HIGH_INTERVAL`, `SCHEDULE_SCAN_DELAY` | Risk-aware monitoring cadence |
| `NOTIFICATIONS_MODE` | Notification provider; this implementation supports `mock` |

Actual secrets will not be committed. `.env.example` will contain safe development placeholders and warnings.

## Implementation phases

1. Maven/Spring Boot foundation and configuration.
2. PostgreSQL enum/JSONB entities and repositories.
3. JWT authentication, security handlers, RBAC, and jurisdiction access service.
4. Consent ledger and audit service.
5. Check-in validation and after-commit asynchronous Redis enqueue.
6. Idempotent Redis worker, retry/backoff/status handling, and recovery scan for persisted unscored check-ins.
7. Existing AI HTTP contract, transcription support, response validation/normalization, and deterministic mock AI.
8. Score persistence, risk-trigger consistency, alert creation, district escalation notification, and live dashboard events.
9. Trend, alerts, acknowledgement, and aggregate dashboard endpoints.
10. Risk-aware scheduler and notification abstraction.
11. OpenAPI, Docker/local development, tests, and build/runtime verification.

## Assumptions

- REST operations are staff-authenticated. The repository defines staff users but no victim identity/session table, so an unauthenticated public check-in API would permit arbitrary victim access and is not safe.
- `national` users receive aggregates only; direct victim trend access is limited to assigned counsellors, matching district/state officials, and admins.
- A Critical score creates one alert assigned to the counsellor and additionally notifies matching district officials. The single `assigned_to` schema column is preserved; no duplicate district alert is created.
- The latest consent row for `mental_health_monitoring`, ordered by creation time, is authoritative. `monitoring_active` is synchronized for efficient scheduling but is never the sole consent check.
- The existing AI service's `distress` label maps to `distress-indicative`, and `neutral/positive` maps to `neutral`, because those are the nearest valid database enum values.
- AI tier boundaries follow the checked-in AI service: Low `<40`, Moderate `<70`, High `<85`, Critical `>=85`. Alert thresholds are configurable and default to High `70`, Critical `85`.
- Real SMS/email providers are outside the repository and credentials are unavailable. Local development uses a logging/WebSocket notification implementation behind an interface.

## Risks and mitigations

- **Seed passwords are undocumented and appear to be placeholder BCrypt values.** Replace only seed hashes with a valid, documented local-demo hash; production credentials remain externally managed.
- **Redis can be unavailable after a check-in commits.** Enqueue occurs after commit and a recovery scheduler finds persisted check-ins without scores, preventing data loss.
- **Duplicate delivery/concurrent workers.** Check-in-to-score and score-to-alert unique database constraints plus application checks provide idempotency.
- **AI response drift.** The integration boundary validates DDS/tier and normalizes known legacy sentiment labels; malformed responses are retryable and never persisted.
- **No existing migration framework.** Docker initializes directly from the authoritative checked-in SQL. Hibernate validation prevents accidental schema creation. Flyway is intentionally not introduced in this first implementation.
- **No Java/Maven installed on the current host.** The build will be verified with an isolated Java 21/Maven runtime (or Docker once available), and the Maven Wrapper will be committed for normal developer use.
