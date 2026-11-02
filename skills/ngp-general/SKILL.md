---
name: ngp-general
description: NGP TechWorld company-wide rules for every project (repo layout, git branches and commit messages, CI, env and secrets, README/CLAUDE.md, naming, dates, money, Arabic/RTL i18n). Use for any work in an NGP TechWorld repository, together with the stack-specific ngp-* skill.
---

# NGP General Standards

> DRAFT v0.2. Items marked ⏳ wait for the approved stack list.

These rules apply to **every** NGP project, whatever the language. The stack skills
(`ngp-laravel`, `ngp-react`, `ngp-nextjs`, `ngp-flutter`) add to them and never contradict them.

## 1. The three principles

1. **Readable beats clever.** Code is written once and read many times. Prefer the obvious
   solution, explicit names, and small files. If a new developer needs you to explain it, rename
   or split it.
2. **One way to do each thing.** Same folder layout, same libraries, same patterns in every
   project. Don't introduce a second library for something we already solved.
3. **Automate the rules.** Anything a tool can check (format, lint, types, tests) is checked in CI,
   not in code review.

## 2. Repository layout

One git repository per product, under the `ngp-techworld` GitHub organization. Every project lives
in git from day one (no local-only folders).

```
<product>/
├── backend/          Laravel API
├── web/              public site (React SPA or Next.js)
├── dashboard/        admin / merchant panel (React SPA)
├── mobile/           Flutter apps (one folder per app + packages/ for shared Dart code)
├── docs/             SPEC.md, ARCHITECTURE.md, API.md, DEPLOY.md, decisions/
├── deploy/           nginx, systemd, scripts (never secrets)
├── .github/workflows/ci.yml
├── CLAUDE.md
└── README.md
```

Only create the folders the product needs. Names are lowercase-kebab (`customer-app` is fine,
`Customer_App` is not). Flutter app folders use snake_case because Dart packages must
(`customer_app`).

## 3. Git

**Branches**

| Branch | Purpose |
|---|---|
| `main` | Production. Protected. Only merged through PRs with green CI. Deploys from here. |
| `dev` | Integration / staging. |
| `feat/<short-name>`, `fix/<short-name>`, `chore/<short-name>` | Work branches, from `dev`. |

**Commits** use [Conventional Commits](https://www.conventionalcommits.org):

```
feat(booking): allow QR booking without an account
fix(orders): stop double charge when retrying payment
chore(deps): update laravel/framework to 13.39
```

- Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`.
- The scope is the module/feature name.
- Subject in English, imperative, max ~72 chars, no full stop.
- **Never** commit with a meaningless message (`.`, `update`, `wip`, `fix`).
- One logical change per commit. Don't mix formatting with behavior changes.

**Pull requests:** small (aim for under ~400 changed lines), one topic, description says *what*
and *why* and how it was tested. Squash-merge into `dev`.

**One identity per developer:** set the same `user.name` and `user.email` on every machine
(`git config --global`). Work email only.

## 4. Never commit

- `.env`, `.env.local`, `.env.production`, keys, certificates, database dumps.
- `node_modules/`, `vendor/`, `dist/`, `build/`, `.next/`, `test-results/`, `tmp/`.
- One-off patch scripts (`patch01.cjs`, `fix-temp.php`). If a script must run once, it is a
  migration, a seeder, or an artisan command, and it is reviewed like any code.

Commit a `.env.example` with every variable the app reads, with safe placeholder values and a
comment per group. Commit lockfiles (`composer.lock`, `package-lock.json`, `pubspec.lock`).

## 5. CI (mandatory)

Every repo has `.github/workflows/ci.yml` that runs on every PR and on pushes to `dev` and `main`.
A PR cannot merge when CI is red. Minimum per stack:

| Stack | CI steps |
|---|---|
| Laravel | `composer audit`, `pint --test`, `phpstan`, `php artisan test` (against PostgreSQL) |
| React / Next.js | `npm ci`, `npm audit --audit-level=high`, `lint`, `typecheck`, `test`, `build` |
| Flutter | `fvm flutter analyze`, `fvm dart format --set-exit-if-changed .`, `fvm flutter test` (CI reads the version from `.fvmrc`) |

Copy the starting file from `templates/ci.yml`. Deploys run from CI on push to `main`, never by hand
from a laptop (manual scripts are a fallback, documented in `docs/DEPLOY.md`).

## 6. Docs every repo must have

- **`README.md`** (replace the framework's default README on day one): what the product is, how to
  run it locally in under 5 commands, how to run tests, where the docs are. Use
  `templates/README.md`.
- **`CLAUDE.md`**: which NGP skills apply, local ports, anything Claude must not do. Keep it short,
  and keep it true (update it in the same PR that changes the setup). Use `templates/CLAUDE.md`.
- **`docs/decisions/NNNN-title.md`**: one short note per important decision (why Redis, why no
  Docker...). Three headings: Context, Decision, Consequences.

## 7. Naming (all languages)

- English for all identifiers, comments, commits and docs. Arabic only in user-facing strings,
  and those live in translation files, never hard-coded in components or controllers.
- Names say what something *is* or *does*: `pendingOrders`, not `data2`, `list`, `tmp`.
- Booleans read as yes/no questions: `isActive`, `hasSubscription`, `canEdit`.
- No abbreviations except well-known ones (`id`, `url`, `api`, `otp`, `qr`).
- Comments explain **why**, not what. Delete commented-out code; git remembers it.

## 8. Shared data rules

- **Dates and times:** store in UTC everywhere (database, API). The API returns ISO 8601 with
  offset (`2026-11-02T09:44:00Z`). Convert to the user's zone (default `Asia/Damascus`, or the
  product's market zone) only for display and for "business day" logic.
- **Money:** integer minor units (`amount_minor`, plus a `currency` code) or `decimal(12,2)`.
  Never floats. ⏳ pick one company-wide.
- **IDs in URLs:** use ULIDs or slugs for public URLs; never expose sequential IDs for things a
  user shouldn't be able to enumerate (orders, bookings).
- **Phone numbers:** stored in E.164 (`+9639...`), normalized in one helper.

## 9. Arabic first, RTL always

- Every UI supports RTL from the first commit. Use logical properties only (`ms-`, `me-`, `ps-`,
  `pe-`, `start`, `end`; `EdgeInsetsDirectional` in Flutter). Never `left`/`right` for layout.
- Default locale `ar`. Extra locales (`en`, `de`) are added through the same translation files.
- Arabic error and validation messages come from the backend translation files (`lang/ar`),
  so every client shows the same text.

## 10. Security basics

- HTTPS in every deployed environment, including staging. No plain-HTTP-by-IP deployments.
- Deploy with a dedicated non-root user and a restricted SSH key (see Dawri's forced-command key).
- Change all seeded/default admin passwords before go-live; never document real ones in a README.
- Rate-limit login, OTP and any public form.
- Run `composer audit` / `npm audit` in CI and fix high/critical issues within a week.
- Production backups are automatic, daily, copied **off the server**, and a restore is tested
  once per month.

## 11. Definition of done

A task is done when: the code follows the stack skill, tests for the new behavior exist and pass,
CI is green, docs/`.env.example`/`CLAUDE.md` are updated if setup changed, and the PR was reviewed.
