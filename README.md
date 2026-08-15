# frugal-kw

[![ci](https://github.com/eephus/frugal-kw/actions/workflows/ci.yml/badge.svg)](https://github.com/eephus/frugal-kw/actions/workflows/ci.yml)

A fork of [ThomasLangbroek/frugal](https://github.com/ThomasLangbroek/frugal) — adds per-main-loop-tier profiles, runner/reviewer worker agents, optimistic down-routing, and tier-mix reporting.

A cost-optimised agent router for [Claude Code](https://code.claude.com). Frugal teaches the main loop to send every sub-task to the cheapest execution strategy that can succeed, and to escalate only on verified failure:

```
deterministic tool → haiku worker → sonnet worker → main model → sage (Opus by default; Fable as opt-in ceiling)
```

The expensive reasoning model plans and judges; commodity work (locating files, extracting data, mechanical edits) runs on cheap tiers. No framework, no runtime, no API keys: frugal is a plugin made of a routing skill, seven agent definitions, and a handful of small hooks. The harness does the rest.

## Install

```
/plugin marketplace add eephus/frugal-kw
/plugin install frugal-kw@frugal-kw-marketplace
```

## How it works

The main model already reads every request, so it acts as the router at zero marginal cost. The routing skill gives it one decision table:

| Task | Agent | Model |
|---|---|---|
| locate, grep, map structure, find usages | `scout` | Haiku |
| extract, classify, summarise one source | `extractor` | Haiku |
| run tests/builds/commands with noisy output, report results | `runner` | Haiku |
| mechanical edits from a complete spec | `mechanic` | Sonnet |
| implement one scoped task from an approved plan | `builder` | Sonnet |
| first-pass review of a diff or file, findings only | `reviewer` | Sonnet |
| design, root-cause debugging, ambiguity, risk, adjudication | main loop | whatever you run |
| beyond the main loop's tier, or isolated deep reviews | `sage` | Opus (Fable via profile) |

Plus a tool-first rule: if grep, jq, git, terraform or any deterministic command solves the task, no model is called at all.

### Recommended profiles per main-loop tier

The right worker tiers depend on what the main loop itself runs. Frugal ships three profiles as ready-made override files (August 2026 pricing: Haiku $1/$5, Sonnet $3/$15, Opus $5/$25, Fable $10/$50 per MTok in/out):

| Agent | Fable main | Opus main | Sonnet main |
|---|---|---|---|
| `scout`, `extractor`, `runner` | Haiku | Haiku | Haiku |
| `mechanic` | Sonnet | Sonnet | Haiku |
| `builder`, `reviewer` | Sonnet | Sonnet | Sonnet |
| `sage` | Opus | Fable | Opus |

`sage` defaults to Opus everywhere because Opus 5 sits at near-parity with Fable at half the price; only an Opus main loop keeps Fable as a genuine capability ceiling, expected on under ~5% of spawns. A Sonnet main loop drops `mechanic` to Haiku because fully specified mechanical edits are deterministically checkable — start low, escalate on a failed check. Every profile targets **≥30% of spawns on Haiku**; `/frugal-kw:router-stats` reports your actual mix against that target.

Apply one with `/frugal-kw:models apply <fable|opus|sonnet>`, or copy `examples/profiles/<model>-main.md` to `.claude/routing-overrides.md` yourself.

### Escalation (verification first)

Workers do not self-grade their way up the ladder. Every worker ends with a fixed footer:

```
RESULT: <one line>
CHECKS-RUN: <commands run and outcomes, or "none">
UNCERTAINTIES: <or "none">
ESCALATE: yes|no - <reason>
```

The router then applies four rules:

1. If a deterministic check exists (tests, compiler, schema validation, `terraform validate`), run it. Pass = done. Fail = escalate one tier, maximum one retry, then the main loop takes over.
2. No check available: the main model spot-reads the result. It receives it anyway, so judging it costs almost nothing.
3. The worker's `ESCALATE: yes` is advisory input, never the sole trigger. Self-reported confidence from a cheap model is poorly calibrated; observable failure is not.
4. Never start at an expensive tier unless the decision table requires it. `sage` (Fable) is reached only via high-risk table rows or after escalation exhausts, one attempt, final.

## Metrics and the cost report

A `SubagentStop` hook logs one jsonl line per worker run (agent, model, token usage, escalation flag) to `~/.claude/frugal-kw/metrics.jsonl`. Run:

```
/frugal-kw:router-stats
```

to get cost per tier, tier mix (haiku share of spawns against the `FRUGAL_HAIKU_TARGET` goal), per-agent escalation rates, and estimated savings versus running the same work on your session's actual main-loop model (recorded per run; older records without it are compared against the top tier). Prices live in `scripts/stats.py` (`PRICES`); update them when Anthropic pricing changes. The report also prints a **delegation floor** per agent: a spawn costs roughly the same whether the task is trivial or large, so the report divides your measured net cost per spawn by your main-loop input rate to say how much reading a delegation has to save before it pays for itself. Under that, do it inline. Learning is deliberately offline: read the report, edit the decision table.

## Enforcement

Routing policy in a skill is advisory: the model follows it well, but a prompt cannot *forcibly* prevent anything. Frugal therefore enforces on four levels, three active out of the box and the fourth as soon as you set a budget:

1. **Policy injection.** A `SessionStart` hook puts the routing policy in context at every session start; a `UserPromptSubmit` hook re-pins a one-line reminder on every prompt. No drift, nothing to invoke manually.
2. **Inline-exploration budget.** A `PreToolUse` guard counts search-type tool calls (Read, Grep, Glob, search-y Bash) in the main loop. Past the budget (default 5 per prompt) further ones are denied with a pointer to the cheap workers. The budget resets on any delegation or new prompt; worker agents are never throttled. Non-search commands (git, test runners, builds) are never blocked.
3. **Expensive-tier guard.** `hooks/guard_expensive.sh` blocks reasoning-tier spawns — `sage`, plus the generic `general-purpose`, `Explore`, `Plan` and `claude` agents, and bare `Agent` calls that name no `subagent_type`. It is registered in the plugin's own `hooks.json` as a `PreToolUse` hook with matcher `Agent`, so it is active on install with nothing to wire up. Cheap workers (`scout`, `extractor`, `mechanic`, `builder`) are never blocked; set `FRUGAL_ALLOW_EXPENSIVE=1` to lift the ceiling for a session.

4. **Budget thermostat.** Set `FRUGAL_BUDGET_USD` and `hooks/guard_budget.py` meters the session against it, counting main-loop spend as well as delegated spend, because the main loop is the bigger line item and a thermostat that watches only workers reads 5% while the plan is empty. Under 80% it says nothing. From 80% a `UserPromptSubmit` warning arrives with each prompt. At 100% the notice turns into a stop-and-confirm, and reasoning-tier spawns are denied even with `FRUGAL_ALLOW_EXPENSIVE=1`. Cheap workers stay allowed throughout: blocking them would push work back into the main loop that emptied the budget.

Judgement lives in prompts; enforcement lives in hooks.

## Knobs

| Knob | Default | Effect |
|---|---|---|
| `FRUGAL_INLINE_BUDGET` | `5` | Inline search ops allowed per prompt before the guard denies |
| `FRUGAL_ALLOW_INLINE=1` | unset | Disables the inline-exploration guard for the session |
| `FRUGAL_ALLOW_EXPENSIVE=1` | unset | Allows `sage` and other reasoning-tier spawns past the expensive-tier guard |
| `FRUGAL_METRICS_PATH` | `~/.claude/frugal-kw/metrics.jsonl` | Where worker-run metrics are written |
| `FRUGAL_BUDGET_USD` | unset | Per-session spend ceiling: warns from 80%, stop-and-confirm plus no reasoning-tier spawns at 100% |
| `FRUGAL_HAIKU_TARGET` | `30` | Target haiku share of spawns (%), reported by `/frugal-kw:router-stats` and session-start advice |
| `/frugal-kw:models apply <profile>` | none | Applies a recommended per-main-loop-tier profile (`fable`, `opus`, `sonnet`) as `.claude/routing-overrides.md` |
| `/frugal-kw:models` | agent defaults | Per-project model overrides, e.g. `/frugal-kw:models scout=sonnet` |
| `.claude/routing-overrides.md` | none | Per-project routing rules; read first, always win |

Too aggressive for your taste? `FRUGAL_ALLOW_INLINE=1` in your environment turns the hard guard off while keeping the advisory policy. Want it gone entirely? `/plugin uninstall frugal-kw` — frugal keeps no state outside the metrics file.

## Configuration

- Defaults are the decision table in `skills/routing/SKILL.md`.
- Per-project overrides: create `.claude/routing-overrides.md` in your project. The skill reads it first and its rules win.
- Per-project model mapping: `/frugal-kw:models` shows it, `/frugal-kw:models scout=sonnet builder=opus` changes it, `/frugal-kw:models reset` restores defaults. Overrides live in the project, not the plugin, and survive updates.
- Recommended per-main-loop-tier profiles: `/frugal-kw:models apply <fable|opus|sonnet>` (templates in `examples/profiles/`).
- `sage` runs Opus by default. Want Fable as the escalation ceiling? `/frugal-kw:models sage=fable`, or apply the opus-main profile.
- Multi-provider: see [docs/litellm-recipe.md](docs/litellm-recipe.md).

## Statusline segment (optional)

Run once:

```
/frugal-kw:setup-statusline
```

It adds a `frugal $0.03/$1.20 saved` badge (session/lifetime) to your statusline: it creates a minimal statusline if you have none, or merges the segment into your existing one (with your consent, smallest possible edit). A plugin cannot configure `statusLine` automatically - that field is user-owned - so this one-time command is as close as it gets.

Manual alternative: call `scripts/statusline.py` from your own statusline command, passing the session id from the statusline stdin JSON:

```bash
FRUGAL_TXT=$(python3 "$(ls -d ~/.claude/plugins/cache/*/frugal-kw/*/scripts/statusline.py 2>/dev/null | head -1)" \
  ${SESSION_ID:+--session "$SESSION_ID"} 2>/dev/null)
```

It prints nothing when no metrics exist yet, so your statusline stays clean.

## Evaluating routing quality

Deliberately no synthetic eval harness: headless scenario evals proved flaky (other plugins' skills win trigger races, model nondeterminism) while measuring little. Evaluate with real usage instead: work normally for a few days, then run `/frugal-kw:router-stats` and read delegation rate, tier mix, and escalation rate. High escalations on one agent means its table row routes too low; near-zero savings means work is not being delegated.

## Privacy

Metrics are agent names, model ids, token counts and an escalation flag — one local jsonl line per worker run, written to `~/.claude/frugal-kw/metrics.jsonl`. No prompt content, no file paths from your projects, no telemetry, nothing leaves your machine. Delete the file at any time; the report simply starts over.

## For teams

Rollout is two commands per person (see Install) and no workflow change; routing is automatic. Work normally for a week, then review `/frugal-kw:router-stats` together and tune the decision table or `FRUGAL_INLINE_BUDGET` if the guard fires too often or too rarely.

Be precise about the cost claim when you pitch it internally: in our measurements delegated work costs **~85% less** than the same work on the top-tier model — cents instead of dollars per task. That saving applies to the *delegated* portion of a session, not the whole bill. Design, debugging and review stay on the expensive model on purpose; what frugal removes is paying reasoning rates for grep. Every install measures itself locally, so nobody has to take this README's word for anything.

## Honest trade-offs

- **Advisory unless the guard hook is enabled.** The skill steers routing; only the hook enforces it.
- **Claude Code only.** The router leans on the harness (Agent tool, hooks, parallel delegation). Multi-provider is a documented recipe, not a tested code path.
- **Metrics are limited** to fields hook events and transcripts expose. Escalations are detected via a prompt marker, so escalations performed without the marker are not counted.

## Extending

Adding a model tier is one agent file plus one table row; see [docs/extending.md](docs/extending.md).

## Releases

Releases are automated with [release-please](https://github.com/googleapis/release-please). Merges to `main` accumulate into a release PR that bumps `.claude-plugin/plugin.json` and regenerates the changelog from conventional-commit history; merging that PR tags the version and publishes a GitHub Release. Contributors never touch the version by hand, and PR titles must be valid conventional commits (a `pr-title` CI check enforces it). See [CONTRIBUTING.md](CONTRIBUTING.md).

## Licence

MIT.
