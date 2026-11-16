# The Brand Finder — a control loop for decisions that can fail

**A case study in execution integrity for a small real-money trading system. The code is private; this is the engineering story, the evidence, and the parts that went wrong.**

> This is not a strategy claim. The system has **no demonstrated edge**, and the measurement below is the argument for saying so. Nothing here is financial advice.

---

## The problem

A trading bot is usually described by its strategy. The harder problem is everything around it: knowing which process owns a position, whether an exit decision was actually executed, whether a price came from a real market or from a fallback, and what to do when the venue, the RPC endpoint or the wallet disagrees with the database.

I built and operated a small Solana system — token discovery, safety gates, a paper arena, a shadow fill simulator, and a real-money canary with a fixed 0.01 SOL stake — because that gap between a **decision** and a **fill** turned out to be where every interesting failure lived.

The canary is not a money-maker. It is an instrument for finding out whether the execution path tells the truth.

---

## Four planes, deliberately separated

| Plane | Money at risk | What it answers |
|---|---|---|
| **Engine** | none | What is this token, and does it pass the safety gates? |
| **Paper arena** | none | What would a rule have done? |
| **Shadow simulator** | none | Would that paper fill have been achievable at a real quote? |
| **Canary** | 0.01 SOL per position | Does the real execution path work, and what does it cost? |

A separate research container tested a graduated-token cohort and lost money over 44 trades. It is documented as a negative result, not presented as a feature.

Conflating these planes is how paper profit gets mistaken for real profit. The shadow layer exists specifically to measure the gap between them, and it labels every row `SIMULATED_NOT_CONFIRMED` because a simulated fill is not a fill.

---

## Five decisions that mattered

### 1. Ownership is explicit, never inferred

The original orphan sweep decided a position was abandoned by looking at the age of its last price mark. That is a heuristic, and it had exactly the failure you would expect: it sold a position that a live watcher still owned, because the mark was stale while the watcher was healthy.

The fix was to make ownership a fact rather than an inference: a watcher registers the position it owns, and the sweep checks whether that task is alive before touching anything. A position with a live owner is never swept; a position whose owner has died is recovered and marked as such.

Covered by 10 checks, including one that asserts the old rule *would* have sold the position — the test proves the regression, not just the happy path.

### 2. A decision is not a fill

Selling can fail after the decision to sell. The original code treated a failed sell as "no exit", which meant a position could keep its stop-loss decision but lose the record of it, and then be re-decided from scratch on the next tick.

The fix is a persistent **pending-exit claim**: the intent to sell is written down before the attempt, survives a crash, and is revalidated when the process restarts — because a decision made against a price ten minutes ago may no longer be warranted. The exit is attempted until it succeeds or the position is provably gone, and it can only be claimed once.

Covered by 21 checks: claim, survive a failed execution, recover after restart, sell exactly once.

### 3. A mark is not a price

Exit decisions need a price. Exits need a *quote*. When the primary price source is unavailable, the system falls back through a three-tier ladder and records, for every valuation, **where it came from and how old it is**.

The rule that matters: a fallback value can trigger a decision, but it can never become the price of a sale. If every provider is down, the system may decide to exit and be unable to fill — and that is the correct outcome, because the alternative is inventing a price and pretending it was a market.

Covered by 23 checks covering tier order, freshness, source disagreement and attribution.

### 4. Blindness is a state, not a signal

A price feed can go silent while the process stays up. Treating that as "price is zero" or "price is unchanged" produces confident, wrong decisions.

Every price read is classified **fresh**, **cached** or **blind**. Transient blindness raises an alert and waits; persistent blindness becomes an exit reason of its own rather than a fabricated stop. The same discipline applies to depth: thin or absent order books are recorded as a state (`no book`), not smoothed into a price.

Covered by 14 checks, including the distinction between transient and persistent blindness, and the case where the position was closed somewhere else entirely.

### 5. Every exit is attributed

The last addition was the one that made the system honest: every exit is classified as

- **market gap** — the price moved against the position faster than the loop could react;
- **execution delay** — the decision was right, the fill arrived late;
- **no book** — there was nothing to sell into.

Each attribution carries a depth ratio and a 30-second post-exit mark, so a bad decision can be told apart from a bad fill. Without this, every loss looks the same and no amount of strategy work can tell you which one you have.

---

## The measurement, stated plainly

From the ledger snapshot of 15 November 2026, over **171 closed rows**:

