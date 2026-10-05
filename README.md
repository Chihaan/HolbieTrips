# HolbieTrips

> [!WARNING]
> HolbieTrips is a deliberately vulnerable application-security training lab. Run it only on your own machine. Never expose it to the Internet or a shared network, and never enter real names, credentials, passport details, payment data, or other personal information.

HolbieTrips is a fictional travel-booking platform designed for hands-on OWASP Top 10:2025 practice. Developers can explore realistic business flows, investigate a challenge in a controlled environment, implement a remediation, and verify that both security and legitimate behaviour still work.

The entire platform runs locally. It sends no real e-mail, performs no real payment, uses no production secret, and requires no external runtime service.

## Technology stack

- React, Vite, and TypeScript for the browser application
- Node.js, Fastify, and TypeScript for the API
- PostgreSQL 17 with `pg`, SQL migrations, and SQL seeds
- Small Fastify simulators for payment and e-mail
- Vitest for API and domain tests
- Docker Compose for the final local environment

## Product features

- travel catalogue and destination search
- customer registration, login, logout, and role-based navigation
- traveller profile management
- booking creation, details, cancellation, and availability tracking
- promotional coupons
- simulated payment processing
- simplified support tickets
- support audit history
- local password-reset request flow

## Quick start

### Requirements

- Docker Desktop or Docker Engine
- Docker Compose v2
- an available local port `8080`

No local Node.js or PostgreSQL installation is required for the standard Docker workflow.

### Start the platform

From the repository root:

```bash
docker compose up --build
```

Wait until Compose reports the services as healthy, then open:

<http://127.0.0.1:8080>

Only this loopback address is published to the host.

### Stop the platform

Stop containers while preserving PostgreSQL data:

```bash
docker compose down
```

### Reset all local data

Remove the PostgreSQL volume, recreate the schema, and reload the demonstration data:

```bash
docker compose down -v
docker compose up --build
```

`docker compose down -v` permanently removes the local lab database volume.

## Demonstration accounts

All accounts and credentials below are fictional and intended only for this repository.

| Role | E-mail | Password |
|---|---|---|
| Customer | `alice.martin@example.test` | `Voyage!Alice2025` |
| Customer | `bruno.dupont@example.test` | `Voyage!Bruno2025` |
| Support | `support@holbietrips.test` | `Support!Holbie2025` |

Additional demonstration values:

- accepted test card: `4242 4242 4242 4242`
- declined-card convention: any fictional 16-digit card ending in `0000`
- coupons: `BIENVENUE10` and `FJORD15`

No payment is transmitted or processed outside the local Docker network.

## Architecture

```text
Browser
  │
  │ http://127.0.0.1:8080
  ▼
local-gateway
  │
  ▼
app ───────────────► PostgreSQL
  │
  ├────────────────► fake-payment
  │
  └────────────────► fake-mail
```

The production-style Vite build is served by the Fastify API. A small gateway is the only service attached to a host port. Application services communicate by Docker DNS on the private `lab` network.

### Compose services

| Service | Responsibility | Host exposure | Storage |
|---|---|---|---|
| `local-gateway` | Accepts loopback traffic and proxies it to the application | `127.0.0.1:8080` | none |
| `app` | Serves the React build and the Fastify API | none | none |
| `db` | Stores users, trips, bookings, payments, tickets, and audit events | none | named volume `postgres-data` |
| `fake-payment` | Produces deterministic local payment outcomes | none | memory only |
| `fake-mail` | Captures up to 50 simulated messages | none | memory only |

The Compose configuration does not use host networking or mount the Docker socket. PostgreSQL and internal HTTP services are not published to the host. Runtime service traffic is constrained to the declared Docker networks.

## Request flow

1. The browser requests the application from `127.0.0.1:8080`.
2. `local-gateway` forwards the request to the `app` service.
3. Fastify handles `/api/*` routes or serves the compiled React application.
4. Authenticated API requests resolve their bearer session from PostgreSQL.
5. Business route modules perform database work and call a simulator when needed.
6. The API returns JSON to React; no browser request is made to an internal service directly.

## Repository structure

