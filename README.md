# Star Media Group — Practical Test

A cookie-consent-compliant 4-page website (Home, About, Privacy
Policy, Terms & Conditions) plus a secured admin portal to review
submitted consent decisions.

Two sibling projects, each its own repo, pulled in here as git
submodules:

| Repo | What it is | README |
|------|------------|--------|
| [`star-be`](https://github.com/kimcys/star-be) | PHP 8 + MySQL JSON API — consent tracking, CSRF, admin auth, PHPUnit tests, OpenAPI docs | [star-be/README.md](star-be/README.md) |
| [`star-fe`](https://github.com/kimcys/star-fe) | Angular 22 + Tailwind v4 SPA — the 4 public pages, the admin portal UI, dark mode, full Vitest coverage | [star-fe/README.md](star-fe/README.md) |

This root README is the fastest path to running **both together**.
For anything specific to one side (environment variables, API
endpoints, component structure, etc.), see that project's own README —
this file intentionally doesn't duplicate that detail.

## Cloning

Because `star-be` and `star-fe` are submodules, a plain `git clone`
leaves both directories empty. Clone with submodules included:

```bash
git clone --recurse-submodules https://github.com/kimcys/Star-Media-Group-Assessment.git
```

Already cloned without that flag? Fetch them into the existing checkout:

```bash
git submodule update --init
```

## Why an Angular frontend?

requirements describes the simplest baseline implementation — a
PHP-rendered website. What's here instead is a deliberate,
production-shaped split: **PHP + MySQL owns every piece of required
logic** (the consent API, cookie handling, GUID generation, DB
persistence, admin auth) behind a real JSON API, and **Angular owns
presentation** on top of it. Every functional requirement in the brief
— the exact consent-box wording, the GUID/timestamp/version cookie,
the 1-year/1-day expiry, the DB logging, the banner reappearance
rules, the Terms & Conditions/Privacy Statement links — is implemented
in `star-be` and verified independently of the frontend.

The reasoning: An API-first backend consumed by a typed SPA is closer to how a real
product would be built than a monolithic PHP-templated site — it
demonstrates API design, CORS/CSRF handling, and frontend architecture
in addition to the core PHP/MySQL requirement, rather than instead of
it. `star-be` also keeps a server-rendered `admin/*.php` portal as a
no-JS fallback, so the one bonus feature the brief explicitly calls
out (a secured admin view) works with or without the Angular layer.

## Quick start — run everything with one command

Requires [Docker Desktop](https://www.docker.com/products/docker-desktop/)
(or another Docker Engine + Compose) — no local PHP, MySQL, or Node
install needed.

```bash
cd star-be
docker compose up -d --build
```

This starts all three services defined in
[`star-be/docker-compose.yml`](star-be/docker-compose.yml):

| Service | What | URL |
|---------|------|-----|
| `web`   | The Angular frontend, built and served via nginx | http://localhost:4200 |
| `app`   | The PHP JSON API | http://localhost:8000 |
| `db`    | MySQL 8 (schema auto-imported on first boot) | internal only — not published to the host |

> If port `4200` is already taken by a local `ng serve`/`npm start`,
> stop that first — the container needs the port free.

### Create an admin account

The admin portal (`/admin/login` on the frontend) needs a user in the
database. Run this once, from `star-be`, after the containers are up:

```bash
docker compose exec app php bin/create_admin.php admin YourPasswordHere123
```

Password must be 8+ characters.

### Using it

- Public site: http://localhost:4200
- Admin login: http://localhost:4200/admin/login
- Backend API root: http://localhost:8000/api/
- Swagger / API docs: http://localhost:8000/docs/api-docs.html

### Stopping

```bash
docker compose down       # stops containers, keeps the database volume
docker compose down -v    # also wipes the database — start fresh next time
```

## Running each side manually (no Docker)

Both are fully documented in their own READMEs:

- **Backend**: [star-be/README.md → Local setup (manual, no Docker)](star-be/README.md#local-setup-manual-no-docker)
  — PHP built-in server + a local MySQL install.
- **Frontend**: [star-fe/README.md → Local setup](star-fe/README.md#local-setup)
  — `npm install && npm start` (`ng serve`, http://localhost:4200).

If you run the backend manually instead of via Docker, the frontend's
API base URL (`star-fe/src/environments/environment.ts`, read via
`api.config.ts`) already defaults to `http://localhost:8000`, so no
change is needed there.

## Things that matter across both sides

- **`localhost` vs `127.0.0.1`** — always use `http://localhost:4200`
  for the frontend and `http://localhost:8000` for the backend, not
  `127.0.0.1`. The backend's session/CSRF/consent cookies are all
  `SameSite=Lax`; `localhost` and `127.0.0.1` count as different sites
  for that purpose, so mixing them means cookies get set but silently
  never sent back — the consent banner would then reappear on every
  reload even after accepting. Both READMEs call this out individually
  too, but it's the single most common way to get stuck.
- **CORS** — the backend only accepts cross-origin requests from
  whatever `CORS_ALLOWED_ORIGIN` is set to (`http://localhost:4200` by
  default, in both `docker-compose.yml` and `.env.example`). Change it
  in one place if the frontend ever runs on a different origin.
- **Map** — the About page embeds the office location via
  OpenStreetMap's free embed — no API key, account, or billing, so it
  renders correctly with zero setup (see
  [star-fe/README.md → Map](star-fe/README.md#map)).
- **Design system / dark mode** — the frontend's whole visual language
  (colors, type scale, spacing, dark mode) is centralized as Tailwind
  v4 `@theme` tokens in `star-fe/src/styles.css`; see
  [star-fe/README.md → Design system](star-fe/README.md#design-system)
  and [→ Dark mode](star-fe/README.md#dark-mode) if touching styling.

## Testing

```bash
# Backend — PHPUnit, in-memory SQLite, no live MySQL needed
cd star-be
composer install
vendor/bin/phpunit --testdox

# Frontend — Vitest + jsdom, via Angular's test runner
cd star-fe
npm test
```

## Status against the requirements

For the full brief. Summary:

- ✅ 4-page public site, mobile-responsive
- ✅ Blocking cookie-consent banner (GUID + timestamp + version cookie,
  1-year accept / 1-day decline expiry, DB-logged)
- ✅ Terms & Conditions / Privacy Statement links wired from the
  consent banner
- ✅ Bonus: secured admin portal (login + dashboard) to view submitted
  consent decisions, with per-account lockout tracking
- ✅ Bonus: OpenAPI/Swagger docs, automated PHPUnit + Vitest test
  suites, Dockerized end-to-end setup, dark mode

Both sub-repo READMEs have a more detailed per-feature status list.
