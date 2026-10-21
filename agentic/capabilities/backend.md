# Capability pack: backend — project `tradingview`

For the `backend-engineer` lane. Here that means **the `.pine` source** — the
whole artifact, inputs and plots and tables included. Where this contradicts
general programming instincts, this file wins; Pine is not the language your
instincts were trained on.

Companion sources: `tradingview/CLAUDE.md`, the script headers, the TradingView
Pine v6 reference.

---

## Index

Read this block first. You are not expected to read this file end to end — read
what your order touches. Reading one extra section is cheap; acting on a
convention you never read is not.

**Always read:** **Your verification command** · **Guardrails, in Pine terms** · **Gotchas that will bite you** · **Definition of done for this lane**

| Section | Read it when |
|---|---|
| Layout and naming | you are adding a script |
| The shape of a script here | you are writing one from scratch |
| Inputs | your order adds or changes a user-facing parameter |
| Symbols | your script reads any instrument other than the chart's |
| Contract notes you must emit | your order changes an input, a plot or an alert |

---

## Your verification command

```bash
python <agenticRoot>/scripts/pine_check.py <the file you changed> --strict
```

Run it. `status: DONE` requires `verification.result: PASS`, and here that means
this command exited 0.

**You cannot compile.** TradingView is the only Pine compiler and no lane in
this network holds browser tools — `project.md` § "Why this project is unusual"
has the full shape. So:

- Never write "compiles cleanly", "tested on a chart", or "verified in
  TradingView" in your report. You did none of those things.
- Anything you are not certain compiles goes in `risks`, specifically: the
  construct, and why you are unsure.
- The static check passing is evidence about the text, not about the script.
  Report it as what it is.

If `--strict` flags something you genuinely should not fix — a
`request.security()` where the explicit `lookahead` would be noise in context —
do not drop the flag. Fix it or report it in `risks` and set `PARTIAL`.

## Layout and naming

Flat. One script per file at the repo root, `<slug>.pine`, lowercase with
underscores. Its doc is `docs/<slug>.md` with the underscores turned into
hyphens (`volume_profile.pine` → `docs/volume-profile.md`), and its fixture,
where one exists, is `fixtures/<slug>-*.csv`.

The `indicator()` / `strategy()` title is the name the user sees in TradingView
and does **not** have to match the slug — it is prose, and the existing files
use prose titles ("Trend + Carry, Risk-Regime Sized (Strategy 13)").

## The shape of a script here

In order, every time:

1. **Licence line and `© antonametov88`.** Copy the two lines from any existing
   script verbatim.
2. **The header comment block** — what it computes, its source, and what was not
   verified. `project.md` § "The script header contract" is binding on this, and
   it is the part most likely to be skipped.
3. `//@version=6`
4. The declaration — `indicator(...)` or `strategy(...)`, wrapped across lines
   when it has more than three arguments.
5. **Inputs**, grouped, under an `// ===== INPUTS =====` banner.
6. **Computation**, in the order a reader would derive it.
7. **Plots, tables and labels.**
8. **Alerts**, last.

Follow the existing files' banner-comment style (`// ===== SECTION =====`).
Consistency with what is there beats any convention this file could invent.

## Guardrails, in Pine terms

| Guardrail | What it means in code |
|---|---|
| 1 — no repainting | `request.security(..., lookahead=barmerge.lookahead_off)`, always explicit. Guard alerts with `barstate.isconfirmed`. Never index a series forward. If a construct is knowingly live-updating, say so in the header |
| 2 — traceability | the header names the source; parameter defaults that came from the source are commented with where |
| 3 — symbols verified | see "Symbols" below — this one has real code consequences |
| 4 — costs declared | `strategy()` passes `commission_type`, `commission_value` and `slippage`. `pine_check.py` blocks on the first and advises on the third |
| 5 — degrade visibly | `na` propagates to the display as `n/a`. Never `nz(x)` a missing input into 0 where 0 is a meaningful value — that is fabrication with extra steps |
| 6 — no execution | no webhook-shaped alert messages, no broker integration, no `strategy.entry` in an indicator |

## Inputs

