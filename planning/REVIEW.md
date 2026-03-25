# REVIEW.md — FinAlly PLAN.md Architectural Review

## Scope

This review covers `planning/PLAN.md` against the existing codebase (`backend/app/market/`, `backend/pyproject.toml`, `backend/CLAUDE.md`, `planning/MARKET_DATA_SUMMARY.md`). It evaluates execution risk, internal consistency, and fitness as an implementation contract for autonomous coding agents. Issues are grouped by severity.

---

## Critical Issues

### 1. SSE broadcasts all tickers to all clients — no per-user filtering, but multi-user future is implied

**Section 6, SSE Streaming:** "Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist."

The existing `stream.py` calls `price_cache.get_all()` and sends every cached ticker to every connected client. The plan acknowledges "in the single-user model this is equivalent," which is accurate today. However, Section 3 (SQLite rationale) says "No auth = no multi-user = no need for a database server" and the schema includes `user_id` columns throughout.

The one-way door: if a second user's watchlist diverges from the first user's, both currently receive identical SSE payloads built from a shared global cache. Adding per-user SSE filtering would require either (a) per-user caches, or (b) filtering at the SSE generator layer keyed by `user_id`. Neither is mentioned. The plan should explicitly state that multi-user SSE is out of scope and document what would need to change, rather than leaving it implied by the `user_id` columns and vague "future-proofing" language.

**Recommendation:** Add a note to Section 6 and Section 13 explicitly marking per-user SSE as a known limitation and out of scope for v1. Remove or qualify the "supports future multi-user scenarios without changes to the data layer" claim — SSE changes *would* be required.

---

### 2. `backend/db/` directory in the spec is contradicted by the actual codebase

**Section 4, Key Boundaries:** "`backend/db/` contains schema SQL definitions and seed logic."

The codebase has no `backend/db/` directory. `backend/CLAUDE.md` and the market data summary both confirm schema initialization lives in code under `backend/app/`. Section 13 acknowledges this ambiguity as Open Question #1, but it has been left unresolved while the market data agent has already made an implementation choice (in-code initialization).

This is a live contradiction that will cause the Database agent to either create `backend/db/` (deviating from existing code) or not create it (deviating from the plan). Both outcomes produce a codebase that doesn't match the spec.

**Recommendation:** Close this open question now. The in-code initialization approach is already established. Remove `backend/db/` from the directory tree in Section 4 and update the Key Boundaries text to reflect that schema and seed logic live in `backend/app/` (likely `backend/app/db/` or `backend/app/core/`). This is a decision, not a question.

---

### 3. Database initialization described as "lifespan startup" but spec text says "lazily initializes on first request"

**Section 4, Key Boundaries:** "The backend lazily initializes the database on first request."

**Section 7, SQLite with Lazy Initialization:** "The backend initializes the SQLite database during FastAPI's lifespan startup event."

These are contradictory. Lazy initialization on first request means the first API call incurs the latency of creating tables and seeding data. Lifespan startup initialization means it happens before the first request, which Section 7 immediately uses to claim "No first-request latency penalty — the app is ready to serve immediately."

Lifespan startup is the correct choice and matches modern FastAPI practice (the `@asynccontextmanager` lifespan pattern). The "lazy on first request" phrasing in Section 4 is a holdover that contradicts the intent. An agent reading Section 4 before Section 7 could implement lazy initialization and then be confused by the latency claim.

**Recommendation:** Remove "lazily initializes the database on first request" from Section 4 Key Boundaries. Replace with "initializes the database during the FastAPI lifespan startup event."

---

### 4. `/api/chat` has no session management endpoints in the API table

**Section 9, Chat Sessions:** "The frontend provides a way to list and switch between previous sessions."

**Section 8, API Endpoints:** The Chat section lists only `POST /api/chat`.

There are no endpoints defined for:
- `GET /api/chat/sessions` — list sessions
- `POST /api/chat/sessions` — create a new session
- `GET /api/chat/sessions/{session_id}/messages` — load a previous session's history

The frontend cannot implement session listing or switching without these endpoints. An agent building the chat UI will either invent endpoints (creating a contract mismatch) or omit the feature.

