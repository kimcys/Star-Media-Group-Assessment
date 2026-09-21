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
