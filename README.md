# LocalFix — Vercel + Supabase

This project keeps the supplied vanilla-JS UI and URLs, serves static files from `public/`, and runs the Flask API as a Vercel Python Function backed by Supabase PostgreSQL.

## Requirements

- Python 3.12 (pinned by `.python-version`)
- A Supabase PostgreSQL project
- A GitHub repository and Vercel account
- Vercel CLI is optional for local development

## 1. Create the Supabase project

1. Create a project in [Supabase](https://supabase.com/dashboard) and choose a region near your users.
2. Save the database password securely. Do not put it in source control.
3. In the project dashboard, open **Connect** and copy the **Transaction pooler** connection string. Use the pooler for serverless connections. It normally uses port `6543` and a username containing the project reference.
4. URL-encode special characters in the database password. The connection should end with `?sslmode=require` (the application also enforces TLS when connecting).

Example shape only (not a credential):

```text
postgresql://postgres.<project-ref>:<URL-encoded-password>@<region>.pooler.supabase.com:6543/postgres?sslmode=require
```

## 2. Initialize the database and first admin

From the repository root, create a local environment file from `.env.example` and fill it locally with your Supabase pooler URL, a random JWT secret (at least 32 characters), and the first admin email/password. Never commit `.env`.

```bash
cp .env.example .env
# Edit .env with your own values; keep it private.
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
set -a
. ./.env
set +a
python init_db.py
```

`init_db.py` applies `schema.sql` and `seed.sql` idempotently, then creates the first admin. A rerun preserves the existing admin password and marketplace rows. If the chosen admin email already belongs to a non-admin user, initialization stops with a clear error; use a dedicated admin email.

Alternatively, paste and run `schema.sql`, then `seed.sql` in the Supabase **SQL Editor**. Run `python init_db.py` once afterward to create the initial admin account; reapplying the schema and seeds is safe.

The initializer requires `DATABASE_URL`, `JWT_SECRET`, `ADMIN_EMAIL`, and `ADMIN_PASSWORD`. The admin password must be at least 12 characters. The app reads `TOKEN_DAYS` (default `7`; allowed range `1–365`) when issuing JWTs.

## 3. Push the project to GitHub

Create an empty repository, then from this project directory:

```bash
git init
git add .
git commit -m "Prepare LocalFix for Vercel and Supabase"
git branch -M main
git remote add origin https://github.com/<your-account>/<your-repository>.git
git push -u origin main
```

Check that `.env` is not staged. `.gitignore` excludes it; `.env.example` contains placeholders only.

## 4. Import and deploy on Vercel

1. In Vercel, choose **Add New → Project**, import the GitHub repository, and leave the detected Python/Flask framework settings at their defaults.
2. Set the project Root Directory to the repository root. Do not set a static output directory or build command.
3. Add these environment variables for **Production**, **Preview**, and **Development**:
   - `DATABASE_URL` — Supabase Transaction Pooler URL, with URL-encoded password and TLS enabled.
   - `JWT_SECRET` — a unique random secret of 32+ characters. For example, generate locally with `openssl rand -hex 32` and paste the result into Vercel; do not commit it.
   - `TOKEN_DAYS` — optional, defaults to `7` (must be between `1` and `365`).
   - `ADMIN_EMAIL` and `ADMIN_PASSWORD` — the same first-admin values used when running `init_db.py`. These are initialization values; the initializer does not reset an existing admin's password.
4. Deploy. Vercel's Flask framework detection loads the `app` object in the root `app.py`; Flask's `/api/...` routes handle API requests. Vercel serves the contents of `public/**` as static assets/CDN files. The `functions` entry in `vercel.json` configures the Flask function and its duration; an extra path rewrite is not used so Flask receives the original `/api/...` URL. Existing root/folder HTML shells and flat redirect pages are retained under `public/`.
5. After deployment, visit `/admin-login.html` and sign in with the first-admin email/password. Then test customer/provider signup and public browsing.

### Account access

Customer and provider signup/sign-in use one **Email address or mobile number** field; enter either an email address or a phone number. Indian 10-digit mobile numbers are normalized for consistent lookup, and duplicate phone numbers cannot be attached to separate accounts. Provider sign-up creates the login first, then the provider applies through `/become-provider/`; existing providers use the Provider login link on `/login/`. The **Log out** action is on each account's My Profile page.

The app adds security headers without a CSP because the retained UI uses inline style attributes. Static assets are served by Vercel; Flask contains API routes only. Requests use a single short-lived PostgreSQL connection per invocation and reconnect when a connection is closed; there is no in-process session or rate-limit state.

### Location options

The marketplace offers manual city/area search and an explicit **Use current GPS** action. GPS is requested only after the user clicks; it takes a one-time position snapshot (not background tracking). Search coordinates are sent to LocalFix only for that request and are not persisted. Provider search uses them to sort providers who have saved a service pin, while providers without pins remain listed with distance unavailable. Customers may keep typing a service address or attach a GPS pin to a booking; a pin is saved with the booking and shared with the selected provider for that booking. Customer profiles and provider profiles can also store their own optional pins; provider pins are used for sorting and are not exposed in public profile responses. No reverse-geocoding service receives the coordinates. Browser location permission is enabled only for the same site origin; manual entry remains available if GPS permission is denied or unavailable.

### Help and Contact Center

The Help Center contains searchable FAQs. The Contact Center accepts authenticated or anonymous support requests, applies a basic IP-hash request limit, and stores tickets in PostgreSQL for admins at `/admin/contact-center.html`. Admins can mark a request in progress, resolve it, or reopen it. **The form does not send email automatically**; connect a mail provider separately if outbound replies/notifications are required. Do not collect passwords, OTPs, or payment-card numbers in support messages.

## 5. Troubleshooting

- **500 / missing configuration:** Confirm all required runtime variables are set in the Vercel environment for the deployment's environment. `DATABASE_URL` and `JWT_SECRET` are required by Flask.
- **Cannot connect to PostgreSQL:** Use the Supabase pooler rather than a direct database URL for serverless, check the pooler username/port, URL-encode password characters, and keep `sslmode=require`.
- **`init_db.py` reports an existing non-admin:** Use a separate admin email, or resolve the account deliberately in Supabase before retrying. The initializer will not silently promote an existing customer/provider.
- **Invalid login / role error:** Confirm the account exists and use the login page for its actual role (`/login/`, `/provider-login.html`, or `/admin-login.html`). Admin signup is intentionally disabled.
- **Empty marketplace:** Run `python init_db.py` against the intended Supabase project. The starter categories/providers are seeded with `ON CONFLICT DO NOTHING`, so reinitialization will not overwrite live ratings or profile edits.
- **Static page missing:** Confirm the Vercel Root Directory is the repository root and that the `public/` folder was pushed. The `public/` tree contains the preserved folder URLs and flat redirects.
- **Database setup or tests touched the wrong project:** Stop and inspect `DATABASE_URL` first. `pytest` wipes the schema only after confirming the database name contains `test`; do not point it at Supabase or any non-test database.

## API and deployment references

- [Deploy a Flask app on Vercel](https://vercel.com/docs/frameworks/backend/flask)
- [Vercel Python runtime](https://vercel.com/docs/functions/runtimes/python)
- [Vercel project configuration](https://vercel.com/docs/project-configuration/vercel-json)
- [Supabase connection strings](https://supabase.com/docs/guides/database/connecting-to-postgres)
