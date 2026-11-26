# LineValue Build Plan

Work **one phase at a time**. Give the AI the phase prompt, check the "done when" line yourself, commit, then move on. Phases 3, 4, and 9 contain the real engineering: read and understand that code before continuing.

Spec IDs (F1.1, F2.3, …) refer to [SPEC.md](SPEC.md).

| # | Phase | Done when | Covers |
|---|---|---|---|
| 0 | Machine and API setup | Every tool reports a version; API key works; plan and quota recorded | — |
| 1 | Skeleton | `docker compose up` runs postgres, redis, api (`/health`), poller, normalizer stubs; CI runs lint, types, tests | — |
| 2 | Poller | Fixture-based tests pass; one live poll stores a raw file, a manifest line, and a stream entry | F1.1, F1.2, F1.6 |
| 3 | Normalizer | Stream → PostgreSQL works; duplicate and crash tests pass | F1.3–F1.5 |
| 4 | Pricing engine + arbitrage | Property and unit tests pass; flags and arbitrage windows appear for live data | F2.1–F2.3, F2.7 |
| 5 | API, cache, dashboard | Endpoints served from cache; dashboard shows flags, arbitrage windows, and line movement | F2.4, F2.5, F2.8 |
| 6 | Live run (28 days, runs in the background) | `bin/live_report` writes `results/live/` (including `arbitrage.md`) with no unexplained gaps | F1.7, F2.9 |
| 7 | Replay | `bin/replay` reproduces the live database and reports `[R]` | F1.8 |
| 8 | API load test | k6 run saved; error rate 0 | F2.6 |
| 9 | Backtest | Frozen τ, matched controls, Wilson intervals saved | F3.1–F3.3 |
| 10 | Explanations | Validator tests pass; explanations stored for test-window flags | F3.4, F3.5 |
| 11 | Write-up | README tables filled from `results/`; diagram; 60-second demo | — |

**Start the live run as soon as phase 3 passes.** It needs 28 calendar days, and pricing (phase 4) can be recomputed later from the raw corpus with `bin/replay`. Keep building phases 4, 5, 7, and 8 while it runs.

At the end of **phase 5**, the early-stage bullets in SPEC.md are true. At the end of **phase 10**, the full bullets can be filled from `results/`.

---

## Phase 0: Set up your machine and API key (manual, once)

