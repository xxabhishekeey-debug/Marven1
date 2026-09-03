# SIH26094 AI-Based Stress Monitoring System

This repository contains the PostgreSQL data model and a Java 21/Spring Boot backend for consent-aware mental-health monitoring, asynchronous AI scoring, risk trends, and human-in-the-loop alert handling.

## Run locally with Docker

```bash
cp .env.example .env
# Replace JWT_SECRET in .env with a local random value of at least 32 bytes.
docker compose --env-file .env up --build
```

The first PostgreSQL startup applies `database/schema.sql`, `database/views.sql`, and the small demonstration seed. Swagger UI is available at <http://localhost:8080/swagger-ui.html> and health at <http://localhost:8080/actuator/health>.

The development seed uses password `Demo@123` for these staff accounts:

- `pooja.counsellor@nhaa.gov.in` — South Delhi counsellor
- `amit.counsellor@nhaa.gov.in` — Pune counsellor
- `rajesh.district@nhaa.gov.in` — Pune district official
- `meena.state@nhaa.gov.in` — Maharashtra state official
- `admin@mosje.gov.in` — national aggregate user

These are local synthetic identities only. Replace credentials and `JWT_SECRET` outside development.

## Run the backend directly

Start PostgreSQL and Redis, apply the database scripts in this order, then run Spring Boot:

```bash
psql "$DATABASE_URL" -f database/schema.sql
psql "$DATABASE_URL" -f database/views.sql
psql "$DATABASE_URL" -f database/seed.sql
cd backend
export JWT_SECRET='replace-with-a-local-random-value-of-at-least-32-bytes'
./mvnw spring-boot:run
```

On Windows PowerShell, set `$env:JWT_SECRET` and use `.\mvnw.cmd spring-boot:run`. Configuration is entirely environment driven; copy `.env.example` as a starting point and use `DB_URL=jdbc:postgresql://localhost:5432/sih26094` when running outside Compose. `AI_MOCK_ENABLED=true` is the default, so the Python intelligence service is not required for a local demo.

## Demo API flow

1. `POST /api/v1/auth/login` with a seeded email and `Demo@123`.
2. Use the returned bearer token for all later requests.
3. `POST /api/v1/checkins` for a victim with active consent. Add `"metadata":{"mock_dds":91}` to deterministically create a Critical mock score.
4. The request returns `202 Accepted` with `PENDING`; Redis processes AI scoring asynchronously.
5. `GET /api/v1/alerts?status=open` and `GET /api/v1/victims/{id}/trend` show the result.
6. `POST /api/v1/alerts/{id}/ack` with an outcome and notes closes the human action loop.

See [backend/README.md](backend/README.md) for request examples, configuration, architecture, and testing.
