# Frugal profile: sonnet-main

Recommended `.claude/routing-overrides.md` when the main loop runs **Sonnet 5** (budget configuration).

| agent | model | why |
|---|---|---|
| sage | opus | One-tier escalation ceiling: near-Fable capability at half the price. Fable is overkill above a Sonnet main loop. |
| mechanic | haiku | Fully specified mechanical edits are deterministically checkable (diff applies, tests pass): start at haiku, escalate to sonnet on a failed check. |

Agents not listed keep plugin defaults: `scout`, `extractor`, `runner` on haiku; `builder`, `reviewer` on sonnet.

Target mix: at least 30% of spawns on haiku (this profile typically lands well above it). Check with `/frugal:router-stats`.