- **Typed constructors only** — `input.int`, `input.float`, `input.bool`,
  `input.source`, `input.symbol`, `input.timeframe`, `input.color`. Bare
  `input()` is v4 style.
- **Every input gets a `tooltip`** when its correct value depends on the
  timeframe, the instrument, or a judgement the user has to make. A lookback of
  20 means something different on a 3-minute chart; the tooltip is where that is
  said.
- **`group=` every input** into the banner section it belongs to, so the
  settings dialog has the same structure as the file.
- **`minval` / `maxval` on every numeric input.** A negative lookback is a
  runtime error, and the settings dialog is the cheapest place to prevent it.
- **Changing an existing input's name, type or default is a contract change.**
  It silently resets that setting on every chart the user already has the script
  on. Emit a contract note; do not do it opportunistically.

## Symbols

Guardrail 3, and the place this repo has already been bitten —
`mm_cot_long_short_index.pine` shipped with CFTC tickers reconstructed from
search results, and carried that warning in its header until a later run
confirmed them against TradingView's own symbol search (2026-09-17).

- **A symbol you did not verify is an `input.symbol()`**, with the reconstructed
  value as the default, so the user can correct it without editing code.
- **Handle non-resolution explicitly.** A symbol that does not resolve returns
  `na`; render that as a visible `n/a` in the table or plot, never as 0, never
  as the last good value. The COT script does this, and it is why its tickers
  were survivable while they were still unverified.
- **Never pattern-match a new ticker from an existing one.** CFTC codes, Cboe
  index tickers and exchange prefixes do not compose the way they appear to.
  If your order needs a symbol nobody confirmed, that is a `risks` bullet and a
  header warning, not a guess.

## Gotchas that will bite you

- **`var` initialises once, not every bar.** A `var` you meant as a local is a
  value that persists across the whole history and produces a plot that is right
  at the last bar and wrong everywhere else.
- **`:=` reassigns, `=` declares.** Declaring inside an `if` shadows rather than
  assigns, and the outer value stays at its initial state. This is the most
  common silent logic bug in Pine.
- **History reference on a series that did not exist yet** returns `na`, and
  arithmetic with `na` is `na` — one missing bar poisons the rest of an
  expression. `nz()` is the tool, but see guardrail 5 before reaching for it.
- **`request.security_lower_tf()` takes no `lookahead` argument.** Do not invent
  one to satisfy a checker; `pine_check.py` does not ask for it.
- **`max_bars_back`.** A series referenced further back than Pine inferred throws
  at runtime, not at compile. If you index a series by a variable amount, set
  `max_bars_back` on the declaration.
- **Plot count is capped at 64**, and `plot()` inside a conditional is not
  allowed at all — plots are declared at the top level and conditioned by
  plotting `na`.
- **Tables are redrawn every bar unless guarded.** Wrap table updates in
  `if barstate.islast` or the script crawls on a long history.
- **A `strategy()` default of `calc_on_every_tick=true`** makes the backtest and
  the live chart disagree. `pine_check.py` flags it; if you need it, justify it
  in the header.

## Contract notes you must emit

There is no server/client seam here, so "contract" means **the surface a chart
already depends on**:

- An input renamed, retyped, or given a new default → the note names the doc at
  `docs/<slug>.md` and says what resets on existing charts.
- A plot added, removed, renamed, or changed in `display=` → the note names the
  doc, and flags that saved chart layouts and templates will not pick it up.
- An `alertcondition` or alert message changed → the note says that existing
  alerts keep firing the **old** message until the user recreates them. This one
  is genuinely surprising and is worth stating every time.
- The script's title changed → it appears as a different script in the user's
  indicator list.

## Definition of done for this lane

- `pine_check.py --strict` exits 0 on every file you touched.
- The header block reflects what the script now does, including anything you
  could not verify. You never weakened an existing admission.
- Every new input has `group`, `tooltip` where it needs one, and bounds.
- Guardrail 5 honoured: every failure path renders visibly, not as a zero.
- A contract note for every change to an input, a plot, an alert or the title.
- Nothing in your report claims the script compiles.
