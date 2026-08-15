# Frugal profile: opus-main

Recommended `.claude/routing-overrides.md` when the main loop runs **Opus 5**.

| agent | model | why |
|---|---|---|
| sage | fable | The only genuine capability escalation above an Opus main loop. Rare by design: expect it on under ~5% of spawns, and only for the longest-horizon frontier work where Fable still leads. If you want isolation without the premium, use `/frugal:models sage=opus` instead. |

Agents not listed keep plugin defaults: `scout`, `extractor`, `runner` on haiku; `mechanic`, `builder`, `reviewer` on sonnet.

Target mix: at least 30% of spawns on haiku. Check with `/frugal:router-stats`.
