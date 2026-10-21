# `mm_cot_long_short_index.pine`

**This documents the net-index redesign done in run
`2026-09-17-fix-mm-cot-long-short-index`.** The script was rewritten below the
header to compute a single net (long − short) COT Index on 0..100, replacing
the two independent gross-long / gross-short indices on -100..+100 that the
prior audit (`2026-09-16-audit-mm-cot-long-short-index`) found broken.

As of this dispatch (after this run's third round, which confirmed the
symbol defaults against TradingView's own live symbol search): **all eight of
the prior audit's findings are now resolved** — six fixed this run, two
dissolved outright by the net-index redesign (the thing they described no
longer exists), and prior FINDING 6 (the reconstructed CFTC symbols) moved
from "reasoned but unconfirmed" to **confirmed**, via TradingView's own live
symbol search on 2026-09-17, this round. See § Known limits for the
finding-by-finding table. This is a change from this run's first round, where
prior FINDING 1 was only partially resolved (correct at or below a weekly
chart timeframe, silently wrong above it); CR-1's fix (degrade to a
visibly-labelled `n/a` above weekly) completes it, so it now counts as fixed
rather than open — an intentional degrade-on-coarse-timeframe behaviour is a
fix, not a defect.

**The TradingView compile check ran on 2026-09-17 and PASSED.** The script
compiles, starts with zero console errors, and computed a live reading of
65.80 in the test session. See § Still open after this run and
`compile.md` of run `2026-09-17-fix-mm-cot-long-short-index` (pruned; in `.agentic` git history)
for the full record.

**Both gates this run judged have PASSED, on 2026-09-17.** The quant-audit
gate (`08-quant-audit.md`) independently re-derived the net COT Index formula
twice — once from the cited Briese/Williams definition directly, once by
emulating the shipped code bar-by-bar across eight chart timeframes — and both
derivations agree with the shipped script wherever it publishes a number. The
integration gate (`09-integration.md`) confirmed the script is statically
clean (`pine_check.py --strict`, 0 findings) and internally coherent: the
header, the inputs, the plot/hline contract and the on-chart table all match
the specification and each other, with no dead code left from the prior
gross-long/gross-short design. **Neither gate is, or claims to be, the
TradingView compile check** — that check ran separately, in the main
session, and PASSED on 2026-09-17; see § Still open after this run.

## Still open after this run

**Symbol reconstruction (prior FINDING 6 / guardrail 3) is RESOLVED, not
open.** The defaults `088691_F_MMP_L` / `088691_F_MMP_S` — bare tickers, no
exchange prefix (`COT2:` and `COT:` are both invalid here) and no
`_ALL`/`_OLD`/`_OTHER` suffix (the aggregate "All" category is the bare
ticker; only the OLD/OTHER crop-year sub-breakouts take an explicit suffix,
and Gold has no crop year) — were confirmed via TradingView's own live symbol
search on 2026-09-17, and separately by pasting both bare strings directly
into the Pine editor with zero compile/runtime errors. This is a genuine
confirmation, not another reasoned guess; guardrail 3 is now satisfied for
this script. See the header's "SYMBOLS — CONFIRMED, GUARDRAIL 3 SATISFIED"
block and § Known limits below.

**The TradingView compile check ran on 2026-09-17 and PASSED.** The
orchestrator (main session, the only place in this network with browser
tools) pasted the final script into TradingView's Pine Editor: it compiled
(reached the "start" stage), started with zero console errors, and computed
a live reading — the chart legend showed `MM COT Long/Short Index  65.80`,
a real, non-`n/a`, non-error value — with the final bare symbol defaults.
Confirmed twice, including once after re-selecting both symbols through
TradingView's own "Change symbol" picker. Full method and result:
`compile.md` of run `2026-09-17-fix-mm-cot-long-short-index` (pruned; in `.agentic` git history).

**What the PASS does not confirm** — from `compile.md` § What this does NOT
confirm, and not to be over-read as "every code path proven":

