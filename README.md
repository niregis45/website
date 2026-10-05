# VoyageRwanda

Inter-provincial bus marketplace: a React SPA talking to a dependency-free Node
API, with MySQL as the system of record.

```
frontend/   React + TypeScript + Vite SPA (source)
backend/    Node HTTP API, zero web framework, mysql2 pool
database/   Schema, seed, views, verification and reset scripts
dist/       Built SPA, served by the backend in production
```

## Data flow

The browser never stores domain data. It calls `/api/*`, the API reads and writes
MySQL, and the response is the new UI state. That is what makes two browsers see
the same seat availability.

- Catalogue, search, facets, fare bounds: computed server-side per request
- Seat maps: polled from `/api/schedules/:id/seats` every 5s while a seat map is open
- Holds, bookings, cancellations, check-ins: all written through the API
- The only browser persistence is the session token and a list of booking
  references so anonymous passengers can reach their own tickets

## Requirements

- Node.js >= 20.6
- MySQL 8 (or MariaDB 10.6+)

## Database setup

Run the scripts in order against a fresh database. `01_database.sql` is
**destructive**: it drops and recreates the schema.

```
mysql -u root -p < database/01_database.sql
mysql -u root -p < database/02_tables.sql
mysql -u root -p < database/03_seed.sql
mysql -u root -p < database/04_views.sql
mysql -u root -p < database/05_verify.sql   # prints PASS
```

Seeded reference data: 6 provinces, 5 operators, 12 buses, 30 routes, one demo
operator, plus a daily schedule generator.

To wipe demo activity and restore the seed:

```
mysql -u root -p < database/06_reset_demo.sql
```

## Configuration

`backend/.env` is loaded by the npm scripts; real environment variables win, so
a hosting platform can inject everything without a file.

| Variable | Default | Purpose |
| --- | --- | --- |
| `DB_DRIVER` | `json` | `mysql` in production. `json` is the legacy fallback. |
| `DB_HOST` / `DB_PORT` | `127.0.0.1` / `3306` | MySQL endpoint |
| `DB_USER` / `DB_PASSWORD` | `root` / empty | MySQL credentials |
| `DB_NAME` | `voyagerwanda` | Schema name |
| `DB_CONNECTION_LIMIT` | `10` | Pool size |
| `HOST` | `127.0.0.1` | Bind address. Set `0.0.0.0` to accept external traffic. |
| `PORT` | `4000` | HTTP port |
| `CORS_ORIGINS` | same-origin only | Comma-separated allowlist when the API is on another origin |
| `AUTH_SECRET` | generated | Signing secret for session tokens. Set explicitly in production. |
| `AUTH_TOKEN_TTL_HOURS` | `12` | Session lifetime |
| `ALLOW_DEMO_ROUTES` | `1` in dev | Set `0` to hide `/api/dev/state` and `/api/admin/reset` |

With `DB_DRIVER=mysql` set, `data/db.json` is ignored entirely.

## Development

Two terminals:

```
npm --prefix backend run dev     # API on http://127.0.0.1:4000
npm --prefix frontend run dev    # Vite on http://127.0.0.1:5173
```

Vite proxies `/api` to the backend. Override with `VITE_DEV_API` if the backend
runs elsewhere. The only frontend dependency on a separate origin is
`VITE_API_BASE`, which should stay unset when using the proxy.

## Production

The backend serves the built SPA from `dist/`, so a single origin and a single
process are enough.

```
npm --prefix frontend ci
npm --prefix frontend run build
npm --prefix backend ci --omit=dev
NODE_ENV=production HOST=0.0.0.0 PORT=4000 DB_DRIVER=mysql \
  DB_HOST=... DB_USER=... DB_PASSWORD=... DB_NAME=voyagerwanda \
  AUTH_SECRET=... ALLOW_DEMO_ROUTES=0 npm --prefix backend start
```

Health check: `GET /api/health`.

Before going live:

1. Run the database scripts against the production schema.
2. Change or delete the seeded operator account
   (`operator@voyagerwanda.rw`), or create your own operator row.
3. Set `AUTH_SECRET` to a random 32+ character string.
4. Set `ALLOW_DEMO_ROUTES=0` so the reset endpoint is unreachable.

## Tests

```
npm --prefix backend test
```

`backend/test/mysql.integration.js` runs the real API against MySQL on a random
port and asserts persistence: 42 checks covering booking, seat conflicts, holds,
cancellation, fare overrides, suspension, counter persistence and reload.

**It performs resets.** Point `DB_NAME` at a throwaway schema, never production.