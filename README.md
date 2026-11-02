# Emmaculate Academy School Platform

A Laravel 12 school platform with a public CMS, admissions, student/parent/teacher portals, results and report cards, fee management, online payments, communications, and role-based administration. School branding, academic structure and portal switches are configured from `/admin`.

## Requirements

- PHP 8.3+ with `bcmath`, `curl`, `ctype`, `dom`, `exif`, `fileinfo`, `gd`, `intl`, `mbstring`, `openssl`, `tokenizer`, `xml`, `zip`, and the PDO driver for your chosen database (`pdo_sqlite`, `pdo_mysql`, or `pdo_pgsql`).
- Composer 2, Node.js 20+ and npm; a supported database: SQLite, MySQL/MariaDB or PostgreSQL 14+.
- Production requires HTTPS, a generated private `APP_KEY`, configured mail, durable file storage, backups, and environment-managed service credentials.

## Local setup

```bash
git clone --branch development https://github.com/addigitaltech/Emmaculate.git
cd Emmaculate
cp .env.example .env
composer install
npm ci
npm run build
php artisan key:generate
# Edit .env for one supported database before migrating.
php artisan migrate --seed
php artisan storage:link
php artisan school:create-super-admin
php artisan serve
```

Open `http://127.0.0.1:8000` and `/admin`. The admin command prompts for a name, email and strong password without echoing it. Passwords must be at least 12 characters and include upper/lowercase letters, a number and a symbol. For Vite live reload, run `npm run dev` in a second terminal. See [`docs/BEGINNER_SETUP_DEPLOYMENT.md`](docs/BEGINNER_SETUP_DEPLOYMENT.md) for beginner steps, XAMPP setup at [`docs/INSTALL_XAMPP.md`](docs/INSTALL_XAMPP.md), or cPanel at [`docs/CPANEL_DEPLOYMENT.md`](docs/CPANEL_DEPLOYMENT.md).

## Databases

Set exactly one connection. A non-empty `DB_URL` takes precedence over individual host fields.

**SQLite:** create `database/database.sqlite`, then configure `DB_CONNECTION=sqlite`, `DB_URL=`, `DB_DATABASE=/absolute/path/to/database/database.sqlite`, and `DB_FOREIGN_KEYS=true`. Best for local development and tests, not shared production use.

**MySQL/MariaDB:** create a database and least-privilege user, then set `DB_CONNECTION=mysql`, `DB_URL=`, `DB_HOST`, `DB_PORT=3306`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `DB_CHARSET=utf8mb4`, and `DB_COLLATION=utf8mb4_unicode_ci`. Do not use the database root account.

**PostgreSQL/Neon:** set `DB_CONNECTION=pgsql`, `DB_URL=postgresql://USER:PASSWORD@HOST:5432/DATABASE?sslmode=require`, and `DB_SSLMODE=require`. Alternatively leave `DB_URL` blank and use `DB_HOST`, `DB_PORT=5432`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, and `DB_SSLMODE`. Percent-encode special characters in URL credentials. Use a Neon pooled connection for web workers and a direct connection for schema work when pooling is unsuitable. Never point tests at production. Back up before `php artisan migrate --force` against a live database.

## Tests

```bash
APP_ENV=testing php artisan test
npm run build
```

SQLite is the default PHPUnit database. `.github/workflows/tests.yml` defines a full suite matrix for SQLite, MySQL and PostgreSQL. Run each database against its own empty test database, with `APP_ENV=testing` and appropriate `DB_*` values. Never use production data or credentials in tests.

## Features and roles

Roles: `Super Admin`, `Website Admin`, `Content Editor`, `Results Admin`, `Finance Admin`, `Teacher`, `Student`, and `Parent`. Server-side permissions protect administrative pages, records, results, fees and linked student information. Features include public CMS/media, admissions, school structure, student/staff management, result entry/publication and printable cards, PIN-based student result access, promotion/graduation, fee assignment, installments/full payments, receipts/refunds/payment reports, email/SMS notifications and audit logs. See [`docs/ADMIN_GUIDE.md`](docs/ADMIN_GUIDE.md).

## Payments (sandbox first)

Paystack and Flutterwave online checkout remain unavailable until their private secret keys are configured. Use provider **test/sandbox** credentials only during testing; automated tests fake provider HTTP calls and need no gateway account. Set keys only in ignored local `.env` or the host's protected environment variables:

```dotenv
PAYSTACK_SECRET_KEY=
FLUTTERWAVE_SECRET_KEY=
FLUTTERWAVE_SECRET_HASH=
FLUTTERWAVE_PUBLIC_KEY=
```

The server calculates fee balances and verifies status, amount, currency and reference before settlement. A browser return is not proof of payment; webhook signatures are checked. Configure the provider callback URLs in [`docs/PAYMENTS.md`](docs/PAYMENTS.md). Never commit or log gateway secrets or use live keys in a test environment.

## Render and Neon

Build using the repository `Dockerfile` and Docker runtime on Render. It builds PHP and Vite assets, listens on Render's injected `PORT`, runs migrations and the idempotent `school:init` bootstrap at startup, then starts Apache. Set Render's health path to `/up`; do not manually define `PORT`. Configure `APP_ENV=production`, `APP_DEBUG=false`, HTTPS `APP_URL`, generated `APP_KEY`, `DB_CONNECTION=pgsql`, private Neon `DB_URL`, `DB_SSLMODE=require`, and mail in Render Environment.

The public disk maps to local container storage, which is **ephemeral** unless you attach a persistent disk at `/var/www/html/storage/app/public` or configure S3/R2 with `FILESYSTEM_DISK=s3` and `AWS_*` values. Back up uploaded files as well as Neon. Do not place confidential records on the public disk. The Docker entrypoint runs migrations whenever a container starts; use a single web instance for schema rollouts, back up first, and avoid simultaneous deployment during schema changes. Detailed instructions and troubleshooting are in [`docs/BEGINNER_SETUP_DEPLOYMENT.md`](docs/BEGINNER_SETUP_DEPLOYMENT.md).

Create the first deployed Super Admin using the interactive `php artisan school:create-super-admin` in Render Shell/one-off command, or temporarily set `SCHOOL_ADMIN_NAME`, `SCHOOL_ADMIN_EMAIL` and a strong unique `SCHOOL_ADMIN_PASSWORD` before the first `school:init`. Remove bootstrap values immediately after provisioning. Never use a default password.

## Configuration and security

- Render/general environment template: [`.env.example`](.env.example).
- cPanel/MySQL template: [`.env.cpanel.example`](.env.cpanel.example).
- Step-by-step local/database/payment/Render/Neon/admin/maintenance guide: [`docs/BEGINNER_SETUP_DEPLOYMENT.md`](docs/BEGINNER_SETUP_DEPLOYMENT.md).
- Payments: [`docs/PAYMENTS.md`](docs/PAYMENTS.md); security: [`docs/SECURITY.md`](docs/SECURITY.md); email: [`docs/EMAIL_SETUP.md`](docs/EMAIL_SETUP.md).

Keep `.env`, database URLs, payment keys, Turnstile secrets, mail passwords, SMS tokens and bootstrap credentials out of Git, issues, screenshots and logs. Use HTTPS, unique individual accounts, least privilege, current dependencies and off-site backups. Set production `APP_DEBUG=false`.
