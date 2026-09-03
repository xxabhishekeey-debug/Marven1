# SIH26094 Spring Boot Backend

## Architecture

`POST /api/v1/checkins` validates staff access and the latest append-only consent event, commits the check-in, then publishes an after-commit Redis job. A scheduled worker obtains a per-check-in Redis lock, calls the mock or real AI service, validates its response, inserts the unique score, and creates an idempotent contextual alert when required. The database trigger updates the victim's current tier. Redis failures do not roll back check-ins; a recovery scan re-enqueues unscored records.

The backend never performs ML inference or autonomous intervention. Critical alerts remain human decisions: the assigned counsellor and matching district officials receive sanitized user-specific dashboard/WebSocket events.

## Important endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/v1/auth/login` | Issue a JWT for an active BCrypt-authenticated user |
| POST | `/api/v1/consent` | Append a grant/withdrawal event |
| POST | `/api/v1/checkins` | Persist and asynchronously enqueue a check-in |
| GET | `/api/v1/victims/{id}/trend` | Authorized trend using `v_victim_distress_trends` |
| GET | `/api/v1/alerts` | Paginated scoped alerts |
| POST | `/api/v1/alerts/{id}/ack` | Record human outcome and audit event |
| GET | `/api/v1/dashboard/summary` | Role-scoped aggregate metrics |
| CONNECT | `/ws/alerts` | STOMP with `Authorization: Bearer <JWT>`; subscribe to `/user/queue/alerts` |

## Example requests

```bash
curl -s http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"pooja.counsellor@nhaa.gov.in","password":"Demo@123"}'
```

```bash
curl -s http://localhost:8080/api/v1/checkins \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "victimId":"22222222-2222-2222-2222-222222222201",
    "channel":"chat",
    "rawText":"I feel unsafe and need immediate help",
    "responseLatencySec":12.4,
    "metadata":{"mock_dds":91},
    "missed":false
  }'
```

```bash
curl -s -X POST http://localhost:8080/api/v1/alerts/ALERT_ID/ack \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{"outcome":"contacted","notes":"Reached the victim and arranged counsellor follow-up."}'
```

## Real AI contract

Set `AI_MOCK_ENABLED=false` and `AI_SERVICE_URL`. The backend calls `POST /ai/v1/transcribe` for audio-only check-ins and then `POST /ai/v1/score` using the checked-in FastAPI contract. Legacy AI labels `distress` and `neutral/positive` are normalized to valid database enum values. DDS, confidence, sentiment, and DDS/tier consistency are validated before persistence.

## Development and tests

```bash
export JWT_SECRET='replace-with-a-local-random-value-of-at-least-32-bytes'
./mvnw clean test
./mvnw spring-boot:run
```

On Windows PowerShell, set `$env:JWT_SECRET` and use `.\mvnw.cmd`. The Testcontainers end-to-end demo test runs automatically when Docker is available and is reported as skipped otherwise.

The application sets `spring.jpa.hibernate.ddl-auto=validate` and `spring.sql.init.mode=never`. It never creates or truncates the production schema. Docker initialization deliberately uses the authoritative scripts under `database/`.

See `.env.example` for every supported environment variable and `IMPLEMENTATION_PLAN.md` for decisions and assumptions.
