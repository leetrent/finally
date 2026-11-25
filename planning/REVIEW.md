# Implementation Review: `PLAN.md`

This review is ranked by implementation impact. It does not propose changes to
`PLAN.md`; it identifies decisions that need to be made before agents can
implement the remaining application consistently.

## Critical

### 1. The plan has no executable API contract

The endpoint table names routes and gives only two request snippets. It does
not define response bodies, HTTP status codes, validation rules, error format,
or whether fields are numbers, strings, nullable, or rounded. This leaves the
frontend, backend, LLM flow, and E2E suite unable to agree on a boundary.

Examples needing an explicit contract include the portfolio/position/P&L
calculation fields; `/api/portfolio/history` ordering and filters; watchlist
price and change fields; chat action-result/error fields; and the SSE envelope
(the existing market implementation sends one event containing a ticker-keyed
map, whereas the plan says an event contains one ticker). Specify Pydantic/
TypeScript schemas, validation errors, and representative JSON for every route
and SSE event before parallel implementation starts.

### 2. Watchlist removal conflicts with valuation and trade requirements

The plan says the poller tracks the union of watched tickers, and the supplied
market implementation removes a ticker from both the source and price cache on
watchlist removal. A user can therefore remove a ticker they still hold, after
which the position has no current price for the positions table, heatmap,
portfolio total, trade execution, or snapshots. The same issue applies to
ticker detail/history after removal.

Define the subscribed universe as at least `watchlist UNION open-position
tickers` (and state when to stop tracking a closed, un-watched ticker), or
forbid removal while holding it. Also define who reconciles that universe at
startup and atomically after manual or LLM watchlist/trade mutations.

### 3. Trade execution is not specified as an atomic financial transaction

Cash, position, immutable trade log, and immediate portfolio snapshot must be
committed together. The plan does not state transaction boundaries, concurrency
handling, idempotency, price freshness, or the exact cost-basis/sell algorithm.
Concurrent manual and chat requests could both pass an insufficient-cash or
insufficient-shares check, and a price update between validation and write
makes the advertised fill price ambiguous.

Specify a single database transaction that obtains one cached price, validates
positive finite quantity and side, changes cash/position, appends the trade,
removes or retains zero-quantity positions, and writes the snapshot. Decide
whether request IDs make retries idempotent. Use fixed-point/`Decimal` storage
(or explicitly defined integer minor units), not SQLite `REAL`, for cash,
prices, quantities, and P&L; binary floating point will otherwise create
visible balance and validation errors.

### 4. LLM integration cannot be implemented from the stated dependencies

Section 9 requires a `cerebras-inference` skill, but no such available skill is
named in the environment. The existing backend dependencies also do not
include LiteLLM or a structured-output/schema-validation library. “LiteLLM via
OpenRouter ... with Cerebras as the inference provider” does not give the
required provider/model configuration, version, request format, or a verified
structured-output capability/fallback.

Define the actual package versions and provider configuration, the exact
validated response model, timeouts/retries, and behavior for API failure,
invalid JSON, unsupported structured outputs, and a missing key. Do not leave
agents to invent a security-relevant execution path from a prompt.

### 5. LLM action semantics are dangerously underspecified

The schema permits arbitrary lists of buys, sells, adds, and removals, but does
not constrain ticker syntax, quantity bounds/precision, max action count,
ordering, duplicate/conflicting actions, or transactional behavior. It also
says failed actions are included “so the LLM can inform the user,” even though
the LLM call already completed; it is unclear whether the backend augments the
message, makes a second call, or merely returns an action failure.

Define server-side allowlists/limits and normalization, execute actions through
the same service as manual trades, return per-action outcomes, and make clear
whether one invalid action rolls back all actions or valid independent actions
continue. The server must treat LLM output as untrusted input; the prompt is
not validation or authorization.

## High

### 6. “Runs immediately” and “required OpenRouter key” are contradictory

First launch promises no setup beyond Docker, while `OPENROUTER_API_KEY` is
marked required and the Docker example always uses `--env-file .env`. No
behavior is defined when `.env` does not exist, the key is absent, or it is
invalid. Docker itself fails if an explicitly supplied env file is absent.

Choose a product behavior: make chat gracefully unavailable while all market
features run, enable mock mode by default for local launch, or require setup
and change the first-launch promise. Document both start scripts' exact env
handling and a health/readiness response that exposes degraded data/LLM state
without exposing secrets.

### 7. “Daily change %” and the requested charts lack a data definition

The cache exposes only the last tick's change; neither source supplies a
previous close/day-open baseline. Thus the required watchlist “daily change %”
cannot be calculated consistently, especially for the simulator. Likewise,
the main chart needs a time series but the cache keeps only the latest update;
the plan says sparklines accumulate client-side only since page load, which
does not cover an initial selected chart, reconnects, or late-added tickers.

Define a market-session baseline and timezone, the source/fallback for it, and
whether “daily” is actual market day, simulator-session, or last-tick change.
Define the detailed-chart data source, retention/window, maximum points, and
reconnect/backfill behavior. Decide whether it is intentionally empty before
enough SSE samples arrive.

### 8. Database initialization and evolution are inconsistent

The plan says initialization happens on startup “(or first request)” and also
says the backend lazily initializes on first request. Those are materially
different for startup market subscriptions, health checks, and concurrent first
requests. “Creates schema if tables are missing” is not a migration strategy:
it gives no schema versioning, upgrade path, or definition of an incompatible
existing database.

Pick one lifecycle (normally initialization before serving requests), serialize
it, and define migrations/version checks and failure behavior. State the
database access approach and connection/locking configuration for FastAPI so
background snapshot writes and requests do not cause `database is locked`.

