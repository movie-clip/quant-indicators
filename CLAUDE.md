# trading-indicators

Pine Script v6 indicators and strategies for TradingView, written for one
trader's own use. The instruments are **ETFs and stocks**; the working style is
**trend, volume and RSI**, with the standing research question being *what
volume behaviour says about when to open and close a position*.

Not a library, not a product, no users but the author. That matters: it means
correctness and legibility beat generality every time, and a script that is
wrong in a flattering direction costs real money.

## Layout

What's actually on disk today:

```
tradingview/
  mm_cot_long_short_index.pine
  range-projection.pine
  strategy13_trend_carry.pine
  systemic_stress_dashboard.pine
  volume_profile.pine
  Trend_+_Carry,_Risk-Regime_Sized_(Strategy_13)_AMEX_VTI_2026-08-24_55c9e.csv
  docs/
    mm-cot-long-short-index.md
    range-projection.md
    volume-profile.md
    testing-methodology.md   the verification method, not a per-script doc
  .agentic.json           binds this repo to the agent network
```

Five scripts, flat at the repo root. Flat is on purpose: there are a handful of
scripts and each is independent, so a directory tree would be structure with
nothing to hold.

**Script docs live at `docs/<slug>.md`, where the doc slug is the script's
filename with underscores turned into hyphens** — `volume_profile.pine` →
`docs/volume-profile.md`. `range-projection.pine` is documented at
`docs/range-projection.md` (its doc slug is already hyphenated). Three scripts
have one; `strategy13_trend_carry` and `systemic_stress_dashboard` do not yet.

The `.csv` above is a Strategy Tester trade-list export for
`strategy13_trend_carry.pine`, kept loose at the root under TradingView's own
export name — there is no `fixtures/` directory yet. `fixtures/<slug>-*.csv` is
the convention for new backtest evidence; when the first one is filed it lands
there.

## Stack

**Pine Script v6 only.** Every file declares `//@version=6` on its first
non-comment line. v4 and v5 idioms (`study()`, bare `security()`, `iff()`, bare
`sma()`) do not compile in a v6 file, and mixing them is the most common way a
script that "looks fine" fails to load.

There is no build, no package manager and no local runtime. **Pine only
compiles inside TradingView's editor.** That single fact shapes everything
about how work is verified here — see "Commands" below.

## Commands

| What | Command |
|---|---|
| Static check one script | `python ../.agentic/scripts/pine_check.py <file>.pine` |
| Static check, blocking on advisories | `python ../.agentic/scripts/pine_check.py <file>.pine --strict` |
| Static check everything | `python ../.agentic/scripts/pine_check.py *.pine` |
| Pin the checker itself | `python ../.agentic/scripts/test_pine_check.py` |

`pine_check.py` is a pattern checker, not a compiler. It catches version
mismatches, repaint patterns, missing strategy costs, unbalanced parens and
v4/v5 leftovers. **It cannot tell you a script compiles.**

**The compile check is the acceptance command**, and it runs in TradingView:
paste the script into the Pine editor, read the error panel, and for a strategy
read the Strategy Tester. Nothing is done until that has happened. It is the
equivalent of a test suite here, and like a test suite it is not optional.

## The six guardrails

Stated here so a session working this repo outside the agent network still sees
them. `../.agentic/projects/tradingview/project.md` carries the numbered copy
that a `REFUSED` cites.

- **No repainting.** A signal that moves after its bar closed is not a signal.
- **Every formula traces to a named source**, or the header declares it original.
- **Symbols are verified, never reconstructed.** A guessed ticker is a wrong
  script that looks right.
- **Costs are declared.** A strategy states commission and slippage.
- **Degrade visibly.** Missing data shows as `n/a`; never substitute a
  plausible value.
- **No execution.** Nothing here places an order or moves money.

## Working on a script

Read the script's header comment first. Each carries, in the header, exactly
what a reader needs to distrust it correctly — `systemic_stress_dashboard.pine`
and `volume_profile.pine` say they are original constructions rather than
reproductions of any published indicator, and `mm_cot_long_short_index.pine`
replaced its "tickers reconstructed, not confirmed" warning with the date and
method by which they were confirmed. **That
convention is the repo's most valuable habit.** Preserve it: when you cannot
verify something, say so in the header rather than quietly shipping it.
