---
name: doc-registry
description: >
  Curated local documentation registry for Hermes agents. Use when the user asks to
  search/lookup/find content in "the docs", "app docs", "project docs", or any
  phrasing that indicates they want to search within a known set of local documents
  before falling back to web search or random filesystem search. Also use when the
  user mentions a specific app/project name alongside "docs" (e.g. "search in the
  docs for my-api"). The skill reads a user-maintained YAML registry of doc paths
  and constrains search to those locations first.
---

# Doc Registry

## What it does

You have a curated set of docs scattered across your filesystem — API docs, READMEs,
specs, notes — that you want Hermes to search **first** when you ask about them,
instead of guessing or web-searching. This skill reads your registry and restricts
search to those paths.

## The registry file

The registry lives at **`~/.hermes/skills/doc-registry/docs.yaml`** by default.
The user can override the path by saying so ("my docs are at /some/other/path").

### Format

```yaml
# docs.yaml — curated doc registry for Hermes agents
# Edit this file to add/remove apps and their doc locations.

apps:
  - name: my-api
    description: "My backend API — OpenAPI spec, deployment notes, and runbooks"
    paths:
      - ~/projects/my-api/docs/openapi.yaml
      - ~/projects/my-api/README.md
      - ~/projects/my-api/docs/runbook.md

  - name: frontend-app
    description: "React frontend — component docs, storybook, architecture notes"
    paths:
      - ~/projects/frontend-app/docs/architecture.md
      - ~/projects/frontend-app/storybook-static/index.html

  - name: infra
    description: "Infrastructure — Terraform, Ansible, Kubernetes manifests"
    paths:
      - ~/projects/infra/README.md
      - ~/projects/infra/modules/*/README.md
      - ~/projects/infra/terraform/*.md

# Optional: default apps to search when the user doesn't name one
default_apps:
  - my-api
  - frontend-app
```

### Rules for the registry

- `name` is a short label the user will reference ("search in my-api docs")
- `description` helps you decide which app matches when the user is vague
- `paths` are glob-capable file paths. Tilde (`~`) expands to `$HOME`. Asterisks (`*`)
  are glob wildcards usable with `search_files`.
- A path can point to a single file or a pattern covering many files
- `default_apps` (optional) lists which apps to search when the user says "search in
  the docs" without naming a specific app

## How to use this skill

### When the skill triggers

Trigger on phrases like:
- "search in the docs", "look in the docs", "check the docs"
- "search the app docs", "find in the project docs"
- "look up in the documentation", "check the docs for X"
- "search my docs for ...", "docs search ..."
- Any request that names an app + "docs" (e.g. "my-api docs say...")

Do **not** trigger when the user is clearly asking about a web page, a specific
file they've already named, or general knowledge.

### Procedure

**Step 1 — Read the registry**

Read `~/.hermes/skills/doc-registry/docs.yaml`. If it doesn't exist or is empty,
tell the user: "No doc registry found at `~/.hermes/skills/doc-registry/docs.yaml`.
Want me to create one? Run `doc-registry-init` or I can scaffold it now."

**Step 2 — Identify the target app(s)**

- If the user named a specific app ("search in my-api docs"), match by `name`.
  Match is case-insensitive substring: "api docs" could match "my-api".
- If the user said "the docs" without naming an app, use `default_apps` if present.
  If no `default_apps`, pick the first app or ask which one.
- If multiple apps match, list the candidates and ask the user to pick.

**Step 3 — Resolve paths**

For each matched app, expand every path:
- `~` → `$HOME` (os.path.expanduser)
- Globs (`*`, `**`) → resolve with `glob.glob` or use `search_files` with the glob

Skip paths that don't exist on disk (the user may have moved things). Note missing
paths quietly — don't spam the user unless nothing is found.

**Step 4 — Search**

The user's query is a natural-language description of what they're looking for.
Extract the key terms and search the resolved paths:

| What you have | Tool to use |
|---|---|
| A specific file path from the registry | `read_file` on that path, then scan for the query terms |
| A glob covering many files | `search_files(pattern=<glob>, path=<dir>, target="content")` with the query terms as regex |
| A directory of docs | `search_files` with `*.md` or `*` pattern |

If the query is vague ("what does the docs say about auth?"), extract "auth" as the
search term. If the query is a direct quote or phrase, use it as-is. When in doubt,
search for the most specific nouns in the query.

**Step 5 — Report findings**

Report what you found, citing the file path and relevant excerpt. If you found nothing
in the registry, say so clearly:

> "I checked [N] files across [apps] and didn't find anything about [query]. Want me
> to web-search instead, or expand the registry?"

Do **not** silently fall back to web search — ask first.

**Step 6 — Suggest registry improvements (optional)**

If the user asked about something that isn't covered by any registered doc, suggest
adding it:

> "That topic isn't covered by your current registry. The docs for [app] could use a
> file on [topic]. Want me to add a path to docs.yaml?"

## Quick examples

**User:** "search in the docs for how auth works"
→ Read registry → match default_apps or ask which app → search matched paths for "auth" → report.

**User:** "check my-api docs for the rate limit"
→ Read registry → match app "my-api" → search its paths for "rate limit" → report.

**User:** "look in the docs"
→ Read registry → if default_apps set, search those → if not, ask "which app's docs?"

## Maintenance

See [references/how-to.md](references/how-to.md) for how to add, remove, and organize
entries in the registry. The user can also just edit `docs.yaml` directly — it's a plain
YAML file.

## Reference

- [Sample registry template](references/sample-docs.yaml) — copy this to `docs.yaml` to start
- [How to maintain your registry](references/how-to.md) — add/remove apps, path patterns, tips
