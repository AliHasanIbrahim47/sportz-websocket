# Sportz WebSocket

Real-time sports commentary API with a REST backend and WebSocket pub/sub for live match updates.

Built with Express, PostgreSQL, Drizzle ORM, and Arcjet for rate limiting and bot protection.

## Prerequisites

- [Node.js](https://nodejs.org/) 20+
- [pnpm](https://pnpm.io/)
- PostgreSQL running locally (or another host you can reach)

## Setup

1. Install dependencies:

```bash
pnpm install
```

2. Create a `.env` file in the project root:

```env
PORT=8000
HOST=0.0.0.0

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/sportz

API_URL=http://localhost:8000
DELAY_MS=500
SEED_MATCH_DURATION_MINUTES=120
SEED_FORCE_LIVE=0

ARCJET_MODE=DRY_RUN
ARCJET_KEY=your_arcjet_key
```

Update `DATABASE_URL` to match your PostgreSQL credentials.

3. Create the database and run migrations:

```bash
# create the database (adjust for your setup)
createdb sportz

pnpm db:migrate
```

## Running the server

Development (auto-restart on file changes):

```bash
pnpm dev
```

Production:

```bash
pnpm start
```

The server starts at `http://localhost:8000` by default.

## Seeding data

The seed script posts match and commentary data from `data.json` through the REST API. **The server must be running first.**

In one terminal:

```bash
pnpm dev
```

In another terminal:

```bash
node seed.js
```

Useful seed environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `API_URL` | — | Base URL of the running API (required) |
| `DELAY_MS` | `250` | Delay between commentary posts (ms) |
| `SEED_MATCH_DURATION_MINUTES` | `120` | Duration for auto-created matches |
| `SEED_FORCE_LIVE` | `0` | Set to `1` to force matches into live window |

If you see `ECONNREFUSED`, the API is not running or `API_URL` points to the wrong host/port.

## REST API

### List matches

```bash
curl http://localhost:8000/matches
curl "http://localhost:8000/matches?limit=10"
```

### Create a match

```bash
curl -X POST http://localhost:8000/matches \
  -H 'Content-Type: application/json' \
  -d '{
    "sport": "football",
    "homeTeam": "Arsenal",
    "awayTeam": "Chelsea",
    "startTime": "2026-06-13T18:00:00.000Z",
    "endTime": "2026-06-13T20:00:00.000Z"
  }'
```

### List commentary for a match

```bash
curl http://localhost:8000/matches/1/commentary
```

### Post commentary

```bash
curl -X POST http://localhost:8000/matches/1/commentary \
  -H 'Content-Type: application/json' \
  -d '{
    "minute": 12,
    "message": "Goal!",
    "team": "Home"
  }'
```

## WebSocket

Connect to:

```
ws://localhost:8000/ws
```

On connect, the server sends:

```json
{ "type": "welcome" }
```

### Subscribe to a match

Send JSON with an integer `matchId`:

```json
{ "type": "subscribe", "matchId": 1 }
```

Response:

```json
{ "type": "subscribed", "matchId": 1 }
```

### Unsubscribe

```json
{ "type": "unsubscribe", "matchId": 1 }
```

### Server events

| Event | Audience | When |
|-------|----------|------|
| `match_created` | All connected clients | A match is created via `POST /matches` |
| `commentary` | Subscribers of that match | Commentary is posted via `POST /matches/:id/commentary` |

Example commentary event:

```json
{
  "type": "commentary",
  "data": {
    "id": 1,
    "matchId": 1,
    "minute": 12,
    "message": "Goal!",
    "team": "Home"
  }
}
```

### Test page

Open the built-in WebSocket test UI:

```
http://localhost:8000/ws-test
```

It connects automatically, lets you load matches, subscribe/unsubscribe, and watch events in a live log.

### CLI testing

```bash
npx wscat -c ws://localhost:8000/ws
{"type":"subscribe","matchId":1}
```

## Database scripts

| Command | Description |
|---------|-------------|
| `pnpm db:generate` | Generate migrations from schema changes |
| `pnpm db:migrate` | Apply pending migrations |
| `pnpm db:studio` | Open Drizzle Studio |

Schema lives in `src/db/schema.js`. Migrations are in `drizzle/`.

## Project structure

```
src/
  index.js           # HTTP server entry point
  arcjet.js          # Arcjet security rules
  db/                # Drizzle schema and connection
  routes/            # REST route handlers
  validation/        # Zod request schemas
  ws/server.js       # WebSocket server
public/
  ws-test.html       # WebSocket test page
seed.js              # Seed script (uses REST API)
data.json            # Seed data
```

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `ARCJET_KEY` | Yes | Arcjet API key |
| `PORT` | No | HTTP port (default `8000`) |
| `HOST` | No | Bind address (default `0.0.0.0`) |
| `ARCJET_MODE` | No | `DRY_RUN` or `LIVE` (default `LIVE`) |
| `API_URL` | For seeding | Base URL for `seed.js` |
| `DELAY_MS` | No | Seed delay between posts |
| `SEED_MATCH_DURATION_MINUTES` | No | Seed match duration |
| `SEED_FORCE_LIVE` | No | Force seeded matches to be live |