**Recommendation:** Add the missing session management endpoints to Section 8. At minimum: `GET /api/chat/sessions` (list sessions with created_at and a message preview) and `POST /api/chat/sessions` (start a new session, returns `session_id`). Optionally add `GET /api/chat/sessions/{session_id}/messages`.

Also clarify: does the frontend pass `session_id` in `POST /api/chat`? The current schema shows no `session_id` in the request body — only in `chat_messages` as a storage field. The frontend-to-backend contract for session tracking is undefined.

---

### 5. Trade execution atomicity is unspecified — race condition on concurrent trades

**Section 7, Schema:** The `positions` table uses `UPSERT` semantics implied by the UNIQUE constraint on `(user_id, ticker)`, and `users_profile` holds `cash_balance` as a plain REAL column.

**Section 8:** `POST /api/portfolio/trade` validates against "sufficient cash for buys, sufficient shares for sells."

The plan does not specify that trade execution must be atomic (read cash balance -> validate -> deduct -> write position in a single transaction). With SQLite's default isolation, two rapid POST requests could both read the same cash balance, both pass validation, and both execute — resulting in a negative balance. While a single-user app makes this less likely, it is not impossible (double-click, concurrent LLM-initiated trade + manual trade).

**Recommendation:** Add an explicit requirement in Section 7 or 8 that trade execution must use a SQLite transaction covering the cash balance check, deduction, position update, and trade log insert as a single atomic operation. This prevents double-spending bugs and is the correct pattern for any financial transaction, even simulated.

---

## Important Issues

### 6. `daily change %` in the watchlist panel has no data source

**Section 10, Watchlist panel:** "ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart"

The SSE `PriceUpdate` model (confirmed in `models.py`) contains `change_percent` — but this is the percent change from the *previous SSE update* (500ms ago), not from the market open or 24-hour baseline. The plan does not specify how to derive a true daily change percentage. The Massive API (`snap.last_trade.price`) provides no daily open or previous close. The simulator has no concept of a "session."

This means "daily change %" as described in the UI cannot actually be a daily figure with either data source. An agent will either display a meaningless value (tick-over-tick change labeled as "daily"), silently omit the column, or invent a field not present in the schema.

**Recommendation:** Either (a) clarify that "daily change %" means "change since page load / first SSE tick for that ticker" (update the label accordingly in the UI spec), or (b) add a `day_open_price` field to the seed data and `PriceUpdate` model to enable a proper daily change calculation. Option (a) is simpler and honest.

---

### 7. Main chart area data source is undefined

**Section 10, Main chart area:** "larger chart for the currently selected ticker, with at minimum price over time."

There is no API endpoint that returns historical price data for a single ticker. `GET /api/portfolio/history` returns total portfolio value snapshots, not per-ticker prices. The SSE stream only holds what has been received since page load.

An agent building the main chart will face a choice with no guidance:
- Use only what the SSE stream has accumulated in-browser (sparse on first load, resets on reconnect)
- Invent a `GET /api/market/history/{ticker}` endpoint not in the plan
- Use the portfolio history endpoint inappropriately

**Recommendation:** Either explicitly state that the main chart is powered by frontend-accumulated SSE data (with a note that it starts sparse on load), or add a `GET /api/market/tickers/{ticker}/history` endpoint and a corresponding backend in-memory price history buffer (e.g., last N price points per ticker in the PriceCache or a separate store). The current plan implies historical data without specifying its source.

---

### 8. `portfolio_snapshots` records `total_value` but the composition is undefined

**Section 7, portfolio_snapshots:** Only `total_value` is stored per snapshot.

**Section 8, GET /api/portfolio/history:** Returns "portfolio value snapshots."

`total_value` is presumably `cash_balance + sum(position.quantity * current_price)`. But at query time, prices in the cache will differ from prices at snapshot time. The plan does not specify whether `total_value` is recorded at snapshot time (correct) or recomputed at query time (incorrect). An agent could implement either.

Additionally, if a user has zero positions and $10,000 cash, the P&L chart will show a flat line at $10,000. The plan does not address what the chart looks like for a new user with no trades, or how to make an empty-state chart visually acceptable.

