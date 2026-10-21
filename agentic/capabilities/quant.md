# Capability pack: quant — project `tradingview`

For the `quantitative-researcher` lane, in both modes. You own **guardrails 1,
2 and 3**: no repainting, every formula traced to a named source, every symbol
verified rather than reconstructed.

Companion sources: `tradingview/CLAUDE.md`, the script headers themselves, and
`docs/<slug>.md`.

---

## Index

Read this block first. You are not expected to read this file end to end — read
what your order touches. Reading one extra section is cheap; acting on a
convention you never read is not.

**Always read:** **What makes this project's mathematics different** · **Repainting is your finding, not the engineer's** · **Definition of done for this lane**

| Section | Read it when |
|---|---|
| The scripts that exist, and what each claims | your order names an existing script |
| Audit mode: the anchors available here | you are in AUDIT mode |
| Research mode: brief sections | you are in RESEARCH mode |
| Volume analysis — the standing research question | the order touches volume |
| Units, bars and annualisation | the order involves a rate, a return or a lookback |

---

## What makes this project's mathematics different

**There is no data layer to interrogate.** In a normal quant repo you can print
the intermediate series. Here the series live inside TradingView and the only
way to see one is to plot it on a chart — which you cannot do. So an audit is
**re-derivation from the definition**, not recomputation from the data, and that
changes what a `PASS` is allowed to mean.

State the limit rather than working around it. "Re-derived the percentile rank
from the CFTC definition and the script's expression agrees, on a worked example
of 12 hand-computed bars" is an audit. "The code looks right" is not.

**The trader acts on these directly.** There is no reviewer downstream of the
chart, no portfolio system that sanity-checks the signal, no second opinion. A
wrong threshold does not get caught by a test — it gets traded. This is why the
lane runs on Opus and why `FAIL` here stops the run.

## Repainting is your finding, not the engineer's

Guardrail 1, and the most under-detected defect class in Pine. `pine_check.py`
catches the two textual cases — `lookahead_on`, and a `request.security()` with
no explicit `lookahead`. It cannot catch the rest, and the rest are yours:

| Pattern | Why it repaints |
|---|---|
| A higher-timeframe value used before that timeframe's bar closes | the value is provisional and rewrites itself intrabar |
| `ta.pivothigh` / `ta.pivotlow` plotted at the pivot bar | the pivot is only knowable `n` bars later; on history it draws in the past |
| A threshold computed from a window that includes the current, unclosed bar | the signal changes as the bar develops and is different after the close |
| `ta.valuewhen` on a condition that itself repaints | inherits it |
| An alert on a condition not guarded by `barstate.isconfirmed` | fires, then un-fires |
| `strategy.entry` on an intrabar condition with `calc_on_every_tick=true` | backtest and live disagree |

A script may legitimately do several of these — a dashboard showing the *current*
provisional state is useful precisely because it is live. The finding is not
"this repaints", it is **"this repaints and the header does not say so"**. Grade
it `CRITICAL` when a backtest or an alert depends on it, `MATERIAL` when a
displayed value misleads, `MINOR` when it is disclosed and harmless.

## The scripts that exist, and what each claims

Read the header before the code; each states what it is and what it is not.

| Script | What it is | The claim to check |
|---|---|---|
| `mm_cot_long_short_index.pine` | a single net (long − short) Managed Money COT Index on 0..100, from the CFTC Disaggregated COT report | the CFTC tickers were once reconstructed from search results; the header now records them as **confirmed against TradingView's live symbol search on 2026-09-17**. Any order touching it keeps that evidence in the header, and a new symbol starts unverified again |
| `strategy13_trend_carry.pine` | Carver, *Advanced Futures Trading Strategies*, Strategy 13 | a **single-instrument adaptation** with IDM, instrument weight and FX all forced to 1. That is a real departure from the book's sizing, disclosed in the header. Carry needs a genuine futures curve and is optional |
| `systemic_stress_dashboard.pine` | VIX / COR1M / credit percentiles into a composite stress reading | an **original approximation**, not a reproduction of any published indicator. It has no external authority and the header says so — so the anchor for auditing it is internal consistency plus the definition of a percentile rank |
| `volume_profile.pine` | volume-by-price histogram and Point of Control over the last N confirmed bars | an **original construction** built from lower-timeframe closes via `request.security_lower_tf()` ("close-binning"), never tick data, and not expected to match TradingView's native Volume Profile. The anchor is a hand-binned worked example; parity with the native tool is explicitly not the test |

