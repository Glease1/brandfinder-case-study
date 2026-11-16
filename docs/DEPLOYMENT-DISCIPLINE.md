# Deployment discipline

The most uncomfortable lesson from this project: **the running build was not the head of the repository — and I did not know it.**

Two fixes were committed, tested and green, and still not running:

| Change | Test |
|---|---|
| Close a token account only when the sale empties it | `test_partial_close` — 6/6 |
| Stop the exit ladder when the wallet holds none, recorded as `inventory_absent` | `test_exit_lifecycle` — 3/3 |

Neither was live. A restart would have replaced a process that was holding positions, so the deploy was deferred — and then forgotten for long enough that the running build and the repository head diverged.

## What the running build could not do

Every partial sale transaction included a close for a token account that still held tokens. The token program rejects that close (`NonNativeHasBalance`), and because a transaction is atomic, **the sale in front of it was rolled back too**.

The partial therefore sold **nothing**: 45 attempts, zero fills. Slippage was never the cause — widening it from 15% to 90% could not have helped, because the transaction never reached the point where slippage mattered.

Once a full sale had emptied an account, later retries asked to sell tokens that were no longer there, and the ladder kept retrying every rung. The lifecycle fix records the empty wallet as `inventory_absent`, states what was observed, deliberately does not assert who moved the tokens, and ends the ladder.

## The rule that came out of it

A deployment is not complete when the process starts. It is complete when the process can prove **what it is running**.

The service now reports a build fingerprint — file hashes plus the runtime configuration — over its own status endpoint. Before a deployment, capture it. After, capture it again and compare. If either the hashes or the configuration differ, the deployment did not take and validation stops there.

One honest caveat, recorded at the time: file hashes describe the code **on disk**, not what an already-running process imported at start-up. A process that predates the fingerprint feature cannot self-report, so its identity has to be established by observation instead — for example, by checking that a function introduced by the fix does not appear in the running image.

## Validation required before a restart

All of these, on live evidence, before a build holding positions is replaced:

1. A partial sell is submitted and **confirmed on chain**.
2. The tokens sold match the requested amount, and the remainder stays in the account.
3. The account is left open after a partial.
4. The position row is not closed by a partial; it stays open with the remaining quantity.
5. A later full sell closes the position and reclaims the rent.
6. A position whose tokens are already gone stops the ladder at once and is recorded as an external sale, not a failed exit.

A simulation, a quote, or a submitted-but-unconfirmed transaction does **not** count as validation.

## Restart procedure

1. Confirm no position is open.
2. Restart the canary so the new build loads.
3. Read the running configuration from the process, not from `.env`.
4. Watch the first partial end to end: submission, confirmation, token delta, remaining balance, position state.
5. Only then leave it unattended.

## The three sentences worth remembering

- A submitted transaction is not a confirmed one.
- A quote is not a fill.
- A restart is a decision, not a recovery.