| Tool | Why | Check |
|---|---|---|
| Git | Version control | `git --version` |
| Docker Desktop (Windows: WSL2 backend) | Runs every service | `docker compose version` |
| Python 3.12 + [uv](https://docs.astral.sh/uv/) | Backend, tools, tests | `python3.12 --version`, `uv --version` |
| Node 20+ | Dashboard | `node --version` |
| k6 | API load test (phase 8) | `k6 version` |
| Claude Code or Codex CLI | Writing the code phase by phase | `claude --version` / `codex --version` |

On Windows, work inside WSL2 and keep the repo in the Linux file system (for example `~/code/Line-Value`).

**API key and quota**
1. Get a key at [the-odds-api.com](https://the-odds-api.com) and put it in `.env` (copy `.env.example`). Never commit `.env`.
2. Read the pricing page and the v4 docs, then write in `results/environment.md`: plan name, monthly credit quota, and the credit cost of one `/odds` call for **1 market × 4 regions**. (Check this yourself; the cost depends on markets and regions.)
3. Estimate: `credits/month = 4 sports × cost_per_call × polls_per_day × 30`. Pick the poll interval that keeps this at or below 90% of quota. The budget guard (phase 2) enforces it automatically.
4. Decide where the poller will run for 28 days. A laptop that sleeps creates gaps. Options: a laptop plugged in with sleep disabled, or a small cloud VM. Gaps are logged either way.

Then:

```bash
git clone https://github.com/CharlieNotChilly/Line-Value.git
cd Line-Value
```

---

## Phase prompts

Start every session with: *"Read AGENTS.md, docs/SPEC.md, and docs/DESIGN.md first. We are on phase N."*

### Phase 1: Skeleton
```
Create the project layout from DESIGN.md "Folder structure". Python 3.12
package `linevalue` managed with uv (pyproject.toml with ruff, mypy strict,
pytest, pytest-asyncio, hypothesis). docker-compose.yml with postgres:16,
redis:7 (appendonly yes, appendfsync everysec), api (FastAPI on uvicorn,
production settings, GET /health), poller and normalizer services that for
now log a heartbeat every 5 s. Settings via pydantic-settings from env;
add .env.example entries. Alembic configured with an empty first migration.
GitHub Actions workflow: ruff, mypy, pytest (unit only for now), and a
`git grep` check that fails if anything looks like an API key.
Done when: `docker compose up --wait` is healthy, `curl localhost:8000/health`
returns ok, and CI passes. Show the commands and output.
```

### Phase 2: Poller
```
Implement linevalue/poller per DESIGN.md: an async httpx client for
GET /v4/sports/{sport}/odds with regions=us,us2,uk,eu, markets=h2h,
oddsFormat=decimal; one asyncio task per sport from SPEC fixed parameters.
For every response: write gzip body + headers to raw/YYYY/MM/DD/, append a
line to raw/manifest.jsonl, insert a raw_responses row (received_at,
credits headers), then XADD odds:raw. Budget guard: read x-requests-used,
x-requests-remaining, x-requests-last on every call; lengthen the interval
so projected monthly use <= 90% of MONTHLY_QUOTA; pause at 0 remaining;
append to results/live/quota.csv. Backoff: exponential with full jitter on
429/5xx/timeouts, per sport, never blocking other sports.
Record 3 real responses into tests/fixtures/ (strip the api key from any
URL). Tests use fixtures and a fake transport only.
Done when: tests for F1.1, F1.2, F1.6 pass, and one live poll produces a raw
file, a manifest line, a raw_responses row, and a stream entry.
```

### Phase 3: Normalizer (hard)
```
Implement linevalue/normalizer per DESIGN.md. Create tables events, books,
raw_responses, quote_changes (range-partitioned by received_at, one
partition per day, created ahead of time), latest_quotes, fair_prices,
flags via Alembic. Consumer group `normalizers` on odds:raw. For each
message, in ONE PostgreSQL transaction: parse the raw file, upsert events
and books, insert quote_changes with ON CONFLICT DO NOTHING on (event_id,
book_key, outcome, book_last_update), update latest_quotes only when
book_last_update is newer, set raw_responses.committed_at. After commit:
XACK. A reclaim loop uses XAUTOCLAIM with min-idle 30 s. Make N consumers
configurable (compose `deploy.replicas` or N processes).
Integration tests against compose services: (a) the same message processed
twice leaves the DB unchanged (F1.4); (b) a consumer killed after commit
but before XACK, and one killed before commit, both end with exactly one
set of rows (F1.5); (c) XACK happens only after commit (F1.3).
Done when: those tests pass and a live poll flows into the tables.
```
Then start the live run: `docker compose up -d poller normalizer` on the always-on machine.

### Phase 4: Pricing engine (hard)
```
Implement linevalue/pricing as pure functions: implied_p(price),
devig(prices_by_outcome) -> fair probabilities summing to 1,
leave_one_out_consensus(fair_by_book, book) -> median of the OTHER books
or None if fewer than 5 others, ev(consensus_p, price). In the normalizer
transaction, recompute fair_prices for every touched event and append to
flags when ev >= FLAG_THRESHOLD (env, provisional until phase 9).
Add arbitrage(event_quotes, window_s=60, m_min=0.005): best price per outcome
across books, using only legs whose book_last_update is within 60 s of the
newest leg; margin m = 1 - sum(1/best); return legs and stake shares
(proportional to 1/best) when m >= m_min. In the same transaction, open an
arbitrages row when one appears and set closed_at when it disappears.
Hypothesis property tests: devig sums to 1 within 1e-9 for any valid odds;
consensus never includes the book itself; ev is 0 when price == 1/p; for any
detected arbitrage, the stake split pays more than its cost on every outcome.
Unit test: an arbitrage that depends on one stale leg is not reported.
Done when: F2.1-F2.3 and F2.7 tests pass and flags appear for live events.
```

### Phase 5: API, cache, dashboard
```
FastAPI endpoints from DESIGN.md API table with pydantic response models.
Read-through cache: cache:events, cache:event:{id}, cache:flags, cache:arbs
with TTL 5 s;
the normalizer deletes affected keys after commit; log hit ratio.
React + TypeScript + Vite dashboard in web/: flags table sorted by EV with
book, price, consensus, EV, and book count; event page with a per-book
line-movement chart from quote_changes; an arbitrage panel from GET /arbs with
margin, legs, and stake split. Add the web service to compose.
Done when: F2.4 and F2.8 integration tests pass, and you save a dashboard screenshot
to results/demo/.
```

### Phase 6: Live report
```
Write bin/live_report: from the database, compute live-run start/end and
days, gaps (no committed response for a sport for > 3x its interval)
written to results/live/gaps.csv, distinct bookmaker keys [B], quotes and
quote changes per day, and ingest latency p50/p95/p99 [F] from
raw_responses (committed_at - received_at). Write results/live/summary.md.
Also write results/live/arbitrage.md: arbitrage windows [A], median lifetime
[D] in seconds, the stale-rejection share [S] (apparent arbitrages that fail
the 60 s freshness rule), and the margin distribution (F2.9).
Done when: it runs on the live database and every number in the summary
comes from a query you can point to.
```

### Phase 7: Replay
```
Write bin/replay: start a fresh compose project with an empty database,
start 4 normalizer consumers, XADD every manifest entry in order, wait for
the pending list to drain, and report quotes processed / wall-clock seconds
[R]. Then compare replayed tables with the live tables (row counts and a
checksum per table, excluding timestamps set at processing time). Save to
results/replay/. Use production settings.
Done when: the comparison matches and [R] is in results/replay/summary.md.
```

### Phase 8: API load test
```
Write load/api.js for k6: 200 constant VUs for 120 s hitting /events,
/events/{id} (ids sampled from a fixture list), and /flags; thresholds
http_req_failed rate == 0. Run k6 in a container on the compose network
(not through localhost port publishing), after warming the cache. Save the
summary export to results/api/ and write results/api/summary.md with p50,
p95, p99 and error rate. Record machine details in results/environment.md.
Done when: F2.6 passes and the files are saved.
```

### Phase 9: Backtest (hard)
```
Write bin/backtest per SPEC "Measurement definitions" and F3.1-F3.3. Split
live-run days 50/50. Load ONLY the calibration window first; choose tau
from a fixed grid (0.00 to 0.10 step 0.005) maximizing the calibration
beat-the-close rate with >= 200 flags; write results/backtest/threshold.json;
only then load the test window. Recompute flags on the test window with the
frozen tau. Controls: for each flag, one unflagged quote from the same event
and snapshot, sampled with seed 42. Compute [P], [Q], 95% Wilson intervals,
[E], and counts. Write results/backtest/summary.md and the per-flag rows to
results/backtest/flags.parquet. Add a unit test proving the test window is
not loaded before threshold.json exists.
Done when: the summary exists. If [P] does not beat [Q], report that
plainly; do not change tau or the split.
```

### Phase 10: Explanations
```
Implement linevalue/explain: build a structured payload per flag (teams,
book, price, consensus_p, ev, n_books, best and worst other prices). Call
the OpenAI API (model name from env) with a short instruction to explain
the flag in 2 sentences using only payload values. Validator: extract every
number from the output; reject if any is not present in the payload
(after rounding to the payload's precision); regenerate once, then drop.
Store accepted explanations; expose GET /flags/{id}/explanation. Unit tests
use stubbed model responses, including one that invents a number.
Done when: F3.4 and F3.5 pass and results/backtest/ records how many
explanations were accepted, regenerated, and dropped.
```

### Phase 11: Write-up
Fill the README results tables from `results/` (run `bin/summarize`), add the architecture diagram from DESIGN.md, and record a 60-second demo: the dashboard updating, a replay run, and the backtest summary. Then fill the brackets in SPEC.md from the same files.
