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

## Working here

A feature moves through `/speckit-specify` → `/speckit-clarify` → `/speckit-plan` →
`/speckit-tasks` → `/speckit-implement`. Branches are named after the feature
(`001-platform-foundation`) in this repo and in the service repos.

The service repos can be checked out inside this folder as git worktrees, so one session sees
the spec and the code together. Git ignores them here (`.gitignore`); code search still finds
them (`.ignore`).

```sh
git -C ../hl-budget-backend  worktree add "$PWD/hl-budget-backend"  <branch>
git -C ../hl-budget-frontend worktree add "$PWD/hl-budget-frontend" <branch>
```

How to set up Auth0, run the stack locally and deploy: the feature's `quickstart.md`.
