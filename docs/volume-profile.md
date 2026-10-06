# Volume Profile (approximation)

Script: `volume_profile.pine`. Indicator, not a strategy — no fixture, no
Strategy Tester export (see § Reproducing the fixture).

## 1. What it computes

A price-anchored, right-edge volume-by-price histogram over the last
`lookbackBars` **confirmed** chart bars (fewer if less history exists; below
`minBars`, output is `n/a` — never padded, never borrowed from another
symbol). The window's own `[low, high]` range is split into `binCount` equal
-height price bins — a bin **count**, not a bin **width**, so the resolution
auto-scales to whatever the instrument and the window are doing rather than
needing per-symbol tuning.

Every lower-timeframe intrabar inside every window bar contributes its
**entire** volume to the single bin containing its **close** ("close
-binning" — the rejected alternative, spreading an intrabar's volume across
every bin its `[low, high]` touches, is not implemented: it smears volume
into prices where nothing necessarily traded and softens the exact Point of
Control this script exists to locate). Each bin's total volume `V_j` is
normalised to the window's busiest bin (`V_max`) to produce a bar length
`L_j` in `[0, 1]`, which the renderer turns into a box width. The profile is
**self-relative**: a full-length bar means "the busiest price in this window
on this symbol", never an absolute quantity, and two charts' profiles (or the
same chart's profile across a change of `lookbackBars`) are not comparable by
bar length.

Point of Control (POC) is the bin with the largest `V_j`; ties resolve to the
lower price bin, implemented explicitly rather than left to
`array.indexof`'s default.

The profile answers "at which **prices** has volume traded over this
window" — a location tool, not a timing one. It says nothing about buying vs
selling pressure and nothing about direction; that is VSA/Wyckoff territory
and is explicitly out of scope (rendering only). For the repo's standing
research question — what volume behaviour says about when to open and close
a position — this script identifies high-volume nodes (prices where a large
amount changed hands, where subsequent trade tends to pause) and low-volume
nodes (prices passed through quickly, which price tends to traverse fast
again), not entry/exit timing itself.

Source: `02-quant.md` § Concept and § Formula
(`C:\projects\investments\.agentic\projects\tradingview\runs\2026-09-18-vertical-volume-indicator\02-quant.md`).

## 2. Inputs

| Input | Type | Default | What it controls | Timeframe-dependence |
|---|---|---|---|---|
| `lookbackBars` | `input.int`, 20-1000 | 120 | Window length in bars: the profile is built from the last N confirmed chart bars, not TradingView's visible range. | Yes — units are bars, so 120 is ~6 months on a daily chart and ~6 hours on a 3-minute chart. Feel-chosen (02-quant § Formula, parameter table): ~6 months of daily bars keeps "where has business been done" a live question rather than archaeology, and stays comfortably inside TradingView's shared intrabar-history budget at the auto-selected 5-minute LTF. |
| `minBars` | `input.int`, 4-500 | 20 | Floor below which the window's range extremes would be set by one or two bars rather than a distribution; below it, output is `n/a`. | No direct timeframe coupling; feel-chosen floor (02-quant § Formula, parameter table), not derived from any source. |
| `includeCurrentBar` | `input.bool` | `false` | Whether the still-developing bar is included in the window. `true` makes the profile PROVISIONAL — every `L_j` and the POC can shift and migrate back within a session — and adds an on-chart PROVISIONAL label. `false` keeps the window to closed bars only (no repaint, guardrail 1). | No. |
| `binCount` | `input.int`, 4-100 | 24 | Number of equal-height price bins the window's `[low, high]` range is split into — a count, not a price width, so it auto-scales per instrument. | No direct timeframe coupling. Feel-chosen (02-quant § Formula, parameter table): below ~12 the POC band is too wide to place a stop against; above ~48 each bin holds too few intrabar prints on a typical daily-chart window and the profile becomes sampling noise rather than a distribution. |
| `ltfResolution` | `input.timeframe` | `""` (blank = auto) | Lower-timeframe override for the intrabar fetch. Blank auto-selects 5 minutes on Daily-and-above chart timeframes, 1 minute below Daily, targeting 50-100 intrabars per chart bar. | Yes, by design — derives from `timeframe.period` via `timeframe.in_seconds()` rather than being hard-coded. Override only with a specific reason: a timeframe not strictly lower than the chart's own returns no intrabar data for every bar, which reads as `n/a` (guardrail 5), not a fabricated value. |

## 3. Plots and outputs

**Profile boxes.** One `box` per bin, drawn right-anchored, width proportional
to `L_j` (max 40 bars wide). Blue fill for ordinary bins, orange fill for the
POC bin. Boxes and the POC label are fully cleared and redrawn from nothing
on every recompute (`barstate.islast` only) — a failed recompute never leaves
a stale shape on screen.

