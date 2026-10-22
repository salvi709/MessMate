# MessMate

Hostel mess feedback and menu management, built on the existing React 19 / TypeScript / Vinext application, Cloudflare Workers, and one shared D1 (SQLite) database. See [IMPLEMENTATION.md](IMPLEMENTATION.md) for the assessment, phased changes, and limitations.

## Local development

Use Node 22.13+ and npm. Install dependencies, check types, and build:

```sh
npm ci
npx tsc --noEmit
npm run build
```

For a fresh local database, execute each SQL migration in `drizzle/` in filename order, once. For an existing database, apply only migrations not previously applied. Example:

```sh
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0000_curly_prism.sql
# Then 0001_gray_sir_ram.sql, 0002_fresh_talon.sql, 0003_live_updates.sql
```

`npm run dev` serves the live development preview. To test the compiled Worker, use:

```sh
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js dev --config dist/server/wrangler.json --local --persist-to .wrangler/state --ip 127.0.0.1 --port 8787 --inspector-port 0
```

Stop the compiled Worker before rebuilding on Windows. Keep local servers on loopback. The legacy account bridge trusts identity headers supplied by Sites dispatch; never expose a bare Worker or development server directly to untrusted clients.

## Accounts and starting a pilot

1. Open the landing page and choose Student or Admin. Returning admins have a direct login link.
2. Students register with email/password and choose their institution/mess, or scan a QR to retain that exact mess through authentication.
3. New admins select a plan and request management access. They remain on the pending page without dashboard or setup privileges until operator approval.
4. After approval, finish institution, mess, serving-time, and QR setup. No sample ratings are created. Plan selection does not collect money or activate a paid subscription.
5. Add weekly menus or date-specific meals. Share the permanent mess QR. Ratings are accepted during serving hours; duplicate ratings and votes are rejected by database constraints.
6. Polls can be drafted, published, edited, or closed. Students see results only when permitted. Complaints and suggestions can receive a response and resolution status.
7. Open student and admin screens receive changes automatically. A 3-second revision check drives SSE notifications; hidden tabs pause updates, and a 60-second fallback keeps meal windows current.

Existing managers who used ChatGPT sign-in can choose “Used MessMate with ChatGPT before?” on the login page. This preserves their old account ID and data ownership. Typed email addresses alone cannot claim existing management grants.

The hosted Site retains its existing private audience. Sites sign-in is the outer access boundary, separate from the app's student/admin session. Guest QR settings never bypass that boundary. Public student access would require an explicit change to the Site audience.

## Approve a management request

Only a trusted database operator with access to this Site's production D1 database may perform this step. There is no public approval endpoint, default admin password, or first-visitor privilege.

1. Verify the applicant's identity and authority to manage the named institution through your institution's established process. Read the `accounts` row and corresponding `onboarding.draft`; match the request reference and email from the pending page.
2. Generate a statement scoped to that exact request and email:

```sh
node scripts/prepare-admin-approval.mjs REQUEST_ID VERIFIED_EMAIL OPERATOR_NAME
```

3. Review the generated SQL, then execute it using the trusted production D1 database administration channel. The generator does not connect to production. An `admin_approvals` record stores the approving operator and timestamp; a database trigger enables the account only if its role is `admin`.
4. The applicant selects “Check approval status” or logs in again to continue setup.

To revoke approval, delete that exact `admin_approvals` row through the trusted database administration channel. Its trigger removes management access on the next server request. Do not approve a request solely because it claims a known institution name or email domain.

Production database-operator access is required for approvals. If that access has not been assigned, new admin requests correctly remain pending. Database credentials must never be added to browser code or committed files.

## Security and data behavior

- Passwords: salted scrypt (`N=16384, r=8, p=5`) with bounded hashing memory; minimum 12 characters. Session cookies are opaque, HttpOnly, SameSite=Lax, Secure over HTTPS; only token hashes are stored, with a seven-day expiry.
- Authority: every admin mutation checks approval and the owned college or explicitly granted location. There is no API accepting an arbitrary role update. Students cannot grant themselves membership. Legacy emailed grants require a verified platform identity.
- All SQL values use prepared bindings. Role, plan, time, calendar date, content size, poll choices, and serving overlap are validated on the server. Same-origin JSON is required for mutations; login/signup/feedback/rating attempts are limited.
- Students receive only their own feedback history; admin ratings and members require authorization. Draft polls and hidden option counts are excluded server-side.
- Menus repeat weekly from an effective date. A dated override applies once. Deleted schedules retain tombstones and original rating snapshots. Serving windows cannot overlap or cross midnight.
- Local storage contains only selected-mess preferences. Menus, votes, feedback, announcements, approvals, and sessions are in D1.
- Anonymous ratings use device cookies plus rate limits. Clearing cookies can evade device-based duplicate detection; require login for stricter pilots.
- Reports preserve the existing bounded history and CSV formula escaping. Analytics are deterministic calculations over stored feedback, not invented AI output.

## Verification

With the built Worker on port 8787 and migrations applied:

```sh
node tests/auth-integration.mjs
node tests/connected-integration.mjs
node tests/connected-browser.mjs
```

The suites create local test accounts and fixtures; they must not target production. The auth suite records ignored `outputs/auth-fixture.json` for subsequent suites. Browser tests use Playwright with local Microsoft Edge; set `PLAYWRIGHT_PATH` if needed. Each fresh full run should start with the auth suite. Repeated runs can trigger genuine rate limits; wait for their ten-minute window instead of weakening production limits.

Auth checks cover registration, pending approval, login, roles, QR returns, setup, hashing, session expiry/logout, legacy migration, and preserved rating workflows. Connected checks cover daily/weekly meals, edits/deletes, poll permissions, duplicate votes, complaints/responses, SSE, persisted values, and cross-institution rejection. Browser checks cover two-account updates without reload, actual form submissions, QR downloads/print, onboarding, and 360/390/768/1440px layouts. Earlier `tests/integration.mjs` and `tests/browser.mjs` document the original platform-header-only prototype and are superseded by these session-based suites.

## Deferred integrations

As requested, Google sign-in, reset-email delivery, email verification delivery, and real payments are not connected. Their screens do not pretend to send emails, authenticate via Google, or collect money. Yearly pricing remains unavailable rather than inventing a price. No API keys or provider credentials are needed for the implemented database features; manual management approval still requires the trusted database operator described above.

## Publication

`.openai/hosting.json` retains the existing Site ID and D1 binding. Publish through the Sites source/build/package workflow, preserving the current audience. Production migrations are additive; local database state and ignored test outputs are excluded from the package. Never deploy test fixture accounts or reset the production database.