**Recommendation:** Add one sentence to Section 7 clarifying that `total_value` is recorded at the time of the snapshot (point-in-time), not recomputed at query time.

---

### 9. The LLM mock mode (`LLM_MOCK=true`) has no specified mock response schema

**Section 9, LLM Mock Mode:** "the backend returns deterministic mock responses instead of calling OpenRouter"

The structured output schema is defined (Section 9), but the content of the mock response is not. An agent implementing mock mode will invent a mock, making E2E tests brittle across agents — a test written by one agent expecting "I've bought 10 shares of AAPL" will fail if another agent's mock returns something different.

**Recommendation:** Define the canonical mock response in the plan. For example:
```json
{
  "message": "I've analyzed your portfolio. You have $10,000 in cash and no open positions.",
  "trades": [],
  "watchlist_changes": []
}
```
This should be checked into `backend/app/` as a constant, not embedded in test files.

---

### 10. `watchlist_changes` action field values are not defined

**Section 9, Structured Output Schema:**
```json
{"ticker": "PYPL", "action": "add"}
```
The `action` field is shown as `"add"` in the example but no exhaustive list of valid values is given. The plan mentions "add/remove tickers" but doesn't state whether `"remove"` or `"delete"` is the correct value. An LLM agent will guess; a backend agent will implement one; a frontend agent will display the other.

**Recommendation:** Add a line to Section 9 explicitly stating: `action` must be one of `"add"` or `"remove"`. This is a simple contract clarification.

---

### 11. Dockerfile copies `backend/` but `.env` is read from the project root at runtime

**Section 5, Behavior:** "The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)."

**Section 11, Docker Volume:** `docker run --env-file .env ...`

