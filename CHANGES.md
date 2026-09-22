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

**Important — this is a suppression on two specific pages, not a change
to the banner itself.** On Home and About, the banner still blocks
scrolling until Accept or Decline is clicked, on purpose: the original
brief (`Practical Test - S. Web Developer.pdf`, requirement 3) says
plainly, *"The user must not be able to scroll the page until the
consent box is addressed (either accepted or declined)."* That's a
stated requirement, not an oversight — only the reviewer's own feedback
(quoted above) carves out an exception for the Privacy Policy and Terms
& Conditions pages specifically, so that's the only place it was
changed.

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

## 4. Actually deployed live, with automatic deploys

Not asked for either, but it's real and it's up:

- **Live site:** https://aimanhakimcy.com (the public site) and https://api.aimanhakimcy.com (the backend API) — both with a real, genuine HTTPS padlock, not a fake or self-signed one.
- **It updates itself.** Every time code is pushed to `main` and passes every check above (tests, security scan, image scan), it automatically builds, ships, and restarts itself on the live server — no manual "upload the files" step, no one has to remember to deploy anything.
- **Branch protection is proven to actually work, not just switched on.** Every direct push made while building this was flagged by GitHub as breaking the "must go through a pull request" rule — it only went through because the repo owner is deliberately allowed to bypass it for now, to keep working fast before the deadline. That's a real enforcement mechanism doing its job, not just a setting nobody's tested.

### Real bugs this deployment caught — in plain words

Going from "works on my computer" to "actually live for a stranger to visit" surfaces problems that never show up any other way. Four real ones came up:

1. **A backend dependency was silently the wrong version.** A lock file had a package locked to a version that needs a newer PHP than the project is actually meant to run on. It worked by accident on the machine it was built on, and only broke the moment real automated testing ran it on the correct, official PHP version — exactly the kind of mismatch automated testing exists to catch. Fixed by regenerating the lock file properly on the right PHP version.
2. **The wrong visitor address was being recorded.** With a reverse proxy now sitting in front of the app (normal for any real deployment), the consent log was recording the *proxy's own* address instead of the actual visitor's — because the code was reading the wrong piece of information. Fixed by reading the correct forwarded-address header instead.
3. **The live site was trying to talk to "localhost."** A setting meant only for local testing was still hardcoded in, so the moment a real visitor loaded the live site, their browser tried to reach a server on *their own computer* — which obviously doesn't exist — breaking every single feature that talks to the backend (the cookie banner, the admin portal, everything). Fixed by making that address a build-time setting instead of a fixed value, so it's correct for wherever it's actually being deployed.
4. **A security cookie wasn't shared correctly between the two live addresses.** Once the site and its API moved to two different addresses (`aimanhakimcy.com` and `api.aimanhakimcy.com`), a cookie needed to protect form submissions from forgery wasn't visible to the website's own code by default — cookies don't automatically cross to a different address unless explicitly told they're allowed to. Fixed by explicitly telling that cookie which addresses it's shared across.

None of these four were guesses — each was actually observed happening on the live site, diagnosed, fixed, tested, and reverified live before moving on.

_Live at https://aimanhakimcy.com. Deploy config lives in the root repo's `deploy/` folder; both repos' CI pipelines handle build, test, scan, and deploy automatically on every push to `main`._
