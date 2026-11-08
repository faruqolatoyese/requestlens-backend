# RequestLens — Real-Time API Observability, Telemetry Ingestion & Alerting Engine (Backend)

**RequestLens** is a real-time API monitoring, telemetry ingestion, and automated alerting platform built for modern engineering teams. This repository (**`requestlens-backend`**) powers the core telemetry ingestion pipeline, real-time WebSocket streaming server, analytical aggregation engine, and asynchronous `5xx` incident alerting worker.

> **Multi-Repository Architecture**
> - **Backend Repository (This Repo):** [faruqolatoyese/requestlens-backend](https://github.com/faruqolatoyese/requestlens-backend)
> - **Frontend Repository:** [faruqolatoyese/requestlens-frontend](https://github.com/faruqolatoyese/requestlens-frontend)

---

## Live Links & Documentation

| Resource | URL | Description |
| :--- | :--- | :--- |
| **Production API Server** | [requestlens-backend.onrender.com](https://requestlens-backend.onrender.com) | Express 5 + Socket.IO Telemetry & REST API |
| **Interactive API Docs** | [requestlens-backend.onrender.com/api-docs](https://requestlens-backend.onrender.com/api-docs) | OpenAPI 3.0 / Swagger UI Documentation |
| **Live Web Application** | [requestlens-frontend.vercel.app](https://requestlens-frontend.vercel.app) | Production Next.js Frontend Dashboard |
| **Frontend Repository** | [github.com/faruqolatoyese/requestlens-frontend](https://github.com/faruqolatoyese/requestlens-frontend) | Next.js 16, React Query, Recharts & Socket.IO Client |

---

## The Problem We Are Solving

Engineering teams frequently face silent API degradations—such as creeping endpoint latency, unexpected traffic spikes, or sudden `5xx` error surges—that go unnoticed until end-users report broken workflows. Traditional Application Performance Monitoring (APM) suites are often heavyweight, expensive, and require complex infrastructure agents.

**RequestLens** solves this with a streamlined, developer-first observability architecture:
1. **High-Throughput, Low-Overhead Telemetry Ingestion:** Client applications (Express, FastAPI, cURL, or the dashboard's built-in Log Simulator) push lightweight HTTP telemetry payloads (`endpointId`, `statusCode`, `responseTime`) to `POST /api/logs` authenticated via a project-scoped `API Key`.
2. **Instant Real-Time WebSocket Streaming:** Instead of polling or waiting for delayed log indexing, the backend broadcasts newly persisted `ApiLog` records directly to authorized browser clients over **Socket.IO** project rooms (`new_log`), updating dashboard charts and log tables the millisecond a request finishes.
3. **Non-Blocking Asynchronous `5xx` Alerting:** Every ingested log triggers an asynchronous threshold evaluation (`evaluateAlerts`) that never blocks the HTTP response. When `5xx` server errors breach user-configured rules over a sliding window, jobs are queued into **BullMQ (Redis)** with exponential backoff and dispatched as rich HTML incident reports via **Resend**.
4. **Built-In Self-Monitoring Middleware:** RequestLens includes an optional `monitorMiddleware` capable of dogfooding its own telemetry pipeline—auto-discovering routes and streaming live performance metrics without external instrumentation.

---

## System Architecture & End-to-End Data Flow

RequestLens is split into two decoupled services (**`requestlens-backend`** and **`requestlens-frontend`**) communicating over REST APIs and persistent WebSocket channels.

```mermaid
flowchart TD
    subgraph ClientApps["Monitored Client Applications"]
        EXT["External APIs (Express / FastAPI / cURL)"]
        SIM["Dashboard Log Simulator"]
    end

    subgraph Backend["RequestLens Backend (Express 5 + TypeScript)"]
        INGEST["POST /api/logs (apiKeyMiddleware)"]
        REST["REST API (/api/auth, /api/projects, /api/stats, /api/alerts)"]
        WS["Socket.IO Server (JWT Handshake + Project Rooms)"]
        ALERT_EVAL["Async Alert Evaluator (evaluateAlerts)"]
        BULL["BullMQ Queue: 'alert-emails'"]
        WORKER["BullMQ Worker (Resend HTML Emails)"]
    end

    subgraph Database["Persistence & Queue Layer"]
        PG[("PostgreSQL (Prisma ORM)")]
        REDIS[("Redis (ioredis)")]
    end

    subgraph Frontend["RequestLens Frontend (Next.js 16 App Router)"]
        UI["Dashboard UI (Recharts + Radix UI)"]
        RQ["TanStack React Query Cache"]
        HOOK["useSocket Hook (socket.io-client)"]
    end

    EXT -->|"Bearer <API_KEY>"| INGEST
    SIM -->|"Bearer <API_KEY>"| INGEST
    INGEST -->|"1. Persist ApiLog"| PG
    INGEST -->|"2. Emit 'new_log' to projectId room"| WS
    INGEST -->|"3. Trigger non-blocking check"| ALERT_EVAL

    ALERT_EVAL -->|"Check window, minRequests & cooldown"| PG
    ALERT_EVAL -->|"Breach detected: Record AlertLog & Enqueue"| BULL
    BULL <--> REDIS
    BULL --> WORKER

    WS ==>|"Real-time 'new_log' event"| HOOK
    HOOK -->|"Invalidate queries"| RQ
    RQ <-->|"Bearer <JWT> REST calls"| REST
    REST <--> PG
    RQ --> UI
```

---

## Core Backend Subsystems

### 1. Dual Authentication & Authorization (`JWT` + `API Key`)
RequestLens separates control-plane management from data-plane telemetry ingestion:
- **Control Plane (`authMiddleware` — `Bearer <JWT>`):**
  - Supports both **Local Email/Password** (`bcrypt` salted hashes) and **Google OAuth 2.0** (`google-auth-library` verifying Google ID tokens on `/api/auth/google`, with automatic account linking if an email already exists).
  - Issues a 24-hour signed JWT (`{ userId }`) used across all `/api/projects/*` REST endpoints and the Socket.IO connection handshake.
  - Every service query strictly scopes lookups by `{ id: projectId, userId }` to enforce multi-tenant isolation.
- **Data Plane (`apiKeyMiddleware` — `Bearer <API_KEY>`):**
  - When a project is created, the backend generates a cryptographically secure 64-character hex secret (`crypto.randomBytes(32).toString("hex")`) stored in the `ApiKey` table (alongside a default `5xx` alert rule).
  - Incoming requests to `POST /api/logs` authenticate via `Authorization: Bearer <API_KEY>`. The middleware resolves and attaches `req.projectId` and verifies that the target `endpointId` belongs to that project.

### 2. Real-Time Telemetry Streaming (`Socket.IO`)
- **Authenticated Handshake:** Clients connect with `socket.handshake.auth.token`. The Socket.IO middleware verifies the JWT and attaches `socket.data.userId`.
- **Authorized Project Rooms:** When the frontend emits `join_project` with a `projectId`, the server queries PostgreSQL to verify that `userId` owns `projectId` before calling `socket.join(projectId)`.
- **Instant Broadcast:** As soon as `logResponse` (or `monitorMiddleware`) creates an `ApiLog` row in PostgreSQL, `emitNewLog(projectId, log)` broadcasts the enriched log (including `endpoint.name`, `endpoint.path`, and `endpoint.method`) to all connected sockets in that project's room.

### 3. Analytical Aggregation Engine (`stats.service.ts`)
`GET /api/projects/:projectId/stats` computes dashboard observability metrics in a single request:
- **Global KPIs:** Total requests, failed requests (`statusCode >= 400`), and overall success rate percentage.
- **7-Day Traffic Time-Series (`chartData`):** Zero-filled daily buckets for the last 7 days breaking down `total`, `successful` (`< 400`), and `failed` (`>= 400`) requests.
- **Per-Endpoint Performance Breakdown (`endpointStats`):** Uses Prisma `groupBy` aggregations (`_count`, `_avg.responseTime`) to compute per-route request volume, failure count, success rate (`%`), and rounded average latency (`ms`).

### 4. Asynchronous `5xx` Alerting Pipeline (`BullMQ` + `Redis` + `Resend`)
- **Non-Blocking Evaluation:** After saving a log, `evaluateAlerts(projectId)` runs asynchronously in the background so `POST /api/logs` returns immediately.
- **Noise-Free Guards:** For each active `AlertRule` on the project:
  1. **Cooldown Check:** Queries `AlertLog` for any trigger within the last `cooldownMinutes` (default `30m`). If found, evaluation skips the rule to prevent inbox flooding.
  2. **Minimum Volume Guard:** Counts total requests across the project within the sliding `windowMinutes` (default `5m`). If `totalRequests < rule.minRequests` (default `10`), evaluation skips to avoid false positives on low traffic.
  3. **Threshold Check:** Calculates the `5xx` server error rate (`statusCode >= 500`). If `errorRate >= rule.thresholdPercentage` (default `5.0%`), the rule triggers.
- **Reliable Queue & Worker Delivery:**
  - Triggering creates an `AlertLog` record (activating the cooldown), aggregates failing endpoints by `method` + `path`, and enqueues a job in the `alert-emails` **BullMQ** queue with **5 retry attempts** and **exponential backoff** (`5s`, `10s`, `20s`, ...).
  - `alertEmailWorker` (`concurrency: 2`) consumes the queue and sends a formatted HTML incident report via **Resend** featuring breached metrics, failing endpoint counts, and cooldown status.

---

## Database Schema (PostgreSQL + Prisma ORM)

```mermaid
erDiagram
    User ||--o{ Project : "owns"
    Project ||--o{ Endpoint : "monitors"
    Project ||--o{ ApiKey : "authenticates via"
    Project ||--o{ AlertRule : "evaluates"
    Endpoint ||--o{ ApiLog : "records"
    AlertRule ||--o{ AlertLog : "triggers"

    User {
        uuid id PK
        string email UK
        string password "Nullable for Google OAuth"
        string name
        string avatar
        string provider "LOCAL | GOOGLE"
        string googleId UK
        datetime createdAt
        datetime updatedAt
    }

    Project {
        uuid id PK
        string name
        string description
        uuid userId FK
        datetime createdAt
        datetime updatedAt
    }

    Endpoint {
        uuid id PK
        string name
        string path
        enum method "GET | POST | PUT | PATCH | DELETE"
        uuid projectId FK
        datetime createdAt
        datetime updatedAt
    }

    ApiLog {
        uuid id PK
        int statusCode
        int responseTime "Latency in ms"
        datetime timestamp
        uuid endpointId FK
    }

    ApiKey {
        uuid id PK
        string key UK "64-char hex secret"
        uuid projectId FK
        datetime createdAt
    }

    AlertRule {
        uuid id PK
        string name
        float thresholdPercentage "Default 5.0 (%)"
        int windowMinutes "Default 5"
        int cooldownMinutes "Default 30"
        int minRequests "Default 10"
        boolean active "Default true"
        uuid projectId FK
    }

    AlertLog {
        uuid id PK
        float errorRate
        int totalRequests
        datetime triggeredAt
        uuid alertRuleId FK
    }
```

---

## REST API & WebSocket Reference

> Interactive documentation is available at [`/api-docs`](https://requestlens-backend.onrender.com/api-docs).

### Authentication (`/api/auth`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :---: | :--- |
| `POST` | `/api/auth/register` | Public | Register a new account with `{ email, password }` and receive a JWT |
| `POST` | `/api/auth/login` | Public | Authenticate with `{ email, password }` and receive a JWT |
| `POST` | `/api/auth/google` | Public | Verify a Google OAuth `{ idToken }`, provision/link user, and receive a JWT |

### Projects & Analytics (`/api/projects`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :---: | :--- |
| `GET` | `/api/projects` | `Bearer <JWT>` | List all projects and associated API keys for the authenticated user |
| `POST` | `/api/projects` | `Bearer <JWT>` | Create a project, generate its 64-char `ApiKey`, and seed a default `AlertRule` |
| `GET` | `/api/projects/:id` | `Bearer <JWT>` | Retrieve details and API keys for a specific project |
| `DELETE` | `/api/projects/:id` | `Bearer <JWT>` | Delete a project and cascade-delete its endpoints, logs, keys, and alerts |
| `GET` | `/api/projects/:id/logs` | `Bearer <JWT>` | Fetch paginated request logs (`?page=1&limit=10`) ordered by newest first |
| `GET` | `/api/projects/:projectId/stats` | `Bearer <JWT>` | Fetch KPI totals, 7-day traffic chart data, and per-endpoint latency/error stats |

### Monitored Endpoints (`/api/projects/:projectId/endpoints`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :---: | :--- |
| `GET` | `/api/projects/:projectId/endpoints` | `Bearer <JWT>` | List all registered HTTP endpoints under a project |
| `POST` | `/api/projects/:projectId/endpoints` | `Bearer <JWT>` | Register a new endpoint (`{ name, path, method }`) |
| `GET` | `/api/projects/:projectId/endpoints/:endpointId` | `Bearer <JWT>` | Get details for a specific registered endpoint |
| `DELETE` | `/api/projects/:projectId/endpoints/:endpointId` | `Bearer <JWT>` | Delete a registered endpoint and cascade-delete its logs |

### Alert Rules (`/api/projects/:projectId/alerts`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :---: | :--- |
| `GET` | `/api/projects/:projectId/alerts` | `Bearer <JWT>` | List all `5xx` alert rules configured for a project |
| `POST` | `/api/projects/:projectId/alerts` | `Bearer <JWT>` | Create a new alert rule (`name`, `thresholdPercentage`, `windowMinutes`, `cooldownMinutes`, `minRequests`) |
| `PATCH` | `/api/projects/:projectId/alerts/:ruleId` | `Bearer <JWT>` | Toggle or explicitly set the `active` boolean state of an alert rule |
| `DELETE` | `/api/projects/:projectId/alerts/:ruleId` | `Bearer <JWT>` | Delete an alert rule |

### Telemetry Ingestion (`/api/logs`)
| Method | Endpoint | Auth | Description |
| :--- | :--- | :---: | :--- |
| `POST` | `/api/logs` | `Bearer <API_KEY>` | Ingest a request log (`{ endpointId, statusCode, responseTime }`), broadcast via Socket.IO, and evaluate alerts |

### WebSocket Events (`Socket.IO`)
| Event | Direction | Payload | Description |
| :--- | :---: | :--- | :--- |
| `join_project` | Client -> Server | `projectId: string` | Verifies project ownership and subscribes the socket to the `projectId` room |
| `leave_project` | Client -> Server | `projectId: string` | Leaves the `projectId` room |
| `joined` / `left` | Server -> Client | `projectId: string` | Confirmation of room subscription state change |
| `new_log` | Server -> Client | `ApiLog` (with `endpoint`) | Broadcasts immediately when a new telemetry log is recorded for `projectId` |
| `error_msg` | Server -> Client | `message: string` | Emitted if room authorization fails or `projectId` is invalid |

---

## Tech Stack

- **Runtime & Framework:** [Node.js](https://nodejs.org/), [Express 5](https://expressjs.com/), [TypeScript](https://www.typescriptlang.org/) (`ES2022` / `NodeNext`)
- **Database & ORM:** [PostgreSQL](https://www.postgresql.org/) + [Prisma ORM v6](https://www.prisma.io/)
- **Real-Time Streaming:** [Socket.IO v4](https://socket.io/)
- **Distributed Queue & Caching:** [Redis (`ioredis`)](https://github.com/redis/ioredis) + [BullMQ v5](https://docs.bullmq.io/)
- **Transactional Email Service:** [Resend](https://resend.com/)
- **Authentication & Security:** [JSON Web Tokens (`jsonwebtoken`)](https://github.com/auth0/node-jsonwebtoken), [bcrypt](https://github.com/kelektiv/node.bcrypt.js), [Google Auth Library (`google-auth-library`)](https://github.com/googleapis/google-auth-library-nodejs)
- **API Documentation:** OpenAPI 3.0 + [Swagger UI Express](https://github.com/scottie1984/swagger-ui-express)

---

## Project Structure

```text
requestlens-backend/
├── prisma/
│   ├── migrations/                       # SQL migration history (init, indexes, alert system)
│   └── schema.prisma                     # PostgreSQL data models & relations
├── src/
│   ├── controllers/
│   │   ├── alert.controller.ts           # CRUD & toggle handlers for 5xx AlertRules
│   │   ├── auth.controller.ts            # Register, Login & Google OAuth controllers
│   │   ├── endpoint.controller.ts        # Endpoint registration & management handlers
│   │   ├── logs.controller.ts            # Telemetry ingestion handler (POST /api/logs)
│   │   ├── project.controller.ts         # Project lifecycle & paginated log controllers
│   │   └── stats.controller.ts           # Dashboard analytics & aggregation controller
│   ├── middlewares/
│   │   ├── apiKey.middleware.ts          # Validates project Bearer <API_KEY> for ingestion
│   │   ├── auth.middleware.ts            # Validates user Bearer <JWT> for dashboard routes
│   │   └── monitor.middleware.ts         # Built-in self-monitoring & route auto-discovery
│   ├── migrations/
│   │   └── backfill-alerts.ts            # One-off utility to backfill default AlertRules
│   ├── queues/
│   │   ├── alertQueue.ts                 # BullMQ 'alert-emails' queue & exponential backoff config
│   │   └── alertWorker.ts                # Background worker rendering & sending HTML emails via Resend
│   ├── routes/
│   │   ├── alert.routes.ts               # /api/projects/:projectId/alerts routes
│   │   ├── auth.routes.ts                # /api/auth routes
│   │   ├── endpoint.routes.ts            # /api/projects/:projectId/endpoints routes
│   │   ├── logs.routes.ts                # /api/logs ingestion route
│   │   ├── project.routes.ts             # /api/projects routes
│   │   └── stats.routes.ts               # /api/projects/:projectId/stats route
│   ├── services/
│   │   ├── alert.service.ts              # Sliding-window 5xx evaluation & cooldown logic
│   │   ├── auth.service.ts               # User registration, password checks & Google account linking
│   │   ├── endpoint.service.ts           # Endpoint CRUD business logic
│   │   ├── logs.service.ts               # Log persistence, Socket.IO broadcast & async alert trigger
│   │   ├── project.service.ts            # Project provisioning, API key generation & log pagination
│   │   └── stats.service.ts              # 7-day chart bucketing & Prisma groupBy endpoint metrics
│   ├── utils/
│   │   ├── google.ts                     # Google OAuth2Client ID token verification
│   │   ├── hash.ts                       # bcrypt password hashing & comparison helpers
│   │   ├── jwt.ts                        # JWT signing utility (1-day expiration)
│   │   ├── prisma.ts                     # Singleton PrismaClient instance
│   │   ├── redis.ts                      # Shared ioredis connection for BullMQ
│   │   └── socket.ts                     # Socket.IO server initialization, JWT middleware & room emitter
│   ├── app.ts                            # Express 5 app setup, middlewares, Swagger UI & route mounting
│   ├── server.ts                         # HTTP server bootstrap, Socket.IO attachment & worker startup
│   └── test-queue.ts                     # CLI script to test BullMQ + Resend email delivery
├── swagger.json                          # OpenAPI 3.0 specification served at /api-docs
├── package.json                          # Dependencies & build/dev scripts
└── tsconfig.json                         # TypeScript compiler configuration
```

---

## Getting Started (Local Development)

### Prerequisites
- **Node.js** `v20+` and `npm`
- **PostgreSQL** database instance (local or hosted, e.g., Neon / Supabase / Render)
- **Redis** instance (local or hosted, e.g., Upstash / Render Redis) for BullMQ background workers

### 1. Clone the Repository
```bash
git clone https://github.com/faruqolatoyese/requestlens-backend.git
cd requestlens-backend
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Configure Runtime Environment Variables
Configure the following environment variables in your local environment before starting the server:

| Variable Name | Required | Description | Example Value |
| :--- | :---: | :--- | :--- |
| `PORT` | No | Port for the Express + Socket.IO HTTP server (defaults to `5000`) | `5000` |
| `DATABASE_URL` | Yes | PostgreSQL connection string used by Prisma ORM | `postgresql://user:pass@localhost:5432/requestlens` |
| `JWT_SECRET` | Yes | Secret key used to sign and verify 24-hour user JWT tokens | `your-super-secret-jwt-key` |
| `REDIS_URL` | Yes | Redis connection URL used by `ioredis` and BullMQ queues/workers | `redis://localhost:6379` |
| `GOOGLE_CLIENT_ID` | Yes | Google OAuth 2.0 Web Client ID (must match the frontend's client ID) | `your-google-client-id.apps.googleusercontent.com` |
| `RESEND_API_KEY` | Optional | Resend API key for dispatching `5xx` alert emails (logs a warning if omitted) | `re_123456789_abcdefghijklmnop` |
| `SELF_MONITORING_API_KEY` | Optional | Project API Key used by `monitorMiddleware` for self-monitoring | `a1b2c3d4...` |

### 4. Run Database Migrations & Generate Prisma Client
```bash
npx prisma migrate dev
npx prisma generate
```

### 5. Start the Development Server
```bash
npm run dev
```
- **API Base URL:** [http://localhost:5000](http://localhost:5000)
- **Swagger UI Docs:** [http://localhost:5000/api-docs](http://localhost:5000/api-docs)

### 6. Build & Run for Production
```bash
npm run build
npm run start
```

---

## Example Telemetry Ingestion (`POST /api/logs`)

Once you create a project and register an endpoint in the [RequestLens Frontend Dashboard](https://requestlens-frontend.vercel.app), you can stream live telemetry logs from any service:

```bash
curl -X POST https://requestlens-backend.onrender.com/api/logs \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_PROJECT_API_KEY>" \
  -d '{
    "endpointId": "<YOUR_ENDPOINT_UUID>",
    "statusCode": 200,
    "responseTime": 142
  }'
```

---

## License

This project is licensed under the MIT / ISC License.