**POC label.** `"POC <price>  raw <volume>"`, placed at the POC bin's
midpoint, price formatted to instrument mintick, volume formatted with
thousands separators. The POC bin's own `[lo, hi)` bounds are not shown
separately (see § Known limits — POC precision).

**PROVISIONAL label.** Appears only when `includeCurrentBar = true`, red,
reading `"PROVISIONAL — includes unclosed bar"`. Rendered even when the
profile itself is `n/a`.

**Diagnostics table** (bottom-right, always visible, never a tooltip — per
guardrail 5):

| Row | Shows | Degraded / flagged state |
|---|---|---|
| Window (bars, M of N) | `M of N` confirmed bars actually in the window | `M < N` until N bars of history exist; below `minBars` the whole profile is `n/a`. |
| Coverage (C of M) | Bars for which `request.security_lower_tf()` returned any intrabar data | Orange cell when `C < M`. `C = 0` → no profile at all, `"n/a — no intrabar data"`, never an empty-looking flat profile. |
| Intrabars/bar (avg) | Realised average intrabars-per-bar over the covered window against the 50-100 target | Orange cell, `"— DEGRADED (target 50-100)"` suffix, when the average is materially below target. See § Known limits — this row counts skipped intrabars too (F7). |
| LTF used | The resolved lower timeframe (auto or override) | — |
| Reconciliation (Σ bins / Σ volume) | Sum of binned volume over the sum of the covered bars' native chart volume; should read ~1.00 when Status is a bare `"OK"` | Red cell, `"— CHECK"` suffix, when more than 2% from 1.00. **Reading this row requires reading the Status row too** — see below. |
| POC price | The POC bin's midpoint, formatted to mintick | `n/a` when the profile is `n/a`. |
| POC raw volume | `V_max`, the POC bin's raw (non-normalised) volume | `n/a` when the profile is `n/a`. |
| Status | `"OK"`, `"OK — PROVISIONAL"`, `"OK — N na intrabar(s) skipped"` (or the PROVISIONAL+skip composite), or one of four `n/a` reason strings | Gray cell on any `n/a` reason; orange cell when one or more na intrabars were skipped and the profile still rendered. |

