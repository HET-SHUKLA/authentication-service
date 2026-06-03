A production-grade authentication and session management service - the auth backbone for a URL Shortener platform - built to be extracted and reused as a standalone microservice.

This service handles everything that happens before a user touches your product: registration, login, Google OAuth, email verification, token refresh, and session revocation. It was designed to be correct by default, not just functional.


## Technical Highlights

- **Hashed refresh tokens with device-level session isolation** - the raw refresh token is never stored, only a `SHA-256` hash is persisted in the `sessions` table. Each device gets its own revokable session, enabling logout-from-all-devices and individual session revocation without invalidating others.

- **Dual-client token delivery** - web clients receive refresh tokens via `HttpOnly; Secure; SameSite=Lax` cookies (XSS-resistant), mobile clients receive them in the response body. The distinction is enforced via an `X-Client-Type` request header, not a config flag.

- **Google OAuth handled server-side** - the frontend never touches the Google token exchange. The client passes a Google ID token, the server validates it against Google's public certs using `google-auth-library`, then upserts a `user_auth` record under the `GOOGLE` provider. A user with the same email on both `EMAIL` and `GOOGLE` providers gets separate `user_auth` rows linked to a single `users` row - no silent account merging.

- **Email delivery decoupled via BullMQ** - verification emails and password-reset emails are dispatched as background jobs through a Redis-backed BullMQ queue with exponential backoff and a dead-letter queue. The HTTP response returns before the email is sent, keeping p99 latency unaffected by AWS SES latency.

- **Full observability out of the box** - structured JSON logs via Pino are shipped to Loki through Grafana Alloy, visualised in Grafana. OpenTelemetry tracing spans the full request lifecycle. The entire observability stack runs in Docker Compose alongside the app with zero external dependencies.

## Architecture

The system is stateless at the application layer - every request carries a signed JWT, and the only mutable state lives in PostgreSQL (users, sessions, verification tokens) and Redis (job queue, cache). This means horizontal scaling requires no sticky sessions, and a full Redis flush doesn't lose user data.

```
Client (Web / Mobile)
        │
        │  HTTPS
        ▼
┌─────────────────────────┐
│    Fastify HTTP Server  │  ← Helmet, Cookie parser, Swagger UI at /docs
│   (Node.js / TypeScript)│
│                         │
│  ┌──────────────────┐   │
│  │  Auth Module     │   │  ← /api/v1/auth
│  │  User Module     │   │  ← /api/v1/user
│  │  URL Module      │   │  ← /api/v1/url
│  └──────────────────┘   │
└──────────┬──────────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
PostgreSQL      Redis
(Prisma ORM)  (BullMQ + Cache)
                  │
                  ▼
            Email Worker
           (AWS SES via BullMQ)

─────────── Observability ────────────
Pino logs → Grafana Alloy → Loki → Grafana
```

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 (Alpine) |
| Framework | Fastify 5 |
| Language | TypeScript 5 |
| ORM / DB | Prisma 7 + PostgreSQL 16 |
| Caching / Queue | Redis + BullMQ |
| Auth tokens | `jsonwebtoken` (JWT) + `bcrypt` |
| Google OAuth | `google-auth-library` |
| Email delivery | AWS SES (v2 SDK) |
| Validation | Zod 4 |
| API Docs | Swagger UI (`/docs`) |
| Logging | Pino → Grafana Loki |
| Metrics / Viz | Prometheus + Grafana |
| Log shipping | Grafana Alloy |
| Containerisation | Docker + Docker Compose |

## Local Setup

**Prerequisites:** Docker, Docker Compose, Node.js 20+

### 1. Clone the repo

```bash
git clone https://github.com/HET-SHUKLA/authentication-service.git
cd authentication-service
```

### 2. Configure environment

```bash
cp .env.sample .env.local
```

Open `.env.local` and fill in the required values:

| Variable | Description |
|---|---|
| `POSTGRES_USER` | PostgreSQL username |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `POSTGRES_DB` | Database name |
| `DATABASE_URL` | Full Prisma connection string, e.g. `postgresql://user:pass@postgres:5432/dbname` |
| `DATABASE_URL_LOCALHOST` | Same but with `localhost` instead of `postgres` - used outside Docker |
| `JWT_SECRET` | Long random string for signing access tokens |
| `JWT_EXPIRES_IN` | Access token TTL in seconds (e.g. `60`) |
| `REFRESH_TOKEN_TTL_DAYS` | Refresh token lifetime in days (e.g. `7`) |
| `COOKIE_SECRET` | Secret for signed cookies |
| `AWS_ACCESS_KEY_ID` | AWS credentials for SES email sending |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `AWS_SES_REGION` | AWS region for SES (e.g. `us-east-1`) |
| `SEND_EMAIL_FROM` | Verified sender address in AWS SES |
| `REDIS_URL` | Redis connection string (default: `redis://redis:6379`) |
| `LOG_LEVEL` | Log verbosity: `debug`, `info`, `warn`, `error` |

### 3. Start all services

```bash
docker compose up -d
```

This starts PostgreSQL, Redis, Loki, Grafana Alloy, and Grafana. The `server` container starts but idles (`sleep infinity`) so you can run the app with hot-reload inside it.

### 4. Run database migrations

```bash
docker compose exec server npx prisma migrate dev
```

### 5. Start the development server

```bash
docker compose exec server npm run dev:watch
```

The API is now live at **`http://localhost:3000`**.

Swagger UI is available at **`http://localhost:3000/docs`**.

Grafana is available at **`http://localhost:3001`** (credentials in `.env.local`).

---

### Adding new dependencies

Because `node_modules` is mounted as a Docker volume, run installs inside the container:

```bash
docker compose run --rm --user root server npm install <package>
```

---

## API Reference

Full request/response documentation is in [`API.md`](./API.md).

**Auth endpoints** (`/api/v1/auth`):

| Method | Path | Description |
|---|---|---|
| `POST` | `/register` | Email + password registration |
| `POST` | `/login` | Email + password login |
| `POST` | `/google` | Google OAuth (pass ID token from frontend) |
| `POST` | `/refresh` | Rotate refresh token, issue new access token |
| `POST` | `/logout` | Revoke current session |
| `POST` | `/logout-all` | Revoke all sessions for the user |
| `GET` | `/me` | Fetch authenticated user profile |
| `POST` | `/verify-email` | Verify email with token from email link |
| `POST` | `/forgot-password` | Trigger password-reset email |
| `POST` | `/reset-password` | Set new password with reset token |