```text
apps/
  api/
    src/
      app.ts                 Fastify composition root
      lib/                   shared HTTP, session, and audit helpers
      routes/                route modules grouped by business capability
      security.ts            reusable cryptographic and network helpers
  web/
    src/
      components/            reusable UI and modal components
      hooks/                 session and notification state
      pages/                 feature-level screens
      utils/                 display formatting helpers
      App.tsx                frontend composition and shared data flow
services/
  fake-payment/              local payment simulator
  fake-mail/                 in-memory message simulator
  local-gateway/             loopback-only reverse proxy
database/
  migrations/                PostgreSQL schema
  seeds/                     fictional demonstration records
  fixtures/                  local destination-pack fixtures
docs/
  challenges/                learner challenge briefs
tests/
  api/                       Vitest suites
compose.yaml                 final local topology
Dockerfile                   shared application image
```

## Database initialization

Compose mounts the migration and seed files into PostgreSQL's `/docker-entrypoint-initdb.d/` directory:

```text
10_001_initial.sql
20_001_demo.sql
```

The official PostgreSQL entrypoint runs these files alphabetically only when its data directory is empty. Consequently:

- `docker compose down` preserves the data and does not replay SQL initialization;
- `docker compose down -v` removes the volume, so initialization runs on the next start;
- editing a seed does not change an existing volume automatically.

## Code organization

### API

`apps/api/src/app.ts` is intentionally a composition root. It configures Fastify, shared hooks, error handling, static frontend delivery, and route registration. Business endpoints live in `apps/api/src/routes/`:

- `auth.ts`: accounts, sessions, and reset requests
- `catalog.ts`: destination data and trip search
- `profile.ts`: traveller information
- `bookings.ts`: booking lifecycle
- `payments.ts`: checkout and payment notifications
- `support.ts`: tickets and audit history
- `system.ts`: health and local operational endpoints

Shared request parsing, authentication, and audit writing live in `apps/api/src/lib/`. Route modules receive their dependencies through a typed context, which keeps them testable without a running PostgreSQL instance.

### Web application

`apps/web/src/App.tsx` coordinates navigation and shared data. Rendering is split between:

- `components/` for reusable layout, feedback, authentication, and booking UI;
- `pages/` for catalogue, bookings, profile, support, and audit screens;
- `hooks/` for session restoration and transient notifications;
- `types.ts` for API-facing view models;
- `utils/` for deterministic presentation helpers.

The browser treats the API as the source of truth for authentication. Cached local browser state is cleared when the server no longer recognizes a session.

## Tests

Run the complete Vitest suite inside the application image:

```bash
docker compose run --rm app npm test
```

Run it directly from a local Node.js installation:

```bash
npm ci
npm test
```

Build every TypeScript workspace and the Vite frontend:

```bash
npm run build
```

Some challenge tests characterize the lab's initial behaviour. When completing an exercise, keep the legitimate business test and turn the relevant characterization into a regression test for the remediation.

## Development workflow

The final Compose setup copies source code into the image and intentionally uses no source-code bind mount. A typical edit-and-test loop is:

```bash
npm test
npm run build
docker compose up --build -d
```

After rebuilding, refresh <http://127.0.0.1:8080>. Database data remains intact unless the named volume is explicitly removed.

For API-only development with a locally reachable PostgreSQL instance, the workspace also provides:

```bash
npm run dev
```

The Docker workflow remains the supported reference environment.

## Configuration

`.env.example` documents the fictional values used by the lab. Compose supplies the same local-only settings directly to its services, so copying the file is not required for the standard startup command.

These values are deliberately non-production credentials. Do not replace them with real secrets or reuse them outside the lab.

## Challenge flags

Exact flags are not committed or stored in PostgreSQL. The API derives them from an in-memory random value when the corresponding runtime condition is reached. They remain stable during one `app` process and change when that process restarts.

This prevents accidental source-code spoilers; it is not a security boundary against the owner of the local Docker environment.

## Safety boundaries

- Keep the published address bound to `127.0.0.1`.
- Do not add real providers, credentials, personal data, or payment details.
- Do not publish internal service ports.
- Do not enable host networking.
- Do not mount the Docker socket.
- Do not deploy this application to a public or shared environment.
- Use only the fictional records supplied by the repository.


n19T1IcOHAKwgcypPbkpDH_NbxTeVowtYMECWmEXFjQ