- Behaviour on chart timeframes above weekly (the `tfTooCoarse` degrade path,
  CR-1's fix) was not separately re-exercised live in this session — that
  path is verified only by `08-quant-audit.md`'s independent hand-derivation,
  not by this compile check.
- Long-run accumulation of the weekly window over real historical bars was
  not observed — a single "add to chart" moment cannot show that, and it
  would need a longer loaded history or more time to exercise.

One item remains for a human decision; it was not fixed by this run, and is
not this doc's or this run's to resolve:

1. **A MINOR, non-blocking residual coarse-timeframe leak** (08-quant-audit.md
   § FINDING 8): a 6D or 7D chart timeframe slips under the script's
   `timeframe.in_seconds() > timeframe.in_seconds("W")` guard and silently
   skips weekly reports (emulated 72.65 / 76.84 against a true 61.42 on the
   auditor's series), rather than degrading to the labelled `n/a` that every
   coarser timeframe already gets. Audited, not fixed — see
   `08-quant-audit.md` § FINDING 8 — residual coarse-timeframe leak.

Two further pre-existing items surfaced by the integration gate belong to the
other two scripts in this repo, not to this one:
`strategy13_trend_carry.pine` (missing declared slippage, guardrail 4; missing
explicit `lookahead`) and `systemic_stress_dashboard.pine` (missing explicit
`lookahead` on four calls). Neither file was touched by this run; see
`09-integration.md` § Repo-wide static check for the human to consider as a
separate follow-up.

## What it computes

The Managed Money **net COT Index**: `net = mmLongPos - mmShortPos` (positive
= net long, negative = net short), rescaled to **0..100** by ranking the
current net position against its own range over the last `lookbackWeeks`
**weekly COT reports**.

- **100** — Managed Money is at the most net long it has been in the window.
- **50** — the middle of the window's range.
- **0** — the most net short it has been in the window.
- **80 / 20** are the conventional "crowded long" / "crowded short" bands.

This describes **positioning**, not a trade signal — the contrarian reading of
a crowded extreme is left to the reader. Gross long and gross short are still
computed and shown, but only as raw numbers in the on-chart table; neither is
indexed on its own anymore (`net` in § ALIGNMENT AND NET, `cotIndex` in
§ INDEX — section names per the script's own banner comments, which is how
this doc cites the code from here on; see § Known limits →
§ Second-round audit findings, FINDING 2, for why).

The rolling window (`netHist`, § WEEKLY WINDOW) is built from **weekly
observations**, not chart bars: it pushes exactly once per new weekly COT
report (gated on `barstate.isconfirmed`). **This holds — and `lookbackWeeks`
means the same span of time — only on chart timeframes at or below "W"
(weekly).** Above a weekly chart timeframe (e.g. monthly),
`request.security(sym, "W", ...)` delivers one value per *chart* bar rather
than per weekly report, so collecting into the window there would silently mix
several weeks into one observation. Instead, on any chart timeframe above "W"
the script suppresses collection and forces both the index and the window
table row to `"n/a (chart TF > W)"` — a visible, labelled degrade rather than
a plausible-looking wrong number. This two-part behaviour (invariant at/below
weekly, visibly `n/a` above it) is this run's second-round fix for the prior
audit's FINDING 1 — see § Known limits and `cr/CR-1.md`.

The current (still-open) week's reading is **provisional**: by design it is
not added to the window until its bar confirms, so the live bar ranks against
`lookbackWeeks + 1` observations until that bar closes, and can move again
before it does. This does not rewrite any already-closed bar's value.

The default symbols target Gold (CFTC code `088691`); both symbol inputs are
user-editable (`input.symbol`).

## Inputs

| Name | Type | Default | Controls | Timeframe-dependent? |
|---|---|---|---|---|
| `mmLongSymbol` | `input.symbol` | `088691_F_MMP_L` | Managed Money gross long position series | No |
| `mmShortSymbol` | `input.symbol` | `088691_F_MMP_S` | Managed Money gross short position series | No |
| `staleAfterDays` | `input.int` | `21` | Days after which the "Data as of" row is flagged `— STALE`; `minval = 8` | Not in count, but its *basis* is: the age is measured from the weekly series bar's OPEN time, which is the current week on a chart at/above weekly and the last closed week on a sub-weekly chart, so the same missing report is flagged STALE about a week later on a weekly chart than on a daily one — see § Known limits → § Second-round audit findings, FINDING 7. |
| `lookbackWeeks` | `input.int` | `156` | Rolling window, **in weekly COT reports**, the current net position is ranked against; `minval = 2` | No — invariant — for chart timeframes **at or below "W"**. Above "W" the index and window both read `n/a (chart TF > W)` rather than silently counting chart bars instead of weeks — see § Known limits, prior FINDING 1. Fixed at 156 (≈3 years); 26/52/260 also in common use, per the tooltip. |

`mmLongSymbol` / `mmShortSymbol` changed from `COT2:088691_F_MMP_L_ALL` /
`COT2:088691_F_MMP_S_ALL` to the bare, confirmed `088691_F_MMP_L` /
`088691_F_MMP_S` across this run (see prior FINDING 6 below — now resolved);
`staleAfterDays` is new this run; `lookbackWeeks` gained `minval = 2` in this
run's second round (0 or 1 left the index permanently `na` with no rejection
at input time).

## Plots and outputs

- `cotIndex` — one blue line, the net COT Index, 0..100 (§ INDEX).
- `hline(80)` — "Crowded long (80)", dashed green.
- `hline(50)` — "Midpoint (50)", dotted gray.
- `hline(20)` — "Crowded short (20)", dashed red.
- On-chart table (top right, 2 columns × 6 rows, § TABLE):
  1. `MM Long Position` — raw gross long contracts.
  2. `MM Short Position` — raw gross short contracts.
  3. `MM Net Position` — signed `net`.
  4. `Net COT Index (0-100)` — `cotIndex`, background tinted green when
     `>= 80`, red when `<= 20`; text colour is white **only** on that same
     tinted branch (fixed this run's second round — it used to be
     unconditional white, unreadable on a light chart theme in the untinted
     mid-range, which is most readings — see § Known limits →
     § Second-round audit findings, FINDING 4).
  5. `Window` — `<collected>/<lookbackWeeks> weeks` while the chart timeframe
     is at or below "W"; `n/a (chart TF > W)` above it (see § Known limits,
     prior FINDING 1). Below "W" it is always a number, never a bare `n/a` —
     this is how a reader tells "still collecting history" apart from
     "symbol did not resolve."
  6. `Data as of (series bar)` — the date of the weekly series bar the reading
     came from, `— STALE` appended past `staleAfterDays`.
- **`n/a` rendering**, and what each case means:
  - A symbol did not resolve → that row's raw position is `n/a`; `net` and
    `cotIndex` are also `n/a`.
  - The two legs report different weekly bars (`legsAligned` false) → `net`
    and `cotIndex` are `n/a`; row 6 shows `n/a (legs misaligned)`, distinct
    from a plain `n/a` (symbol failure) — see prior FINDING 8 below.
  - Fewer than `lookbackWeeks` weekly observations collected → `cotIndex` is
    `n/a`; row 5 shows the count so far (e.g. `9/156 weeks`).
  - The chart timeframe is above "W" (weekly) → `cotIndex` and the `Window`
    row both read `n/a (chart TF > W)`, distinct from every other `n/a` case
    above — this is CR-1's fix, this run's second round; see § Known limits,
    prior FINDING 1.
  - The net position was constant across the whole window (`hi == lo`) →
    `cotIndex` is `n/a`, **never `50`** — a degenerate range has no meaningful
    "position within range" to report.
- **The current (still-open) week's reading is provisional**, not an `n/a`
  case: by design (`01-quant.md` § Formula — specification, step 4) the
  in-progress week is not added to `netHist` until its bar confirms, so the
  live-bar value ranks against `lookbackWeeks + 1` observations and can move
  again before that bar closes. See § Known limits → § Second-round audit
  findings, FINDING 6. No already-closed bar's plotted value changes after
  the fact.

## Sources

The header now cites, at book/publisher/year granularity (no chapter — neither
book was opened during this run, and an invented chapter would be a fabricated
citation):

- Stephen Briese, *The Commitments of Traders Bible*, Wiley, 2008.
- Larry Williams, *Trade Stocks & Commodities with the Insiders: Secrets of the
  COT Report*, Wiley, 2005.

Both define the COT Index as a stochastic rescale of a trader category's
**net** position over a rolling window onto **0..100**, with 80/20 as the
conventional extremes — this script now follows that definition on the net
Managed Money position, resolving the prior audit's FINDING 5 (the header
previously named no source) and prior FINDING 3 (the script previously
computed a gross-long/gross-short variant, not this definition).

