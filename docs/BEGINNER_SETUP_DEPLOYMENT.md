# Beginner Setup, Deployment, and Maintenance Guide

This guide covers a clean local installation and a Render + Neon PostgreSQL deployment. Work in a development/staging environment first. Never test against a live school database or configure live payment keys until the system has been reviewed and verified.

## 1. What you need

Install PHP 8.3+, Composer 2, Node.js 20+, npm, Git, and one database engine. PHP needs `bcmath`, `curl`, `ctype`, `dom`, `exif`, `fileinfo`, `gd`, `intl`, `mbstring`, `openssl`, `tokenizer`, `xml`, `zip`, and the PDO driver for your selected database. On Windows, XAMPP/Laragon can provide PHP and MySQL; confirm PHP is 8.3 or newer. On macOS/Linux, use a current PHP package and Composer installation.

Create an empty project directory by cloning the repository, then enter it:

```bash
git clone --branch development https://github.com/addigitaltech/Emmaculate.git
cd Emmaculate
```

## 2. Local installation

Copy the template and install dependencies/build browser assets:

```bash
cp .env.example .env
composer install
npm ci
npm run build
php artisan key:generate
```

On Windows PowerShell, use `Copy-Item .env.example .env` in place of `cp`. Open `.env` in a text editor. Keep it private and do not upload it to GitHub. Set `APP_ENV=local`, `APP_DEBUG=true`, and `APP_URL=http://localhost` for local work. Choose one database below before running migrations.

### Option A: SQLite (quickest local setup)

Create an empty database file:

```bash
mkdir -p database
touch database/database.sqlite
```

Use these settings:

```dotenv
DB_CONNECTION=sqlite
DB_URL=
DB_DATABASE=/absolute/path/to/Emmaculate/database/database.sqlite
DB_FOREIGN_KEYS=true
```

On Windows, use an absolute Windows path with forward slashes, such as `C:/projects/Emmaculate/database/database.sqlite`.

### Option B: MySQL or MariaDB

Create a database and a dedicated application user in your MySQL administration tool. Grant that user privileges only on this application database. Do not connect as `root`.

```dotenv
DB_CONNECTION=mysql
DB_URL=
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=emmaculate
DB_USERNAME=emmaculate_app
DB_PASSWORD=use-a-long-unique-password
DB_CHARSET=utf8mb4
DB_COLLATION=utf8mb4_unicode_ci
DB_FOREIGN_KEYS=true
```

A cPanel database often has a hosting-account prefix in both the database and user names. Follow [`CPANEL_DEPLOYMENT.md`](CPANEL_DEPLOYMENT.md) for those steps.

### Option C: PostgreSQL

Create an empty database and a non-superuser database role. Grant the role access to that database and its schema. Set:

```dotenv
DB_CONNECTION=pgsql
DB_URL=
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=emmaculate
DB_USERNAME=emmaculate_app
DB_PASSWORD=use-a-long-unique-password
DB_CHARSET=utf8
DB_SSLMODE=prefer
```

For a hosted PostgreSQL provider, require TLS: set `DB_SSLMODE=require`. You may use `DB_URL=postgresql://USER:PASSWORD@HOST:5432/DATABASE?sslmode=require` instead; if using a URL, escape/percent-encode reserved characters in username/password. `DB_URL` takes precedence over the separate `DB_*` host fields.

### Run migrations and seed initial data

After selecting a database, build the schema and standard starter content, create the public storage link, and create the first administrator:

```bash
php artisan migrate --seed
php artisan storage:link
php artisan school:create-super-admin
php artisan serve
```

The admin command asks for the name/email/password; the password is entered without being displayed. Use a unique random password stored in a password manager. Open `http://127.0.0.1:8000` for the site and `http://127.0.0.1:8000/admin` for administration. To use the Vite development server for hot reload, run `npm run dev` in another terminal.

## 3. How to switch databases safely

Change `.env` to exactly one connection. Before pointing a local checkout at a different database, confirm the hostname and database name; never paste production credentials into a test file. Clear Laravel's cached config after changing environment values:

