# Project profile: `tradingview`

Repo: `C:\projects\investments\tradingview` (trading-indicators)
Bound via `tradingview/.agentic.json`.

**`CLAUDE.md` is already in your context.** Claude Code loads the repo's
`CLAUDE.md` into every dispatched lane at startup, so the product surface, the
layout, the Pine version rule and the command list reach you without a read.
Treat it as read, and go to it for anything about the code this file does not
carry. (If it is genuinely not in your context, read it — that is the one case
where it is worth a tool call.)

This file carries what `CLAUDE.md` does not: the lane map, the **numbering** a
`REFUSED` cites, and the friction the repo docs do not record. Phase bindings
live in `phases.md`, which the orchestrator reads at intake and no specialist
lane needs.

---

## Index

Read this block first. You are not expected to read this file end to end — read
what your order touches. Reading one extra section is cheap; acting on a
convention you never read is not.

**Always read:** **Hard guardrails — no plan or lane may violate these** · **Mechanical gates — never bypass**

| Section | Read it when |
|---|---|
| Why this project is unusual | it is your first order here, or you are about to write a `verification` field |
| Lane routing | you are choosing which lane owns a piece of work |
| Delivery model | you are the producer, or shaping a request into work |
| The script header contract | your order creates or edits a `.pine` file |

Elsewhere: **`phases.md`** — which lane fills each phase and the clause that
fires it (orchestrator, at intake). **`CLAUDE.md`** — product, layout, stack,
commands (already in your context).

---

## Hard guardrails — no plan or lane may violate these

1. **No repainting.** A value that changes after its bar has closed makes every
   backtest a lie and every alert unreliable. `request.security()` passes
   `lookahead=barmerge.lookahead_off` explicitly; no series is indexed forward;
   anything that must wait for a close waits on `barstate.isconfirmed`. If a
   construct might repaint and you cannot prove it does not, say so in `risks`
   and in the script header.
2. **Formula traceability.** Every computed value traces to a named source — a
   book and chapter, a paper, an exchange or regulator spec — **or** the script
   header declares it an original construction. Untraceable and undeclared →
   do not ship.
3. **Symbols are verified, never reconstructed.** A ticker assembled from search
   results or pattern-matched from another symbol is a guess, and a guess that
   resolves to *something* produces a chart that looks correct and is not. A
   symbol that has not been confirmed against TradingView's own symbol search
   must be an `input.symbol()` the user can correct, must handle non-resolution
   as visible `n/a`, and must be flagged as unverified in the header.
4. **Costs are declared.** Every `strategy()` states `commission_value` and
   `slippage`, even where slippage is 0. An uncosted backtest is wrong in the
   flattering direction, which is the direction that gets acted on.
5. **Degrade visibly.** Missing, stale or unresolved data renders as `n/a` or an
   explicit gap. Never substitute a plausible value, never carry the last good
   one forward silently, never let a failed symbol read as a zero.
6. **No execution.** Nothing here places an order, sizes a live position
   automatically, or emits an alert wired to a broker webhook. Signals and
   analysis only.

An order that requires breaking one of these gets `status: REFUSED` naming the
guardrail. **This is the system's only numbered copy** — `CLAUDE.md` restates
the same six unnumbered, so a session working the repo outside the network still
sees them and a cited number still resolves to exactly one rule.

## Why this project is unusual

**There is no local toolchain.** No compiler, no test runner, no type checker,
no package manager. Pine compiles in exactly one place: TradingView's editor.
Every instinct the network carries from a repo with a green suite has to be
re-derived here, and the field it lands on is `verification`.

So verification is split in two, and the split is not a compromise — it is the
honest shape of the problem:

| Layer | What it is | Who runs it | What it proves |
|---|---|---|---|
| **Static check** | `pine_check.py` | any lane with `Bash` | the text has no version mismatch, repaint pattern, missing cost, unbalanced paren or v4/v5 leftover |
| **Compile check** | paste into TradingView, read the error panel and the Strategy Tester | **the main session**, driving the browser | the script loads, plots, and — for a strategy — backtests |

**A lane's `verification` is the static check, and it must actually run it.**
`status: DONE` still requires `verification.result: PASS`, and here that means
`pine_check.py` exited 0. That rule keeps its teeth.

**The compile check is this project's acceptance command**, and it sits outside
the lanes for a structural reason: no agent in the roster holds browser tools.
It is run in the main session and its result is written to
`<run_dir>/compile.md` before close-out. This is the same position
`run_all_tests.py` occupies in the `portfolio` project — the mechanical
authority, run by the human at the boundary, never by an agent claiming it ran.