| Measure | Value |
|---|---|
| Realised result | +0.1557 SOL |
| Wins / losses | 62 / 109 |
| Profit factor | 1.22 |
| Top three trades as a share of profit | **154%** |
| Result excluding those three | **−0.0833 SOL** |
| Worst losing streak | 14 |
| Rows that are paper or legacy, not live | **74 of 171** |

Read that table as a failure of the strategy, not a success of the system. The book makes money only because three trades paid for everything else; remove them and it loses. Seventy-four of those rows are not live trades at all — they are paper or legacy records that share the table — which is why the live figure has to be filtered out before quoting anything.

The honest sentence is: **I can defend how the system decides and executes. I cannot defend an edge, because there isn't one.**

---

## The incident worth reading

The migration canary — a second book, with its own tab and a 0.01 SOL stake — bought eleven positions overnight and recorded none of them. It could not price, guard or sell any of them, and it kept buying because the open-position query returned zero every time.

The cause was a schema migration that silently did nothing: `CREATE TABLE IF NOT EXISTS` against a table left over from an earlier revision, plus an `ALTER` list that covered only some of the new columns. Every buy succeeded on chain and failed at the bookkeeping insert, the loop died, the supervisor restarted it ten seconds later, and the cycle repeated.

Eleven buys, 0.11 SOL committed, 0.049 SOL recovered (including rent reclaimed by closing token accounts), roughly 0.07 SOL lost, one position unsellable until it was burned to reclaim rent.

The remediation is the useful part:

- the book **asserts every required column at start-up**, renames an incompatible table and rebuilds, and **refuses to trade** if it still cannot hold a position;
- if the insert fails after a successful buy, the position is **sold straight back** instead of being left unmanaged;
- the trader runs under a supervisor and logs what it did.

And the prevention rules, which now govern every live arming:

1. one position, watched end to end, before any concurrency above one;
2. prove the whole cycle — buy, persist, price, exit, reconcile — not just the parts that are convenient to test;
3. a start-up schema check is mandatory for any table the trader writes;
4. nothing is armed at a concurrency that has not been tested at that concurrency.

The full write-up, with the accounting treatment that keeps this incident out of the strategy's expectancy, is in [docs/INCIDENT-INC-001.md](docs/INCIDENT-INC-001.md).

---

## Deployment discipline

The most uncomfortable lesson: **the running build was not the head of the repository.**

Two fixes — a Token-2022 account-close bug and a lifecycle fix for an emptied wallet — were committed, tested and green, and still not running, because a restart would have replaced a process that was holding positions. Meanwhile the running build could not bank a partial sale at all: every partial transaction included a close for an account that still held tokens, the venue rejected it, and atomicity rolled the sale back too. Forty-five attempts, zero fills. Slippage was never the cause.

The response was to stop trusting "it's deployed" as a claim. The service now reports a build fingerprint — file hashes plus the runtime configuration — and a deployment is only complete when the fingerprint matches what was intended. The restart checklist requires a partial sale to be **confirmed on chain**, with the token delta and remaining balance checked, before the process is left unattended.

A submitted transaction is not a confirmed one. A quote is not a fill. Those two sentences caused most of the bugs in this project.

---

## What I would do next

- Convert the seven bespoke check scripts into `pytest` with fixtures and run them in CI. Today they are invisible to a standard collection, which is a reproducibility problem.
- Kill the remaining programmatic arming path: a supervisor can currently restart into a real-money mode with the confirmation phrase supplied in code. Human arming should be a human action.
- Make the reports artifacts rather than scripts: the attribution, performance and robustness routines exist, but their output was never committed, so the numbers above had to be re-derived from a database snapshot.

---

## What this is not

- Not a profitable strategy, and not a claim of one.
- Not production infrastructure: one operator, one machine, one wallet.
- Not a claim of sole authorship: AI tools were used heavily for implementation; the requirements, the failure analysis and the acceptance were mine. See the portfolio's AI-assistance note.
- Not the source code. The repository stays private because it contains a live wallet configuration and strategy parameters that are not worth publishing.

---

## Evidence

The test inventory, what each check proves, what was **not** run, and how to reproduce it: [EVIDENCE.md](EVIDENCE.md).
The process and data model: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).
The failure taxonomy: [docs/FAILURE-TAXONOMY.md](docs/FAILURE-TAXONOMY.md).
The incident: [docs/INCIDENT-INC-001.md](docs/INCIDENT-INC-001.md).
The deployment rules: [docs/DEPLOYMENT-DISCIPLINE.md](docs/DEPLOYMENT-DISCIPLINE.md).

---

*Written by Ryan Glease, November 2026. Shared as a portfolio case study; no licence is granted to the underlying system.*
