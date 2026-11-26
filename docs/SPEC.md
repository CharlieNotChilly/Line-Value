# LineValue Spec (the contract)

This file turns each résumé bullet into acceptance criteria. **The code must satisfy every criterion before the matching bullet goes on a résumé.** If the code and this spec disagree, fix the code or change the bullet. Never leave them out of sync.

## Target résumé bullets

Bracketed values are **measured, never chosen**. Fill them only from files in `results/` (see [results/README.md](../results/README.md)).

```latex
\resumeProjectHeading
  {LineValue -- Real-Time Odds Analytics \& Risk Platform}{Python, FastAPI, asyncio, PostgreSQL, Redis, React, Docker}
  {https://github.com/CharlieNotChilly/Line-Value}
\resumeItemListStart
  \resumeItem{Built an \textbf{async ingestion pipeline} that normalizes odds from \textbf{[B] sportsbooks} into one PostgreSQL schema through at-least-once \textbf{Redis Streams} consumers and idempotent upserts, processing \textbf{[R]} quotes/s in replay at \textbf{[F] ms} p95.}
  \resumeItem{Designed a \textbf{pricing engine} that removes bookmaker margin and scans each updated event for \textbf{cross-book arbitrage}, detecting \textbf{[A]} arbitrage windows (median lifetime \textbf{[D] s}) and rejecting \textbf{[S]\%} of apparent ones as stale quotes.}
  \resumeItem{Backtested mispricing flags on \textbf{[E]} live-captured events with a frozen threshold: flagged prices beat the closing consensus \textbf{[P]\%} of the time vs. \textbf{[Q]\%} for matched controls, served from Redis-cached FastAPI at \textbf{[L] ms} p99.}
\resumeItemListEnd
```

**Early-stage version** (true once the F1 and F2.1–F2.4 criteria pass, before load tests and the backtest):

```latex
\resumeItem{Built an \textbf{async ingestion pipeline} that normalizes odds from \textbf{[B] sportsbooks} into a unified PostgreSQL schema through at-least-once \textbf{Redis Streams} consumers and idempotent upserts.}
\resumeItem{Designed a \textbf{pricing engine} that removes bookmaker margin to compute leave-one-out consensus probabilities and flags mispriced lines and cross-book arbitrage, served through Redis-cached FastAPI endpoints and a React dashboard.}
```

**If the backtest does not beat the control** (`[P]` ≤ `[Q]`, or the confidence intervals overlap), bullet 3 cannot claim an edge. Rewrite it to report the measured result honestly (for example, "measured closing-line value on [E] events and found no significant edge over matched controls"), or improve the model on the calibration window only and rerun.

## Fixed parameters (do not change without changing the bullets)

| Parameter | Value |
|---|---|
| Data source | **The Odds API v4 only** (`/v4/sports/{sport}/odds`, optional `/v4/historical/...` backfill). No scraping |
| Sports | `basketball_nba`, `icehockey_nhl`, `americanfootball_nfl`, `soccer_epl` |
| Market | `h2h` (moneyline) only, decimal odds |
| Regions | `us,us2,uk,eu` |
| `[B]` | Distinct bookmaker keys with at least one stored quote during the live run (measured, not assumed) |
| Live run | At least **28 consecutive days** of polling, gaps logged |
| Quota | Projected monthly credit use stays at or below **90%** of the plan quota, computed from the `x-requests-used`, `x-requests-remaining`, and `x-requests-last` response headers |
| Replay corpus | Every raw response captured during the live run |
| Normalizer consumers | **4** in the replay test |
| API load | k6, **[C] = 200** constant VUs for **120 s**, warmed cache, k6 running inside the Compose network |
| Runtime mode | Production settings for every measured run (no debug logging, no auto-reload) |
| Backtest split | Calibration window = first 50% of live-run days; test window = last 50%. Threshold chosen on calibration only, then frozen |
| Confidence intervals | 95% Wilson intervals for `[P]` and `[Q]` |
| Arbitrage freshness window | Every leg's `book_last_update` within **60 s** of the newest leg |
| Minimum arbitrage margin | `m_min` = **0.5%** |

## Measurement definitions

- **Quote**: one (event, bookmaker, outcome, decimal price, bookmaker `last_update`) tuple inside a raw response.
- **Quote change**: a quote whose price differs from the latest stored price for the same (event, bookmaker, outcome). Only changes are written to `quote_changes`.
- **`[R]` replay throughput** = quotes processed ÷ wall-clock seconds from the first message read to the last commit, replaying the full corpus with 4 consumers.
- **`[F]` ingest latency** = `committed_at − received_at` for each raw response (from the HTTP response arriving to its last row committing), p95 over the live run.
- **`[L]` API latency** = k6 `http_req_duration` p99 across the fixed load run.
- **Fair probability**: for each bookmaker, implied probabilities `1/price` normalized to sum to 1 across the event's outcomes (multiplicative de-vig). The consensus for a bookmaker's quote is the **median of the other bookmakers'** fair probabilities (leave-one-out), requiring at least `k_min = 5` other books.
- **Expected value (EV)** of a quote = `consensus_fair_p × price − 1`. A quote is **flagged** when EV ≥ threshold τ.
- **Closing consensus**: the leave-one-out consensus from the last snapshot before `commence_time`.
- **Beats the close**: the flagged price is greater than `1 / closing_consensus_fair_p`.
- **`[P]`** = share of test-window flags that beat the close. **`[Q]`** = the same share for a control set of unflagged quotes, one per flag, sampled from the same event and snapshot (seeded RNG).
- **`[E]`** = distinct events in the test window with at least one flag and a closing consensus.
- **Arbitrage**: for one event, take the best price for each outcome across all books, using only legs inside the freshness window. The margin is `m = 1 − Σ 1/best_price_i`. An arbitrage exists when `m ≥ m_min`. Stakes `s_i ∝ 1/best_price_i` give the same payout whichever outcome wins, and that payout exceeds the total stake by the factor `1/(1 − m)`.
- **Arbitrage window**: the interval from the first snapshot where an event's arbitrage holds to the first later snapshot where it doesn't. Its **lifetime** is the length of that interval.
- **`[A]`** = arbitrage windows detected during the live run; **`[D]`** = their median lifetime in seconds.
- **`[S]`** = share of apparent arbitrages (margin ≥ `m_min` using the latest stored price for every book) that are rejected because at least one leg falls outside the freshness window.

