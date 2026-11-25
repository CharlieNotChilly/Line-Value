# Rules for AI coding agents (Claude Code, ChatGPT/Codex, and others)

This project backs specific résumé claims. Correctness and honesty matter more than speed.

## Before writing code
1. Read `docs/SPEC.md` (the contract), `docs/DESIGN.md` (architecture and decisions), and `docs/PLAN.md` (phases).
2. Work on **one phase at a time**, the one the user names. Do not start later phases or add features outside the spec.
3. If something in the spec seems wrong or impossible, stop and say so. Do not silently change behavior.

## While writing code
- Follow the table names, Redis keys, stream names, and endpoints in DESIGN.md exactly.
- The Odds API is the only data source. Never add scraping.
- Tests never call the live API or OpenAI. Use recorded fixtures and stubs.
- Keep `linevalue/pricing/` pure (no I/O) so it stays property-testable.
- Every phase ends with tests for its spec criteria (the IDs in SPEC.md). Run them and show the output.
- Prefer small, readable, typed code. Add a comment only where the reason is not obvious (for example, why `XACK` must come after the commit).
- Never commit secrets. Keys come from environment variables.

## Results and numbers
- **Never invent, estimate, or round up results.** Every number in the README or SPEC must come from a file in `results/`.
- Do not change the fixed parameters in SPEC.md (sports, market, regions, 28-day live run, 4 consumers, 200 VUs for 120 s, 50/50 split, frozen threshold).
- In the backtest, never look at the test window before the threshold is written. If the result is weak or negative, report it plainly.
- Measured runs use production settings and run load generators inside the Compose network.

## When a phase is done
- Summarize what changed, which spec criteria now pass, and anything still failing.
- Suggest a commit message (plain text, no AI attribution).
