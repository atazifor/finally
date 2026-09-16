# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: Claude Haiku 4.5 via the official Anthropic Python SDK, with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── app/db/               # Schema definitions, seed data, initialization logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/app/db/`** contains schema SQL definitions and seed logic. The backend initializes the database at startup — creating tables and seeding default data if the SQLite file doesn't exist or is empty (see §7).
- **Database file location**: set by `DB_PATH`. The default is `db/finally.db` under the **project root** (resolved from the backend package location, not the working directory), so running the backend from `backend/` still writes to `finally/db/`. The backend creates the directory if missing; `db/` is gitignored. In Docker, `DB_PATH=/app/db/finally.db` is set in the Dockerfile and that directory is a named volume (see §11).
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Anthropic API key for AI chat. Without it the app still runs; the chat panel says AI is not configured.
ANTHROPIC_API_KEY=your-anthropic-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Optional: SQLite file location (default: <project root>/db/finally.db; set to /app/db/finally.db in Docker)
DB_PATH=
```

Frontend (development only, in `frontend/.env.local`):

```bash
# Prefix for API calls when running `next dev`; leave unset for the production build
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

`.env.example` at the project root is committed with the backend variables above and empty/placeholder values.

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests); no API key needed
- If `LLM_MOCK` is not true and `ANTHROPIC_API_KEY` is empty **or rejected by the API** → the app starts normally and `POST /api/chat` returns 200 with the assistant message "AI chat is not configured — set ANTHROPIC_API_KEY in .env." (stored like any other reply; see §9)
- The backend reads `.env` from the project root: in Docker via `--env-file`; in development the backend loads it with python-dotenv, searching upward from the backend package so it is found when running from `backend/`

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.) for the default tickers
- Tickers without seed data (added by the user) start at a random price between $50 and $300 with default volatility
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all tracked tickers (see "Tracked Tickers" below) on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- Accepted limitation: prices only change as often as the poll interval, and not at all outside market hours, so flashes and sparklines are sparse in this mode

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, timestamp, and **session start price** for each ticker: the first price seen since the ticker became tracked. It resets if the ticker is removed and later re-added. (Implementation note: `PriceUpdate` is a frozen dataclass whose last field has a default, so the new field needs a default too; `PriceCache.update()` carries it forward.)
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### Tracked Tickers

Tracked tickers = **watchlist ∪ tickers with an open position**. A ticker you still own keeps getting prices after you remove it from the watchlist. The backend calls `add_ticker()` when a ticker becomes tracked and `remove_ticker()` only when it is in neither set (`remove_ticker()` already clears the ticker from the cache).

### Ticker Rules

- Tickers are uppercased and trimmed everywhere (REST endpoints and LLM actions)
- Valid format: 1–6 characters, letters and `.` only, starting with a letter; anything else is rejected with 400
- Any validly formatted ticker is accepted (the simulator invents a price). In Massive mode an unknown ticker is accepted but never gets a price
- A ticker with no price in the cache yet cannot be traded (400: "No price available for TICKER yet")

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The first message is `retry: 1000` (browser reconnects after 1s)
- The server checks the cache every ~500ms and sends an event only when prices changed
- Each event is **one JSON object holding every tracked ticker**:

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.5, "previous_price": 190.2, "session_start_price": 190.0, "timestamp": 1757000000.0, "change": 0.3, "change_percent": 0.1577, "direction": "up"}, "MSFT": {...}}
```

- `change` / `change_percent` compare with the previous update (tick-to-tick); `direction` is `"up"`, `"down"` or `"flat"`. `session_start_price` is new and must be added to the existing market data module
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite, Initialized on Startup

The backend initializes the database at startup (FastAPI lifespan). Startup order: (1) initialize/seed the database; (2) start the market data source with the tracked tickers from the database (watchlist ∪ positions), which seeds the price cache; (3) record the startup portfolio snapshot; (4) start the 30-second snapshot task. If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`, except `users_profile`, whose `id` is the user id. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded once at startup (so the P&L chart has a point on first launch), every 30 seconds by a background task, and immediately after each trade execution. The background task deletes snapshots older than 7 days.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — the action results from the chat response, see §9; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |
| GET | `/api/chat/history` | Stored conversation, oldest first (reloads the chat panel after a page refresh) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

