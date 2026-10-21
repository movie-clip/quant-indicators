# Capability pack: docs — project `tradingview`

For the `docs-engineer` lane. Two surfaces here, and the first is unusual: the
**script header block inside the `.pine` file** is documentation, is the most
-read documentation in the repo, and is yours to keep true.

Companion sources: `tradingview/CLAUDE.md`, `project.md` § "The script header
contract".

---

## Index

Read this block first.

**Always read:** **The two surfaces** · **Never weaken an admission** · **Definition of done for this lane**

| Section | Read it when |
|---|---|
| `docs/<slug>.md` — the shape | your order reconciles or creates a script's doc |
| Auto-update vs flag-for-human | you found a discrepancy your order did not name |
| The second close-out order: applying pack corrections | your order names `pack-corrections.md` |

---

## The two surfaces

| Surface | Lives in | Who reads it |
|---|---|---|
| **Script header** | the comment block at the top of each `.pine` file, above `//@version=6` | anyone who opens the script — including the trader, months later, and anyone it is shared with |
| **Script doc** | `docs/<slug>.md` | whoever is deciding whether to trust or change the script |

The header is the one that matters most, because it travels with the code. A
script pasted into TradingView carries its header and nothing else.

**You may edit the header block of a `.pine` file.** That is a deliberate
exception to "`backend` owns the `.pine` files" — the header is prose, it is the
part most likely to drift, and the implementation lane is the lane least likely
to notice it drifted. You do not touch anything below `//@version=6`.

## Never weaken an admission

`project.md` § "The script header contract" requires each header to say **what
was not verified**. The existing headers do this well and specifically: a
stripped position-sizing formula, an original construction with no external
authority, an estimate built from lower-timeframe closes rather than ticks —
and, in the COT script, a reconstructed-ticker warning replaced by the date and
method of confirmation, which is the shape the rule below asks for.

The failure mode is quiet erosion. A run improves a script, the header is
rewritten to describe the improvement, and the warning goes with it — nobody
decided to remove it and nobody notices it is gone.

So:

- **An admission is removed only when the thing it warns about was actually
  fixed**, and the run's artifacts say who fixed it and how. Replace it with what
  was verified — "symbols confirmed against TradingView symbol search,
  2026-09-14" — rather than deleting the paragraph.
- **If you cannot find evidence in this run that it was resolved, it stays**,
  verbatim, even if the surrounding paragraph is rewritten.
- A header admission you believe is now false, with no evidence in the run, is a
  `risks` bullet and a `should_fix`. It is not yours to remove on judgement.

## `docs/<slug>.md` — the shape

One file per script, named for the script's slug with underscores turned into
hyphens (`volume_profile.pine` → `docs/volume-profile.md`); the existing docs
use this form. `docs/testing-methodology.md` is the one doc that is not
per-script. Sections, in order:

1. **What it computes** — the paragraph from the header, expanded.
2. **Inputs** — a table: name, type, default, what it controls, and whether its
   correct value depends on the timeframe.
3. **Plots and outputs** — what is drawn, what each colour and state means, and
   what `n/a` looks like when it appears.
4. **Sources** — the citation, or the explicit statement that it is original.
5. **Known limits** — the header's admissions, expanded, plus anything a gate
   found and the run accepted.
6. **Reproducing the fixture**, where one exists — symbol, timeframe, date
   range, and every non-default setting.

Section 6 is the one that gets skipped and the one that makes a fixture worth
keeping. A backtest export whose settings nobody recorded cannot be compared
against anything.

## Auto-update vs flag-for-human

**Update directly** when the change is mechanical and the run's artifacts prove
it: an input's default changed, a plot was added, a citation gained a page
number, a fixture was regenerated with recorded settings.

**Flag for the human** — `risks` plus a `should_fix` — when:

- The discrepancy is about what a number *means* rather than what it is. That is
  the quant lane's, not yours.
- A header admission looks stale but nothing in the run resolved it.
- Two sources in the repo disagree and neither is obviously newer.
- The fix would require knowing what the trader intended.

## The second close-out order: applying pack corrections

At close-out you may receive a second order naming
`<run_dir>/pack-corrections.md`. `protocol/packs.md` is the authority; read it.

Scope is `agentic/capabilities/**` and nothing else. Each correction names a pack file, the false premise, and
the replacement wording — apply the replacement, do not improvise a better one.
A correction you disagree with goes back as a `risks` bullet with the pack and
the premise named; it does not get silently dropped.

This is the one order in which a lane writes a pack. The corrected packs
are committed with the run's code changes and reviewed in its pull request.

## Definition of done for this lane

- Every script the run touched has a header that matches what it now does.
- No admission was removed without evidence in this run that it was resolved.
- `docs/<slug>.md` exists for every script the run touched, with section 6
  filled in wherever a fixture exists.
- Discrepancies you could not resolve are in `risks`, named specifically enough
  that the next run can act on them.
- Nothing below `//@version=6` in any `.pine` file was changed.
