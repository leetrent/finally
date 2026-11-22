# FinAlly Plan Review

## Overall assessment

The plan gives a strong product vision and sensible baseline architecture for a single-user demo. Its embedded review in `PLAN.md` identifies many important implementation gaps. Before implementation begins, the team should turn the remaining ambiguous behavior into an API contract and make a few deployment/runtime decisions explicit. Without that, frontend, backend, and E2E work are likely to diverge.

## Must resolve before parallel implementation

1. **Publish the API and SSE contract.** Define request bodies, response bodies, error bodies/statuses, timestamp format, rounding rules, and the precise SSE event schema. In particular, specify initial-load behavior and whether a stream event contains a single ticker or a batch. This is the highest-value shared artifact for independent frontend/backend work.

2. **Choose one persistence mount strategy.** The plan currently describes both a Docker named volume and a project-directory bind mount. Pick one and make the Docker command, compose configuration, scripts, and DB-path defaults agree. A bind mount is more inspectable for a course project; a named volume is less sensitive to host path differences.

3. **Define the authoritative price at trade time.** Trades need a clear rule: use the most recent cached price, reject prices older than a configured threshold, and execute the balance/position update plus trade-log insert in one SQLite transaction. State behavior when a price is absent or stale.

4. **Specify chat-action execution semantics.** The plan should say that actions execute in listed order and independently, include their actual execution result in the HTTP response, and persist those results. The initial LLM message cannot accurately describe validation failures that occur after it is generated.

5. **Define startup and background-task lifecycle.** Use FastAPI lifespan to initialize the DB, select/start the market source, create an initial portfolio snapshot, and cancel tasks cleanly at shutdown. Explicitly guard against duplicate pollers/snapshot tasks when reload or multiple worker processes are used.

6. **Decide the intended deployment concurrency.** An in-memory price cache and in-process simulator are correct only for a single process. Pin the production command to one Uvicorn worker, or move price state/coordination to a shared service before advertising multi-worker/container deployment.

## Important design corrections

- The document contains mojibake (corrupted em-dash, arrow, and box-drawing characters), which will make the shared spec harder to read and copy. Save `PLAN.md` as UTF-8 and replace the corrupted sequences.
- `OPENROUTER_API_KEY` is required only when mock mode is disabled. Startup should fail clearly when a real LLM is selected without a key, rather than failing on the first chat request.
- The default simulated price source should be labeled clearly in the UI (for example, `SIMULATED`) and the app should include a no-investment-advice/demo disclaimer.
- Normalize symbols to uppercase, validate their format, and define behavior for unpriceable/unknown symbols. Continue pricing held symbols even if they are removed from the watchlist.
- Use decimal arithmetic for cash, prices, and order values where feasible. If floating point remains a deliberate simplification, define tolerances, display rounding, and the zero-position cleanup rule consistently.
- Add a `GET /api/chat` history endpoint (or explicitly make history frontend-only). The current persistent `chat_messages` schema otherwise has no retrieval path.
- Add pagination/limits to trade history, chat history, and portfolio-history endpoints before the data grows indefinitely.

## Security and operational concerns

- No login is acceptable for a local demo, but it means anyone who can reach the port can mutate the same portfolio and use the API key indirectly through chat. Bind the container to localhost by default and state that this is not a multi-user or internet-exposed deployment.
- Ensure `.env` is ignored and never copied into the image. Pass secrets at runtime only. Redact provider errors and never return API-key-bearing details to the client or logs.
- The LLM must not be trusted to bypass business validation. Validate every generated ticker, side, quantity, cash/share constraint, and watchlist mutation using the same service used by manual requests.
- Add health/readiness semantics: `/api/health` should report DB readiness and market-source state without contacting the LLM provider. Consider a separate readiness status if startup tasks can fail.

## Testing gaps to make measurable

Add acceptance criteria for:

- atomic transaction behavior when a trade update fails partway through;
- stale/missing prices and invalid/unknown symbols;
- duplicate SSE connections and client reconnects without leaked server loops;
- DB persistence across a container restart;
- mock LLM exact fixtures and malformed real-provider output;
- static-export deep-link fallback (a direct request to an app route must serve the SPA entry point if client routes are used);
- an offline, reproducible test command that does not require API keys or live Massive responses.

## Suggested implementation order

1. Correct encoding; settle volume, local-development, price-source, and one-worker decisions.
2. Write `planning/API_CONTRACT.md`, including JSON examples and shared error/status rules.
3. Build and test backend domain services (prices, portfolio, watchlist) before routes and LLM wiring.
4. Add routes/SSE and backend integration tests using mock data.
5. Build the frontend against the contract, then add deterministic mock-mode E2E coverage.
6. Containerize and verify the documented fresh-start and persistence flows.

## Release checklist

- [ ] Fresh start displays seeded watchlist, simulated/source label, $10,000 cash, and connected status.
- [ ] Valid buy/sell operations are atomic and persist across restart.
- [ ] Invalid, stale, or unavailable prices cannot execute trades.
- [ ] Watchlist removal does not make an owned position unpriced.
- [ ] Mock mode works with no provider keys and all automated tests pass offline.
- [ ] The image contains no secrets and the default launch is local-only.
- [ ] Docker launch, scripts, DB location, and documentation all describe the same persistence model.
