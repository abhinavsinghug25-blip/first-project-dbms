# Roomwise — Classroom Occupancy & Scheduling System

Frontend (HTML/CSS/JS, original design kept) + **FastAPI** backend + **MySQL 8**, built from the
*Database Lab Project Roadmap*. Three login roles only: **ADMIN**, **CR**, **VIEWER**
(TEACHER is data, not a login role).

```
frontend/   original UI + api.js (auth/bootstrap) + integration.js (API wiring) + integration.css
backend/    FastAPI app (app/), unit + integration tests (tests/), requirements.txt, .env.example
db/         001_schema.sql  002_seed.sql  003_dcl.sql  004_demo_queries.sql  005_migration_example.sql  006_displaced_classes.sql  reset.sql
tools/      generate_seed.py, source_timetable.json, mock_server.py + browser_smoke_test.js (dev only)
docs/       API.md  DESIGN.md  TESTING.md  DEMO_SCRIPT.md  ER_DIAGRAM.png/.mmd
```

## Setup (≈10 minutes)

Prerequisites: Python 3.10+, MySQL 8.0.16+ (CHECK constraints are only enforced from 8.0.16).

```bash
# 1. Database - as a privileged MySQL account (schema owner), NOT the runtime account
mysql -u root -p < db/001_schema.sql
mysql -u root -p < db/002_seed.sql          # regenerate with: python tools/generate_seed.py
# edit the password in db/003_dcl.sql first, then:
mysql -u root -p < db/003_dcl.sql           # creates least-privilege 'roomwise_app'

# 2. Backend
cd backend
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                   # set RW_DB_PASSWORD to the roomwise_app password

# 3. Provision the first administrator (password is prompted - nothing is hard-coded)
python -m app.cli create-admin --username admin --email you@example.edu --name "Administrator"

# 4. (optional, lab demo) fictional CRs + a cancellation + pending/approved/rejected requests
python -m app.cli seed-demo        # prints the demo password ONCE

# 5. Run (serves the API at /api and the frontend at /)
uvicorn app.main:app --reload
# open http://127.0.0.1:8000   (API docs: /api/docs)
```

Reset everything: `mysql -u root -p < db/reset.sql` then repeat step 1.
Dated classes are generated automatically from the *published* timetable for any week that is
requested; `python -m app.cli materialize --weeks 12` pre-generates them.

## What was built, mapped to the roadmap
| Stage | Delivered |
|---|---|
| 1 Audit | Frontend was localStorage-only with 189 timetable rows hard-coded in `app.js`; those rows became the seed data. See DESIGN.md §1. |
| 2 Database | 15 baseline tables + `SECTION_CR_ASSIGNMENT`, `NOTIFICATION`, `AUDIT_LOG`, `USER_SESSION`; PK/FK/unique/check, indexes, seed, DCL, migrations. |
| 3 Auth | Argon2id, DB-backed sessions in HttpOnly cookies, per-session CSRF token, signup ⇒ VIEWER only, CR via admin assignment only. |
| 4 APIs | rooms, availability, week events, published timetable, dated classes, CR-scoped timetable, dashboard, admin CRUD. |
| 5 Requests | cancel/release, CR→CR room/slot request, room change, decisions, notifications, audit. |
| 6 Engine | `backend/app/scheduling.py` — one place for every clash/capacity/feature/availability rule + row-lock concurrency. |
| 7 Frontend | login, live data, cancel/request/approve dialogs, notifications, role-aware UI, ADMIN console. |
| 8 Testing | see docs/TESTING.md (what was run vs. what still needs your MySQL). |

## Important notes
* **Placeholder data:** the timetable images did not contain capacities, features, student strengths,
  branch names for DU/CH/HS, teacher availability or term dates. Those seed values are *fictional* and
  flagged in `tools/generate_seed.py`. Replace them with real data.
* Parallel whole-semester groups in the images (e.g. several DU Sem-5 electives at the same hour) are split
  into "Group 2/3" sections so no section is double-booked.
* The UI is whole-hour based (08:00–18:00, Mon–Fri) like the original; the API accepts any `HH:MM` inside those hours.
