# HL Budget — specifications

HL Budget is a personal budgeting app that runs in a homelab. This repository holds no
application code: it is where each feature is specified, planned and tracked with
[Spec Kit](https://github.com/github/spec-kit), and where the project's rules live.

The code is in two service repositories, one service each:

| Repository | What it is |
|---|---|
| [hl-budget-backend](https://github.com/valyalkin/hl-budget-backend) | Kotlin + Spring Boot API under `/api`. Validates Auth0 tokens, owns the data (Postgres, Flyway) and publishes the OpenAPI contract |
| [hl-budget-frontend](https://github.com/valyalkin/hl-budget-frontend) | Next.js web app at `/`. Holds the Auth0 session server-side and calls the API from the server |
| [hl-helm](https://github.com/valyalkin/hl-helm) | The homelab's Helm charts. `charts/apps/hl-budget` deploys both services through Argo CD |

## Layout

| Path | Holds |
|---|---|
| `.specify/memory/constitution.md` | The project's principles (spec-first, separate services, Auth0, test gates, homelab operations, simplicity, API design). Every plan is checked against it |
| `specs/<NNN-feature>/` | One folder per feature: `spec.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`, `tasks.md` |
| `.specify/` (rest) | Spec Kit templates and scripts |
| `.claude/skills/` | The `/speckit-*` commands for Claude Code |

## Features

| Feature | Status |
|---|---|
| [001-platform-foundation](specs/001-platform-foundation/spec.md) | Implemented 2026-10-26: sign-in, profile, deployed to the homelab |
| [002-spending-tracking](specs/002-spending-tracking/spec.md) | Implemented 2026-10-27 (backend only): spendings, accounts and categories with 17 fixed defaults, in CAD; deployed to the homelab |
| [003-spending-screens](specs/003-spending-screens/spec.md) | Implemented 2026-10-29: the Spendings screen (month line, table with inline add/edit/remove, autofill and keyboard entry, month total, By category table and pie) and Settings for categories and accounts; backend totals, merchants and spendings by name; deployed to the homelab |
| [004-csv-import](specs/004-csv-import/spec.md) | Implemented 2026-10-30: Import CSV above the month total — drop or choose a file (or download the sample), read in the browser; review table with the Spendings table's editing, every field fixable before Add; all rows saved at once, all or nothing, retry-safe; deployed to the homelab |
| [005-home-and-remote-access](specs/005-home-and-remote-access/spec.md) | Implemented 2026-10-31: HL Budget at https://budget.valyalkin.com on any home device and over Tailscale away from home; one shared wildcard certificate for all homelab apps (a new app needs only its host name); no ports open to the internet |

## Working here

A feature moves through `/speckit-specify` → `/speckit-clarify` → `/speckit-plan` →
`/speckit-tasks` → `/speckit-implement`. Branches are named after the feature
(`001-platform-foundation`) in this repo and in the service repos. Once a feature is deployed,
add or update its row under **Features** above.

The service repos can be checked out inside this folder as git worktrees, so one session sees
the spec and the code together. Git ignores them here (`.gitignore`); code search still finds
them (`.ignore`).

```sh
git -C ../hl-budget-backend  worktree add "$PWD/hl-budget-backend"  <branch>
git -C ../hl-budget-frontend worktree add "$PWD/hl-budget-frontend" <branch>
```

How to set up Auth0, run the stack locally and deploy: the feature's `quickstart.md`.
