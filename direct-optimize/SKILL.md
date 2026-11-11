---
name: direct-optimize
description: Deep-scan a codebase's hot paths for waste and fix it while preserving behavior. Covers async work that waits in sequence without needing to (independent awaits, duplicate in-flight fetches, redundant round trips), database queries that fetch columns or rows the code never reads, repeated parsing (URLs, regexes compiled per call), and hot keys buried in large JSONB/JSON columns. Use when the user asks for direct-optimize, a hot-path or performance pass, to parallelize or cut round trips, or to trim over-fetching queries.
---

# Direct Optimize

Find the work a hot path does that it doesn't need to do: waiting it doesn't need, data it doesn't read, and parsing it repeats. Fix it without changing what the code does. Fix the code by default. The user asked for the fixes, so make them, verify them, and report.

Work through the four passes below in order. Earlier passes change code that later ones look at.

## Map the hot paths

Start from the entry points that run often or that users wait on:

- Server request handlers and the routes they dispatch to: the busiest list and search routes, detail views, submits. Include the auth, session, routing and logging code that runs on every request.
- Data-access, validation and crypto helpers those routes call per request or per row.
- Client startup (config, identity, first page load), each page's initial data load, and anything that runs per keystroke or per render.
- Background work that delays a response or is billed by time (deferred or `waitUntil` work, telemetry), and scheduled jobs. Scheduled jobs are lower priority.

Read the code itself. Follow calls through helpers until you reach the actual I/O (fetch, database, file, crypto, sockets) or the actual repeated work. Names don't tell you whether something waits or what it fetches.

Survey the codebase for implicit state machines, including lifecycles, workflows, retries, connection handling and async coordination expressed through flags, status values or scattered branches. Trace their states, events, transitions and guards across modules; use them to identify redundant work and ordering constraints before optimizing. Report the machines found and any uncertain or incomplete traces. Preserve their behavior; don't rewrite them as explicit state machines solely for this pass.

## Pass 1: needless waiting

