# TBI App — Deployment Guide

## Overview

This Flask + PostgreSQL application (Employee Feedback System) can be hosted for **free** using a combination of cloud platforms. This guide covers three deployment options ranked by ease and cost.

---

## Recommended Option 1: Render.com (Best Free Tier for Flask + PostgreSQL)

Render offers a free web service tier and a free PostgreSQL database (1 GB, free for 90 days, then \$7/mo). For **permanent free hosting**, combine Render's free web service with **Supabase** (see Option 3).

### Prerequisites

1. **Push to GitHub first:**
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/YOUR_USERNAME/tbi-app.git
   git push -u origin main
   ```

### Step-by-Step: Deploy to Render

#### Step 1 — Sign up at Render

- Go to [render.com](https://render.com) and sign up (free, no credit card required for free tier).

#### Step 2 — Create a PostgreSQL Database

1. In the Render dashboard, click **New → PostgreSQL**.
2. Set:
   - **Name:** `tbi-db`
   - **Database:** `employee_feedback`
   - **Plan:** Free
3. Click **Create Database**.
4. After creation, copy the **Internal Database URL** (starts with `postgresql://`).

> [!NOTE]
> The free PostgreSQL database on Render expires after **90 days**. After that it's \$7/mo. For a truly free-forever database, use Supabase (see Option 3).

#### Step 3 — Create a Web Service

1. In the Render dashboard, click **New → Web Service**.
2. Connect your GitHub repository (`tbi-app`).
3. Configure:
   - **Environment:** Python 3
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `gunicorn app:app`
   - **Plan:** Free
4. Under **Environment Variables**, add:
   | Key | Value |
   |-----|-------|
   | `SECRET_KEY` | Generate a random string (e.g. `openssl rand -hex 32`) |
   | `DATABASE_URL` | The Internal Database URL from Step 2 |

5. Click **Create Web Service**.

#### Step 4 — Run the SQL Schema

After the first successful deploy:

1. In Render dashboard, go to your **PostgreSQL database → Shell** (or use the external connection URL with `psql`).
2. Run the schema file:
   ```bash
   psql "YOUR_DATABASE_URL" -f employee_feedbaxk_schema.sql
   ```
3. The app automatically creates the `question_departments` table on first startup via `ensure_question_department_table()`.

#### Step 5 — Auto-Deploy

Render automatically redeploys whenever you push to `main`. That's it!

---

## Option 2: Railway.app

Railway provides \$5 free trial credit (no time limit — lasts until credit runs out), then \$5/mo on the Hobby plan.

### Step-by-Step: Deploy to Railway