`--env-file` injects variables into the container environment. This is correct. However, some Python dotenv libraries also look for a `.env` file on disk (e.g., `python-dotenv`'s `load_dotenv()`). If the backend agent adds `python-dotenv` and calls `load_dotenv()`, the `.env` file won't exist inside the container (it's gitignored, not copied), and the app will silently fall back to environment variables — which is fine via `--env-file`, but the behavior is fragile and agent-dependent.

The plan does not specify whether the backend should use `python-dotenv` or rely purely on environment variables injected by Docker. `pyproject.toml` does not currently include `python-dotenv`.

**Recommendation:** Add a note to Section 5 or 11 stating: "The backend reads configuration from environment variables only (`os.environ`). Do not use `python-dotenv` or `load_dotenv()` — the `.env` file is not present inside the container." This prevents a subtle misconfiguration.

---

### 12. `positions` table has no row for zero-quantity after a full sell

**Section 7, positions:** "one row per ticker per user"

The plan does not specify what happens to a `positions` row when all shares are sold (`quantity` reaches 0). Options:
- Delete the row (cleaner, but complicates avg_cost history)
- Set `quantity = 0` and keep the row (simpler, but `GET /api/portfolio` must filter these out)

An agent will make this choice silently. If rows are kept at `quantity=0`, the portfolio heatmap agent must know to exclude them. If rows are deleted, the trade history agent must not rely on the positions table for historical cost basis.

**Recommendation:** Explicitly state: "When a sell brings quantity to zero, delete the positions row." This is the cleaner choice and consistent with the spec's implied behavior (positions table shows current holdings only).

---

## Ambiguities (Lower Risk but Actionable)

### 13. No specification for how `GET /api/watchlist` handles tickers without prices yet

When the Massive API is used and the first poll hasn't completed, `cache.get(ticker)` returns `None`. The watchlist endpoint would need to return the ticker with a null price. The frontend must handle this. Neither the API response shape nor the frontend behavior is defined for this state.

### 14. The "connection status indicator" reconnection state is not tied to SSE events

**Section 2:** "yellow = reconnecting." `EventSource` fires `onerror` on disconnect and reconnects automatically. The transition from green to yellow to green requires the frontend to track `EventSource` state changes. This is standard but the plan doesn't specify the timeout before showing "reconnecting" vs "disconnected." A frontend agent will pick an arbitrary timeout.

### 15. No chat endpoint specifies how `session_id` is managed in the request

`POST /api/chat` request body is described as just a message. There's no mention of whether `session_id` is part of the request, a header, or managed entirely server-side. This is a contract gap between frontend and backend agents.

### 16. `portfolio_snapshots` pruning frequency is oddly specific without justification

"Prune once per ~100 writes" (every 50 minutes). The math: 30-second interval x 100 = 50 minutes. This is a reasonable heuristic but documenting it as ~50 minutes would be clearer than "every ~100 writes" since the 30-second interval is itself configurable.

---

## One-Way Doors

These decisions, once implemented, are difficult to reverse without significant rework.

### A. SQLite WAL mode not specified

SQLite's default journal mode (`DELETE`) serializes all writes. With concurrent SSE reads, snapshot writes, and trade writes, this can cause `SQLITE_BUSY` errors under load. WAL mode allows concurrent readers and a single writer. Not specifying WAL mode means the database agent will likely use the default, and enabling WAL mode later requires schema migration awareness. **Recommendation:** Add `PRAGMA journal_mode=WAL;` to the initialization sequence in the spec.

### B. `user_id` column defaults without a users table foreign key

All tables have `user_id TEXT DEFAULT "default"` but there is no foreign key constraint to `users_profile`. This means orphaned rows are possible if a user is ever deleted. For v1 this doesn't matter, but the "future multi-user" framing makes this a real concern. Enforce the constraint now or document explicitly that it is not enforced.

### C. `trades` table has no `session_id` or link to the chat action that triggered a trade

The plan says `chat_messages.actions` stores executed trades as JSON. But `trades` rows themselves have no reference back to the chat message that triggered them. If the frontend wants to show "this trade was executed by the AI" vs. "this trade was manual," there's no way to determine this from the database. Adding a nullable `source` column to `trades` now is trivial; adding it later requires a schema migration.

### D. In-memory price history is not specified for the main chart

If the frontend accumulates price history in-browser from SSE (the most likely implementation given no history API), a page refresh loses all chart data. This is a UX decision with a one-way door: once users expect chart continuity across refreshes, a backend price history store must be added. The plan should explicitly acknowledge this limitation.

---

## Open Questions Status

The plan's Section 13 lists three open questions. Recommended resolutions:

1. **`backend/db/` ambiguity** — Close it: remove `backend/db/` from the tree, use `backend/app/` for all initialization code (already the de facto standard from the market data implementation).
2. **Database path resolution** — Add `DATABASE_PATH` to Section 5 with default `db/finally.db`. This is a one-line env var change that prevents hardcoded-path bugs in local dev.
3. **LLM model fallback** — For a course demo, define a single fallback: if `openrouter/openai/gpt-oss-120b` fails, log the error and return an HTTP 503. Do not silently degrade. The user should know the chat is down rather than get a hallucinated response from a fallback model with a different capability profile.

---

## Summary

The plan is well-structured and covers the happy path clearly. The architecture choices are sound. The main risks for agent execution are:

| # | Issue | Impact |
|---|-------|--------|
| 1 | SSE multi-user claim contradicts single-cache architecture | Misleads agents on scope |
| 2 | `backend/db/` vs `backend/app/` contradiction | Agent creates wrong directory structure |
| 3 | "Lazy on first request" vs "lifespan startup" | Agent implements wrong initialization pattern |
| 4 | No session management endpoints defined | Frontend-backend contract gap for chat |
| 5 | No trade atomicity requirement | Silent double-spend bug |
| 6 | "Daily change %" has no computable data source | Frontend agent invents or omits feature |
| 7 | Main chart data source undefined | Agent invents incompatible solution |
| C | No `source` column on `trades` | Manual vs AI trades indistinguishable forever |
| A | SQLite WAL mode unspecified | Concurrent access errors under light load |

The most actionable fix is to close the three open questions in Section 13 before the next agent starts — they are already causing divergence between the spec and the built code.

---

*Review written against commit `14550e1` (main branch). Codebase state: market data subsystem complete (73 tests passing), all other components unbuilt.*
