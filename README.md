# NGP TechWorld Engineering Standards

> **Status: DRAFT v0.2** (2026-11-02). Flutter is approved (GetX template); the other stacks are drafts. Built from the stack audit of 16 projects.
> Choices marked **⏳ pending approval** may change once the approved stack list ("filters") is final.

One rulebook per stack, so that every NGP project looks the same and a new developer can open
any repo and know where things are.

## What is in here

| Stack | Used in | Standard |
|---|---|---|
| All projects (git, commits, CI, env, docs, dates, i18n) | everything | [`skills/ngp-general/SKILL.md`](skills/ngp-general/SKILL.md) |
| Laravel API (PHP) | all backends | [`skills/ngp-laravel/SKILL.md`](skills/ngp-laravel/SKILL.md) |
| React + Vite + TypeScript SPA | dashboards, apps behind login, most web frontends | [`skills/ngp-react/SKILL.md`](skills/ngp-react/SKILL.md) |
| Next.js (App Router) | public sites that need SEO | [`skills/ngp-nextjs/SKILL.md`](skills/ngp-nextjs/SKILL.md) |
| Flutter mobile (GetX, from `ngp_flutter_clean_architectuer`) | customer / merchant / worker apps | [`skills/ngp-flutter/SKILL.md`](skills/ngp-flutter/SKILL.md) |

Templates you copy into a new repo live in [`templates/`](templates/).

## Starter (architecture) repos

Every new project starts from a starter repo, so the structure is right from the first commit.
The skill for a stack describes the rules; the starter repo shows them in working code.

| Stack | Starter repo | Status |
|---|---|---|
| Flutter | [`ngp-techworld/ngp_flutter_template`](https://github.com/ngp-techworld/ngp_flutter_template) (from `NGPTechWorld/ngp_flutter_clean_architectuer`) | ready |
| Laravel API | [`ngp-techworld/ngp_laravel_api_template`](https://github.com/ngp-techworld/ngp_laravel_api_template) | ready |
| React + Vite | [`ngp-techworld/ngp_react_vite_template`](https://github.com/ngp-techworld/ngp_react_vite_template) | ready |
| Next.js | [`ngp-techworld/ngp_nextjs_template`](https://github.com/ngp-techworld/ngp_nextjs_template) | ready |

Each starter repo contains: a working example feature end-to-end (e.g. login + one list
screen), the folder layout from the skill, lint/format/test config, `ci.yml`, `.env.example`,
`README.md` and `CLAUDE.md` from `templates/`, and an `ARCHITECTURE.md` like the Flutter one.
Mark them as GitHub "template repositories" so a new project is created with
"Use this template".

## Why the format is "a Claude Code skill per stack"

Each standard is a plain Markdown file (`SKILL.md`). A person reads it like any doc. Claude Code
loads the same file automatically whenever it works on a matching project, so the AI writes code
that follows the same rules as the team. One file, two readers, no copy drifting out of date.

## How to use it

**People:** read `ngp-general` once, then the file for the stack you work on.

**Claude Code (whole team, recommended):** this repo is a Claude Code plugin. Each developer runs
once:

```
/plugin marketplace add https://github.com/ngp-techworld/ngp-standards.git
/plugin install ngp-standards@ngp
```

The short form `ngp-techworld/ngp-standards` clones over SSH and fails without a GitHub SSH key;
the HTTPS URL works for everyone.

Where `/plugin` isn't available (e.g. the VS Code extension), run the same in a terminal, then
restart Claude Code:

```
claude plugin marketplace add https://github.com/ngp-techworld/ngp-standards.git
claude plugin install ngp-standards@ngp
```

Updates to the standards then reach everyone with `/plugin marketplace update ngp`.

**Claude Code (single repo, no plugin):** copy the needed folders into the project at
`.claude/skills/ngp-laravel/SKILL.md` etc. and commit them.

In every project's `CLAUDE.md`, add one line so Claude always applies the right one:

```
Follow the NGP standards: ngp-general plus ngp-laravel (backend) / ngp-react (frontend).
```

## Rules for changing a standard

1. Open a PR against this repo. Explain *why* in the PR, with a real example from a project.
2. One approver (tech lead) merges it. Bump `version` in `.claude-plugin/plugin.json`.
3. Existing projects are migrated when they are next touched, not in one big rewrite.

The rules here describe **new code**. Old code is fixed when you are already editing that file
("leave the file cleaner than you found it"), not in separate refactor-only PRs.
