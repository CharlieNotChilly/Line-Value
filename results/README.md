# Results

Every number on the résumé and in the README must trace to a file here. Commit these files.

```
results/
  environment.md           machine, Docker resources, versions, Odds API plan and quota
  live/
    summary.md             live-run days, [B] sportsbooks, quotes per day, [F] ingest latency
    gaps.csv               polling gaps per sport
    quota.csv              credits used/remaining per call
  replay/
    summary.md             [R] quotes/s, consumer count, live-vs-replay comparison
  api/
    k6-<timestamp>.json    k6 summary export
    summary.md             p50/p95/p99 [L], error rate
  backtest/
    threshold.json         frozen tau, chosen on the calibration window only
    flags.parquet          per-flag and per-control rows
    summary.md             [E], [P], [Q], Wilson intervals, counts, explanation stats
  demo/                    dashboard screenshots
```

## Where each résumé number comes from

| Résumé value | Source |
|---|---|
| `[B]` sportsbooks | `live/summary.md` |
| `[R]` quotes/s | `replay/summary.md` |
| `[F]` ms p95 ingest latency | `live/summary.md` |
| `[L]` ms p99, `[C]` clients | `api/summary.md` |
| `[E]` events, `[P]%`, `[Q]%` | `backtest/summary.md` |

Rules: don't edit result files by hand, don't delete failed runs, and regenerate summaries with the scripts in `bin/`.
