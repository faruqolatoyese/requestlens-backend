# RequestLens — Real-Time API Observability & Alerting Engine (Backend)

**RequestLens** is a real-time API monitoring, telemetry ingestion, and automated alerting platform built for modern engineering teams. This repository (**`requestlens-backend`**) powers the telemetry ingestion API, real-time WebSocket streaming server, analytics aggregation engine, and asynchronous `5xx` incident alerting worker.

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

Engineering teams frequently face silent API degradations—such as creeping endpoint latency, unexpected traffic spikes, or sudden `5xx` error surges—that go unnoticed until end-users report broken workflows. Traditional APM tools are often heavyweight, expensive, and complex to set up.

**RequestLens** solves this with a lightweight, real-time backend architecture:
1. **Low-Overhead Telemetry Ingestion:** Client apps (Express, FastAPI, cURL, or the dashboard's Log Simulator) send lightweight request metrics (`endpointId`, `statusCode`, `responseTime`) to `POST /api/logs` using a project `API Key`.
2. **Instant Real-Time Streaming:** Instead of polling, the backend broadcasts newly recorded logs directly to authorized browser clients over **Socket.IO** project rooms (`new_log`), updating dashboard charts and log tables immediately.
3. **Noise-Free Asynchronous Alerting:** Every ingested log triggers a non-blocking `5xx` threshold check (`evaluateAlerts`). When an error spike breaches a rule, jobs are queued into **BullMQ (Redis)** with exponential backoff and sent as HTML incident emails via **Resend**.

---

## System Architecture & How It Works

RequestLens separates **Control Plane** (dashboard management) from **Data Plane** (telemetry ingestion):

1. **Dual Authentication (`JWT` + `API Key`):**
   - **Dashboard Auth (`Bearer <JWT>`):** Users authenticate via Email/Password (`bcrypt`) or **Google OAuth 2.0** (`google-auth-library`). A signed 24-hour JWT secures all `/api/projects/*` routes and the Socket.IO handshake.
   - **Ingestion Auth (`Bearer <API_KEY>`):** Creating a project provisions a unique 64-character hex `ApiKey` (plus a default `5xx` alert rule). `apiKeyMiddleware` validates this key on `POST /api/logs` and resolves the target `projectId`.
2. **Real-Time WebSocket Rooms (`Socket.IO`):**
   - Dashboard clients connect with their JWT and emit `join_project` with a `projectId`.
   - The server verifies project ownership in PostgreSQL before joining the socket to the isolated `projectId` room, where `new_log` events are broadcast in real time.
3. **Analytics Aggregation (`GET /api/projects/:projectId/stats`):**
   - Computes total requests, failed requests (`>= 400`), overall success rate, 7-day daily traffic buckets (`chartData`), and per-endpoint average latency and failure rates using Prisma `groupBy` queries.
4. **Asynchronous `5xx` Alerting Pipeline (`BullMQ` + `Redis` + `Resend`):**
   - Evaluates active `AlertRule` records over a sliding time window (`windowMinutes`).
   - Enforces a minimum traffic guard (`minRequests`) to avoid low-volume false positives and a cooldown period (`cooldownMinutes`) via `AlertLog` to prevent email spam.
   - Enqueues breached alerts into the `alert-emails` BullMQ queue (5 retries with exponential backoff), processed by `alertEmailWorker` to send endpoint failure summaries via Resend.

---

## Tech Stack

- **Runtime & Framework:** Node.js, Express 5, TypeScript
- **Database & ORM:** PostgreSQL + Prisma ORM
- **Real-Time Server:** Socket.IO
- **Queue & Background Jobs:** Redis (`ioredis`) + BullMQ
- **Transactional Emails:** Resend
- **Authentication:** JWT (`jsonwebtoken`), `bcrypt`, Google OAuth 2.0 (`google-auth-library`)
- **API Documentation:** OpenAPI 3.0 / Swagger UI (`swagger-ui-express`)

---

## REST API Overview

> Full interactive documentation and request/response schemas are available at [`/api-docs`](https://requestlens-backend.onrender.com/api-docs).

| Group | Method & Route | Auth | Purpose |
| :--- | :--- | :---: | :--- |
| **Auth** | `POST /api/auth/register` | Public | Register with `{ email, password }` and receive a JWT |
| | `POST /api/auth/login` | Public | Log in with `{ email, password }` and receive a JWT |
| | `POST /api/auth/google` | Public | Authenticate with a Google `{ idToken }` and receive a JWT |
| **Projects** | `GET /api/projects` | `JWT` | List all projects and API keys for the current user |
| | `POST /api/projects` | `JWT` | Create a project, generate its `ApiKey`, and seed a default alert rule |
| | `GET /api/projects/:id` | `JWT` | Get project details and API key |
| | `DELETE /api/projects/:id` | `JWT` | Delete a project and cascade-delete all associated data |
| **Analytics & Logs** | `GET /api/projects/:projectId/stats` | `JWT` | Get KPI totals, 7-day traffic chart, and per-endpoint metrics |
| | `GET /api/projects/:id/logs` | `JWT` | Get paginated request logs (`?page=1&limit=10`) |
| **Endpoints** | `GET /api/projects/:projectId/endpoints` | `JWT` | List registered HTTP endpoints for a project |
| | `POST /api/projects/:projectId/endpoints` | `JWT` | Register an endpoint (`{ name, path, method }`) |
| | `DELETE /api/projects/:projectId/endpoints/:endpointId` | `JWT` | Delete a registered endpoint |
| **Alerts** | `GET /api/projects/:projectId/alerts` | `JWT` | List `5xx` alert rules for a project |
| | `POST /api/projects/:projectId/alerts` | `JWT` | Create a custom `5xx` threshold alert rule |
| | `PATCH /api/projects/:projectId/alerts/:ruleId` | `JWT` | Toggle `active` status of an alert rule |
| | `DELETE /api/projects/:projectId/alerts/:ruleId` | `JWT` | Delete an alert rule |
| **Ingestion** | `POST /api/logs` | `API Key` | Ingest `{ endpointId, statusCode, responseTime }`, emit via Socket.IO & evaluate alerts |

---

## Project Structure

```text
requestlens-backend/
├── prisma/
│   ├── migrations/              # SQL migration history
│   └── schema.prisma            # PostgreSQL models (User, Project, Endpoint, ApiLog, ApiKey, AlertRule, AlertLog)
├── src/
│   ├── controllers/             # Express route handlers (auth, project, endpoint, logs, stats, alert)
│   ├── middlewares/             # JWT auth, API key validation, and self-monitoring middleware
│   ├── queues/                  # BullMQ queue ('alert-emails') and Resend email background worker
│   ├── routes/                  # Express routers mounted in app.ts
│   ├── services/                # Business logic, Prisma queries, and sliding-window alert evaluation
│   ├── utils/                   # Prisma client, Redis connection, Socket.IO server, JWT, bcrypt & Google OAuth
│   ├── app.ts                   # Express app configuration & Swagger UI setup
│   └── server.ts                # HTTP server entry point, Socket.IO init & worker bootstrap
├── swagger.json                 # OpenAPI 3.0 specification
└── package.json
```

---

## Getting Started (Local Development)

### Prerequisites
- **Node.js** `v20+` and `npm`
- **PostgreSQL** database and **Redis** instance

### 1. Clone & Install
```bash
git clone https://github.com/faruqolatoyese/requestlens-backend.git
cd requestlens-backend
npm install
```

### 2. Configure Environment Variables
Set the following environment variables in your local environment:

| Variable Name | Required | Description |
| :--- | :---: | :--- |
| `PORT` | No | Server port (defaults to `5000`) |
| `DATABASE_URL` | Yes | PostgreSQL connection string for Prisma ORM |
| `JWT_SECRET` | Yes | Secret key used to sign and verify JWT tokens |
| `REDIS_URL` | Yes | Redis connection URL for BullMQ queues and workers |
| `GOOGLE_CLIENT_ID` | Yes | Google OAuth 2.0 Web Client ID |
| `RESEND_API_KEY` | Optional | Resend API key for dispatching `5xx` alert emails |
| `SELF_MONITORING_API_KEY` | Optional | Project API Key used by `monitorMiddleware` for self-monitoring |

### 3. Run Database Migrations & Start Server
```bash
npx prisma migrate dev
npm run dev
```
- **API Server:** `http://localhost:5000`
- **Swagger Docs:** `http://localhost:5000/api-docs`

---

## License

This project is licensed under the MIT / ISC License.
