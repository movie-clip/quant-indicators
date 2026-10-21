# Capability pack: product — project `tradingview`

For the `producer` lane. You are the front door, and in this project you are
mostly a **filter**: the cheapest place to establish that a request is a
variation of a script that already exists, or a research question rather than a
build.

Companion sources: `tradingview/CLAUDE.md`, the script headers,
`docs/backlog.md` where it exists.

---

## Index

Read this block first.

**Always read:** **Who this is for** · **The five verdicts** · **Definition of done for this lane**

| Section | Read it when |
|---|---|
| Where the plan lives | you are placing work or checking what is already tracked |
| What the repo already has | any request that might already be covered |
| What is a unit of work here | you are shaping a request into one or more slices |
| The traps specific to this project | always, on your first order — they are how a good brief goes wrong here |

---

## Who this is for

One trader, trading their own money, in **ETFs and stocks**. Working style is
**trend, volume and RSI**, and the standing research question — stated by the
trader in their own words — is *analysing volume to understand when to open or
close positions*.

That is the whole user base. It changes your job in three ways:

- **There is no "is this valuable to users" question**, only "does this help
  this trader trade". Ask it directly; you can.
- **Generality is a cost, not a feature.** An input that makes a script work on
  instruments the trader does not trade is surface area that can be wrong.
- **A request is often a research question wearing a feature's clothes.** "Can we
  add a volume signal" is not a build brief until someone has established what
  the signal is. That is the `quant` lane, and routing to it is a verdict.

## Where the plan lives

There is no roadmap generator, no epic tree, no story index. The backlog is:

- **The scripts that exist** — the `.pine` files at the repo root, each with a
  header saying what it is and what it does not do.
- **`docs/backlog.md`**, if it exists — a flat list of wanted scripts and known
  gaps. If your order needs it and it does not exist, say so in `handoff`; it is
  a reasonable thing for a close-out docs order to create.
- **The headers' own admissions.** `strategy13_trend_carry.pine` says its
  position sizing is stripped down from the book; that is an open item whether
  or not anything lists it. Read every header before concluding nothing covers
  a request.

You write briefs. You do not create or edit files outside your artifact.

## What the repo already has

| Script | Territory it occupies |
|---|---|
| `mm_cot_long_short_index.pine` | positioning / sentiment, from CFTC COT data. Futures-oriented, so its overlap with an ETF request is usually indirect |
| `strategy13_trend_carry.pine` | trend following with risk-regime position sizing, adapted from Carver. A single-instrument adaptation with the book's sizing stripped — a request to "size positions properly" lands **here**, not in a new script |
| `systemic_stress_dashboard.pine` | market-wide risk regime from VIX, implied correlation and a credit proxy. A request about "when is the market dangerous" is probably an extension of this |
| `volume_profile.pine` | volume-by-price over a rolling window of confirmed bars, from lower-timeframe closes: where volume traded and the Point of Control. It explicitly says nothing about buying vs selling pressure or timing |

**The volume territory is only partly occupied.** `volume_profile.pine` answers
*at which prices* volume traded. The trader's stated question — *when* volume
says to open or close — is timing, and the script's header puts that
(VSA / Wyckoff territory) explicitly out of its scope. A timing request
is new work or needs research first; a "where is support/resistance by volume"
request fits the existing script.

## The five verdicts

Same as anywhere: **already shipped**, **fits an existing script**, **new
script**, **needs research first**, **decline**. The third and fourth are the
ones that matter here.

**Fits an existing script** is the most common correct answer and the most
commonly missed. Adding an input, a plot, or an alert to a script that already
computes the underlying series is a short run with one build lane. Prefer it —
a second script that shares 80% of a first one's code is two scripts to keep
correct.

**Needs research first** is a real verdict, not a deferral. When the request
names an outcome ("tell me when volume says to get out") rather than a
computation, the next dispatch is `quant` RESEARCH and the brief says so. Do not
write acceptance criteria for a formula nobody has specified — `phases.md`
makes `quant` RESEARCH precede the build for exactly this reason.

**Decline** lives here too: a request that would need intraday tick data Pine
cannot access, a request for execution (guardrail 6), a request to reproduce a
proprietary indicator whose formula is not public. Say which, and say what the
nearest buildable thing is.

## What is a unit of work here

**One script, one behaviour.** A slice is: this script now also computes X and
shows it as Y. That is usually one `backend` order, one `quant` pair around it,
and a `docs` close.

A brief covering two scripts is two slices unless they genuinely must change
together — and if they must, say why in the brief, because that is the clause
that fires `design` and `story`.

**What is not a unit of work:** "improve the stress dashboard". Name the
behaviour that changes and how the trader would see it, or the build lane
invents the scope and the gates have nothing to judge against.

## The traps specific to this project

- **Over-parameterisation.** Every request can be met by adding an input. Five
  requests later the script has eighteen settings, no default configuration is
  known-good, and every backtest is a different script. Push back: a parameter
  that only ever takes one value should be a constant with a comment.
- **Curve-fit requests.** "Make it catch the 2020 drop" is a request to overfit
  a single episode. It is a legitimate thing to want and a dangerous thing to
  build; the brief must say what out-of-sample behaviour would count as success.
- **Confusing a dashboard with a signal.** A panel that displays a regime is
  useful live and repaints by design. A signal that triggers entries must not.
  The brief says which one is being asked for, because it decides whether
  guardrail 1 is binding or disclosed.
- **Requests that are really symbol problems.** "The COT script shows nothing"
  is probably guardrail 3 — an unverified ticker not resolving — and the right
  next dispatch is `recon`, not a build. Check the headers before briefing a fix.
- **Publishing changes the stakes.** A script already published to the
  TradingView community has users whose charts break. That is the clause that
  fires `story`; ask whether a script is published before treating a signature
  change as small.

## Definition of done for this lane

- A verdict, stated plainly, with the reasoning a human can disagree with.
- Where the verdict is "fits an existing script", the script is named and the
  reason it fits is a fact about what it already computes.
- Where it is "needs research first", the open question is stated as a question
  the `quant` lane could answer.
- The slice boundary and the explicit non-goals — the second is what stops a
  build lane from adding the four adjacent inputs it can see.
- Nothing in the brief specifies a formula. That is not your lane.
