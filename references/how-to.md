# How to maintain your doc registry

## Adding an app

Edit `docs.yaml`. Add a new entry under `apps:`:

```yaml
  - name: my-new-app
    description: "What this app is and what its docs cover"
    paths:
      - ~/projects/my-new-app/docs/spec.md
      - ~/projects/my-new-app/README.md
```

The `name` is what you'll say to the agent ("search in my-new-app docs").
The `description` helps when you're vague ("check the docs for the new app").
`paths` is a list of files or glob patterns.

## Path patterns

Paths are expanded before search:

| Pattern | What it matches |
|---|---|
| `~/projects/app/README.md` | One specific file |
| `~/projects/app/docs/*.md` | All .md files directly in docs/ |
| `~/projects/app/docs/**/*.md` | All .md files recursively under docs/ |
| `~/projects/app/docs/*` | All files directly in docs/ |

Use `glob.glob` semantics. Tilde expands to `$HOME`. If a path doesn't exist on
disk, it's silently skipped (the docs may have moved — check your registry).

## Removing an app

Delete the entry from `apps:`. If it was in `default_apps`, remove it from there too.

## Setting default apps

If you often say "search in the docs" without naming an app, list those apps under
`default_apps:`:

```yaml
default_apps:
  - my-api
  - frontend-app
```

When no app is named, the agent searches these in order. If nothing is found, it asks
whether to expand the search.

## Organizing large registries

If you have many apps, group them with descriptive names and descriptions. The agent
matches by substring, so "search the api docs" will match any app whose name contains
"api". Use distinctive names when you have multiple similar apps.

Example:

```yaml
apps:
  - name: payments-api
    ...
  - name: auth-service
    ...
  - name: billing-frontend
    ...
```

## Tips

- Keep descriptions short but specific. "Backend API docs" is too vague; "Payment API —
  OpenAPI spec, webhooks, and deployment runbook" helps the agent disambiguate.
- Register the docs you actually search for. A registry with 50 unused paths is noise.
- If a doc moves, update the path. Missing paths are skipped silently, so a stale
  registry degrades search quality without warning.
- You can have the agent suggest additions: when a search finds nothing, it will offer
  to add a path to the registry.