**Two departures are declared in the header, and stay declared:**

- Briese and Williams work primarily from the legacy report's Commercial and
  Large Speculator categories; this applies the same index to **Managed
  Money** from the **Disaggregated** report — the closest modern analogue, not
  the same series, and the 80/20 thresholds were not calibrated on it.
- `lookbackWeeks = 156` (3 years) is a convention choice, not a number either
  author derives; 26/52/260 weeks are equally in use.

## Known limits

The header's former unverified-symbol admission has been replaced, not
weakened: the symbols are now CONFIRMED (bare `088691_F_MMP_L` /
`088691_F_MMP_S`, no prefix, no `_ALL`/`_OLD`/`_OTHER` suffix), via
TradingView's own live symbol search on 2026-09-17, and the header states
what was verified and how (see prior FINDING 6 below).

**Numbering note.** This section names findings from three different audits,
and they do not share a numbering scheme. "Prior FINDING N" below is always
the 2026-09-16 audit's eight numbered findings. "Second-round audit FINDING
N", in its own subsection further down, is this run's own first audit
(`05-quant-audit.md`, 2026-09-17) re-checking the prior eight and raising six
of its own. A third audit (`08-quant-audit.md`, 2026-09-17, PASS) re-verified
CR-1 and all six second-round findings and raised one new one of its own,
labelled "FINDING 8" in that report and referred to here as "08-quant-audit's
FINDING 8" — see § Still open after this run. Read each by the label given,
not by the bare number.

