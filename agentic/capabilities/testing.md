# Capability pack: testing — project `tradingview`

For the `test-engineer` lane. **This project has no test framework for its
product code**, and pretending otherwise is the failure this pack exists to
prevent. Read the first section before deciding what your order can possibly
deliver.

Companion sources: `tradingview/CLAUDE.md`, `<agenticRoot>/scripts/pine_check.py`.

---

## Index

Read this block first.

**Always read:** **What "testing" means here** · **Definition of done for this lane**

| Section | Read it when |
|---|---|
| The three things you can actually deliver | always, before planning your order |
| Fixtures — Strategy Tester exports | your order involves a strategy's backtest results |
| Worked examples | your order is to pin a formula's behaviour |
| Testing `pine_check.py` itself | your order edits the checker |

---

## What "testing" means here

Pine cannot be unit tested. There is no runner, no assertion library, no way to
call a function with an input and inspect its output outside a chart. TradingView
is the only execution environment and no lane holds browser tools.

**So there is no "write tests for this script" order that can be satisfied as
written.** If you receive one, the honest response is `PARTIAL` with the three
deliverables below named in `handoff`, not a file of Pine that looks like tests
and executes nothing.

What you must never do: write a `.pine` file containing assertions, comparison
plots, or a "test harness" that only runs if someone loads it on a chart. It
cannot be run by anything in this network, nobody will load it, and it will rot
into a file that asserts a behaviour the script no longer has. A test nobody
runs is worse than a documented gap, because it reads as coverage.

## The three things you can actually deliver

1. **A fixture** — a Strategy Tester export capturing what a strategy actually
   produced on a named symbol, timeframe and date range. Evidence, recorded.
2. **A worked example** — a hand-computed trace of a formula over a small number
   of bars, in Markdown, that a human or the quant lane can check the script
   against. This is the closest thing to a unit test that exists here.
3. **A `pine_check.py` rule** — when the defect your order is about is textual,
   the durable fix is a new rule in the checker plus the case that pins it. This
   is the only one of the three that runs automatically, and where it applies it
   is worth more than the other two combined.

Prefer (3) whenever the defect is expressible as a pattern. A repaint you catch
once by reading is caught forever by a rule.

## Fixtures — Strategy Tester exports

`fixtures/<slug>-<symbol>-<date>.csv`, exported from TradingView's Strategy
Tester "List of Trades" tab. No `fixtures/` directory exists yet: the one export
in the repo, from `strategy13_trend_carry` on AMEX:VTI, sits at the repo root
under TradingView's own filename
(`Trend_+_Carry,_Risk-Regime_Sized_(Strategy_13)_AMEX_VTI_2026-08-24_55c9e.csv`).

Its columns: `Trade number, Type, Date and time, Signal, Price USD, Size (qty),
Size (value), Net PnL USD, Return %, Commission USD, Favorable excursion USD/%,
Adverse excursion USD/%, Cumulative PnL USD/%, Duration (bars)`. Note that each
trade occupies **two rows** — the exit first, then the entry — and both rows
repeat the trade's summary figures. A naive row count doubles every statistic.

**You cannot generate a fixture.** It comes out of TradingView, which means the
main session produces it. Your order can specify exactly what is needed — symbol,
timeframe, date range, settings — and name it in `handoff`; it cannot produce it.

**A fixture is evidence of a run, not a golden file.** Nothing compares against
it automatically. Its value is that a later change can be checked against what
the strategy used to do, by a human, with the settings recorded. Record the
settings alongside it in `docs/<slug>.md` or the fixture is uninterpretable six
months later.

## Worked examples

Written into your artifact, or into `docs/<slug>.md` when the order says so:

- Eight to fifteen bars, with real or plausible OHLCV values stated explicitly.
- The formula applied by hand, one step per column, so a reader can see where a
  discrepancy enters.
- The boundary rows included deliberately: the first bar, a bar with `na`, a
  constant stretch, the bar where a threshold is crossed exactly.

This is what the `quant-audit` gate reads when the anchor is a hand computation,
and it is often the most valuable artifact a run produces.

## Testing `pine_check.py` itself

When your order edits the checker:

```bash
python <agenticRoot>/scripts/test_pine_check.py
```

It is a dependency-free script, not a pytest module — run it directly; it prints
`N/M passed` and exits non-zero on any failure.

- **Add the case that would have caught the bug**, and add it before the fix, so
  you have seen it fail.
- The suite already pins a real regression — anchored `^\s*` patterns matching
  the newline and dragging every reported line number towards 1. Keep that class
  of case; line numbers being right is most of what makes the tool usable.
- A new rule owes at least three cases: it fires on the bad pattern, it does not
  fire on the good one, and it does not fire on the near-miss that looks like the
  bad one (a user function named `my_sma`, a paren inside a string).
- Severity is part of the contract. A rule graded `BLOCKING` stops a lane; pin
  the grade in the test, not just the code.

## Definition of done for this lane

- Nothing you wrote pretends to execute Pine.
- Every fixture you called for names the symbol, timeframe, date range and
  settings needed to reproduce it — or you did not call for it.
- A worked example, where you wrote one, includes its boundary rows.
- The checker suite green where you touched the checker, with a case that fails without
  your fix.
- Where the order asked for something this project cannot do, you said so in
  `risks` and set `PARTIAL` rather than delivering a shape that looks like the
  answer.
