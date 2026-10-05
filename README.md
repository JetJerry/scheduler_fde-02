# Meeting Assistant

An AI-powered meeting assistant that lets authenticated users connect Google Calendar and manage their schedule through a conversational interface. The application can list upcoming meetings, inspect availability, create meetings with optional attendees and Google Meet links, reschedule events, and cancel events.

The repository contains:

- **Next.js frontend**: sign-in, calendar connection status, threaded chat, and streamed assistant responses.
- **Express/TypeScript backend**: authentication, calendar connection lifecycle, agent orchestration, SSE streaming, persistence, and an MCP endpoint.
- **PostgreSQL**: durable application data for users and calendar connection status.
- **Mastra + LibSQL**: local agent memory and chat thread storage.
- **Docker Compose**: a development PostgreSQL instance.
- **Python root project**: an initial scaffold currently containing only a greeting entry point; it is not part of the active web application runtime.

> **Status:** This is an active application scaffold. It is suitable for local development and further hardening, but it should not be treated as production-ready until the operational, security, and deployment items in [Production checklist](#production-checklist) are addressed.

## Contents

- [Product capabilities](#product-capabilities)
- [Architecture](#architecture)
- [Repository layout](#repository-layout)
- [Technology stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Local development](#local-development)
- [Configuration](#configuration)
- [Runtime flows](#runtime-flows)
- [HTTP API](#http-api)
- [Agent tools](#agent-tools)
- [Data model and persistence](#data-model-and-persistence)
- [MCP integration](#mcp-integration)
- [Build and verification](#build-and-verification)
- [Deployment guidance](#deployment-guidance)
- [Security considerations](#security-considerations)
- [Production checklist](#production-checklist)
- [Troubleshooting](#troubleshooting)

## Product capabilities

After signing in, a user can:

1. Connect or reconnect Google Calendar through Descope outbound applications.
2. View the current calendar connection state.
3. Start a new conversation or resume one of the latest saved threads.
4. Ask for today's or upcoming meetings.
5. Check whether a time range is busy.
6. Create a calendar event with a title, ISO-8601 start and end times, description, invitees, and an optional Google Meet link.
7. Reschedule an existing event by event ID.
8. Cancel an existing event by event ID.

The assistant uses working memory for preferences such as timezone, default meeting length, preferred hours, and usual invitees. It is instructed not to invent meeting details that are absent from Google Calendar.

## Architecture

```mermaid
flowchart LR
    Browser[Next.js browser client] -->|Descope session| Identity[Descope]
    Browser -->|Authenticated session + JSON/SSE| API[Express API]
    API -->|validateSession| Identity
    API -->|users and connection status| PG[(PostgreSQL)]
    API -->|thread history and working memory| Memory[(Mastra LibSQL file)]
    API -->|agent tools| Agent[Mastra meeting agent]
    Agent -->|outbound application token| Identity
    Agent -->|Calendar API| Google[Google Calendar]
    MCP[MCP clients] -->|POST /mcp| API
    MCP -->|Descope MCP auth| Identity
```

### Request boundaries

- **Frontend to backend:** requests use the HTTP `Authorization` header with the Descope session credential. Chat responses are Server-Sent Events (SSE).
- **Backend to Descope:** the backend validates user sessions and uses Descope management/outbound APIs to obtain Google Calendar access tokens. Google refresh tokens are not stored by this application.
- **Backend to Google:** calendar operations use short-lived access tokens and the Google Calendar v3 API.
- **Agent memory:** Mastra stores the latest messages and resource-scoped working memory in `backend/mastra.db` by default.
- **Application database:** PostgreSQL stores the local user identity mapping and calendar connection status only.

### Backend layers

1. **Routes** validate request shape and select the operation.
2. **`requireSession` middleware** validates the Descope session and upserts the local user.
3. **Services** contain agent, calendar, token, and connection business logic.
4. **Repositories** isolate PostgreSQL reads and writes.
5. **Configuration modules** construct Descope, database, and Mastra clients.

## Repository layout

```text
.
|-- backend/
|   |-- src/
|   |   |-- config/                 # Descope, agent instructions, and memory setup
|   |   |-- db/                     # PostgreSQL pool
|   |   |-- mcp/                    # Descope-authenticated MCP endpoint and tools
|   |   |-- middleware/             # Session authentication
|   |   |-- repositories/           # PostgreSQL persistence
|   |   |-- routes/                 # Express HTTP routes
|   |   `-- services/               # Agent, calendar, token, and connection logic
|   |-- scripts/migrate.ts          # Ordered SQL migration runner
|   |-- sql/                        # Numbered schema migrations
|   `-- package.json
|-- frontend/
|   |-- src/app/                   # Next.js App Router pages
|   |-- src/components/             # Auth, dashboard, chat, and UI components
|   |-- src/lib/                    # API, agent, connection, and shared types
|   `-- package.json
|-- docker-compose.yml             # Local PostgreSQL 16
|-- main.py                         # Initial Python scaffold, currently unused
|-- pyproject.toml                 # Root Python project metadata
`-- README.md
```

## Technology stack

| Area | Technology |
| --- | --- |
| Web UI | Next.js 16, React 19, TypeScript, Tailwind CSS 4 |
| Authentication | Descope Next.js SDK and Node SDK |
| API | Express 5, TypeScript, Zod |
| Agent runtime | Mastra Agent, Mastra Memory, OpenAI-compatible model configuration |
| Calendar | Google APIs / Google Calendar v3 |
| Durable relational data | PostgreSQL 16, `pg` |
| Agent storage | `@mastra/libsql` and a local SQLite-compatible file |
| Local infrastructure | Docker Compose |
| MCP | `@descope/mcp-express` |

## Prerequisites

- Node.js compatible with the installed Next.js and TypeScript toolchains.
- npm.
- Docker Desktop with Compose support, or a PostgreSQL 16 instance.
- A Descope project with the `sign-up-or-in` flow enabled.
- A Descope management key with the permissions required by outbound application token retrieval.
- A configured Descope Google Calendar outbound application. Its ID must match `DESCOPE_CALENDAR_CONNECTION_ID`.
- An OpenAI API key, unless the configured Mastra model provider is changed.

## Local development

### 1. Start PostgreSQL

From the repository root:

```bash
docker compose up -d postgres
```

The default Compose configuration exposes PostgreSQL on `localhost:5442` and creates:

```text
database: agentic_calendar_app_db
user:     postgres
password: postgres
```

Do not use these development credentials in a shared or production environment.

### 2. Configure the backend

Create `backend/.env` from the template below and fill in the provider-specific values:

```dotenv
PORT=4000
DATABASE_URL=postgresql://postgres:postgres@localhost:5442/agentic_calendar_app_db
APP_URL=http://localhost:3000

DESCOPE_PROJECT_ID=your-descope-project-id
DESCOPE_MANAGEMENT_KEY=your-management-key
DESCOPE_CALENDAR_CONNECTION_ID=google-calendar

OPENAI_API_KEY=your-openai-api-key
AI_MODEL=gpt-4o-mini

# Optional MCP configuration
SERVER_URL=http://localhost:4000
DESCOPE_MCP_SERVER_WELL_KNOWN_URL=
```

Run the numbered SQL migrations:

```bash
cd backend
npm ci
npm run migrate
```

### 3. Configure the frontend

Create `frontend/.env.local`:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_DESCOPE_PROJECT_ID=your-descope-project-id
```

Install dependencies and start the development server:

```bash
cd frontend
npm ci
npm run dev
```

Open [http://localhost:3000](http://localhost:3000), sign in, connect Google Calendar, and start a chat.

### 4. Start the backend

In a second terminal:

```bash
cd backend
npm run dev
```

The API listens on [http://localhost:4000](http://localhost:4000). Verify it can reach PostgreSQL:

```bash
curl http://localhost:4000/health
```

Expected response when the database is available:

```json
{
  "status": "ok",
  "service": "agentic-calendar-app",
  "database": "up"
}
```

The backend creates `backend/mastra.db` on first use. This file contains agent memory and must be stored on durable, private storage if it is used outside local development.

## Configuration

### Backend variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `PORT` | No | Express listen port; defaults to `4000`. |
| `DATABASE_URL` | Yes | PostgreSQL connection string. |
| `APP_URL` | Yes | Allowed browser origin for CORS and default calendar redirect. |
| `DESCOPE_PROJECT_ID` | Yes | Descope project identifier. |
| `DESCOPE_MANAGEMENT_KEY` | Yes for calendar operations | Server-side key used to fetch outbound application tokens. |
| `DESCOPE_CALENDAR_CONNECTION_ID` | Yes for calendar operations | Descope outbound application ID; defaults to `google-calendar`. |
| `OPENAI_API_KEY` | Yes for agent chat | API key required by the configured AI model provider. |
| `AI_MODEL` | No | Mastra model name suffix; defaults to `gpt-4o-mini` and is prefixed with `openai/`. |
| `SERVER_URL` | MCP only | Public backend URL used in MCP endpoint metadata. |
| `DESCOPE_MCP_SERVER_WELL_KNOWN_URL` | MCP only | Enables the Descope MCP router when set. |

### Frontend variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_API_URL` | No | Backend base URL; defaults to `http://localhost:4000`. |
| `NEXT_PUBLIC_DESCOPE_PROJECT_ID` | Yes | Descope project ID used by the browser SDK. |

Only variables prefixed with `NEXT_PUBLIC_` should be exposed to the browser. Never place management keys, API keys, or database credentials in frontend environment files.

## Runtime flows

### Authentication

1. The frontend renders the Descope `sign-up-or-in` flow.
2. Descope issues a session token and refresh token managed by the SDK.
3. Protected API calls send the session credential in the HTTP `Authorization` header.
4. `requireSession` validates the token, extracts the Descope subject, and upserts a local `users` row.
5. The request receives both the external `authUserId` and internal PostgreSQL `userId`.

### Calendar connection

1. The user clicks **Connect**.
2. The frontend obtains the Descope refresh token and posts it to `/api/connections/connect`.
3. The backend asks Descope to start the outbound Google connection and marks the local status as `pending`.
4. The browser follows the returned redirect URL.
5. The user can click **Refresh**; the backend asks Descope for the outbound access token and stores `connected` or `disconnected` status.
6. Calendar tools fetch an access token from Descope when they execute; access tokens are not persisted in PostgreSQL.

### Chat streaming

1. The frontend creates a UUID thread ID for a new conversation.
2. It posts `{ message, threadId }` to `/api/agent/chat`.
3. The backend validates the payload with Zod and opens an SSE response.
4. Mastra streams planning, tool-call progress, and text-delta events.
5. The frontend appends token events to the assistant message as they arrive.
6. Mastra persists messages and resource-scoped working memory in LibSQL.

## HTTP API

All routes under `/api` require an `Authorization` header containing the authenticated Descope session credential unless noted otherwise.

### Health

`GET /health`

Checks database connectivity. Returns `200` when PostgreSQL is reachable and `503` otherwise.

### Connections

`GET /api/connections`

Returns the current Google Calendar connection label and status:

```json
{
  "connection": {
    "label": "Google Calendar",
    "status": "connected"
  }
}
```

`POST /api/connections/connect`

Starts the Descope outbound connection:

```json
{
  "refreshToken": "descope-refresh-token",
  "redirectUrl": "http://localhost:3000/dashboard"
}
```

Returns `{ "url": "..." }` for the browser to follow.

`POST /api/connections/refresh-status`

Refreshes the local connection status by checking Descope for the user's outbound token.

### Agent threads

`GET /api/agent/threads`

Returns up to 30 most recently updated threads for the authenticated Descope resource.

`GET /api/agent/threads/:threadId`

Returns the messages for a UUID thread owned by the authenticated resource. Cross-user thread access is rejected by the service layer.

### Agent chat

`POST /api/agent/chat`

Request:

```json
{
  "message": "What's on today?",
  "threadId": "00000000-0000-0000-0000-000000000000"
}
```

The response is `text/event-stream`. Each event is sent as an SSE `data` record:

```json
{ "type": "started", "message": "Agent is planning" }
{ "type": "progress", "message": "Running listUpcomingMeetings" }
{ "type": "token", "token": "You have..." }
{ "type": "completed", "message": "done" }
```

Errors are emitted as `{ "type": "error", "message": "..." }` before the stream closes.

## Agent tools

The Mastra agent exposes these tools:

| Tool | Purpose |
| --- | --- |
| `listUpcomingMeetings` | Lists future events, optionally restricted to today and capped at 20 results. |
| `checkCalendarBusy` | Queries Google free/busy data for an ISO-8601 interval. |
| `createMeeting` | Creates an event, sends attendee invitations, and adds Google Meet by default. |
| `rescheduleMeeting` | Patches the start and end of an existing event and sends updates. |
| `cancelMeeting` | Deletes an event and sends cancellation updates. |

The agent instructions define response conventions, including short agenda lists, concise summaries, Markdown links, and no invented details. Tool inputs are validated with Zod before execution.

## Data model and persistence

### PostgreSQL

`users`

| Column | Description |
| --- | --- |
| `id` | Internal UUID primary key. |
| `auth_user_id` | Unique Descope subject identifier. |
| `email` | Latest known email claim, when present. |
| `created_at` | Row creation timestamp. |

`connections`

| Column | Description |
| --- | --- |
| `user_id` | Foreign key to `users`, cascading on delete. |
| `provider` | Currently constrained to `calendar`. |
| `status` | `connected`, `disconnected`, or `pending`. |
| primary key | `(user_id, provider)`. |

Migrations are plain SQL files in `backend/sql`, applied alphabetically by `backend/scripts/migrate.ts`. The migration runner is currently intentionally simple and does not maintain a migration-history table; run it only against a controlled database and make migrations idempotent.

### Mastra LibSQL

The backend creates `mastra.db` in its current working directory. It stores:

- Thread metadata and titles.
- Conversation messages.
- The latest message context.
- Resource-scoped working memory for scheduling preferences.

For multiple backend replicas, move this store to a shared, supported Mastra storage backend or provide a shared filesystem strategy. A local file is not sufficient for horizontally scaled production deployments.

## MCP integration

When `DESCOPE_MCP_SERVER_WELL_KNOWN_URL` is set, the backend mounts a Descope-authenticated MCP server. The endpoint is:

```text
POST /mcp
```

The current MCP surface exposes `listUpcomingMeetings`, including the optional `maxResults` and `todayOnly` arguments. `GET /mcp` intentionally returns `405 Method Not Allowed` with `Allow: Post`.

Set `SERVER_URL` to the externally reachable HTTPS API URL in deployed environments. Do not enable the endpoint without configuring authentication and reviewing the provider scopes.

## Build and verification

### Backend

```bash
cd backend
npm ci
npm run build
npm run start
```

### Frontend

```bash
cd frontend
npm ci
npm run build
npm run start
```

There is currently no automated test suite configured in either package. At minimum, validate:

- `npm run build` succeeds in both `backend` and `frontend`.
- `GET /health` returns database status.
- Sign-in and sign-out work.
- Calendar connect, redirect, and refresh work.
- Chat SSE events stream and persist.
- A second request can resume an existing thread.
- Calendar create, reschedule, cancel, and free/busy operations behave as expected.

## Deployment guidance

Deploy the frontend and backend as separate services:

1. Build the backend with `npm run build` and run `npm run start`.
2. Build the frontend with `npm run build` and run `npm run start`.
3. Run PostgreSQL as a managed database or a separately operated stateful service.
4. Configure `APP_URL` and `NEXT_PUBLIC_API_URL` with the final HTTPS origins.
5. Configure Descope allowed origins, redirect URLs, and outbound Google Calendar application settings.
6. Provide durable shared storage for Mastra memory, or replace the default LibSQL file store.
7. Terminate TLS at the platform ingress and forward SSE responses without buffering or aggressive idle timeouts.
8. Add metrics, structured logs, error tracking, and alerting around health, auth failures, provider failures, and stream termination.

The current repository does not include a Dockerfile, CI workflow, deployment manifests, structured logging setup, or automated migrations at service startup. These should be added as part of the target hosting platform implementation rather than assumed.

## Security considerations

- Keep `DESCOPE_MANAGEMENT_KEY`, `OPENAI_API_KEY`, and `DATABASE_URL` server-side only.
- Use HTTPS for every non-local environment.
- Restrict CORS to the exact frontend origin; do not use a wildcard with credentials.
- Configure Descope redirect and origin allowlists narrowly.
- Use a secret manager instead of committed `.env` files.
- Rotate management keys and API keys according to organizational policy.
- Apply least-privilege scopes to Descope MCP and Google Calendar integrations.
- Add request rate limiting and abuse controls to `/api/agent/chat`.
- Add payload and date-range limits for calendar operations.
- Avoid logging bearer tokens, refresh tokens, calendar access tokens, prompts containing sensitive information, or raw provider responses.
- Treat SSE as a long-lived authenticated request and enforce server and proxy timeouts intentionally.
- Encrypt and restrict access to the Mastra memory store; it may contain user conversations and scheduling preferences.
- Review tenant isolation whenever adding new memory queries or tools. Current thread reads are scoped by `authUserId`.

## Production checklist

### Application

- [ ] Replace the root Python placeholder or remove it from the production build if it is not needed.
- [ ] Add unit and integration tests for routes, repositories, agent events, and calendar operations.
- [ ] Add centralized error handling and structured logging.
- [ ] Add rate limiting, request IDs, and provider retry/backoff policies.
- [ ] Decide whether agent memory remains file-backed or migrate it to shared production storage.
- [ ] Validate all date/time inputs and define an explicit timezone policy.

### Infrastructure

- [ ] Use managed PostgreSQL with backups, TLS, monitoring, and least-privilege credentials.
- [ ] Replace default Compose credentials outside local development.
- [ ] Add CI for dependency installation, type checking, builds, tests, and dependency auditing.
- [ ] Add a deployment artifact such as a Dockerfile and health-checked service definition.
- [ ] Configure graceful shutdown for Express and database pool connections.
- [ ] Configure an ingress that supports unbuffered SSE and suitable idle timeouts.

### Identity and integrations

- [ ] Configure Descope production origins and redirect URLs.
- [ ] Verify Google Calendar consent, scopes, token expiry, and revocation behavior.
- [ ] Store secrets in a managed secret store.
- [ ] Configure and test MCP only if it is required.

## Troubleshooting

### `401 Unauthorized` from the API

Confirm the frontend and backend use the same Descope project, the session credential is current, and the request contains the HTTP `Authorization` header.

### Calendar status remains disconnected

Check `DESCOPE_MANAGEMENT_KEY`, `DESCOPE_CALENDAR_CONNECTION_ID`, the Descope outbound application configuration, and the user's completed redirect flow. Then use **Refresh** in the dashboard.

### Chat fails before streaming

Check `OPENAI_API_KEY`, `AI_MODEL`, backend logs, and that the backend can create or write `mastra.db`. Also confirm the request contains a valid UUID `threadId`.

### Health returns `503`

Check that PostgreSQL is running, port `5442` is available, `DATABASE_URL` matches the Compose service, and `npm run migrate` has been executed.

### Browser reports a CORS error

Ensure `APP_URL` exactly matches the frontend origin, including scheme and port. Restart the backend after changing environment variables.