### Response Shapes

All numbers are JSON numbers; all timestamps are ISO 8601 strings except SSE `timestamp` (Unix seconds).

- `GET /api/portfolio` → `{cash_balance, positions_value, total_value, unrealized_pnl, positions: [{ticker, quantity, avg_cost, current_price, market_value, unrealized_pnl, unrealized_pnl_percent}]}`. A position with no price yet uses `avg_cost` as `current_price`.
- `POST /api/portfolio/trade` → `{trade: {id, ticker, side, quantity, price, executed_at}, portfolio: <same as GET /api/portfolio>}`
- `GET /api/portfolio/history` → `[{recorded_at, total_value}]`, oldest first
- `GET /api/watchlist` → `[{ticker, price, session_start_price, added_at}]` (`price` fields are null until the first tick)
- `POST /api/watchlist` → 201 with `{ticker, added_at}`; 409 if already on the watchlist
- `DELETE /api/watchlist/{ticker}` → 204; 404 if not on the watchlist
- `POST /api/chat` → the stored assistant message: `{role: "assistant", content, actions: [{type, ticker, side?, quantity?, price?, status: "executed"|"failed", error?}], created_at}`. `price` is the fill price, present on executed trades
- `GET /api/chat/history` → `[{role, content, actions, created_at}]` (last 100 messages; same item shape as `POST /api/chat`, `actions` is null for user messages)
- `GET /api/health` → `{status: "ok"}`

**Errors** use FastAPI's standard body `{"detail": "human-readable message"}`: 400 for business-rule failures (insufficient cash, invalid ticker, no price), 404 for missing resources, 409 for duplicates, 422 for malformed request bodies.

### Trade Rules

- Fractional shares allowed; `quantity` is rounded to 4 decimal places and must be > 0 after rounding
- Fill price is the current cache price at execution time
- Buy: requires `cash_balance >= quantity × price`; new `avg_cost` is the weighted average of old and new shares
- Sell: requires owned quantity ≥ `quantity` (tolerance 1e-9); `avg_cost` does not change; the position row is deleted when remaining quantity < 1e-9
- The cash update, position update and trade insert happen in **one SQLite transaction**, so a manual trade and an AI trade cannot interleave
- After each trade, record a portfolio snapshot and update the tracked tickers (§6)

---

## 9. LLM Integration

The chat uses **Claude Haiku 4.5** through the official `anthropic` Python SDK (`uv add anthropic pydantic`), with Structured Outputs so the reply is validated against a pydantic model. The ANTHROPIC_API_KEY goes in the .env file in the project root; the SDK reads it from the environment.

### Calling Claude

