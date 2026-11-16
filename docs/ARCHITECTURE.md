# Architecture

**Scope:** process boundaries, the control loop and the durable state. No strategy parameters, no endpoints, no wallet values — those are why the source stays private.

---

## 1. Processes

| Process | Role | Money | State |
|---|---|---|---|
| Engine | Discovery, safety gates, paper arena, API and dashboard | none | SQLite |
| Shadow simulator | Replays paper decisions against real quotes; labels rows `SIMULATED_NOT_CONFIRMED` | none | SQLite |
| Canary | Real-money execution, own guard, own watchdog | 0.01 SOL per position | SQLite (separate book) |
| Research container | Graduated-token cohort experiment | none | own volume |

The canary is deliberately a **separate process with a separate book**. A defect in the engine cannot spend money, and a defect in the canary cannot corrupt the research record. The engine's "live execution" module exists but its live path is hard-disabled.

## 2. The control loop

One `asyncio` task per open position. Each task:

1. registers ownership of its position;
2. reads a price, classified **fresh / cached / blind**;
3. evaluates exit rules against that price;
4. if a rule fires, claims a **pending exit** and revalidates the decision;
5. attempts the sale and records the outcome — including a failure;
6. writes telemetry and, on close, an attribution row;
7. deregisters ownership.

A position is never sold by two paths at once: the pending-exit claim is exclusive and persistent, and a restart re-reads it rather than starting over.

## 3. Position state

```
discovered → evaluated → opened → (pending_exit) → closed
                                    ↘ failed
```

`pending_exit` is a durable state, not a flag in memory. If the process dies between the decision and the fill, the claim survives and is revalidated on restart against a fresh price — because a decision made ten minutes ago may no longer be warranted.

## 4. Ownership

The orphan sweep exists to recover positions whose watcher died. It answers one question: **is the owning task alive?** It does not infer ownership from timestamps, marks or age, because that inference is exactly what sold a live position in an earlier revision.

A position with a live owner is skipped. A position whose owner is gone is recovered, marked `orphaned`, and handled by a fresh watcher.

## 5. Valuation ladder

| Tier | Source | Used for |
|---|---|---|
| 1 | On-chain quote | decision **and** sale price |
| 2 | Local price series | decision only |
| 3 | Aggregator quote | decision only |

Every record carries `source`, `age_seconds` and a disagreement flag. Tiers 2 and 3 can trigger an exit decision; they can never supply the price of a sale. If no tier is usable, the position is marked blind and the guard waits or exits without a price.

## 6. Durable state

| Table | Purpose |
|---|---|
| positions | one row per position, with timestamps, stake, state and exit reason |
| evaluations | why the guard did or did not act, per tick |
| curve / graduations | discovery evidence for tokens that were considered |
| depth observations | order-book depth at decision time, so "no book" is a measurement |
| attributions | exit class, depth ratio, post-exit mark |
| wallet trades | the owner's own trades, imported and matched to the ledger |

SQLite in WAL mode is the only durable store. In-memory state is assumed lost on every restart, which is why ownership, claims and decisions are all persisted.

## 7. Supervision and RPC

- A watchdog checks liveness and price freshness; a supervisor restarts a dead trader.
- Restarting is not automatically safe — see [DEPLOYMENT-DISCIPLINE.md](DEPLOYMENT-DISCIPLINE.md). A restart into an unvalidated build is how one bad buy became eleven.
- RPC access uses a primary, a secondary and a last-resort endpoint, with a keyless rescue path and per-provider statistics. Failover is recorded, not silent.
- A single-instance lock prevents two traders from sharing a book.
