# Quarterly StdDev Fan (range-projection)

Script: `range-projection.pine` (indicator title "Quarterly StdDev Fan", short
title `Q-StdDev`). Indicator, not a strategy — no fixture, no Strategy Tester
export (see § 6).

Run evidence lives at
`C:\projects\investments\.agentic\projects\tradingview\runs\2026-10-05-range-projection-quarter-anchor\`
(`02-quant.md`, `06-quant.md`, `04-`/`08-quant-audit.md`, `compile.md`,
`decisions.md`); file names below refer to that directory.

## 1. What it computes

At the first trading session of every calendar quarter (Jan / Apr / Jul / Oct),
the script centres a volatility envelope on **A(q), the open of that session**,
and draws +/-1 and +/-2 sigma bands that start at zero width and widen as the
square root of elapsed calendar time, reaching exactly +/-1 and +/-2 sigma_Q at
the quarter's end. Decisions behind it: D-1 keep the widening fan, D-2 anchor on
the first day's open, D-3 follow the selected chart timeframe, nothing hard-coded
(`decisions.md`).

It is a **volatility envelope, not a forecast**: the centre line assumes zero
drift.

### Formula (02-quant § Formula)

- `q_id(d) = 4*year(d) + floor((month(d) - 1) / 3)`.
- `T_q` = 00:00 on the first calendar day of quarter q, exchange timezone.
- `r_j = ln(H_j / L_j)`, from TradingView's own **3M** bar for completed quarter
  j (log units). Valid only if H and L are non-na, > 0, and H >= L.
- `K = sqrt(8 / pi) = 1.5957691`, the expected range of a standard Brownian path
  over unit time.

```
sigma_Q(q) = ( mean of r_j over the L most recently completed quarters q-L..q-1 ) / K
tau_b      = ( min(time_close_b, T_{q+1}) - T_q ) / ( T_{q+1} - T_q ),  clipped below at 0
U_k(b)     = A(q) * exp( +k * sigma_Q * sqrt(tau_b) )      k = 1, 2
D_k(b)     = A(q) * exp( -k * sigma_Q * sqrt(tau_b) )
M(b)       = A(q)
```

sigma_Q is the standard deviation of `ln(P_end / P_start)` over one calendar
quarter under driftless Brownian motion; it is displayed as a percent. sigma_Q
is **n/a** (never a substitute) unless all of: the L quarters are consecutive;
none is the symbol's first (inception, possibly partial) 3M bar; every high/low
is valid; the range is non-zero.

The time axis is timestamps. There is no bars-per-quarter constant and no
intraday cap, so the same code follows whatever time-based timeframe (3M or
shorter) is selected.

### Worked example (02-quant § Worked example; re-derived independently in 04 and 08)

L = 4, A = 100.00, quarter 2027Q2 (1 Apr to 1 Jul, 91 days). 3M log ranges for
2026Q2..2027Q1 are 0.10, 0.20, 0.15, 0.25 (2026Q1 is the symbol's inception bar
and is excluded; the 2027Q1 fan is therefore n/a, "need 4 completed quarters,
have 3").

`sigma_Q = 0.175 / 1.5957691 = 0.10966499` (10.97%).

| tau | U_1 | D_1 | U_2 | D_2 |
|---|---|---|---|---|
| 0.25 | 105.6364 | 94.6644 | 111.5904 | 89.6134 |
| 0.50 | 108.0631 | 92.5386 | 116.7763 | 85.6338 |
| 1.00 | 111.5904 | 89.6134 | 124.5242 | 80.3057 |

Check identity: U_2 at tau 0.25 equals U_1 at tau 1.00 (`2*sqrt(0.25) = 1`); an
implementation that fails it is not using sqrt(tau). Daily bar 2027-05-12 closing
16:00 exchange time: tau = 41.6667 / 91 = 0.457875, U_1 = 107.7029, D_1 =
92.8480. Validation case: if the 3M value read on 2027-04-01 carries id 2026Q4,
sigma_Q for 2027Q2 must be n/a, not 0.1097.

### Weekly worked example (06-quant § Weekly worked example; re-derived in 08)

Same data. Weekly start bar B is the week of Mon 2027-03-29 (Good Friday is
26 Mar). Its sessions are 29, 30, 31 Mar (2027Q1) and 1, 2 Apr (2027Q2).
`idx_b = 3`, the anchor is the open of the **1 Apr daily session = 100.00**, not
the week's open on 29 Mar, and sigma is the same 0.10966499 as the daily chart.

| weekly bar (time_close) | tau | U_1 | D_1 | U_2 | D_2 |
|---|---|---|---|---|---|
| B, if time_close = Fri 2027-04-02 16:00 | 0.018315 | 101.4952 | 98.5268 | 103.0128 | 97.0754 |
| B, if time_close = Mon 2027-04-05 00:00 | 0.043956 | 102.3258 | 97.7270 | 104.7058 | 95.5057 |
| week to Fri 2027-04-09 16:00 | 0.095238 | 103.4423 | 96.6723 | 107.0030 | 93.4553 |
| week to Fri 2027-05-14 16:00 | 0.479853 | 107.8926 | 92.6847 | 116.4082 | 85.9046 |
| week to Fri 2027-06-25 16:00 | 0.941392 | 111.2270 | 89.9062 | 123.7144 | 80.8313 |

Which `time_close` convention a weekly bar uses (Friday close or next Monday
00:00) has not been observed; the cap at `T_{q+1}` bounds either. The following
week (28 Jun to 2 Jul) ends in 2027Q3, so it draws 2027Q3's fan, never 2027Q2 at
tau > 1.

## 2. Inputs

| Input | Type | Default | What it controls | Timeframe-dependence |
|---|---|---|---|---|
| "Lookback Quarters" (`i_lookback`) | `input.int`, 4-20 | 8 | Completed quarters in the sigma window. Relative standard error of sigma is about 30%/sqrt(L): 10.5% at 8, 15% at 4. Judgement-chosen. | No. |
| "Project Quarters Ahead" (`i_project`) | `input.int`, 1-4 | **1** | How far the fan extends. 1 = to the end of the current quarter. Levels at P quarters are sigma*sqrt(P), not sigma; labels name the horizon (e.g. "+1σ @2Q") when P > 1. Default changed from 2 to 1 in this run (03 contract note); only newly added chart instances pick it up. | No. |
| "Show Historical Fans" (`i_hist`) | `input.bool` | true | Draw completed quarters as well as the current one. | No. |
| "Show Labels" (`i_labels`) | `input.bool` | true | Labels at the projected quarter end. | No. |
| "Historical Quarters Shown" (`i_maxfans`) | `input.int`, 1-50 | 20 | Completed quarters drawn in addition to the current one. Display only; the oldest fan drops off when a new quarter starts. Retitled from "Max Historical Fans" (03 contract note); saved settings may reset. | No — but see below. |
| "±1σ Color (green)", "±2σ Color (dark)", "Midline" | `input.color` | as in script | Colours. | No. |
| "Log to Pine Logs" (`i_log`) | `input.bool` | true | Writes the Pine Logs entries in § 3. Does not change any plotted value. Appended after "Midline", so no saved setting resets. | No. |

All inputs now carry `group=` ("Fan" or "Display") and tooltips, so the settings
dialog layout changed (03 contract note).

**Why the daily chart's first fan is 2021-10-01.** That is the display cutoff
(06-quant § Daily earliest fan), not a data limit and not a defect. `shown`
requires `f_q >= cur_qid - Historical Quarters Shown`. With the last bar in
2026Q4 and the default 20, the oldest drawn quarter is 2026Q4 - 20 = 2021Q4, whose
first session is 2021-10-01. "Every quarter" therefore means 21 fans, 2021Q4 to
2026Q4: 20 completed quarters plus the current one with its projected polyline.
Raising the input moves the daily and weekly charts back together. Weekly depth
beyond the default depends on `request.security_lower_tf` daily history, which
was not checked (§ 5).

## 3. Plots and outputs

**Elapsed bars.** 10 `plot()` lines (+2σ, +1σ, centre, -1σ, -2σ, twice: even and
odd quarters) and 6 `fill()`s. Alternate quarters go to alternate series with
`plot.style_linebr`, so a band is never joined across a quarter boundary. Values
are exact per bar (no straight chords). The elapsed centre line cannot be dashed
(a `plot()` limitation); only the projected centre line is.

**Projected part (last bar only).** 8 polylines (3 fills, 5 lines) from the last
bar out to the calendar start of quarter q+P, drawn in `xloc.bar_time`, plus 4
labels (±1σ, ±2σ levels with price at that x). Everything is deleted and recreated
on the last bar on every tick, with identical geometry.

**Projected tau is piecewise calendar time** (04 contract note). Inside quarter q
it is the elapsed fraction above. Beyond it, each further quarter is worth exactly
1, so the polyline ends at precisely sigma*sqrt(P) on the start of quarter q+P.
The single-length reading `(t - T_q) / len_q` would miss P by a day or two for
P > 1 (it gives 3.0224 at P = 3). The polyline is sampled uniformly in sqrt(tau)
over M = 32 segments; chord error is at most `sqrt(P)/(4M)` of k*sigma, i.e.
0.0078 at P = 1 and 0.0156 at P = 4 (04).

**Quarter-start detection.**
- Time-based charts of 1D or shorter: a start is the first chart bar whose
  **trading day** (`time_tradingday`, UTC) falls in a different quarter from the
  previous bar's. Not `time("3M")`. The anchor is that bar's open: on a daily
  chart the first session's open; on an intraday chart with extended hours shown,
  the first pre-market print.
- Charts with bars longer than a day (weekly, monthly, 3M): the quarter's first
  daily session is found from `request.security_lower_tf(..., "1D", ...)`, carrying
  the last daily element's quarter id across chart bars. The anchor is that daily
  element's open. On a monthly or 3M chart it coincides with the bar's open; on a
  weekly chart it is the only correct source.
- The first chart bar of history never starts a quarter.

**Weekly behaviour (06, 07, 08).** Before the weekly fix, fans appeared only on
quarters whose first session was also the first session of its week (9 fans on
SPX, 2022Q1 to 2024Q3, which is what the user saw), because the 3M value read on
the spanning bar was not for q-1. Now:
- Every quarter draws a fan on weekly, including quarters that begin mid-week;
  21 fans on SPX are expected (2021Q4 to 2026Q4), with sigma identical to the daily
  chart's. The user accepted the weekly and daily result ("it looks fine",
  `compile.md` § Re-check after 07-backend) without an item-by-item breakdown.
- A bar draws the band of the quarter of its **last daily session** (`bq`), and
  only if `bq == f_q`. A bar whose daily array is empty draws nothing.
- A bar's `time_close` at or past `T_{q+1}` is a timestamp convention, so tau is
  computed with `min(time_close, T_{q+1})`. This also caps 3M and monthly bars.
- **A live spanning week redraws once.** Until the first new-quarter session
  prints, the current week plots the previous quarter's fan at tau = 1. When that
  session prints, the bar switches to the new quarter's fan, before the bar
  closes. Its committed value equals its history value, so this is not a repaint
  under guardrail 1, but it is visible (06 § Repaint analysis of the fix).
- If the stale-carry rule applies (a bar longer than a day followed an empty daily
  array), the anchor is n/a (code 6). See § 5.

**3M chart.** Each bar is one whole quarter, so each quarterly band is a single
isolated point per series and `plot.style_linebr` draws no segment from it: **no
historical band lines appear on a 3M chart**. Only the projected polyline, the
labels and the status table show (05-integration § handoff). The tau cap makes a
3M bar's band equal the status cell's quarter-end values; this has not been
observed on a chart. A visual seam to expect on every chart: the last plotted
band point sits at the last bar's open x with the value at its close, while the
projected polyline starts at the last bar's `time_close`, so there is a
one-bar horizontal gap (one week on a weekly chart).

**Status cell** (top right, always present). Either the fan summary (σQ percent, L
and the quarter window used, A, and the quarter-end ±1σ / ±2σ levels), or
"n/a — reason". Reasons as they exist now:

| Code | Cell / log text | Cause |
|---|---|---|
| 1 | need L completed quarters, have n | fewer than L post-inception 3M bars |
| 2 | quarterly data not aligned | neither 3M set carries quarter id q-1 |
| 3 | a quarter is missing inside the sigma window | hole in the 3M history |
| 4 | invalid high/low in the sigma window | H or L na, <= 0, or H < L |
| 5 | zero range | window of identical highs and lows |
| 6 | anchor open missing | first-session open unknown, including the stale-carry rule |
| — | quarter start not in loaded history | no start detected in the loaded bars |
| — | unsupported chart (needs a standard time-based chart, timeframe 3M or shorter) | see § 5 |

**There is no code 7.** It existed in the first version as a guard that forced
n/a on a weekly start bar spanning the previous quarter; it was removed when the
latch was rebuilt (07). Any text listing "quarter-start bar spans the previous
quarter" is stale.

**Sigma latch (what makes the weekly fix work).** `f_htf` runs in the 3M context
and returns the 3M bar's quarter id, mean log range, validity code and bar index,
plus the same four values as they were on the previous 3M bar. On the quarter's
first chart bar itself (chart [0], not one bar back) the script takes the set
whose id is q-1: the current set first (history on every timeframe), the lagged
set second (a 3M chart, or realtime, where the 3M bar for q is already developing),
else code 2. It applies to all timeframes (D-6). Every q-1 session has ended
before a start can be detected, so a q-1 set is always the completed one. This
stands on TradingView's documented `lookahead_off` merge, which no lane observed
directly; if the merge differs, the result is n/a code 2, not a wrong value.

**Pine Logs** (`i_log`, default on; Pine Logs panel, More menu). Format (10-backend
§ handoff):

```
2026Q4 quarter start: <date> | A <price> | 3M set: current|lagged|none | σQ x.xx% (L=8, 2024Q4 to 2026Q3) | quarter end: +1σ .. -1σ .. +2σ .. -2σ ..
```

- `<date>` is `first session yyyy-MM-dd (chart bar yyyy-MM-dd)` on charts longer
  than a day, `start bar yyyy-MM-dd HH:mm` otherwise.
- For an n/a quarter the info line carries only the head (date, A, 3M set), then a
  separate warning `2026Q4 n/a — <reason>`, using the same reason text as the
  status cell.
- Unsupported chart: one error with the status-cell text, on the first bar.
- No quarter start in loaded history: one warning.
- A guard (`varip`) keyed on quarter and outcome stops repeats on realtime ticks.
  A change of outcome mid-bar logs again by design. A reload of a live chart
  replays history and logs again. Old entries can scroll out of the panel, and
  the toggle is all-or-nothing.
- Observed (compile.md): weekly SPX shows one info line per quarter 2021Q4 to
  2026Q4 with no repeats on live ticks. The `3M set:` field and n/a warnings were
  not individually reported.

## 4. Sources

- **Range constant.** Feller, W. (1951), "The asymptotic distribution of the
  range of sums of independent random variables", *Annals of Mathematical
  Statistics* 22(3), 427-432: expected range of a Brownian **path** over unit time
  is `sqrt(8/pi) * sigma` (a path, not a normal sample).
- **Range-based sigma.** Parkinson, M. (1980), "The Extreme Value Method for
  Estimating the Variance of the Rate of Return", *Journal of Business* 53(1),
  61-65, first moment `E[ln(H/L)] = sqrt(8/pi) * sigma` on log prices. His
  published estimator uses the second moment; this script uses the first-moment
  form, a **declared judgement**: unbiased for sigma rather than sigma^2 and less
  sensitive to one extreme quarter. With L = 8 the simulated relative SD of sigma
  is 10.5% for the range estimator against 25.7% for the sample SD of eight
  quarterly returns (02).
- **sqrt(t) scaling of a log-normal price.** Textbook, e.g. Hull, *Options,
  Futures and Other Derivatives*, lognormal property of stock prices.
- **The fan as a whole is an ORIGINAL construction for this trader.** The
  combination of Parkinson-style quarterly sigma, first-session-open anchor and
  calendar-time sqrt widening is not a published indicator and has no external
  authority.

The estimator was changed from dollar range `H - L` to log range `ln(H/L)` in this
run: the dollar form averages ranges at different price levels and applies the
mean to today's price, which on a stock trending 50 to 100 with constant 8%
quarterly log sigma comes out 31% too narrow (02 § Source). The constant itself
was checked (`2 * 0.7979 = 1.5958` against `sqrt(8/pi) = 1.59577`, 1.9e-5
relative; Brownian simulation with the Broadie-Glasserman-Kou discrete-monitoring
correction is consistent, 04).

## 5. Known limits

From the header's admissions, expanded.

- **Assumptions.** Driftless Brownian motion in log price, constant volatility
  across the window and the projected quarter. Real tails are fatter, so the ±1σ
  and ±2σ bands will **not** contain 68.3% and 95.4% of outcomes. Falsification
  (02): over many quarters and liquid ETFs, quarter-end closes inside ±1σ (±2σ)
  should be near 68% (95%), lower with fat tails; far above means sigma is
  overstated, far below means understated or a units or stale-window error.
- **Sigma bias.** H/L miss overnight extremes, so sigma is biased low by roughly
  1-3% (simulation, -1.1% to -1.7% across overnight variance shares of 0-30%,
  -1.4% at 5-minute monitoring). Drift biases it slightly high.
- **Calendar time, not trading time.** Holidays are ignored. Against a
  trading-time tau, sqrt(tau) differs by up to about **0.07** in units of k*sigma,
  either sign (04 F2 measured -0.041 to +0.074, largest on the first session after
  a quarter that starts on a holiday Friday; approximate NYSE holidays
  2005-2029). The earlier header figure, 0.034, was understated.
- **Anchor.** First session's open. With extended hours shown on an intraday chart
  it is the first pre-market print, which can differ slightly from the official
  daily open. On a weekly chart the start bar's OHLC contains prior-quarter days,
  but the anchor is the first daily session's open, not the bar's.
- **Display cutoff.** "Historical Quarters Shown" drops the oldest fan when a new
  quarter starts. A display change, not a value change; it is tied to
  `last_bar_time`, so on a weekly chart whose last bar contains a quarter start,
  switching "Show Historical Fans" off can also show the previous quarter's fan
  (03 risks).
- **Unsupported (n/a, nothing computed):** timeframe above 3M, and non-standard
  chart types: Renko, Kagi, Line Break, Point and Figure, Range, Heikin Ashi, tick
  charts. Heikin Ashi is excluded because its open is synthetic and so would be
  the anchor (an addition beyond the quant spec, accepted in 04).
- **Inception quarter excluded**, and L consecutive quarters required.
- **Carried SHOULD_FIX, not fixed (08 F1, MINOR; backend order owed).** The
  stale-carry rule makes the anchor n/a (code 6) for **any** quarter start on a
  bar that follows a bar whose daily array was empty, including a start at
  `idx_b > 0` where the bar's own data already proves which element is the first
  session. Example: empty daily array for the week of 2027-03-22, then week B
  (`idx_b 3`) shows n/a although A = 100.00 is in B's data. A correct fan is
  withheld; no wrong number is shown. 08 proposed the narrower rule
  `a_b := (i < n_o and not (stale and i == 0)) ? array.get(d_o, i) : na`.
  Reachable only after a lower-timeframe data hole.
- **A data hole inside a non-empty weekly daily array** (first session of the
  quarter missing, later ones present) anchors on the next session's open, the
  same as a missing daily bar on the daily chart, and is indistinguishable from a
  holiday (08 risks).
- **Futures and overnight symbols:** the 3M bar is labelled by `time` in exchange
  timezone; if a 3M bar's time is the prior evening, ids could shift. Pre-existing,
  outside the ETF and stock scope, not changed (08 risks).
- **Vendor corrections.** A post-close correction to a 3M high or low would differ
  between a live latch and a reload; inherent, not this script's (06 risks).
- **The worked examples are the quant lane's own constructed inputs.** The
  formula is anchored externally (Feller, Parkinson, sqrt-t scaling); the window
  and anchor rules only by decisions D-1 to D-6.

### Verification status

Static check: `pine_check.py range-projection.pine --strict` exits 0 with no
findings. The repo-wide `--strict` run exits 1 on seven advisories in
`strategy13_trend_carry.pine` and the systemic-stress script, which this run did
not touch (05-integration).

Compile check (`compile.md`): run by the user in TradingView on 2026-10-05, not by
an agent: daily and weekly SPX plus the Pine Logs panel. Observed: the script
loads and runs; daily showed no defect; the first version's weekly chart drew fans
only on 2022Q1 to 2024Q3 with gaps, which led to the latch rebuild; after the
rebuild the user reported "it looks fine" for weekly and daily; after logging was
added the user confirmed it compiles with no error and that the weekly SPX Pine
Logs shows one info line per quarter 2021Q4 to 2026Q4 with no live-tick repeats.
Error panel text, exact symbol detail and screenshots were not captured.

**Not verified** (nothing in `compile.md` covers these):
- The latch selection item by item, including that no "quarterly data not
  aligned" appears; the user's acceptance was an overall judgement.
- Which `time_close` convention a weekly bar uses; `time_tradingday` semantics on
  pre-market bars; that `xloc.bar_time` points beyond the last bar render at the
  expected place; that `bq == f_q` is false when `bq` is na.
- Intraday charts (anchor on a 5-minute chart; whether `request.security` on "3M"
  from intraday supplies history older than the chart's first bar, else sigma is
  n/a), a 3M chart, and "Historical Quarters Shown" above 20 (daily-data depth on
  weekly; 06 expects back to 2014Q2 at 50, unobserved).
- No hand-computed ETF quarter has been compared with the status cell.
- Realtime on a live spanning week (next: Thu 2027-04-01); live-versus-reload
  equality is by design only.
- Constructs the backend lanes were unsure of that compiled by implication
  (loads, no error) but were not individually reported: `na(array)`, history
  references inside `f_htf`, polyline fills, `varip` at top level.

## 6. Reproducing the fixture

**There is no fixture for this script.** It is an indicator, not a strategy, and
`fixtures/*.csv` is a Strategy Tester convention. The nearest evidence is the
hand-computed worked examples in § 1, independently re-derived in
`04-quant-audit.md` (literal Python port of the script, all 12 band values) and
`08-quant-audit.md` (weekly rows under both `time_close` conventions, 21 modelled
SPX weekly fans), and the user's compile check above.

To compare a chart against the worked examples: weekly or daily SPX, Lookback 8,
Project Quarters Ahead 1 (default), Historical Quarters Shown 20 (default), "Log
to Pine Logs" on; the status cell's σQ for 2026Q4 should equal a hand value of
`mean(ln(H/L)) / 1.5957691` over the 3M bars 2024Q4 to 2026Q3, and the weekly and
daily status cells should agree exactly.