```bash
php artisan config:clear
php artisan migrate:status
```

Use an empty disposable database for migrations and feature tests. Migrations alter schema; never run them against an existing production database unless you have a verified backup and explicit change plan.

## 4. Run the tests

Run tests in testing mode and build assets:

```bash
APP_ENV=testing php artisan test
npm run build
```

The test suite defaults to in-memory SQLite. The GitHub Actions workflow also has MySQL and PostgreSQL services so the same suite can be exercised on the three database families. Use separate disposable databases for each DB run. Payment tests fake provider HTTP responses: they should not require real credentials or create real charges.

For test runs, ensure the test process really uses `APP_ENV=testing`; never let a production environment leak into test commands. If login tests show 419/session errors, check the effective `APP_ENV`, `APP_URL` (tests expect localhost), session driver, and CSRF/session setup before changing production security settings.

## 5. Configure email, payments, Turnstile and SMS

### Email

A working mail transport is required for password reset and payment receipt messages. Create an SMTP credential at your mail provider, then fill `MAIL_MAILER`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_SCHEME`, `MAIL_FROM_ADDRESS` and `MAIL_FROM_NAME` in the private environment. Do not use a personal mailbox password if the provider supports app-specific credentials. See [`EMAIL_SETUP.md`](EMAIL_SETUP.md).

### Payment provider sandbox

1. Create a test/sandbox integration in Paystack and/or Flutterwave.
2. Store the provider's **test secret key** and webhook verification secret in local `.env` or the staging host's encrypted environment settings. Never add secrets to GitHub or client-side JavaScript.
3. Keep `PAYSTACK_BASE_URL=https://api.paystack.co` and `FLUTTERWAVE_BASE_URL=https://api.flutterwave.com/v3`; use test-mode credentials from the provider dashboard. The same API hosts support test credentials.
4. Enable the corresponding gateway in `/admin` only after configuration is ready. Configure provider callbacks to `https://YOUR-HOST/api/webhooks/payments/paystack` and `https://YOUR-HOST/api/webhooks/payments/flutterwave`.
5. Use provider test cards/accounts only. Verify an intentionally failed/cancelled transaction remains unpaid, a success is verified server-side and recorded once, and replayed webhooks do not double-credit a fee. Do not send real money for a test.
6. Before real payments, replace test keys with production keys in a controlled change, verify webhook signature settings, run a low-value authorized live transaction, and have finance reconcile it.

Blank gateway keys disable online checkout. Never trust a user's browser return as proof of payment. Read [`PAYMENTS.md`](PAYMENTS.md) for school fee setup, installments, refunds and callback details.

### Turnstile and SMS

Turnstile is optional. Set both `TURNSTILE_SITE_KEY` and `TURNSTILE_SECRET_KEY`, list the deployed hostnames in `TURNSTILE_HOSTNAMES`, and keep `TURNSTILE_STRICT_CONTEXT=true`. Use Cloudflare's documented test keys only in a test environment. Termii SMS is optional; set `SMS_DRIVER=termii`, `TERMII_API_KEY`, and `TERMII_SENDER_ID` to enable it. Keep all secret keys private.

## 6. Create all role accounts for testing

The eight supported roles are `Super Admin`, `Website Admin`, `Content Editor`, `Results Admin`, `Finance Admin`, `Teacher`, `Student`, and `Parent`. Create accounts through the administration area after the first Super Admin has been provisioned. Assign exactly the role(s) needed; a student should not also be a staff user. Teachers need valid teacher profiles and class/subject assignments to test score entry. Parents need a parent profile and linked student records; students need a student record and student ID/PIN setup. Finance and results roles need corresponding school data to exercise their pages.

For a role acceptance test, verify the permitted route/action works, a different role cannot access it by entering the URL directly, and the record scope is correct. Use disposable test identities with fake names/emails and nonproduction school data. Do not expose temporary passwords in a public report. Provision real user accounts only with an explicit approved identity/email list and deliver generated credentials through an approved private channel; force a password change when the product supports it.

## 7. Render + Neon PostgreSQL deployment

### 7.1 Prepare a Neon database

