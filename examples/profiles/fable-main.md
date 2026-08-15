# Frugal profile: fable-main

Recommended `.claude/routing-overrides.md` when the main loop runs **Fable 5**.

| agent | model | why |
|---|---|---|
| sage | opus | The main loop already sits at the frontier, so `sage` exists here for context isolation, and Opus 5 does isolated deep reviews and synthesis at half Fable's price. Escalate a single task to fable only by explicit per-task decision. |

Agents not listed keep plugin defaults: `scout`, `extractor`, `runner` on haiku; `mechanic`, `builder`, `reviewer` on sonnet.

Target mix: at least 30% of spawns on haiku. Check with `/frugal:router-stats`.