```python
from typing import Literal
from anthropic import AsyncAnthropic
from pydantic import BaseModel

MODEL = "claude-haiku-4-5"  # exact ID, no date suffix
client = AsyncAnthropic()   # async: FastAPI handlers are async


class Action(BaseModel):
    type: Literal["trade", "watch", "unwatch"]
    ticker: str
    side: Literal["buy", "sell"] | None = None  # trade only
    quantity: float | None = None               # trade only


class ChatReply(BaseModel):
    message: str
    actions: list[Action] = []


response = await client.messages.parse(
    model=MODEL,
    max_tokens=2048,
    system=SYSTEM_PROMPT + "\n\n" + portfolio_context,
    messages=messages,  # built as in "How It Works" step 2
    output_format=ChatReply,
)
if response.stop_reason != "end_turn" or response.parsed_output is None:
    ...  # treat as a failed reply: friendly message, no actions
reply = response.parsed_output
```

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the last 20 messages from `chat_messages` (before storing the new one), drops leading assistant messages so the history starts with a user message, then appends the new user message
3. Builds the request: `system` = the fixed system prompt followed by the portfolio context; `messages` = the history from step 2
4. Calls Claude Haiku 4.5 via the Anthropic SDK, requesting structured output (see "Calling Claude")
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — replies are short and Haiku is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "actions": [
    {"type": "watch", "ticker": "PYPL"},
    {"type": "trade", "ticker": "AAPL", "side": "buy", "quantity": 10},
    {"type": "unwatch", "ticker": "NFLX"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `actions` (optional, default empty): One list of things to do, executed **in list order**
  - `trade`: `ticker`, `side` (`buy`/`sell`), `quantity`. Same validation as manual trades (§8 Trade Rules). A trade action missing `side` or `quantity` fails with an error
  - `watch` / `unwatch`: `ticker`. Adding a ticker already watched, or removing one not watched, counts as success

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

Each action runs independently: a failed action (e.g., insufficient cash) does not stop the ones after it. Every action gets a result (`status: "executed"` or `"failed"` with `error`) in the chat response and in `chat_messages.actions`, and the frontend shows each one inline. The stored message is the LLM's `message` as returned; failed actions are shown by the result list, not rewritten into the text.

### Context and Failure Handling

- Context sent to the LLM: see steps 1–3 above
- Missing key (checked before calling) or rejected key (`anthropic.AuthenticationError`): reply "AI chat is not configured — set ANTHROPIC_API_KEY in .env." (§5)
- If the API call fails for any other reason (`RateLimitError`, other `APIStatusError`, `APIConnectionError`; the SDK already retries these twice) or the reply cannot be parsed into the schema, `POST /api/chat` still returns **200** with an assistant message such as "Sorry, I couldn't process that — please try again" and no actions. No trades are executed. The user message and this reply are both stored
- The chat endpoint never returns a raw 500 to the chat panel

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling the Anthropic API. Mock rules (case-insensitive, whole message after trimming, checked in order):

1. `^(buy|sell)\s+(\d+(?:\.\d+)?)\s+([A-Za-z.]{1,6})$` → `message: "Executing <side> <quantity> <TICKER>."` with that one trade action
2. `^(watch|unwatch)\s+([A-Za-z.]{1,6})$` → `message: "Adding <TICKER> to watchlist."` or `"Removing <TICKER> from watchlist."` with that one action
3. Anything else → `message: "This is a mock response."` with no actions

This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), **session change %** (`price` vs `session_start_price`, i.e. since the app started; labelled "Chg (session)"), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here. History starts when the page loads (accepted limitation: a reload clears it).
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by unrealized P&L % (green = profit, red = loss, intensity capped at ±5%)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, unrealized P&L % (`unrealized_pnl_percent`)
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history (loaded from `GET /api/chat/history` on page load), loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Data Flow

- Watchlist rows come from `GET /api/watchlist`, never from the SSE keys (the stream also includes tickers that are only held as positions)
- Positions and cash come from `GET /api/portfolio`, fetched on page load and again after every manual trade and every chat response that contains actions. The watchlist is refetched the same way
- Live values are computed on the frontend: header total value, position current price / market value / unrealized P&L, and the heatmap all use the latest SSE price × quantity (+ cash for the total)
- One shared per-ticker price history store, fed by SSE and capped at ~1,000 points per ticker, backs both the sparklines and the main chart

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- **Recharts** for all charts: `LineChart` for sparklines, the main chart and the P&L chart; `Treemap` for the heatmap. Turn off animations on charts fed by the live stream
- Price flash effect: when a ticker's `price` differs from the last price the client received for it, briefly apply a green (higher) or red (lower) CSS class with a background color transition, then remove it. Do not flash on the first price or when the price is unchanged (every event repeats all tickers)
- All API calls use relative paths (`/api/*`) — in the built container the frontend and API share one origin, so no CORS is needed in production
- Development: run `next dev` (port 3000) and the backend (port 8000) separately. The frontend prefixes API calls with `NEXT_PUBLIC_API_BASE_URL` (unset in production, `http://localhost:8000` in development), and the backend allows CORS from `http://localhost:3000` only
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - WORKDIR /app; backend copied to /app
  - Copy frontend build output (`frontend/out`) to /app/static
  - ENV DB_PATH=/app/db/finally.db, STATIC_DIR=/app/static
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves all API routes and the static frontend on port 8000. The static files are mounted from `STATIC_DIR` after the `/api` routers so API routes take priority; if the directory doesn't exist (development), the mount is skipped.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

`finally-data` is a Docker-managed named volume (no folder in the repo). The backend writes `/app/db/finally.db` inside it, so data survives container restarts and rebuilds. To reset to a fresh $10,000 account, remove the volume: `docker volume rm finally-data`.

There is no top-level `docker-compose.yml`; the start/stop scripts are the supported way to run the app.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing (including an empty `actions` list), graceful handling of API errors and malformed responses, actions executed in order with independent failures, mock-mode rules
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism. The test app container uses **no persistent volume**, so every test run starts from a freshly seeded database. The "fresh start" scenario runs first; later scenarios assert changes relative to the state they observe, not absolute balances.

**Determinism**: Simulated prices are random, so tests assert relationships, never literal prices — e.g. cash after a buy equals cash before minus quantity × the executed price returned by the API.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send `buy 1 AAPL`, receive a response, trade execution appears inline and the position appears
- Chat history: reload the page and the previous messages are still shown
- SSE resilience: break the open stream connection, verify the status dot leaves green, then comes back to green and prices resume. Use a method that actually closes an already-open EventSource, and show the test fails when the frontend's reconnect handling is disabled before trusting it

---

## 13. Decisions Log

The plan was reviewed against the built market data code. The decisions below are already written into the sections above; this list records what was decided.

- **LLM**: Claude Haiku 4.5 via the Anthropic SDK replaces LiteLLM → OpenRouter → Cerebras (§9)
- **Default watchlist**: 10 tickers; PLTR removed because the simulator has no seed data for it (§7)
- **SSE format**: documented as the code sends it — one event with all tickers; `session_start_price` added (§6)
- **Change % in the watchlist**: since the app started, not daily (§6, §10)
- **Tracked tickers**: watchlist ∪ open positions (§6)
- **Ticker rules**: format-validated, uppercased; any valid ticker accepted in simulator mode; no trading before the first price (§6)
- **Startup without an API key**: app runs, chat says AI is not configured (§5)
- **Database**: initialized at startup; named Docker volume; snapshot at startup and 7-day retention (§7, §11)
- **API**: response shapes, error format, trade rules and chat history endpoint defined (§8)
- **LLM response**: one ordered `actions` list replaces separate `trades` / `watchlist_changes`; failures are independent and reported; 200 with a friendly message on LLM errors; mock rules defined (§9)
- **Frontend**: Recharts for all charts; live values computed client-side from SSE; one shared price history store; dev CORS for `localhost:3000` (§10)
- **Paths**: schema code in `backend/app/db/`; database at `<project root>/db/finally.db` (dev) or `/app/db/finally.db` (Docker); `db/` gitignored; `.env.example` committed (§4, §5, §11)
- **Chat details**: history window built before storing the new message; portfolio context goes in `system`; POST and history share one message shape; fill price in action results; anchored mock regexes (§8, §9)
- **Kept**: `user_id` columns, for future multi-user support
- **Removed**: top-level `docker-compose.yml` (duplicate of the start scripts)

### Known issue in existing code

`backend/app/market/stream.py` creates its `APIRouter` at module level, so calling `create_stream_router()` twice registers `/prices` twice on the same router. Move the router creation inside the factory when wiring up the FastAPI app.