## Bullet 1: ingestion

| ID | Criterion | How it is verified |
|---|---|---|
| F1.1 | The poller calls only The Odds API v4 with the fixed sports, market, and regions, and stores every raw response (gzip) with `received_at` before publishing it | Unit test with a recorded fixture; `raw/` manifest |
| F1.2 | A budget guard reads the `x-requests-*` headers on every call and lengthens the poll interval so projected monthly use stays at or below 90% of quota | Unit test with fake headers; `results/live/quota.csv` |
| F1.3 | The poller publishes one Redis Streams message per raw response to `odds:raw`; normalizers read through consumer group `normalizers` and `XACK` only after the PostgreSQL commit | Integration test |
| F1.4 | Normalization is idempotent: processing the same message twice leaves the database unchanged (unique key on event, bookmaker, outcome, `book_last_update`) | Integration test that processes one message twice |
| F1.5 | A message whose consumer dies before `XACK` is reclaimed with `XAUTOCLAIM` after 30 s and committed exactly once | Integration test that kills a consumer mid-message |
| F1.6 | Per-source failures (HTTP 429/5xx, timeouts) retry with exponential backoff and jitter, and never block other sports | Unit test with a fake transport |
| F1.7 | `[B]`, live-run days, gaps, and `[F]` are produced by `bin/live_report` from the database | `results/live/summary.md` |
| F1.8 | `bin/replay` replays the full corpus through 4 consumers into an empty database and reports `[R]`; the replayed database matches the live database row for row | `results/replay/summary.md` |

## Bullet 2: pricing engine and serving

| ID | Criterion | How it is verified |
|---|---|---|
| F2.1 | De-vigged probabilities for each bookmaker and event sum to 1 (±1e-9) | Property-based tests (Hypothesis) |
| F2.2 | Consensus is leave-one-out: a quote never contributes to its own consensus; events with fewer than 5 other books get no consensus and no flag | Unit tests |
| F2.3 | EV and flags are recomputed for an event whenever any of its quotes change, within the same consumer transaction | Integration test |
| F2.4 | `GET /events`, `GET /events/{id}`, and `GET /flags` serve from Redis (TTL 5 s, invalidated on change) and fall back to PostgreSQL on a miss | Integration tests; cache hit ratio logged |
| F2.5 | The React dashboard lists live flags and shows per-book line movement for an event | Screenshot in `results/demo/` |
| F2.6 | `load/api.js` runs the fixed k6 load; `[L]` and the error rate come from its summary export, and the error rate is 0 | `results/api/summary.md` |
| F2.7 | Arbitrage detection follows the definition above, ignores legs outside the freshness window, and returns the stake split; for any detected arbitrage, the equal-payout stakes return more than they cost for every outcome | Property-based tests (Hypothesis) + a unit test with one stale leg |
| F2.8 | Arbitrage windows are opened and closed in the `arbitrages` table inside the normalizer transaction; `GET /arbs` serves open windows from the cache, and the dashboard lists them with margin and stake split | Integration test; screenshot in `results/demo/` |
| F2.9 | `bin/live_report` writes `[A]`, `[D]`, `[S]`, and the margin distribution of arbitrage windows to `results/live/arbitrage.md` | File |

## Bullet 3: backtest and explanations

| ID | Criterion | How it is verified |
|---|---|---|
| F3.1 | `bin/backtest` reads only stored data, splits days 50/50, chooses τ on the calibration window (maximizing the calibration beat-the-close rate with at least 200 calibration flags), and writes τ to `results/backtest/threshold.json` before loading the test window | Code review + unit test that the test window is not loaded before τ is written |
| F3.2 | The control set is sampled with a fixed seed, one unflagged quote per flag from the same event and snapshot | Unit test |
| F3.3 | `[P]`, `[Q]`, their 95% Wilson intervals, `[E]`, and flag counts are written to `results/backtest/summary.md` | File |
| F3.4 | Explanations are generated from a structured payload (prices, fair probabilities, EV, book count). A validator rejects any explanation containing a number not present in the payload; a rejected output is regenerated once, then dropped | Unit tests with stubbed model responses |
| F3.5 | No API keys are committed; keys come from environment variables | `git grep` check in CI |

## Engineering bar (applies to every phase)

- CI (GitHub Actions) runs lint (ruff, eslint), type checks (mypy, tsc), unit tests, and integration tests against Compose services on every push.
- Tests never call the live API. Use recorded fixtures in `tests/fixtures/`.
- Measured runs record the machine, Docker resources, and versions in `results/environment.md`.
- Raw results are committed. Failed runs are kept, not deleted.