1. **Independent sequential awaits.** `a = await x(); b = await y();` where `y` uses nothing from `a`, its ordering doesn't matter, and it doesn't run conditionally on `a` in a way that matters. Run them together with `Promise.all` (or the language's equivalent).
2. **Awaits in loops** over independent items, such as `for … await` doing one request or decrypt per item. Map the items to promises and run them together, unless ordering, rate limits or quotas require otherwise.
3. **Redundant round trips.** A second query for data the first one already returned (a view or join that already includes the columns), or data fetched twice in one request. Delete the extra call rather than parallelizing it.
4. **Duplicate in-flight work.** Caches that store only the result, so concurrent cold callers each compute the same thing: key derivation, token or JWKS fetches, config loads. Cache the in-flight promise instead, share it, and clear it on failure so an error isn't remembered.
5. **Waterfalls across layers.** The client fetching B only after A returns when B doesn't depend on A. Start B early. If B's failure must not trigger side effects (sign-out, error screens), make it a quiet prefetch and fall back to the normal request when it doesn't succeed.
6. **Work that can start before a dependency resolves.** Kick off the independent part first and await it where it's needed.

Leave the code sequential, and say why, when:

- The second operation uses the first one's result, or only runs depending on it, and running it anyway would have side effects, cost real money or quota, or be expensive.
- Order is part of the contract: protocol exchanges (SMTP, handshakes), writes that must land in sequence, read-after-write, locks or transactions.
- External limits apply: rate limits, per-run request caps, daily quotas, connection pools.
- Doing them together changes failure semantics in a way that matters. If an early failure used to prevent a later write, check that running both is still safe, and keep error handling (try/catch, fallbacks, "never throws" helpers) behaving the same.
- The path is cold (setup, rare first-run branches, one-off scripts) and the change would only add complexity.

Concurrency must not create races. Check what two requests or two tasks touching the same row, cache, or token-refresh path would do at once.

## Pass 2: over-fetching queries

For every query on a hot path (`select *`, ORM loads of whole models, REST `select=*`, `RETURNING *`, `return=representation`), trace each column from the query to its consumers. That includes everything that reaches the browser: check what the page actually renders, not just what the API returns.

- **Select only what's read.** Replace `*` with an explicit column list. Watch for large values (JSON/JSONB, text blobs, ciphertext, images), legacy columns, and columns used only in filters.
- **Writes that echo the row back.** An insert or update that returns the full row when the caller needs only the `id` (or nothing) sends every column back, sometimes megabytes of it. Return just what's used.
- **Checks that load full rows.** Existence or duplicate checks, counts, and "latest date" lookups should select the one or two columns they compare.
- **List vs. detail.** If a list endpoint returns the detail shape, give it its own narrow shape and update the API docs. Removing a field from a response changes the contract, so grep the clients and tests for every use before dropping it.
- **Leave alone:** queries that genuinely use every column (backups, exports, admin tables), tiny single-row lookups where an explicit list adds nothing but upkeep, and one-off scripts that touch real data.

Make the safety net real: if the test mock or fake database ignores the column list, make it honor one, so a column the code reads but no longer selects fails a test. Then check the narrowed column names against the real schema (`information_schema.columns` or the migrations). Read column names only, never row values.

## Pass 3: repeated parsing and setup

Look for work that produces the same result on every call:

- **URL parsing.** The same request URL parsed in several layers (router, middleware, logging or timing wrappers). Parse it once at the outermost layer and pass the parsed object down. Keep a fallback so callers that don't pass one still work.
- **Regexes built per call.** `new RegExp(...)` from a constant or config string inside a function, loop, validator or render path. Compile it once at module level, or cache it by source string. Plain regex literals are already compiled once by JS engines. Only share a compiled regex if it has no `g` or `y` flag: those keep `lastIndex` state between calls.
- **Other per-call setup** whose result never changes: splitting the same env var, building lookup maps from static config, JSON-parsing constants, re-creating formatters (`Intl.*`), or importing key material. Hoist it, or memoize it with a cache that is invalidated when its input changes.

## Pass 4: hot keys in large JSON columns

When hot queries pull a few keys out of a JSON/JSONB column (`data->>'x'` in views, filters, aggregates or sorts), measure before changing anything. Aggregate sizes only, never values:

```sql
select count(*), avg(pg_column_size(col)), max(pg_column_size(col)),
       count(*) filter (where pg_column_size(col) > 2000), pg_size_pretty(sum(pg_column_size(col)))
from tbl;
```

Postgres moves values over about 2 KB out of line (TOAST), and every query that reads one key has to fetch and decompress the whole value. Only if a meaningful share of rows is over that size, or the totals are large, should you propose promoting the hot keys to real columns: a migration, a backfill, writes that keep both in sync, and switching readers over. That is a schema change, so propose it with the numbers and wait for approval rather than doing it unasked. If the values are small, report the measurements and leave the schema alone.

## Make the changes

- Keep each change small and local, in the codebase's existing style, with a short comment where the reason for the change isn't obvious.
- When a store or API contract changes, update its docs and every implementation (real, local or file-based, and the test mocks), then grep for other callers.
- Avoid unhandled rejections from promises started early: attach a handler, or await them on every path.
- Don't add dependencies or new abstractions just to run things concurrently or to cache.
- Keep the files' existing line endings and encoding.

## Verify and report

Syntax-check the edited files and run the tests that cover the changed paths. Run the full suite if the project's rules allow it, and confirm the changed behavior is actually exercised (grep the tests). Never claim a speedup you didn't measure. Describe it as fewer round trips, fewer bytes, or less repeated work.

Report, grouped by pass:
- The implicit state machines found, their locations and relevant transitions, including any traces that could not be completed.
- Each fix: location, what it did before, what it does now, and whether it removed a request, removed data, or only overlapped work.
- What you deliberately left alone, and why.
- Measurements taken (sizes, counts) and schema checks run.
- What was tested and what wasn't.

Don't commit or deploy unless the user asks; follow the project's own release rules.
