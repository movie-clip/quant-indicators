# Testing this repo's Pine scripts

**General reference, not a per-script doc.** This does not follow the
`docs/<slug>.md` convention `CLAUDE.md` describes — it isn't about one
script, it's the method every script in this repo is tested by, including
how to verify the symbols any script references.

If you are a human or an agent about to paste a Pine script into
TradingView's browser editor, this is written for you.

## The three layers of verification

This project has no local compiler, so "did it work" is answered in three
separate layers, each proving something the others cannot:

1. **`pine_check.py --strict`** — static, pattern-based, fast. Catches
   version mismatches, repaint patterns, missing strategy costs, unbalanced
   parens and v4/v5 leftovers. It is text analysis, not compilation: it
   cannot tell you a script compiles, cannot see whether a symbol string
   actually resolves, and cannot tell you an arithmetic formula is right.
   Run it from any lane that holds `Bash`.

2. **Independent hand-derivation** — for formula and arithmetic correctness.
   This is the quant-audit gate's job: it re-derives the computed values
   against synthetic inputs that neither the original spec nor the shipped
   code has seen, and checks the shipped script's output against that
   independent derivation. Nothing in this doc substitutes for it — a script
   can compile cleanly and still compute the wrong number.

3. **The live TradingView compile check** — the only layer that proves the
   script actually *loads*, resolves every symbol it references, and
   computes. It is mandatory before any run closes. Only the browser-driving
   session can perform it: no automated lane in this project's roster holds
   browser tools (see this project's capabilities pack), so this step runs
   in whichever session is actually driving a browser against
   TradingView.com — nothing else in the pipeline touches the real editor.

None of the three layers is optional, and none substitutes for another: a
clean static check plus a passing hand-derivation still leaves "does it
actually load in TradingView" unanswered, and that question can only be
answered by layer 3.

## Pasting a multi-line script into the Pine Editor without corrupting it

TradingView's Pine Editor is Monaco-based (the same editor VS Code uses), and
Monaco runs `onEnter` auto-indent logic on every newline it receives as
*input*. That logic is exactly what corrupts a naively-inserted script — see
below. The reliable method avoids ever sending Monaco a stream of individual
keystrokes:

1. Create a hidden `<textarea>` element in the page.
2. Set its `.value` to the exact script text via JS.
3. Call `.select()` on it, selecting the full text.
4. Send a **real, trusted `Ctrl+C` keystroke** — an actual input-device
   event dispatched by the automation layer, not a JS API call. This step is
   the crux: `document.execCommand('copy')` and
   `navigator.clipboard.writeText()` are both JS-dispatched rather than
   genuine user gestures, and both either silently fail or hang forever on a
   permission prompt for exactly that reason. Only a real keystroke event
   reliably populates the system clipboard from an automated session.
5. Click into the Pine Editor to focus it.
6. Send a real `Ctrl+A` keystroke (select all existing content), then a real
   `Ctrl+V` keystroke (paste).

This preserves indentation exactly, because Monaco treats a paste as one
atomic insertion rather than a sequence of keystrokes that could each trigger
auto-indent.

## Why the naive alternatives fail

Two approaches look like they should work, get accepted without any visible
error, and produce a script with the *correct line count* — and both are
wrong:

- **Typing the script character-by-character** (simulating individual
  keystrokes for every character, including every newline).
- **`document.execCommand('insertText', false, text)`** (inserting the full
  text as one JS call, but still landing as editor input rather than a
  paste).

In both cases, Monaco's `onEnter` auto-indent logic fires on every embedded
newline in the inserted text, the same as it would for a human typing live.
Each firing adjusts the new line's leading whitespace based on the *previous*
line's indentation and Monaco's own guess at Pine's indentation rules — and
because Pine is indentation-significant, those adjustments compound. The
result is a **staircase corruption pattern**, worst inside nested blocks: an
`if` inside an `if`, or continuation lines after a multi-line function call,
drift furthest from their correct indentation because each nested level
compounds the previous line's auto-indent error.

The dangerous part is that this produces a script that *looks* right at a
glance — right line count, right-looking structure in a quick scroll — but
compiles wrong or doesn't compile at all. **Always specifically re-check any
nested-block section after a non-clipboard paste**, and prefer the clipboard
method above entirely rather than relying on that check to catch a failure.

## Reading the result: layer 3, the live TradingView compile check

`pine_check.py` cannot see compile or runtime failures at all — it never
runs the script, so it has no opinion here. Ground truth for this layer is
the Pine Editor's own error state, the browser console filtered for
`Study|Invalid|error`, and — the recommended primary technique — **Pine
Logs**.

### Pine Logs: the recommended live-verification technique

Reading the legend or plot only tells you "a number is showing, not `n/a`".
Pine Logs proves more: not just that the script loaded, but that it
**computed the right intermediate values, traced through every step** of a
formula, on real production data.

- **How to open it.** In the Pine Editor, use the "•••" (More) menu →
  "DEVELOPER TOOLS" section → "Pine logs". It's a toggle, not an
  obviously-labelled button — easy to miss on a first look. Turning it on
  opens a docked panel showing a live, scrollable log list.
- **How to use it.** Call `log.info("...{0}...{1}", val0, val1)` (also
  `log.warning()`, `log.error()`) anywhere in the script, including inside a
  `request.security()` expression. Logs are produced on historical,
  realtime, and replay bars alike. Gate log calls with
  `barstate.isconfirmed` to avoid a flood of intra-bar repeats.
- **Why this is stronger than "check the legend".** It lets you trace every
  intermediate value in a formula — not just the final plotted number —
  against hand-derived arithmetic, on real production data rather than only
  synthetic test inputs. It also lets you directly observe branch/guard
  logic (e.g. a timeframe gate) taking the path you expect, rather than
  inferring it from the absence of a wrong-looking output.
- **Never commit instrumentation to the actual script file.** A temporary
  `log.*` block belongs only in a scratch copy pasted into the browser's
  Pine Editor buffer (via the clipboard method above), and should be removed
  from the chart/editor once verification is done. Nothing is saved unless
  "Save" is explicitly clicked — see "Experimenting safely" below.

### Two distinct failure shapes

- **A syntax/compile error** — the script never reaches TradingView's
  "start" stage. This is what a broken paste (see above) or a genuine code
  error produces.
- **`Chart.Study:ERROR: ...start failed, Invalid symbol, <ticker>`** — the
  script *compiled successfully* (it reached the "start" stage; compilation
  had already succeeded) but a `request.security()` or `input.symbol()`
  call's symbol argument failed to resolve at runtime. This is a distinct
  failure mode from a syntax error, and it's the one that matters for symbol
  verification (see "Verifying symbols, especially COT data" below).
  **`ignore_invalid_symbol = true` does not suppress this class of
  failure** — it does not make an invalid symbol resolve, it only changes
  how the script degrades once resolution has already failed, and
  TradingView still surfaces the `start failed, Invalid symbol` error in
  this situation.

A clean pass shows **no console error** and the indicator's legend or plot
displays a real computed value — not `n/a`, and no error icon on the
indicator itself. If you see `n/a` where a number is expected, that's either
expected degrade behaviour (check the script's own `n/a` cases first) or a
resolution failure worth investigating with the console open.

## Worked example: verifying mm_cot_long_short_index.pine

First-hand evidence from **2026-09-18**, using the Pine Logs method above
against the current repo script. This is a point-in-time demonstration of
the method, not a standing regression test — re-run the same technique after
any future change to this script's formula.

- **Formula cross-check on real data.** On the as-of date 2026-09-14, the
  Pine Logs trace read: `mmLong=145804`, `mmShort=10832`, `net=134972`,
  `netLo=-26767`, `netHi=219029`, `cotIndex=65.802`. Hand-computing
  `100 * (134972 - (-26767)) / (219029 - (-26767))` independently gives
  `65.80`, matching the logged value and the on-chart display. This is the
  concrete demonstration that "trace every intermediate value against hand
  arithmetic" (above) actually catches what it claims to catch.
- **`newWeekly` fires only on a genuine new report, not every bar.** Across
  three consecutive daily bars the logged `newWeekly` values were `false,
  false, true` — the `true` landing exactly on the bar where the as-of date
  advanced to a new COT report date, not on every bar.
- **CR-1's timeframe guard, confirmed directly rather than inferred.**
  Switching the chart to a Monthly timeframe and reading the logs showed
  `tfTooCoarse=true` and `histSize=0` on every bar back to 2020 (e.g.
  `bar=0 time=2020-05-01`, `bar=1 time=2020-06-01`, ...) — the weekly-
  observation array never accumulates above the weekly timeframe, and
  `cotIndex` correctly stays `na`/NaN throughout, exactly as CR-1's fix (see
  the fix run's `compile.md`) was designed to do. This is a stronger check
  than the original fix run performed, since that one only confirmed the
  Daily-chart "computes a plausible number" path, not this degrade path.

## Verifying symbols, especially COT data

This exists because of guardrail 3: **symbols are verified, never
reconstructed.** A ticker assembled from search results or pattern-matched
from a similar market is a guess, and a guess that happens to resolve
produces a chart that looks correct and isn't verified at all.

### The finding: CFTC/COT symbols resolve bare, with no exchange prefix

CFTC/COT data symbols on TradingView — the data provider shown as "COT" in
symbol search — resolve as **bare ticker IDs with no exchange prefix at
all**. For example: `088691_F_MMP_L`.

Neither `COT2:` nor `COT:` is a valid prefix when used literally inside
`input.symbol(...)` or `request.security(...)`. Both were tested directly
and both produced runtime `Invalid symbol` errors.

**The `COT2:` label shown in TradingView's own indicator-settings dialog,
next to an already-configured symbol input, is that dialog's own display
convention for naming the data vendor** — it is not part of the resolvable
ticker string, and copying it into source literally is exactly the mistake
that produces the `Invalid symbol` error above.

### The suffix convention

The aggregate "All" category — the one most scripts want by default — is
the **bare ticker with no suffix**. Only `_OLD` and `_OTHER` (crop-year
sub-breakouts, relevant to agricultural futures with a harvest cycle) take
an explicit suffix.

Those two suffixes exist as *selectable options in TradingView's symbol
search* even for symbols with no real crop year — Gold (CFTC code `088691`)
being the example this finding was made against. That makes guessing `_ALL`
as an explicit suffix a **plausible-looking wrong guess**: it reads like the
natural counterpart to `_OLD`/`_OTHER`, but it isn't a real suffix at all —
the aggregate case simply has none.

In TradingView's search-result descriptions (e.g. "G(Old) - COMEX - Futures
- Managed Money Positions Long/Short **(All)**"), "(All)" is prose describing
the row, not part of the ticker string.

### Why this matters here

Guardrail 3 exists because a plausible-looking wrong ticker has exactly two
outcomes, and only one is recoverable: it either **errors loudly** (a
symbol-resolution failure at runtime, recoverable the moment someone runs the
compile check) or, worse, **silently resolves to something adjacent-but-
wrong** — a different contract, a different vendor's series, a stale mapping
— which is not recoverable without an independent check, because nothing
about the chart looks wrong.

This specific case took **two rounds of reasoned-but-wrong guessing**
(`COT2:..._ALL`, then `COT:...`) before the bare form was found — and both
guesses were reasonable extrapolations from adjacent markets' conventions,
not careless ones. The only reason the correct form was found at all is that
the mandatory compile check (above) forced someone to actually open
TradingView's own symbol search, rather than continuing to reason from
pattern-matching.

### The general verification method

This generalizes past the one script the finding above was made on:

1. Search the raw identifying code — e.g. the CFTC contract code — with
   **no prefix or suffix guessed**. Start from the bare identifying number
   or string, not from a convention carried over from a similar market.
2. **Read the actual result rows** TradingView's symbol search returns,
   rather than assuming a prefix or suffix convention from an adjacent
   symbol family. The vendor label shown in the UI (see `COT2:` above) is
   not necessarily part of the ticker.
3. Apply the candidate bare ticker as a chart's primary symbol, or through
   the indicator's own symbol picker, and confirm real data actually loads.
4. Actually run the script — see the compile-check method above.
   `pine_check.py` cannot catch a wrong symbol; it has no way to know a
   ticker string is invalid, only that it's syntactically present.

### Dated caveat

This reflects TradingView's symbol database **as of 2026-09-17**, and vendor
databases change — renumbering, retirement, and new sub-breakouts all
happen without notice. If this note is being relied on again after a long
gap, **re-verify rather than trusting it blindly**; the method in the
section above is the durable part, the specific ticker strings above are
not guaranteed to still be current.

## Experimenting safely

Nothing you do in the Pine Editor or on the chart is saved to the
TradingView account unless "Save" is explicitly clicked on the chart layout.
That makes it safe to freely change the chart's primary symbol, add or
remove indicators, and edit Pine source (including temporary `log.*` calls)
while testing — there is no need to precisely undo each change afterward.
Abandon the tab or reload it when you're done; the account's saved layout is
untouched by any of it.
