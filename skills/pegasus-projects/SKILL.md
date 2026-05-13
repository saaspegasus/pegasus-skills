---
name: pegasus-projects
description: |
  Use when the user asks to create, view, or modify a SaaS Pegasus project
  via the `pegasus` CLI — e.g. "create a new Pegasus project for X",
  "add subscriptions to my Pegasus project", "show me my project settings",
  "switch the front-end framework to React", "what features can I use on
  my license tier". This skill covers the `pegasus projects create / update /
  show / fields` commands and the underlying API shape.
---

# Managing SaaS Pegasus projects

You are managing a SaaS Pegasus project on behalf of the user. Pegasus is a
Django boilerplate; a "project" is a configuration of features (frontend
framework, auth providers, billing, AI, etc.) that the Pegasus build pipeline
later renders into a real Django codebase. Your job is to translate the user's
intent into the right CLI calls.

## Setup (one time)

Authentication uses an API key from saaspegasus.com.

- Check if `pegasus auth` is already configured: a key lives at
  `~/.pegasus/credentials`, or in `$PEGASUS_API_KEY`.
- If not, ask the user to run `pegasus auth` themselves. Don't try to guess
  or generate a key — they must paste their own.

## Commands

```
pegasus projects list                              # list all projects
pegasus projects fields --json                     # schema for a new project
pegasus projects fields --for <id> --json          # schema for an existing project
pegasus projects show <id> --json                  # full config of one project
pegasus projects create --json [--set k=v ...] [--config-file path]
pegasus projects update <id> --json [--set k=v ...] [--config-file path]
pegasus projects push <id>                         # push to GitHub (separate flow)
```

**ALWAYS pass `--json` when you (an agent) are inspecting output.** The
default is a Rich table for humans that may truncate or scroll past your
visible viewport — a 60+ field schema looks like fields are missing when
they aren't. JSON output is always complete and parseable. Treat tables as
human-only.

## The standard workflow

For any non-trivial create or update, work in this order:

1. **Get the schema.** The endpoint is **project-aware** — call the variant
   that matches your task:
   - **Creating a new project**: `pegasus projects fields --json`
   - **Updating an existing project**: `pegasus projects fields --for <id> --json`

   The schema omits fields the project's release/state can't configure. For
   a new project on a modern release, expect fields like `bundler`,
   `css_framework`, `database`, and `python_package_manager` to be absent
   — there's only one valid choice and the server applies it. For an
   existing legacy project (e.g. one still on `bundler=webpack`), those
   fields reappear with both choices so you can see and change them.
   **Trust what the schema shows.** If `bundler` isn't in the response,
   don't try to set it.

   Response shape:
   ```json
   {
     "user_tier": "free" | "basic" | "pro" | "unlimited",
     "fields": {
       "project_name":      { "type": "string", "max_length": 100 },
       "use_celery":        { "type": "boolean", "min_tier": "free" },
       "use_subscriptions": { "type": "boolean", "min_tier": "pro" },
       "ai_chat_mode":      { "type": "choice", "choices": ["llm", "none"], "min_tier": "pro" },
       ...
     }
   }
   ```

   - `min_tier` only appears on fields gated by a license tier. Compute
     "can I use this?" client-side: `field.min_tier <= user_tier` per the
     ordering `free < basic < pro < unlimited`. No feature requires
     `unlimited` today — treat unlimited as ≥ pro for the math.
   - Fields without `min_tier` (project_name, project_slug, etc.) are
     tier-agnostic.
   - Choice fields list **only the currently valid choices** for this
     context — don't hardcode the choice list, read it from the schema.

2. **If the user asked for a tier-gated feature they can't use**, surface
   that *before* attempting the call. Say something like: "Subscriptions
   requires a Pro license; you're on free tier. Want to skip it, or upgrade?"

3. **Construct the payload.** Two ways to provide settings, combinable:
   - `--set key=value` (repeatable) for individual fields. Booleans accept
     `true`/`false`/`yes`/`no`/`y`/`n`. `null`/`none`/empty parse to None
     on the client side, but **most string fields reject null server-side**
     (you'll get `field: This field may not be null.`). Use null only on
     fields explicitly documented as nullable — `pegasus_version` and
     `license` are the main ones.
   - `--config-file path` to load a YAML or JSON file. If the file has a
     `default_context:` top-level key (real `pegasus-config.yaml` shape),
     it's unwrapped automatically. `--set` values override file values.

4. **Call create or update.** The response is the full project in
   `pegasus-config.yaml` shape (see "The config shape" below).

5. **On a 400**, read the response body. Two shapes are possible:
   - **Field-keyed** (DRF serializer errors — bad slug, invalid choice,
     missing required field):
     ```json
     { "project_slug": ["Sorry, your project ID must be a valid Python module name..."] }
     ```
     The CLI renders each as `field: message`.
   - **Flat with optional `help_url`** (business / license errors, e.g.
     a license tier that doesn't support a requested feature):
     ```json
     {
       "error": "Subscriptions is not available on your current license...",
       "help_url": "https://www.saaspegasus.com/billing/"
     }
     ```
     When `help_url` is present, **always relay it to the user** — it's
     where they go to fix the underlying issue (upgrade their license,
     set up a GitHub repo, etc.). The CLI prints it as a second line
     prefixed `More info: <url>`.

   Either way, adjust and retry, or report to the user.

## The config shape

The API speaks the same key shape as a project's local `pegasus-config.yaml`,
with a few specifics:

- **JSON booleans on output**, but input also accepts `"y"`/`"n"` strings —
  so an agent can paste yaml back without translating.
- **Required on create**: only `project_name` and `project_slug`. Everything
  else uses model defaults. `author_name`, `email`, and `license` auto-populate
  from the user's profile if omitted.
- **`project_slug`** must be a valid Python identifier, lowercase, no leading
  or trailing underscore, and not a reserved name (`apps`, `templates`,
  `pegasus`, stdlib module names, etc.). Server normalizes/validates.
- **Renamed wire keys** (different from the model field name):
  - `project_name` ↔ model `name`
  - `use_auto_reload` ↔ model `use_browser_reload`
- **`pegasus_version`** is the *pinned* version (e.g. `"2026.5"` or
  `"2026.5.0.2"`) or `null` to track latest. The value must match an
  actual released version — the server validates against its release
  list and rejects guessed strings like `"2026.5.0"` with
  `Unknown Pegasus version`. If the user just wants the latest, use
  `null`; don't try to construct a version string. Output also includes
  `_pegasus_version` (read-only, the resolved version that would be
  used at build time).
- **`css_theme`** is a read-only output field derived from `css_framework`.
- **`license`** is a UUID string. Pass `null` for free tier. Must belong to
  the requesting user.

### Read-only fields (output only, ignored on input)

- `id`
- `_pegasus_version` (resolved version)
- `github_username` (computed from the linked GitHub repo or user profile)
- `css_theme` (computed from `css_framework`)

You can safely PATCH the entire GET response back — read-only keys are
silently dropped.

## Defaults you should usually leave alone

The schema decides which choice fields you can touch. Don't try to be clever
about deprecated alternatives — if `bundler` isn't in the schema, the
question "vite or webpack?" doesn't exist for this context. Similarly,
don't ask the user "should we use tailwind or bootstrap?" when the schema
only lists `tailwind`.

For **feature booleans** the user hasn't mentioned, omit them from the
payload — the server applies the model default. Don't enumerate them when
proposing the call; it bloats the conversation. The booleans that default
*on* because they're recommended for typical SaaS apps: `use_sentry`,
`use_health_checks`, `use_impersonation`, `use_async`, `use_celery`,
`use_translations`, `use_browser_reload`, `use_dark_mode`, `use_api_keys`,
`post_process` (ruff). Only flip these to `false` if the user explicitly
asks.

For **AI coding-tool rules** (`use_ai_rules_*`), all default off. If the
user asks for "AI rules" / "agents" generically without naming a tool,
prefer `use_ai_rules_claude=true` (its UI label is "Claude Code
(Recommended)"). Only set `use_ai_rules_agents`, `use_ai_rules_cursor`, or
`use_ai_rules_junie` when the user names that specific tool.

## Field interdependencies

The server enforces several couplings. Knowing them keeps you from proposing
conflicting settings:

- `bundler=vite` forces `include_static_files=false`.
- `css_framework != tailwind` forces `use_flowbite=false` and `use_shadcn=false`.
- `use_subscriptions=false` clears `subscription_billing_model` and
  `subscription_pricing_ui`.
- `use_teams=false` clears `use_teams_example`.
- `use_async=false` clears `use_async_example`.
- `docker_mode=full` requires `database=postgres` (rejected otherwise).
- `deploy_platform=kamal` requires `database=postgres`.

## License × feature gating

License tiers (low to high): `free`, `basic`, `pro`, `unlimited`.

The server validates feature compatibility at create/update time and again
at build time. If a project has features its license can't support, the
API rejects with a 400 keyed per offending feature.

You should pre-check via the schema's `min_tier` rather than discovering
through 400s. If the user wants something their tier can't do:

- If a license upgrade unblocks them, surface that as an option.
- If they want to drop the gated features instead, propose the specific
  subset to remove.

If the user has no license at all and the free tier flag is active for them,
their tier is `free`. Otherwise no license means they can't build at all
(the API will create projects but `pegasus push` will refuse).

## Common patterns

**"Create a project for me with X, Y, Z":**
1. Get schema → check user_tier supports X, Y, Z.
2. If anything's gated, tell the user and confirm before proceeding.
3. `pegasus projects create --json --set project_name="..." --set project_slug=... [--set k=v ...]`
4. Show the resulting project to confirm.

**"Show me my project / what's in it":**
- `pegasus projects show <id> --json` and present relevant subset to user.

**"Add feature X" / "switch to React" / etc:**
1. `pegasus projects show <id> --json` to see current state.
2. `pegasus projects fields --for <id> --json` to confirm the field is
   configurable for this project's release/tier.
3. `pegasus projects update <id> --json --set key=value`.

**"Apply these settings from this yaml file":**
- `pegasus projects update <id> --json --config-file path/to/pegasus-config.yaml`.
- Combine with `--set` to override specific values.

**"What can I configure?" / "What features are available?":**
- For a new project: `pegasus projects fields --json`.
- For an existing project: `pegasus projects fields --for <id> --json` —
  the response will reflect that project's release and current values.
- Parse JSON; don't rely on the table.

## Gotchas

- **Always parse JSON, never the table.** Repeating because it bites:
  `pegasus projects fields` without `--json` is a Rich table that can be
  truncated by terminal height. If you think the schema is "missing" a
  field, you're almost certainly reading truncated output — re-run with
  `--json`.
- **Slug uniqueness is per-user.** Two users can both have a project with
  slug `my_app`. You can't have two on one account.
- **PATCH is partial.** Unspecified fields stay as-is. To "reset" a field,
  you must explicitly set it (e.g., `--set ai_chat_mode=none`).
- **Pegasus version is finicky.** Pinned version (`"2026.5.0"`) or null. If
  the user wants the latest, use null — don't try to discover the latest
  version string yourself.
- **License downgrade can break a project.** If the user PATCHes to a lower
  license tier while pro features are on, the API will 400. Either remove
  the pro features in the same PATCH or do it as two steps.
- **Don't try to discover available choices.** The schema endpoint lists
  every choice for every choice-typed field. Use it.
- **Pegasus build is a separate step.** Creating/updating a project doesn't
  generate any code — that happens via `pegasus projects push` (creates a
  GitHub PR with the rendered project). Build-time validation is stricter
  than create-time validation; if it passes the API it might still fail at
  build with a license/feature/release combo issue.

## Output for the user

Pegasus CLI commands print Rich tables by default. When you're acting as an
agent for a human:

- Show meaningful state changes (the project's new name, the feature you
  changed, etc.), not the entire 60-field config dump.
- Surface license/tier issues clearly — these are conversion moments where
  the user might want to upgrade.
- For multi-step flows (check tier → propose payload → confirm → execute),
  pause at each step rather than running blind.
