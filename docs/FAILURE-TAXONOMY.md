# Failure taxonomy

Every class below was found in operation, not in a design review. The right-hand column is the test that now pins it.

| # | Failure class | How it showed up | The guard | Pinned by |
|---|---|---|---|---|
| 1 | **Ownership theft** | The orphan sweep sold a position that a live watcher still owned, because it inferred ownership from the age of the last price mark | Ownership is a registration; the sweep checks whether the owning task is alive, and marks a recovered position `orphaned` | `test_sweep_ownership` — 10 checks |
| 2 | **Lost intent** | A sell failed and the decision to sell was erased, so the next tick started from scratch | The intent is written to a persistent claim before the attempt, survives a crash, and is revalidated on restart | `test_pending_exit` — 21 checks |
| 3 | **Phantom price** | A fallback value was used as though it were a tradable quote | Three-tier ladder; a fallback can trigger a decision but never price a sale; every value carries source and age | `test_fallback_price` — 23 checks |
| 4 | **Confident blindness** | A silent feed was read as "price unchanged", producing decisions with no market behind them | Every read is fresh / cached / blind; transient blindness alerts, persistent blindness is its own exit reason | `test_blind_policy` — 14 checks |
| 5 | **Unattributed loss** | Every loss looked the same, so strategy work could not tell a bad decision from a bad fill | Every exit is classified market gap / execution delay / no book, with a depth ratio and a post-exit mark | `test_exit_attribution` — 18 checks |
| 6 | **Silent schema drift** | A trader bought eleven positions it could not record, because `CREATE TABLE IF NOT EXISTS` did nothing and the `ALTER` list was incomplete | Start-up schema assertion, rebuild of an incompatible table, **refusal to trade** if the book cannot hold a position, and sell-back when the insert fails after a buy | INC-001, prevention rules |
| 7 | **Zombie deployment** | The running build was not the repository head; two committed, green fixes were not live | Build fingerprint (file hashes + runtime configuration) reported by the service, compared before and after every deployment | Restart checklist |
| 8 | **Unbounded restart loop** | A supervisor restarted a failing process into the same failure | Supervisor plus a trader that refuses to start without a valid book; concurrency above one is not armed until tested at that concurrency | INC-001 |
| 9 | **Atomic rollback by account close** | Every partial sale included a close for an account that still held tokens; the venue rejected it and atomicity rolled the sale back — 45 attempts, zero fills | Close the token account only when the sale empties it | `test_partial_close` — 6 checks |
| 10 | **Selling inventory that is gone** | After a full sale emptied the wallet, later rungs asked to sell tokens that were no longer there | An empty wallet is recorded as `inventory_absent` and the ladder ends; the record states what was observed, not who moved the tokens | `test_exit_lifecycle` — 3 checks |

---

## The pattern

Nine of the ten are the same mistake in different clothes: **treating an observation as a fact.**

- a mark is treated as a price (3);
- a heartbeat is treated as a working feed (4);
- a timestamp is treated as ownership (1);
- a submitted transaction is treated as a fill (9);
- a deployment is treated as a running build (7);
- a `CREATE TABLE IF NOT EXISTS` is treated as a schema (6).

The fix is always the same shape: record what was actually observed, give the observation a provenance and an age, and make the decision path refuse to proceed when the observation is not good enough. That is also why every close path has to write a reason — a system that cannot explain a loss cannot improve one.

## What is deliberately *not* guarded

- **Adverse selection.** A stop can be right and still fill badly. Attribution separates the two; it does not prevent the loss.
- **Rug pulls and malicious metadata.** The safety gates check authorities, holders, liquidity and age. They cannot read intent.
- **Venue outage longer than the position's horizon.** The system can decide to exit and be unable to; it records `no book` rather than inventing a fill.
