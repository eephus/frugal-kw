---
name: models
description: Show or change which model each frugal agent runs on, per project. Use when the user wants to see the current tier-to-model mapping, move an agent to a different model (e.g. scout to sonnet), or reset overrides. Triggers on "frugal models", "change router models", "which model does scout use".
disable-model-invocation: true
---

# Frugal model overrides

Manage the per-project model mapping for frugal's agents. Defaults live in each
agent's frontmatter (`agents/*.md` under the plugin root); overrides live in the
project's `.claude/routing-overrides.md`, which the routing policy reads first.
Overrides work because the Agent tool accepts a `model` parameter per spawn - no
plugin files are ever edited.

Valid agents: `scout`, `extractor`, `mechanic`, `builder`, `runner`, `reviewer`, `sage`.
Valid models: `haiku`, `sonnet`, `opus`, `fable`.

## No arguments: show the mapping

1. Read each agent's default from the `model:` frontmatter line in
   `<plugin-root>/agents/*.md` (plugin root is two directories up from this
   skill's base directory).
2. Read the "Model overrides" section of `.claude/routing-overrides.md` if it
   exists.
3. Show one table: agent, default, override (or "-"), effective model. Nothing
   else.

## Arguments like `scout=sonnet builder=opus`

1. Validate every pair against the lists above. Invalid agent or model: reject
   with the valid options, change nothing.
2. Create or update the managed section in `.claude/routing-overrides.md`
   (create the file if missing, keep any unmanaged content above it intact):

   ```markdown
   ## Model overrides (managed by /frugal-kw:models)

   | Agent | Model |
   |---|---|
   | scout | sonnet |

   When spawning a frugal agent listed above, pass its listed model as the
   Agent tool's `model` parameter.
   ```

3. Merge with existing overrides: new pairs win, unmentioned pairs stay.
4. Setting an agent to its default removes its row. Empty table: remove the
   whole section.
5. Confirm with the resulting mapping table and apply it immediately to any
   spawns later in this session.

## Argument `reset`

Remove the managed section. If the file is then empty, delete the file.
Show the default mapping.

## Argument `apply <fable|opus|sonnet>`

Applies a whole-mix recommendation for the given main-loop tier, rather than
editing individual agent mappings.

1. Reject any value outside `fable`, `opus`, `sonnet` with the valid options,
   change nothing.
2. Locate `<plugin-root>/examples/profiles/<name>-main.md` (plugin root is two
   directories up from this skill's base directory). If it is missing, say so
   and stop.
3. If `.claude/routing-overrides.md` does not exist, copy the profile file to
   `.claude/routing-overrides.md` and confirm with its resulting content.
4. If `.claude/routing-overrides.md` already exists, do not overwrite it
   silently: show the user a diff between the existing file and the profile
   being applied, and ask before replacing it. Only copy the profile over on
   explicit confirmation.
5. Profiles are recommendations, not the only valid mapping for that tier -
   the user can still fine-tune with individual `agent=model` pairs afterward.
   Mention that `/frugal-kw:router-stats` verifies the resulting model mix
   against the profile's target.