**These headers are the repo's best habit.** An order that improves one of these
scripts and quietly drops its admission has made the repo worse. Say so.

## Audit mode: the anchors available here

`protocol/gates.md`'s rule applies with force: a gate that reads only the source
the work was built from is not a gate. Name the anchor in
`verification.detail`. In descending order of strength:

1. **The cited primary source.** Carver's book for Strategy 13, the CFTC's own
   report definitions for COT fields, Cboe's methodology page for VIX and
   COR1M. If the script cites it, re-derive from it — not from the script.
2. **A hand-computed worked example.** Eight to fifteen bars, written out in the
   artifact, run through the formula by hand. This is the strongest anchor
   available for an original construction, and it is usually the only one.
3. **A second data source** for the same quantity, where one exists.
4. **Dimensional and boundary analysis.** What does the value do at 0, at 1, at
   a constant series, at a single bar of history, when the symbol fails to
   resolve? A percentile that returns 50 on an all-`na` window is a bug that no
   amount of code reading reveals.

**Re-reading the script's own comments is not an anchor.** Neither is agreeing
with `docs/<slug>.md`, which was written from the same script.

## Research mode: brief sections

Below the report block comes the `## Orchestrator brief` (PROTOCOL.md § 4), then:

- **Concept** — what the number means to a trader, in one paragraph.
- **Formula** — stated in mathematics, then in Pine terms, with every parameter
  named and its default justified. A default chosen by feel is declared as such.
- **Source** — book and chapter, paper, or spec. Or an explicit **ORIGINAL**
  with the reasoning for building rather than borrowing.
- **Data requirements** — every symbol, its timeframe, and what happens when it
  does not resolve. Guardrail 3 lives here: say whether each symbol is
  *confirmed* or *reconstructed*.
- **Repaint analysis** — walk the table above against this construction.
- **Failure modes** — insufficient history, gaps, holidays, a constant series, a
  symbol that resolves to the wrong instrument.
- **What would falsify it** — the observation that would show the metric is not
  measuring what it claims. A metric with no such observation is not a metric.

## Volume analysis — the standing research question

The trader's stated aim is *using volume to decide when to open and close a
position*. Expect orders in this territory, and know the traps before you write
a brief:

- **Volume is not comparable across instruments or eras.** Raw share volume on
  an ETF says nothing next to another's. Normalise — relative volume against a
  rolling median, or a z-score — and say which, because the choice changes the
  signal.
- **The rolling window must exclude the current bar** when the comparison is
  "unusual versus normal", or every bar is compared against a baseline it is
  itself inflating.
- **The current bar's volume is incomplete until it closes.** A relative-volume
  reading intrabar is a fraction of a bar compared against whole ones. This is
  the most common volume repaint and it is invisible on historical bars.
- **ETF volume is a creation/redemption artefact as much as a demand signal.**
  Unlike a single stock, an ETF's volume can spike from authorised-participant
  activity that says nothing about directional interest.
- **Holidays and half-days** produce low-volume bars that are not low-interest
  bars. Any threshold crossing on a half-day is noise.

## Units, bars and annualisation

- **Everything is in bars, and a bar is whatever the chart says.** A lookback of
  20 is 20 bars — an hour on a 3-minute chart, a month on a daily. Any parameter
  whose correct value depends on the timeframe must say so in its `input`
  tooltip, or be derived from `timeframe.period` rather than hard-coded.
- **Annualisation uses 252 trading days** for daily bars in this repo. State the
  factor wherever it appears; a volatility figure with an unstated basis is not
  comparable to anything.
- **Percentages are expressed as percent, not fractions, wherever the user sees
  them**, and the script is consistent within itself. Mixed conventions inside
  one script is a `MATERIAL` finding.
- **Sign convention:** positive is bullish/long/risk-on across every script here.
  A stress index that rises when things get worse is an inversion and needs to
  be labelled as one.

## Definition of done for this lane

- RESEARCH: a brief whose formula someone could implement without asking you a
  question, and whose every parameter default has a stated reason.
- AUDIT: a verdict with the anchor named in `verification.detail`, and a worked
  example in the artifact where the anchor was a hand computation.
- Every finding graded `CRITICAL | MATERIAL | MINOR`, and `severity` on the
  change request derived from the verdict rather than copied from the grade
  (PROTOCOL.md § 5).
- Repaint analysis present in every artifact that touched a computed series,
  even when the answer is "it does not".
- `verdict: PASS` only where you could state what would have made it fail.