### 9. Real-data startup and failure behavior is missing

The existing `MassiveDataSource.start()` performs an immediate poll. A bad key,
network outage, quota response, or provider API change can therefore prevent
application startup rather than merely leave stale/no prices. The plan also
claims “live” market data but does not define session/holiday behavior,
delayed/free-tier data, stale-price labeling, poll interval configuration, or
what happens when an added symbol is invalid/not returned.

Specify startup degradation/retry semantics, stale-data timestamps and UI
states, configurable interval and rate-limit policy, invalid-symbol response,
and whether the app ever falls back to the simulator after a real-data failure.

### 10. The single-container persistence description contradicts itself

The command mounts a named volume (`finally-data:/app/db`), but the text then
says the project-root `db/` directory maps to `/app/db`, which would be a bind
mount (`./db:/app/db`). These have different visibility, lifecycle, backup, and
test-reset behavior. The directory structure calls `db/` the volume-mount
target, further obscuring the intended deployment.

Choose one primary mount form and use it consistently in Dockerfile, compose,
scripts, tests, and documentation. Also define the image/container name,
network/container name, port-conflict behavior, and whether scripts use a
fresh or retained database.

### 11. In-memory market state limits the deployment topology

Price cache and source are in-process. Multiple Uvicorn workers or multiple
container replicas produce different simulations/caches and can execute
competing trades against the same SQLite database. The plan simultaneously
mentions cloud deployment without stating this limitation.

Explicitly require one worker and one replica for the core build, or redesign
state ownership. The health endpoint should distinguish process liveness from
market-source readiness.

## Medium

### 12. Ticker normalization and eligibility are unspecified and inconsistent

SQLite's uniqueness treatment, manual API input, LLM output, and simulator
behavior can diverge on `aapl`, whitespace, punctuation, duplicates, and
unknown symbols. The supplied Massive source normalizes ticker input, whereas
the simulator source does not. Unknown symbols get random simulator prices but
may be rejected or absent in Massive.

Set one canonical ticker format and validation policy, normalize before every
database lookup/write, set a watchlist size limit, and specify source-specific
unknown-symbol handling. Also decide whether a ticker must be watched before
it can be traded, and how a first price is obtained before an immediate order.

### 13. Portfolio semantics remain incomplete

Define whether total value includes cash (the text implies yes but does not
say), whether unrealized P&L includes realized gains, how percentage P&L is
defined at zero cost, and what value/color is used for missing/stale prices.
Specify heatmap behavior for all-cash/zero positions, zero or negative P&L,
and min-sized tiles. The current schema has no realized-P&L field, so its
calculation must be explicitly derived from the trade ledger if shown.

### 14. Snapshot scheduling has operational gaps

The 30-second task has no defined lifecycle, initial snapshot, alignment,
deduplication with post-trade snapshots, retention/downsampling, ordering, or
timezone format. On a fresh launch the P&L chart may have no points until the
first scheduled tick unless initialization writes one. Over time, permanent
30-second rows grow without bound.

Define the initial point, timestamp standard (UTC ISO 8601 with `Z` is
recommended), query order/limits, and retention/downsampling policy.

### 15. Static Next export hosting needs routing details

The plan calls the UI a SPA and asks FastAPI to serve an exported Next build,
but does not state static mount ordering, an `index.html` fallback for direct
client routes, asset base path, cache headers, or the Next configuration needed
for static export. This commonly produces API routes shadowed by static files
or 404s on refresh. If the app truly has one page, say so and avoid client
routes; otherwise specify the fallback behavior.

### 16. SSE requirements do not define production/client behavior

Native EventSource reconnects, but the plan does not say how the UI distinguishes
initial connection, retrying, a permanently failing endpoint, and a stale last
event. Nor does it define heartbeats for quiet/empty streams, cache-control for
proxies, cleanup, backoff, or how to prevent a reconnect from duplicating chart
points. “All tickers known to the system” contradicts the earlier union of
watched tickers and must be reconciled with the position rule above.

### 17. Testing is a list of intentions, not a reproducible test contract

`docker-compose.test.yml` lacks service definitions, build contexts, readiness
checks, test commands, artifact handling, fixed browser image/version, and a
fresh database policy. Mock LLM responses are called deterministic but their
request-to-response behavior is unspecified, so no agent can write stable chat
assertions. “Disconnect and verify reconnection” lacks a controllable failure
mechanism and observable acceptance criterion.

Add test fixtures/contracts for mock prompts/actions, deterministic simulator
configuration (seed/clock or tolerant assertions), selectors/accessibility
requirements for Playwright, and precise end-to-end assertions.

## Low

### 18. Documentation encoding is corrupted

The plan contains mojibake throughout (for example `FinAlly â€”`, `â†’`, and box
drawing characters). If agents copy these strings or diagrams, generated files
and UI text can inherit corrupted characters. Restore UTF-8 before treating
this document as a canonical shared contract.

### 19. Some implementation choices are left open where consistency matters

“Canvas-based charting library preferred (Lightweight Charts or Recharts)” is
not a decision (Recharts is primarily SVG), and package/license/SSR-static
export compatibility is unaddressed. The responsive/tablet requirement also
does not define the collapse/reflow behavior for the many simultaneous panels.
Choose the chart library and provide a minimal layout breakpoint specification
to prevent agents from building incompatible component assumptions.

### 20. Security and operational limits are absent even for localhost use

No auth is a deliberate product choice, but the plan should still set request
body limits, CORS/host policy, logging/redaction rules for keys and chat
content, dependency/security-update expectations, and rate/concurrency limits
for chat. These are particularly relevant because arbitrary users of a
deployed cloud instance can spend the configured OpenRouter quota and trigger
auto-trades in its shared default account.
