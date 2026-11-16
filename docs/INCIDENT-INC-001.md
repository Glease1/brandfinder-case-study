# INC-001 — the canary that bought positions it could not record or exit

**Severity:** critical (real money lost)
**Status:** fixed, not re-deployed at the time of writing
**Date:** 15–16 November 2026
**Component:** the migration canary (a second book), not the original canary

---

## Summary

A second canary — a separate book with its own stake and its own tab, built to test young tokens priced from the curve — bought eleven positions with real funds overnight. **None were written to its book**, so nothing could price, guard or sell them. All eleven were left in the wallet until they collapsed.

0.11 SOL was spent; 0.049 SOL was recovered.

## Root cause

A table called `migration_live` existed from an earlier revision of the script, with a different schema: `position_id` as the primary key, and no `entry_lamports` or `banked_lamports` columns.

The new code used:

```sql
CREATE TABLE IF NOT EXISTS migration_live (...)
```

which **silently does nothing** when the table already exists, plus an `ALTER` list that covered only some of the new columns.

Every buy therefore did this:

1. build and send the buy — real SOL spent, real tokens received;
2. attempt the bookkeeping insert — fails, column does not exist;
3. the exception propagates and kills the loop;
4. the supervisor restarts it ten seconds later;
5. the open-position query returns zero, because nothing was ever recorded;
6. buy again.

The exit loop iterated rows that never existed. No take-profit, stop or timeout could ever fire.

## Contributing factors

- **Live arming without an end-to-end test of buy → record → exit on a single position.** The sell lane had been tested separately; the entry logic had been tested in dry run; the persistence step in between was assumed.
- **A supervisor that restarted a failing process into the same failure**, turning one bad buy into eleven.
- **No schema assertion at start-up.**

## Consequence

| | |
|---|---|
| Buys | 11 × 0.01 SOL = 0.11 SOL committed |
| Recovered by manual sale | 0.0262 SOL |
| Rent reclaimed by closing token accounts | 0.0226 SOL |
| Lost to token collapse | ≈ 0.07 SOL |
| Unsellable | one position had to be burned to reclaim rent |

## Remediation (in place)

- `connect()` now asserts that every required column exists, renames an incompatible table to `migration_live_old` and rebuilds, and **refuses to trade** if the book still cannot hold a position.
- If the insert fails after a successful buy, the position is **sold straight back** rather than left unmanaged.
- The trader runs under a supervisor and logs to its own file.

## Prevention rules for future live arming

1. One position, watched end to end, before any concurrency limit above one.
2. Prove the whole cycle — buy, persist, price, exit, reconcile — not just the parts that are convenient to test.
3. A start-up schema check is mandatory for any table the trader writes.
4. Nothing is armed with a concurrency limit that has not been tested at that concurrency.

## Accounting treatment

This is an **infrastructure incident**, not a strategy result. It is recorded separately and must not be blended into the original canary's expectancy, which is measured only from its own positions table and never included these trades.

## What I would tell an interviewer

> "I armed a second book without testing the persistence step end to end. It bought eleven positions, couldn't record any of them, and a supervisor kept restarting it into the same failure. I lost about 0.07 SOL. The fix was a start-up schema assertion that refuses to trade, a sell-back when the insert fails after a buy, and a rule that nothing gets armed at a concurrency it hasn't been tested at. The deeper lesson is that a restart is a decision, not a recovery."
