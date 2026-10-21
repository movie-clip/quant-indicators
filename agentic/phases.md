<!-- Profile extension: ORCHESTRATOR only, read at intake.
     `check_report.py` reads the `verify` rows here; see `## Phases` below. -->

# Project phases: `tradingview`

The phase→lane binding for this project. Read this at intake, alongside
`project.md`. No specialist lane needs it.

## Contents

- **Phases** — the binding table: which lane fills each phase, in what order,
  and the clause that decides whether it fires. Copied into the run ledger.
- **Three orderings that are not negotiable** — dependencies a plan may not
  reorder.
- **When the brief is the specification** — why the `story` trigger is much
  narrower here than in `portfolio`.
- **What the gates run** — the command each `verify` row owes.
- **The short run** — what most work in this repo actually looks like.
- **Where the compile check sits** — the one phase-shaped thing that is not a
  phase, and why it runs before the gates.

## Phases

`protocol/orchestrator.md` § 2 defines the phases and § 3 the loop that walks
them. This table is the binding: which lane fills each phase **here**, in what
order, and the clause that decides whether it fires at all. The orchestrator
copies it into the run ledger at intake and records a verdict beside every row.

| Phase | # | Lane | Fires when | Human stop |
|---|---|---|---|---|
| ground-truth | 1 | recon | the request names an existing script, or rests on an unverified claim — a symbol, a built-in's behaviour, "this indicator already does X" | — |
| framing | 1 | product | the request is a new script, or changes what an existing script is *for* rather than how well it does it | **yes** — the brief is relayed and the verdict is the human's |
| specification | 1 | quant (RESEARCH) | the work introduces or changes a formula, a signal rule, a normalisation, a threshold or a position-sizing rule | — |
| specification | 2 | story | framing produced scope no approved story covers, **and** the slice spans more than one script or changes a signature already published to TradingView | **yes** — the one hard stop |
| design | 1 | design | more than one script changes together, or a published input/plot/alert signature changes | — |
| build | 1..n | backend, test, docs | anything that edits a `.pine` file, a fixture or a script header | — |
| verify | 1 | quant-audit | any lane touched a formula, a threshold, a sizing rule, a normalisation or a symbol mapping | — |
| verify | 2 | integration | any build lane was dispatched | — |
| verify | 3 | review | the run carries a story whose acceptance criteria someone must accept | — |
| close | 1 | docs | always — `docs/<slug>.md` and the script header against what shipped, and `pack-corrections.md` against `capabilities/` **when a lane emitted one** | — |

### Three orderings that are not negotiable

**`quant` RESEARCH before `backend`.** The formula is settled before it is
typed. Written the other way round, the implementation *is* the specification,
and the audit gate that follows has nothing independent to check it against —
it re-reads the code that it is supposed to be judging. That collapse is the
single most likely way this project's strongest gate becomes theatre.

**`quant-audit` before `integration`.** A repainting, mis-sourced or wrongly
normalised script can be flawless Pine. The engineering gate would pass it,
correctly, because it is not looking at the arithmetic. Read the audit's
`verification.detail` for **which anchor it checked against**: an audit that
re-derived the number from the same script it is auditing is a consistency
check, not an independent one.

**`recon` before `framing`.** When both fire, recon's artifact is an `inputs`
on the producer's order. Run the other way round, the producer establishes the
facts itself and a later scout can contradict it with nothing that notices.

### When the brief is the specification

The `story` row's trigger is far narrower here than in `portfolio`, and
deliberately so. The unit of work in this repo is usually **one script**, owned
by one lane, with no contract crossing and no consumer to break. The delivery
brief states the outcome and the boundary; the human approved it at the framing
stop; a ticketed story after that is a dispatch and a second human stop spent
restating a decision already made.

Record the row as `not triggered (single-script slice; the delivery brief in
<nn> is the specification)`, and `review` does not fire either — its clause asks
for a story to accept, and there is none.

**What still fires it, however small the change looks:** a change spanning more
than one script, and any change to a signature — inputs, plots, alert
conditions — of a script that has already been published to the TradingView
community. Published means other people's charts break, and "what does done
mean for them" is a question no delivery brief here has ever answered.

### What the gates run

**`integration` runs `python <agenticRoot>/scripts/pine_check.py <every .pine
file in the repo> --strict`.** Not only the files this run touched. The check is
cheap, the repo is small, and a change that breaks a script nobody dispatched
against is exactly what a whole-repo gate is for. Write that command into the
gate's order, and nothing narrower.

**`quant-audit` runs whatever re-derives the number independently** — named in
its order, and its `verification.detail` says which anchor it checked against.
Here the anchors are real and worth naming: the cited book or paper, the
exchange or regulator's own published definition, a hand-computed value on a
handful of bars, or a second data source. `capabilities/quant.md` § "Audit mode"
carries the list.

**`review`** verifies the story's acceptance criteria one by one. It fires
rarely; when it does, the compile evidence at `<run_dir>/compile.md` is what it
reads for "does it actually work", not a lane's claim.

### The short run

Most work here is a short run, and that is the system working rather than the
system being skipped. A wording fix to a header, a repaint fix inside one
script, an added input, a `pine_check.py` finding cleared — `framing`,
`specification`, `design` and `story` all fail their clauses, and the run is
`build → verify(integration) → close`. Four dispatches, and each false clause
written into the ledger with its reason.

**What makes them true, no matter how small it looks.** Anything that changes a
number the script plots — the quant rows in `specification` and `verify` both
fire, always; mathematics never takes the short route, and that is guardrail 2
made operational. Any new script at all — `framing` fires and the verdict is
the producer's. Any symbol the script resolves — guardrail 3, and `recon` owes
the run a confirmation or an explicit "unverified" before `backend` types it.

**A false clause is re-read every turn.** When the quant lane returns a formula
whose parameters need a new input, `design`'s clause has stopped being false:
that is a re-plan (`orchestrator.md` § 3), with a `## Replans` row and a bumped
`plan:`.

### Where the compile check sits

It is not a phase, and it has no lane, because **no agent in the roster holds
browser tools**. It is this project's acceptance command, run in the main
session against TradingView's editor. It runs **after each build dispatch that
changes a `.pine` file and before that change's `verify` gates**, and once more
before close if anything changed after it last ran.

Why it runs early: the gates read the source, and the compile check is the
only thing that loads the script. Gates that rule before it has run are judging
code nobody has loaded, and when the compile check then contradicts them the
run pays for a re-plan and for the gates a second time.

The orchestrator drives it, writes what the error panel and the Strategy Tester
actually said to `<run_dir>/compile.md`, and names that file in the `inputs`
of the `quant-audit`, `integration`, `review` (when it fires) and `close`
orders. **This is the one thing the orchestrator does that looks like
lane work and is not** — it is the boundary check, the same role the human's
`run_all_tests.py` run plays in `portfolio`, and the reason it is written down
here is so that doing it never gets mistaken for licence to do anything else in
the main session.

A run that closes with no `compile.md` has shipped text nobody loaded. Say so at
close-out rather than letting the ledger imply otherwise.
