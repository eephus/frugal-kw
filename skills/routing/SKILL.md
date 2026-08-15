---
name: routing
description: Cost-optimised task routing. Use at the start of any coding, search, extraction, review, or multi-step task to pick the cheapest execution strategy (deterministic tool, haiku worker, sonnet worker, main model, sage escalation) and to handle escalation when a worker fails. Also triggers on mentions of cost, budget, routing, or delegation.
---

# Frugal routing

Route every sub-task to the cheapest strategy that can succeed. Priorities, in order: correctness, cost, latency, extensibility, simplicity.

## Step 0: overrides

If `.claude/routing-overrides.md` exists in the project, read it first. Its rules win over everything below.

## Step 1: tool first

Before any delegation: if a deterministic command solves the task (grep, rg, jq, yq, sed, awk, git, terraform, kubectl, helm, docker, a compiler, a test runner), run it. No LLM call. Reasoning models are for reasoning.

Step 1 covers **one-shot** commands only: you know the exact command and its output answers the question directly. If the command is right but its output is long or noisy — test suites, builds, linters, verbose scripts — hand the run to `runner` instead: haiku reads the noise, you read the summary. The moment discovery turns iterative — a second search informed by the first, listing directories to decide what to read next, reading files to summarise them — it is no longer a tool call, it is a locate/extract task. Bright line: the third search/list/read operation on the same question means you are exploring inline; stop and hand the whole question to `scout` or `extractor`, including what you already learned. Every raw tool result you ingest is paid at main-loop rates; a haiku worker reads the same bytes at a fraction of the cost and returns a summary.

## Step 1.5: sensitivity gate

Sensitivity is not a task-type signal, so it cannot live in the decision table below. Some data must never leave the main loop for a worker, however cheap the task looks. Decide this **before** tier selection, not by letting a worker fail into it.

If `.claude/frugal-sensitivity.json` exists in the project, it declares rules (content regexes and path globs) and, per rule, which workers may still receive matching data. The `guard_sensitive.py` hook enforces it at spawn: a matching Agent delegation to a non-allowed worker is blocked, and you handle that sub-task inline. The gate fails closed (a broken config blocks delegation), unlike the cost hooks which fail open. With no such file the gate is off. See `examples/frugal-sensitivity.example.json`.

This is an enforced default, not a substitute for judgement: for regulated data the human owns the final call on where it may go.

## Step 2: decision table

Decompose the request into sub-tasks. For each, match signals to the cheapest capable agent:

| Task signals | Required capabilities | Route |
|---|---|---|
| "where is X", "what uses Y", map directory, grep logs | locate | `scout` (haiku) |
| pull fields from docs/logs, classify against given categories, summarise one file or diff, format conversion | extract | `extractor` (haiku) |
| run a given test suite, build, linter, or command with long or noisy output; report status and failures | run-and-report | `runner` (haiku) |
| rename, boilerplate, apply known pattern, config value change, test scaffold from example, with a complete spec | mechanical-edit | `mechanic` (sonnet) |
| implement one scoped task from an approved plan, write tests from given cases, fix a simple reproduced bug | implement-from-plan | `builder` (sonnet) |
| first review pass over a diff or file: correctness, obvious bugs, conventions — findings only, you adjudicate | first-pass-review | `reviewer` (sonnet) |
| design, root-cause debugging, ambiguous requirements, adjudicating review findings, trade-offs, anything regulated or risky | reasoning | main loop (you) |
| task exceeds the main loop's own tier, or top-tier work needs an isolated fresh context (parallel deep reviews, synthesis over merged summaries) | deep-reasoning | `sage` (opus; fable via profile) |

**Start low when checkable.** When a deterministic check exists for the output (tests, compiler, schema validation, diff applies) and you are torn between two tiers, start at the cheaper one. At current prices (haiku $1/$5, sonnet $3/$15, opus $5/$25, fable $10/$50 per MTok in/out) a failed haiku attempt plus the sonnet retry still costs less than starting at sonnet whenever haiku's success odds beat roughly 15%. The escalation protocol caps the downside at one retry.

`sage` is never a routing default, and it runs **Opus 5 by default**: near-Fable capability at half the price. Fable stays available behind an explicit override (`/frugal-kw:models sage=fable`, or the opus-main profile below) for the rare longest-horizon task where the frontier model still leads. Check what the main loop actually runs before reaching for `sage`: from a Sonnet main loop it is a real capability escalation; from an Opus or Fable main loop it is mostly context isolation, not capability.