**Reading Reconciliation and Status together — this is the corrected
behaviour as of this run's CR-1 fix.** A single lower-timeframe intrabar
whose close or volume comes back `na` from `request.security_lower_tf()` is
now skipped rather than allowed to poison a bin or crash the script: it is
never binned, never allowed to win or contaminate a bin's `V_j`, and it is
counted. When one or more intrabars are skipped this way, Status reads
`"OK — N na intrabar(s) skipped"` instead of a bare `"OK"`, and the
Reconciliation ratio legitimately falls below 1.00 **by construction** — the
skipped intrabar's volume leaves the numerator (`Σ bins`) while its bar's
full native volume stays in the denominator (`Σ volume`). That shortfall,
next to the na-skip note, is the expected combination, not a defect: it says
"a known, counted, deliberately-excluded quantity of volume is missing",
not "the profile is broken". **A low ratio is only cause to distrust the
*whole* profile when Status reads a bare `"OK"` with no na-skip note** — in
that combination, intrabars are being dropped, double-counted, or the
close/volume arrays are misaligned, and every bin is suspect, not just the
skipped one. Source: `06-backend.md` § changed and `07-quant-audit.md`
Finding 6 (both at
`C:\projects\investments\.agentic\projects\tradingview\runs\2026-09-18-vertical-volume-indicator\`),
which is also what this run's fix to the script header's Reconciliation
bullet now says.

## 4. Sources

**ORIGINAL, no external authority.** Declared per guardrail 2: there is no
regulator/exchange spec for volume-at-price the way there is for a COT field,
and no single named methodology to cite. The nearest named method, J. Peter
Steidlmayer's Market Profile (CBOT), is TPO — time-at-price, a genuinely
different measurement — so citing it here would be a false citation, worse
than declaring ORIGINAL. TradingView's own native Volume Profile is computed
server-side from tick data with an unpublished exact construction, so it
cannot be reproduced, only approximated, and a header claiming to implement
it would be claiming an authority that does not exist. Built from
lower-timeframe candle closes via `request.security_lower_tf()`, never tick
data — an estimate, never tick-exact even when intrabar data is fully
available. This is not TradingView's native Volume Profile tool and will not
agree with it bar for bar. (02-quant.md § Source.)

## 5. Known limits

From the header's own admissions, expanded:

- **Candle-derived approximation, not tick data.** Built from lower
  -timeframe candle closes via `request.security_lower_tf()`. Never
  tick-exact, even when intrabar data is fully available.
- **Not TradingView's native Volume Profile tool**, and will not agree with
  it bar for bar. Falsification tests disagreement, not parity.
- **Close-binning, not range-spreading** — a definitional choice, not a
  correctness one. Each intrabar's entire volume goes to the one bin
  containing its close; the alternative (spreading across every bin an
  intrabar's `[low, high]` touches) was considered and rejected because it
  manufactures apparent high-volume nodes at bin boundaries.
- **Rolling window, not a stable series.** Exactly one profile is rendered,
  anchored at the right edge, describing the last N bars as of now. Bin
  boundaries depend on the window's own `[low, high]` and are not stable
  across bars — a new extreme re-lays every bin. Today's profile is not the
  profile that existed three weeks ago even though it is drawn over the same
  chart. Bar-replay review of this indicator is misleading unless the replay
  itself re-anchors the window.
- **Overnight gaps** render identically to a genuine low-volume node — bins
  spanning a gap legitimately receive zero volume; different facts, same
  appearance, not imputed across.
- **Holidays/half-days** need no correction — the profile is volume-weighted,
  not bar-weighted.
- **Chart session setting** (RTH vs extended hours) changes which intrabars
  exist and therefore the profile, particularly around the closing-auction
  print that carries a large share of a US equity's daily volume.
- **ETF creation/redemption volume** can produce a high-volume node that is
  authorised-participant activity, not accepted value at that price.
- **Tick-count volume** (common on FX/CFDs) produces a well-defined profile
  of something else — ticks at price, not size at price. Only the trader can
  tell; the script cannot detect it.
- **`request.footprint()` not implemented.** It would give TradingView-native
  intrabar binning instead of this candle-close approximation. Considered,
  not implemented: its signature, plan/symbol availability and history
  limits were not verified against the official Pine v6 reference by the
  research this script was built from.
- **POC precision (04-quant-audit F5, open, non-blocking).** The POC label
  and table row publish the bin midpoint formatted to full instrument
  mintick precision, for a level that is only resolved to one bin's width —
  neither the bin height nor its `[lo, hi)` bounds appear on the chart, so
  the POC reads as a price rather than as the band it actually is.
- **Coverage row can misattribute a non-binning failure (04-quant-audit F4,
  open, non-blocking).** "Coverage (C of M)" reads `0 of M` both when
  intrabar data genuinely failed to return and when the failure happened
  before the binning loop ran at all (below-`minBars` or degenerate-range
  cases) — the Status row disambiguates, so this is a legibility defect, not
  a wrong number.
- **Intrabars/bar truncates on integer division (04-quant-audit F3, open,
  non-blocking).** `totalIntrabars / c` truncates rather than rounding,
  which can flag a window as DEGRADED on rounding alone near the 50
  threshold.
- **Coverage-branch na-guard unverified (04-quant-audit F2, open,
  non-blocking).** Whether `request.security_lower_tf()` returns an empty
  array or an `na` array id when a bar has no intrabar data at all was not
  verified against the Pine v6 reference; if it is `na` rather than empty,
  `array.size()` may raise rather than degrade to the specified `n/a`.
  Confirm at the compile check.
- **"Intrabars/bar (avg)" counts skipped intrabars in its numerator
  (07-quant-audit Finding 7, MINOR, informational).** `totalIntrabars`
  accumulates every intrabar `request.security_lower_tf()` returns,
  including the ones the na-guard then skips before binning — so the row
  reports intrabars **returned** per bar, not intrabars **binned** per bar.
  In the normal case the gap is small and the skipped count is visible
  separately in the Status row, so it can be subtracted by hand; the chart
  does not state which of the two quantities the row is. Not fixed in this
  run — a docs-only record, per this run's close-out order; a code fix
  (either exclude skipped intrabars from the count, or state in the header
  that the count is returned-not-binned) would be a new backend order.

## 6. Reproducing the fixture

**There is no fixture for this script.** `volume_profile.pine` is an
indicator, not a strategy — it has no Strategy Tester trade list to export,
and `fixtures/*.csv` in this repo is a strategy-backtest convention. The
repo's one existing CSV
(`Trend_+_Carry,_Risk-Regime_Sized_(Strategy_13)_AMEX_VTI_2026-08-24_55c9e.csv`)
belongs to `strategy13_trend_carry.pine`, not this script. The nearest
equivalent evidence for this indicator is 02-quant.md's hand-computed worked
example (3 bars, 4 intrabars each, `B = 4`, `R_lo = 10.00`, `R_hi = 12.50`),
independently re-derived against the shipped code in 04-quant-audit.md § 1
and again, with a partial-`na` mutation, in 07-quant-audit.md § 1 — both at
`C:\projects\investments\.agentic\projects\tradingview\runs\2026-09-18-vertical-volume-indicator\`.