**A lane must never claim a script compiles.** You did not compile it and you
cannot. Report what the static check said, and put anything you are unsure
compiles into `risks`. "It should compile" in a report body is the exact failure
this section exists to prevent.

## Lane routing

| Lane | Agent | Pack | Owns |
|---|---|---|---|
| recon | `scout` | — | read-only exploration: which script does this, what does it already compute |
| product | `producer` | `product.md` | which scripts exist, which are wanted, what is already covered, and where "no" lives |
| quant | `quantitative-researcher` (RESEARCH) | `quant.md` | the formula, its source, its parameters and its failure modes, before code |
| story | `story-author` | — | drafts a ticketed story on the rare slice that needs one |
| design | `tech-lead` (DESIGN) | — | the input/plot/alert signature, and the split when more than one script moves |
| backend | `backend-engineer` | `backend.md` | **the `.pine` source.** Every script file in the repo |
| test | `test-engineer` | `testing.md` | `fixtures/**`, and the evidence a change did what it claimed |
| docs | `docs-engineer` | `docs.md` | `docs/**`, script header blocks, and at close-out applying `pack-corrections.md` |
| quant-audit | `quantitative-researcher` (AUDIT) | `quant.md` | **mathematics gate.** Independently re-derives the computed values |
| integration | `tech-lead` (INTEGRATION) | — | **engineering gate.** Runs the static check across the repo; PASS / CHANGES_REQUESTED |
| review | `reviewer` | — | **acceptance gate.** Only where a story exists |
| protocol-lint | `protocol-linter` | — | on request only: audits this project's profile and packs, at onboarding or when an order asks. Not a delivery phase |

**`frontend` is not bound in this project.** Pine has no client/server seam —
the script *is* the whole artifact. A plan that routes work to `frontend` here
has misread the repo; the plot, the table and the inputs all belong to
`backend`, because they are all the same file.

**`backend` owning `.pine` is a naming compromise, not a description.**
`check_report.py`'s `LANES` is a closed vocabulary shared with every other
project, so the lane that writes the primary source artifact is called
`backend` here as it is everywhere. Read it as "the implementation lane".

**Three gates judge a delivery.** `phases.md` § Phases carries their
triggers. `protocol-lint` judges the network's own files, not this repo,
so it is not a gate here.

## Delivery model

**Request → script.** There is no PRD and no epic tree here; the backlog is the
set of scripts that exist plus the ones the trader wants next, and `product.md`
says where it is written down.

A ticketed story is the exception rather than the rule. One script, one
behaviour change, a formula the quant lane has already specified — the delivery
brief is the specification and `story` does not fire. `phases.md` § "When the
brief is the specification" carries the clause. What does earn a story: a change
across more than one script, or a change to a script that has already been
published to the TradingView community, where "what breaks for existing users"
is a question the brief did not answer.

## The script header contract

Every `.pine` file opens with a comment block, before `//@version=6`, carrying:

- **What it computes**, in one paragraph a reader can check against the code.
- **Its source** — book and chapter, paper, spec — or an explicit statement that
  it is an original construction and therefore has no external authority.
- **What was not verified.** Symbols reconstructed rather than confirmed,
  adaptations that strip part of a published method, parameters chosen by feel.
  Named, in the header, where the next reader cannot miss them.

This is guardrails 2 and 3 made visible, and the repo already does it well: read
`systemic_stress_dashboard.pine` and `volume_profile.pine` before writing a new
header, and `mm_cot_long_short_index.pine` for what replacing a resolved warning
with its evidence looks like. **Weakening an existing header's admissions is a
regression**, even when the code around it improved. If you verified something
the header warns about, replace the warning with what you verified and how —
never just delete it.

## Mechanical gates — never bypass

- **Static check.** `python <agenticRoot>/scripts/pine_check.py <file> --strict`
  is the build lanes' and the integration gate's command. `--strict` makes
  advisories blocking; a lane that cannot satisfy one reports it in `risks`
  rather than dropping the flag.
- **The checker has its own suite.** Any order that edits `pine_check.py` also
  runs `python <agenticRoot>/scripts/test_pine_check.py` (a dependency-free
  script, not a pytest module — it exits non-zero on any failure), and adds
  the case that would have caught the bug. A checker nobody checks is a checker
  whose findings are believed without evidence.
- **Compile check in TradingView** is the acceptance command, run in the main
  session, recorded at `<run_dir>/compile.md`. A run that closes without it has
  shipped text nobody loaded.
- **No agent commits.** The human commits.