1. Go to [railway.app](https://railway.app) and sign up with GitHub.
2. Click **New Project → Deploy from GitHub Repo** → select `tbi-app`.
3. Railway auto-detects Python and uses `Procfile` / `railway.json`.
4. Add a **PostgreSQL plugin**:
   - In your project, click **New → Database → PostgreSQL**.
   - Railway automatically sets the `DATABASE_URL` environment variable.
5. Add `SECRET_KEY` environment variable in the Railway dashboard.
6. Railway deploys automatically on every push to `main`.
7. After first deploy, run the schema via Railway's **PostgreSQL → Connect → Query** interface, or use `psql`:
   ```bash
   psql "YOUR_RAILWAY_DATABASE_URL" -f employee_feedbaxk_schema.sql
   ```

> [!TIP]
> The `railway.json` file in this repo pre-configures the build and restart policy for Railway.

---

## Option 3: Supabase (Database) + Render (Web) — Truly Free Forever

This combination gives you **permanent free hosting** with no expiry:
- **Supabase** free tier: PostgreSQL, 500 MB, **no expiry**
- **Render** free tier: Web service, **no expiry**

### Step 1 — Set up Supabase

1. Go to [supabase.com](https://supabase.com) and sign up (free, no credit card).
2. Click **New Project** → name it `tbi-app`.
3. Choose a region close to your users.
4. Wait for the project to provision (~2 minutes).
5. Go to **Settings → Database** and copy the **Connection String** (URI format):
   ```
   postgresql://postgres:[YOUR-PASSWORD]@db.xxxx.supabase.co:5432/postgres
   ```
6. Go to **SQL Editor** and paste + run the contents of `employee_feedbaxk_schema.sql`.

> [!IMPORTANT]
> Supabase uses `public.uuid_generate_v4()` by default — the `uuid-ossp` extension is pre-installed. The schema is compatible out of the box.

### Step 2 — Set up Render Web Service

Follow **Option 1, Steps 1–3** but instead of using Render's PostgreSQL:
- Set `DATABASE_URL` to your **Supabase connection string** (from Step 1, Step 5 above).
- No need to create a Render PostgreSQL database.

### Step 3 — Deploy

Push to GitHub → Render auto-deploys. The app connects to Supabase.

---

## Local Development Setup

1. **Install Python 3.11** from [python.org](https://python.org).

2. **Create a virtual environment:**
   ```bash
   python -m venv venv
   venv\Scripts\activate   # Windows
   # source venv/bin/activate   # Linux/Mac
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

4. **Set up a local PostgreSQL database:**
   ```bash
   createdb employee_feedback
   psql employee_feedback -f employee_feedbaxk_schema.sql
   ```

5. **Create a `.env` file** (copy from `.env.example`):
   ```bash
   copy .env.example .env
   ```
   Edit `.env`:
   ```
   SECRET_KEY=any-local-secret-key
   DATABASE_URL=postgresql://postgres:postgres@localhost:5432/employee_feedback
   ```

6. **Load environment variables and run:**
   ```bash
   # Windows PowerShell:
   $env:SECRET_KEY="any-local-secret-key"
   $env:DATABASE_URL="postgresql://postgres:postgres@localhost:5432/employee_feedback"
   python app.py
   ```
   Or use a tool like `python-dotenv` to auto-load `.env`.

7. Open [http://127.0.0.1:5000](http://127.0.0.1:5000) in your browser.

---

## Post-Deployment Checklist

After deploying to any platform:

- [ ] **Run the SQL schema** against your production database (`employee_feedbaxk_schema.sql`)
- [ ] **Verify `question_departments` table** was auto-created (check app logs for "Could not ensure..." errors)
- [ ] **Create your first superadmin** — insert directly into `admin_users` table:
  ```sql
  INSERT INTO public.admin_users (name, email, password_hash, role)
  VALUES (
    'Admin Name',
    'admin@example.com',
    -- Generate hash with: python -c "from werkzeug.security import generate_password_hash; print(generate_password_hash('yourpassword'))"
    'pbkdf2:sha256:...',
    'superadmin'
  );
  ```
- [ ] **Test login** at `/login`
- [ ] **Test language switching** (English ↔ French)
- [ ] **Set `SECRET_KEY`** to a secure random value in production (never use the default)
- [ ] **Set `DATABASE_URL`** to your production database URL

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `SECRET_KEY` | **Yes** (in production) | Flask session signing key. Use a long random string. |
| `DATABASE_URL` | **Yes** (in production) | PostgreSQL connection string. Format: `postgresql://user:pass@host:5432/dbname` |

> [!CAUTION]
> Never commit a real `SECRET_KEY` or `DATABASE_URL` to version control. The `.gitignore` excludes `.env` files, but double-check before pushing.

---

## File Reference

| File | Purpose |
|------|---------|
| `app.py` | Main Flask application |
| `employee_feedbaxk_schema.sql` | PostgreSQL schema — run once on new database |
| `requirements.txt` | Python dependencies |
| `Procfile` | Process definition for Render/Heroku |
| `runtime.txt` | Python version hint for Render |
| `.python-version` | Python version for pyenv / Railway |
| `render.yaml` | Render blueprint (IaC deploy config) |
| `railway.json` | Railway deployment config |
| `.env.example` | Template for local environment variables |
| `.gitignore` | Files excluded from git |
| `babel.cfg` | Flask-Babel extraction config |
| `translations/` | i18n translation files (French) |