The prior audit found eight numbered findings. Resolution status per finding,
as of this dispatch (after this run's third round):

| Finding | Status | Note |
|---|---|---|
| **Prior FINDING 1** — window counted chart bars, not weeks | **Fixed this run** | `netHist` (§ WEEKLY WINDOW) is an array of confirmed weekly observations, pushed once per weekly COT report and gated on `barstate.isconfirmed`. `ta.lowest`/`ta.highest` are removed entirely. This run's first round left the fix correct only at or below a weekly chart timeframe and silently wrong above it (a monthly chart mixed ~4 weeks per push); this run's second round (CR-1) closed that gap by detecting `timeframe.in_seconds() > timeframe.in_seconds("W")` and forcing the index and window to a visibly-labelled `n/a (chart TF > W)` instead. The falsification test — weekly chart vs. a sub-weekly chart of the same instrument, same date, must be identical — holds; above weekly, both now read the labelled `n/a` rather than a silently wrong number. |
| **Prior FINDING 2** — the +80/-80 guide labels were inverted for the short plot | **Dissolved by the redesign** | There is now one signed series, so "80 = crowded long" is unambiguous; the second plot the inverted reading depended on no longer exists. |
| **Prior FINDING 3** — index applied to gross long/short separately, not net | **Fixed this run (decision D-1)** | `net = mmLongPos - mmShortPos` (§ ALIGNMENT AND NET), rescaled 0..100, replaces the two per-side indices. Gross long/short survive as raw table rows only. |
| **Prior FINDING 4** — ±80 was the 90/10 extreme, not the conventional 80/20 | **Dissolved by the redesign** | The scale is now 0..100, so the shipped 80/20 guide lines are the cited authors' actual convention, not a rescaled approximation. |
| **Prior FINDING 5** — "several COT analysts" named no source | **Fixed this run** | The header now cites Briese (2008) and Williams (2005) at book/publisher/year granularity, with the two departures (Managed Money vs. legacy categories; 156 weeks as convention) stated explicitly. |
| **Prior FINDING 6** — the `_OLD` suffix does not apply to Gold | **Resolved this run (round 3)** | Defaults are now the bare, confirmed `088691_F_MMP_L` / `088691_F_MMP_S` — no exchange prefix (`COT2:`/`COT:` are both invalid) and no `_ALL`/`_OLD`/`_OTHER` suffix (the aggregate "All" category is the bare ticker; the OLD/OTHER crop-year sub-breakouts don't apply to Gold). Confirmed via TradingView's own live symbol search on 2026-09-17, and separately by pasting both bare strings into the Pine editor with zero compile/runtime errors. Guardrail 3 (symbols verified, never reconstructed) is now satisfied for this script. |
| **Prior FINDING 7** — no explicit `lookahead=` on `request.security()` | **Fixed this run** | Both calls now pass `lookahead = barmerge.lookahead_off` explicitly (§ DATA), which is also what clears `pine_check.py --strict`'s two prior `P008` hits. |
| **Prior FINDING 8** — no as-of date, no staleness test | **Fixed this run** | The table's row 6 shows the weekly series bar's date and appends `— STALE` past the new `staleAfterDays` input (default 21, justified against CFTC's Tuesday-as-of / Friday-publish cycle). A misaligned-legs case is now distinguishable from a plain unresolved symbol (`n/a (legs misaligned)` vs. `n/a`). |

**As of this dispatch: six of the prior audit's eight findings are fixed,
two were dissolved by the redesign, and none remain open** — prior FINDING 6
/ guardrail 3 is now resolved (round 3), confirmed against TradingView's own
live symbol search on 2026-09-17. Prior FINDING 1 moved from "partially
resolved" (first round) to "fixed" (second round, CR-1) — the
chart-TF-above-weekly case is now an intentional, visibly-labelled degrade,
which is a fix, not an open defect, and it is not counted among any "open"
total above.

**The TradingView compile check ran on 2026-09-17 and PASSED**, separately
from finding status above. `pine_check.py --strict` passing (`02-backend.md`,
re-run independently in `05-quant-audit.md`) is evidence about the *text* —
no version mismatch, no repaint pattern, no missing `lookahead`, no
unbalanced parens — and on its own would not be evidence the script loads,
plots or resolves any symbol. The compile check closes that gap: the script
was pasted into TradingView's editor, compiled, started with zero console
errors, and computed a live reading of 65.80 with its final symbol defaults.
See § Still open after this run and `compile.md` (path above) for the full
record and its two "what this does not confirm" caveats. This doc's claim is
now: the static check passed, an independent hand-derivation of the formula
agrees with what the code implements, **and** the script compiles and
computes in TradingView itself.

The unresolved bar-stamping question carried from the prior audit — whether
TradingView stamps the weekly bar at CFTC's Tuesday as-of date or the Friday
release date — is also still open. The table's row label says "Data as
of **(series bar)**", deliberately not "CFTC as-of date," because that
question was never settled.

### Second-round audit findings (`05-quant-audit.md`, 2026-09-17)

This run's second round dispatched a fresh audit against the first round's
code and docs. Its findings, by its own numbering (not the prior-audit table
above):

| Finding | Owner | Status |
|---|---|---|
| **FINDING 1** — prior FINDING 1 only partially resolved (chart TF > "W" branch still wrong) | backend | **Fixed** — CR-1, remedy (a); see prior FINDING 1 row above. |
| **FINDING 2** — this doc's `file:line` citations were 26-34 lines short of the current file | docs | **Fixed this dispatch** — citations re-taken as section names (the script's own `// ===== ... =====` banner comments), which do not drift the way line numbers do. |
| **FINDING 3** — this doc's summary line ("six of eight... two remain open") contradicted its own table | docs | **Fixed this dispatch** — see the top-of-file summary and the "As of this dispatch" line above: five fixed, two dissolved, one open. |
| **FINDING 4** — `text_color = color.white` on the index cell was unconditional, unreadable on a light theme outside the tinted branch | backend | **Fixed** — now fires only alongside the `bgcolor` tint. |
| **FINDING 5** — `lookbackWeeks` had no `minval`, so 0 or 1 silently produced a permanently-`na` index | backend | **Fixed** — `minval = 2`. |
| **FINDING 6** — the header's "WHAT DEGRADES" section didn't disclose that the current week's reading is provisional until its bar confirms | docs | **Fixed this dispatch** — one line added to the header's "WHAT DEGRADES, AND HOW YOU SEE IT" section and mirrored in § Plots and outputs above. |
| **FINDING 7** — the staleness tooltip's "~13 days" justification holds only on sub-weekly charts; on a weekly chart the same basis runs about a week younger | docs | **Fixed this dispatch** — the header's staleness bullet and this doc's `staleAfterDays` Inputs-table row both now state that the age is measured from the weekly bar's open time and that basis shifts by about a week depending on chart timeframe. |

## Reproducing the fixture

No `fixtures/` directory or CSV exists for this script — `mm_cot_long_short_index.pine`
is an `indicator()`, not a `strategy()`, and this run created no fixture.

Independent worked examples exist as hand computations, not executed tests
(there is no Pine test runner in this project):

- `01-quant.md` of run `2026-09-17-fix-mm-cot-long-short-index` (pruned; in `.agentic` git history)
  § Worked examples — two series (5-week and 12-week) hand-traced against the
  formula, including the degenerate `hi == lo` case.
- `03-test.md` of run `2026-09-17-fix-mm-cot-long-short-index` (pruned; in `.agentic` git history)
  § Worked example — an independently constructed 12-week, `lookbackWeeks = 4`
  series exercising window-not-full, first-full-bar, a legs-misaligned bar, a
  symbol-unresolved bar, a constant-net `hi == lo` stretch, and recovery, with
  every non-`na` value and the reasoning for every `na` traced by hand.

Both agree with each other and with the formula in
`01-quant.md` § Formula — specification; neither was produced by loading the
script on a chart. The TradingView compile check (§ Still open after this
run) covers that gap separately, and it PASSED on 2026-09-17 — see
`compile.md` (path above).
