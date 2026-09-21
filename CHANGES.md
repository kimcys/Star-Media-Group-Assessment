# Resubmission changes

Changes made in response to the review feedback on the first submission
(see `feedback.md`). Two issues were named explicitly; both are fixed
below.

## 1. Consent banner appeared on Terms & Conditions / Privacy Policy pages

**Feedback:** "The accept/decline consent box currently appears before
the T&C and Privacy Statement pages... the consent box should not
appear on either page, allowing users to read the content before
deciding whether to accept or decline."

**Cause:** `ShellComponent` (`star-fe/src/app/layout/shell/shell.ts`)
wraps every public route — Home, About, Privacy Policy, Terms &
Conditions — and rendered `<app-consent-banner>` unconditionally.
`ConsentBannerComponent` only asked the backend "should the banner
show?" with no awareness of which page it was currently on, so the
banner (and its scroll lock) sat on top of the Privacy Policy and
Terms & Conditions pages before a visitor could read them.

**Fix:** Added a `hideConsentBanner` route-data flag to the
`privacy-policy` and `terms-conditions` routes
(`star-fe/src/app/app.routes.ts`). `ConsentBannerComponent` now tracks
Angular Router navigation events and checks the active route's data
before deciding whether to render, suppressing itself only on those
two routes. The banner's behavior on every other route — Home, About,
and reappearing after a decision expires — is unchanged.

**Files:** `star-fe/src/app/app.routes.ts`,
`star-fe/src/app/shared/components/consent-banner/consent-banner.ts`,
`star-fe/src/app/shared/components/consent-banner/consent-banner.html`.
Two new tests added in `consent-banner.spec.ts` covering the
suppressed and reappearing cases; full frontend suite (91 tests)
passes.

_star-fe commit: `9e39a45`_

## 2. PHP backend missing try/catch

**Feedback:** "The PHP backend code does not currently implement any
try/catch blocks."

**Cause:** Most endpoints already had try/catch around their
DB-touching logic, but five did not: `api/consent-status.php`,
`api/csrf-cookie.php`, `api/admin/me.php`, `api/admin/logout.php`, and
`admin/logout.php`. If any of them threw, execution fell through to
the global `set_exception_handler` in `includes/bootstrap.php`, which
replies in **plain text** — breaking the JSON contract every other API
endpoint follows, and leaving the plain-page logout with no
graceful fallback.

**Fix:** Added try/catch to all five, matching the existing pattern
used elsewhere in the codebase (e.g. `consent-handler.php`): log the
real error server-side via `error_log`, then return a clean JSON error
with an appropriate HTTP status (or, for the non-JSON `admin/logout.php`
page, log and still redirect to the login page rather than surface a
raw error).

**Files:** `star-be/api/consent-status.php`,
`star-be/api/csrf-cookie.php`, `star-be/api/admin/me.php`,
`star-be/api/admin/logout.php`, `star-be/admin/logout.php`. All five
pass `php -l`; the existing 19-test PHPUnit suite still passes
unchanged (those tests cover the `AdminAuth`/`ConsentManager` classes
directly, not these entry-point scripts, since the scripts have no
class to instantiate in a unit test).

_star-be commit: `61022f2`_

## 3. Extra improvements (not asked for by name — feedback invited "any other improvements")

These go beyond the two specific fixes above. In plain words, here's what changed and why it's better:

### Backend (`star-be`)

- **The app no longer runs as an admin/root user, and uses a real web server.** Before, the Docker container ran PHP's built-in test server — which PHP itself says is "not intended for production" — and it ran as the most powerful user on the system. Now it uses Apache (a proper web server) running as a low-privilege user. Think of it like the difference between giving a shop assistant a key to just the till versus a master key to the whole building — if something ever goes wrong, the damage it could do is much smaller.
- **Added a "health check" page** (`api/health.php`) — a simple page that just says "yes, I'm alive and can reach the database" or not. Docker checks this automatically to know if the app is actually working, not just switched on.
- **Cleaned up how errors get logged.** Before, error messages were scattered free-text notes. Now there's one small `Logger` helper that writes every error the same tidy way (as one line of structured data), so if something breaks, it's much easier to search the logs and see what happened.
- **Added tests that prove the try/catch fix actually works.** Instead of just trusting the code, there are now automated tests that call the real endpoints and check they respond correctly.
- **Added automatic testing on every code change (CI).** Every time code is pushed, a robot now automatically: installs everything, checks for known security issues in dependencies, runs all the tests, builds the Docker image, and scans that image for security problems — all without a human having to remember to do it.

### Frontend (`star-fe`)

- **The app now retries automatically if the server is slow or down**, instead of just giving up silently. If the connection times out, it quietly tries again a couple of times before showing a small "having trouble reaching the server" message — better than the user just seeing nothing happen.
- **The app's Docker container also no longer runs as root**, same reasoning as the backend above.
- **Added a linter** — a tool that automatically checks the code for common mistakes and bad patterns every time it runs. This project didn't have one wired up before.
- **Added automated browser tests (end-to-end tests).** These are robots that actually open the real website in a real browser and click around — checking the cookie banner appears/disappears correctly, and that admin login/logout works — just like a real visitor would, instead of only testing small pieces of code in isolation.
- **Added the same kind of automatic testing on every code change (CI)** as the backend: lint, unit tests, the new browser tests, a security check on dependencies, and a Docker image build + scan.

- **Turned on branch protection for `main` on both repos.** In plain words: nobody (other than the repo owner, for now, to keep solo iteration fast before the deadline) can push code straight to `main` anymore — it has to go through a pull request, and that pull request can't be merged unless the automatic checks above are all green. This is a real, live GitHub setting, not just something described in words.

Together, these make the two apps closer to how a real company would run them in production — safer containers, automatic checks before anything gets merged, branch protection backing that up, and tests that actually exercise the real app instead of just the code in isolation.

_All of the above is committed and pushed to `main` on both `star-be` and `star-fe`, and both repos' CI pipelines are green._