1. Sign in at [Neon Console](https://console.neon.tech/) and create/select the intended project in the correct organization and region. Confirm you are not reusing an unrelated project.
2. Keep a dedicated production branch and separate development/test branches. For testing, create a Neon child branch or a wholly separate project; never run tests on production.
3. Ensure the production branch has a running compute endpoint and a database/role for this application. Restrict who can read connection details.
4. Copy the **pooled** connection string for the web service. Keep it private. The direct connection can be used for migration/schema work if your provider/network settings require it. Require TLS with `sslmode=require`.
5. Before applying migrations to an existing database, inspect its schema, take a restore point/backup, and confirm with the site owner whether existing school data must be preserved. Never drop or reset tables to “fix” a deployment.

### 7.2 Create the Render service

1. In Render, choose **New > Web Service** and connect the GitHub repository `addigitaltech/Emmaculate`.
2. Select the tested development branch (or an approved release branch), not the source repository's `main` branch. Turn auto-deploy off until the first deployment is reviewed if you prefer manual release control.
3. Choose the **Docker** runtime and use the repository root as the Docker context and `Dockerfile` as the Dockerfile path. Do not add a separate PHP build command; the Dockerfile already builds frontend and PHP dependencies.
4. Set Render's health-check path to `/up`. Render provides `PORT`; do not set it manually. Select a region near the Neon compute region to reduce latency.
5. Start with the free/lowest-cost service only if its resource and persistence limitations are acceptable. Persistent disks, uninterrupted uptime, custom domains and higher capacity can have a charge; check Render's current pricing before selecting a paid plan or disk.

### 7.3 Add Render environment variables

In Render **Environment**, add values via the protected UI (never paste them into source code, a support issue, or deployment logs):

- `APP_ENV=production`, `APP_DEBUG=false`, `APP_URL=https://YOUR-SERVICE.onrender.com`, and a fresh `APP_KEY` generated once with `php artisan key:generate --show` in a private local environment. Keep this exact key unchanged between deploys; changing it can invalidate sessions/encrypted data.
- `DB_CONNECTION=pgsql`, `DB_URL=<private Neon pooled URL>`, `DB_SSLMODE=require`.
- `SESSION_DRIVER=database`, `CACHE_STORE=database`, `QUEUE_CONNECTION=database`, `SESSION_ENCRYPT=true`, `SESSION_SECURE_COOKIE=true`.
- Mail provider settings (`MAIL_MAILER`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_SCHEME`, `MAIL_FROM_ADDRESS`, `MAIL_FROM_NAME`).
- Optional production Turnstile keys and hostname only after the exact service domain has been added in Cloudflare.
- Leave payment and SMS secrets blank until provider sandbox verification and webhook registration are complete. Use test keys only on staging.

`DB_URL` overrides individual DB connection fields. If a service URL is rotated, update Render's environment and redeploy. Do not print environment values while troubleshooting.

### 7.4 Persist uploads and provision the administrator

The default `public` filesystem is local to the container. Uploads can disappear when an ephemeral container restarts or deploys. Choose one:

- Attach a Render persistent disk mounted at `/var/www/html/storage/app/public`, and keep one web instance if files are stored on that disk; or
- Configure S3/R2-compatible object storage with the `AWS_*` values and `FILESYSTEM_DISK=s3`, checking bucket CORS/access policy and public URL behavior.

Public-disk assets are public by design. Do not upload confidential student documents there. Back up media separately.

The Docker entrypoint runs `php artisan migrate --force`, `php artisan storage:link`, `php artisan school:init`, and Laravel caches before Apache starts. First startup can take longer. For the initial Super Admin, prefer a protected Render Shell/one-off command: run `php artisan school:create-super-admin` and enter the password interactively. If you temporarily use `SCHOOL_ADMIN_NAME`, `SCHOOL_ADMIN_EMAIL`, `SCHOOL_ADMIN_PASSWORD` for `school:init`, use a unique strong password and remove those three values immediately after successful initialization. Do not leave bootstrap credentials configured.

### 7.5 Deploy and verify

1. Trigger the initial deploy after all required environment variables are present.
2. Review the build log for Composer/npm failure and the runtime log for database migration or bootstrap errors. Check that no secrets appear in logs.
3. Visit `/up` and the homepage over HTTPS. Test `/admin` login, a public page, password reset email delivery, portal login, a non-destructive admin workflow and a test payment on the staging environment.
4. Confirm a test checkout is verified by the provider and the application records it once. Confirm no successful payment is created from a browser return alone.
5. Verify upload persistence after a controlled restart/redeploy before relying on media uploads.
6. Check Render events/health status and Neon connection/compute status. Record the exact deployed commit and environment, excluding secrets.

The entrypoint currently applies migrations whenever the web container starts. Use a single service instance for a schema rollout; do not run multiple concurrent app instances against a migration that is not safe for concurrent execution. Back up first. The `/up` health endpoint indicates framework reachability; separately check whether migrations and database-backed pages work.

## 8. Admin and day-to-day maintenance

- Visit `/admin` and create individual users under the user/role controls. The Super Admin should not be shared. Grant only the role required for each person's job.
- Use CMS controls for website content and media; school settings for branding and terms; academic structure for classes/subjects/sessions; Results for entry/publish; Fees & payments for fee types, assignments, receipts, reports and gateway settings.
- Review audit logs after important actions. Remove accounts promptly when access is no longer needed.
- Back up PostgreSQL and user-uploaded files before migrations or bulk edits. Test restores to a separate branch/database. Keep at least one backup outside the hosting account.
- Apply dependency and OS updates in a development branch first; run tests and build; review the diff; deploy at a low-traffic time; verify `/up`, login, mail, database and payment callbacks. Keep rollback instructions and previous known-good commit.
- Rotate credentials after suspected exposure and update the hosting environment store. Preserve `APP_KEY` unless you have a planned key-rotation/migration procedure.
- Monitor Render events/logs and Neon usage, compute, connection limits, backups and slow queries. Redact personal data, payment references and secrets from diagnostic reports.

## 9. Troubleshooting

| Symptom | Checks and safe fix |
|---|---|
| `could not find driver` | Install/enable the PDO module for the selected database (`pdo_sqlite`, `pdo_mysql`, or `pdo_pgsql`), restart PHP, and verify with `php -m`. |
| Database connection refused | Check DB host, port, firewall/allowlist, service status, Neon compute, username, password and database name. A non-empty `DB_URL` overrides the separate DB fields. |
| Migration fails | Read the first migration error, verify the DB role has schema/table permissions, check version compatibility, and back up first. Never drop production tables as a shortcut. |
| Render restarts before becoming healthy | Review startup logs for DB reachability, migration and permission errors. Ensure Render's injected `PORT` is not overridden and health check is `/up`. |
| Login test returns 419 | Set `APP_ENV=testing`, use `APP_URL=http://localhost`, array session driver and a disposable test DB. Do not weaken production CSRF/session controls to silence a test. |
| Turnstile rejects a valid local/deployed login | Check that site and secret keys are both present; hostname and form action must match the exact configured widget; keep strict context on for real keys. |
| Password reset/receipt does not arrive | Verify SMTP host/port/TLS/user/from-address and provider logs; check spam and sender-domain verification. Do not put SMTP credentials in source. |
| Gateway option is missing or checkout disabled | Confirm gateway is enabled in admin, test secret key is in the environment, Render redeployed after a secret change, and the environment uses sandbox/test keys. Never copy live keys into staging. |
| Uploaded images disappear after deploy | Local container storage is ephemeral. Configure a persistent disk or S3/R2, verify bucket/disk permissions and public URLs, and restore existing media from backup. |
| `APP_KEY`/sessions behave differently after deploy | Keep one stable private `APP_KEY` across instances/deploys. Do not regenerate it on every startup. |
| Changes to `.env` seem ignored | Run `php artisan config:clear` locally; on Render update Environment and redeploy. Do not commit a generated config cache containing secrets. |

For security-sensitive incidents, disable affected integration credentials at the provider, rotate them, inspect audit and hosting logs, preserve evidence, and contact the application owner before restoring service.
