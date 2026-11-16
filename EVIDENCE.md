# Evidence

What was run, what it proves, and what it does not. The source is private; the counts and commands below are reproducible on request.

---

## 1. The control-loop checks

Seven bespoke scripts, each printing pass/fail lines and exiting non-zero on failure. Run on 13 November 2026 against Python 3.12 with bytecode writing disabled; every exit code was 0.

| Script | Result | What it proves |
|---|---|---|
| `test_sweep_ownership.py` | **10/10** | A live watcher is never swept; a dead watcher is recovered. Includes a check that the old rule *would* have sold the position |
| `test_pending_exit.py` | **21/21** | A claim survives a failed execution, recovers after restart, and sells exactly once |
| `test_fallback_price.py` | **23/23** | Tier order, freshness, source disagreement and attribution; a fallback never prices a sale |
| `test_blind_policy.py` | **14/14** | Transient blindness alerts, persistent blindness exits, closed-elsewhere detaches |
| `test_exit_attribution.py` | **18/18** | Every close path records a reason; a failed sell stays open |
| `test_guard_control_loop.py` | **14/14** | The watcher evaluates repeatedly, persists telemetry, issues exits for flash/stop/trail and handles failure |
| `test_ui_config.py` | **17/17** | The UI reads live configuration instead of hard-coded thresholds |

**Total: 117 checks, all passing.**

A later addition, run on 16 November 2026:

| Script | Result | Note |
|---|---|---|
| `test_tp_partial.py` | **27/27** | Partial take-profit ladder. Of the 27, six are source-shape assertions (they check the guard calls the right helper), not behavioural tests |

## 2. What was not run

| Check | Why not |
|---|---|
| A full `pytest` suite | `pytest` is not installed in the project venv, and there is no `pyproject.toml` or `pytest.ini` |
| `test_live_execution.py` (66 unittest methods) | Needs network, RPC and state |
| `test_shadow_execution.py` (30 methods) | Not run in this audit |
| `test_solana_infra.py` (21 methods) | Known pre-existing failures |
| `test_multi_instance.py` | Not run |
| On-chain reconciliation | Would require wallet and network access |
| The guard self-test against the live canary | Must not disturb a live service |

**The correct statement is: "117 targeted checks pass; the wider suite is not standardised and was not run."** Not "the test suite passes".

## 3. Test quality, honestly

**Strengths**

- The tests target failure paths, not happy paths.
- Docstrings name the production incident that motivated each check.
- Several assert that a bad outcome is *recorded*, not merely that an exception is raised.
- The ownership test proves the old rule would have caused the bug.

**Weaknesses**

- No single command runs everything; the harnesses are invisible to standard collection.
- No CI, coverage or flaky-test policy.
- Some tests use stubs that could drift from the production classes.
- The largest modules have little direct unit coverage.

## 4. The measurement

Derived from a read-only query of the canary book snapshot of 15 November 2026 (`state = 'closed'`): 171 rows, 62 wins, 109 losses, profit factor 1.22, +0.1557 SOL realised, top three trades = 154% of profit, −0.0833 SOL excluding them, 74 of 171 rows paper or legacy.

The performance, robustness and attribution routines exist as scripts; their output was never committed as an artifact, which is why the figures above had to be re-derived. That is a documentation debt, and it is the first thing on the fix list.

## 5. Provenance

- Single operator, AI-assisted implementation. Requirements, failure analysis, acceptance and the negative conclusions are human decisions.
- The ledger includes paper and legacy rows alongside live ones; every figure above states which population it describes.
- No wallet address, endpoint, key or strategy threshold appears in this case study. That is deliberate, and it is why the source is not published.