**Generic agents are never routing targets.** `Explore`, `general-purpose`, `claude`, and `Plan` are reasoning-tier and bill at main-loop rates — never spawn them for locate, extract, or summarise work, no matter how broad the fan-out. Map them down: locate/map -> `scout`; extract/summarise/classify -> `extractor`; mechanical edits -> `mechanic`/`builder`; reasoning stays in the main loop. A bare `Agent` call with no `subagent_type` defaults to `general-purpose` (expensive) — always name a cheap agent explicitly. The `guard_expensive.sh` hook blocks these at spawn time; `FRUGAL_ALLOW_EXPENSIVE=1` is the deliberate, per-session override for when a task genuinely needs main-loop breadth.

Workers pin their own reasoning effort in frontmatter: `low` for `scout`, `extractor`, `runner` and `mechanic`, `medium` for `builder` and `reviewer`, `high` for `sage`. Model tier is one axis of cost and thinking is another, so a session running at high effort no longer makes a haiku scout deliberate over a grep.

## Profiles: recommended tiers per main-loop model

Worker tiers should key off what the main loop itself runs. Apply a profile by copying `examples/profiles/<model>-main.md` to `.claude/routing-overrides.md` in the project, or with `/frugal-kw:models apply <fable|opus|sonnet>`:

| Agent | Fable main | Opus main | Sonnet main |
|---|---|---|---|
| scout, extractor, runner | haiku | haiku | haiku |
| mechanic | sonnet | sonnet | haiku |
| builder, reviewer | sonnet | sonnet | sonnet |
| sage | opus | fable | opus |

Why the deviations: `sage` stays opus almost everywhere because Opus 5 is near-parity with Fable at half the price; only an Opus main loop keeps fable as a genuine capability ceiling, and even there expect it on under ~5% of spawns. A Sonnet main loop drops `mechanic` to haiku because fully specified mechanical edits are deterministically checkable — start low, escalate on a failed check.

Whatever the profile, the target mix is **at least 30% of spawns on haiku** (`FRUGAL_HAIKU_TARGET` to change). `/frugal-kw:router-stats` reports the actual mix; below target usually means runnable or summarisable work is still being done inline or routed to sonnet.

## Never delegate

Security-sensitive changes, destructive operations, ambiguous requirements, anything needing user judgement. These stay in the main loop, always.

## Delegation rules

- Delegate only self-contained sub-tasks the prompt can fully specify. If specifying takes longer than doing: do it inline.
- Delegation has a floor. A spawn costs about the same whether the task is trivial or large, so a job too small to repay that fixed cost is cheaper done inline. `scripts/stats.py` prints the measured floor per agent ("delegating pays off above ~N tokens of reading") from your own runs; under it, read it yourself.
- Context handoff: pass pointers (`path:line` ranges, commit SHAs, URLs), never pasted file content. Pasting is billed as main-loop output tokens (fable ~$50/MTok, and generating them takes wall-clock time); a worker reads the same bytes as haiku input (~$1/MTok) in one round trip. Paste only what the worker cannot retrieve itself — text that exists solely in the conversation (user message, prior tool output, fetched page) — or trivially small snippets (<~200 tokens).
- Batch independent delegations in one message so they run in parallel. Large fan-outs (e.g. review 50 modules): fan out `scout`/`extractor` workers, merge their summaries, do one final reasoning pass yourself.
- Workers end with a footer (`RESULT:` / `CHECKS-RUN:` / `UNCERTAINTIES:` / `ESCALATE:`). A worker reporting ambiguity: resolve it yourself; never re-prompt the worker to guess.

## Escalation protocol (verification first)

1. A deterministic check exists for the worker's output (tests, compiler, schema validation, diff applies, `terraform validate`): run it. Pass = done. Fail = re-dispatch one tier up (haiku worker -> the equivalent sonnet worker, or `builder` when none exists; sonnet worker -> take over yourself), maximum one retry. Prefix the retry prompt with `[frugal-escalation from <agent>]` and include the failed attempt's footer.
2. No deterministic check: spot-read the result yourself. You receive it anyway; judging it costs almost nothing.
3. The worker's `ESCALATE: yes` is advisory input to rules 1 and 2, never the sole trigger.
4. If the task still exceeds your own tier after you take over: hand it to `sage` with the full failure history, prefixed `[frugal-escalation from main]`. One attempt, final.
5. Never start at an expensive tier unless the decision table sends you there.

## Budget

`FRUGAL_BUDGET_USD` sets a per-session ceiling, counting main-loop spend as well as delegated spend. Under 80% of it nothing is said. From 80% a warning arrives with each prompt: delegate harder, keep worker replies terse, leave `sage` alone. At 100% stop and confirm with the user before starting new work, and reasoning-tier spawns are denied even with `FRUGAL_ALLOW_EXPENSIVE=1`, because they cost more than the main loop does. Cheap workers stay allowed at every level: they are how you get back under.

Unset means no ceiling and no warnings.
