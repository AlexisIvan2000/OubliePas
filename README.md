# OubliePas

A subscription and bill tracker. It answers one question — **what am I paying
for, and what is about to be charged?** — and then makes sure nobody has to go
looking for the answer.

![Dashboard](docs/screenshots/Dashboard.png)

---

## What it does

### Two kinds of commitment

A **subscription** is a recurring service you signed up for. A **bill** is a
fixed cost you owe anyway. Same shape, different meaning, and the distinction
runs all the way down to the reminder wording.

Each line carries its category, its cadence, its own reminder lead time and both
the per-charge and yearly amounts — a $16.99 monthly line reads very differently
as $203.88 a year.

![Subscriptions](docs/screenshots/Subscriptions.png)

### A calendar of instalments

Occurrences are **materialised rows, not computed on read**: a generator writes
them ahead of time, so a due date is a fact in the database rather than the
result of a calculation nobody can audit. Marking one paid takes one click.

![Calendar](docs/screenshots/Calendar.png)

On a phone the grid keeps its seven columns and drops the text — a day becomes a
square with coloured dots, and tapping it opens that day's instalments below the
grid. Seven columns of text are unreadable at 390 px; the shape of the month is
not.

### Where the money goes

Every category, not just the top five, and the two numbers that matter for a
decision: what you are committed to per year and per month. The heaviest lines
are ranked across both types together — rent at $8,400 a year and Netflix at
$262 belong on the same scale.

![Breakdown](docs/screenshots/Breakdown.png)

![Heaviest lines](docs/screenshots/Breakdown_2.png)

### Reminders that arrive

Three families of alert, each switchable on its own — **before the due date**,
**past due**, **trial end or cancellation notice** — over two channels, email
and web push, plus a weekly digest every Monday.

![Reminders](docs/screenshots/Reminders.png)

![Reminder email](docs/screenshots/reminders_email.png)

A cron job runs once a day: it purges, generates the occurrences that entered the
horizon, and sends. Each reminder is keyed so it goes out exactly once, and a
failed send is left unmarked so the next run retries it.

---

## How it is built

Two deployables from two repositories: a FastAPI service and a React SPA.

| | Backend | Frontend |
|---|---|---|
| Language | Python 3.13 | JavaScript |
| Framework | FastAPI + Starlette | React 19 |
| Build | — | Vite 8 |
| Data | PostgreSQL, SQLAlchemy 2 (async), asyncpg | — |
| Migrations | Alembic | — |
| Validation | Pydantic 2 | — |
| Auth | JWT + argon2, Google OAuth | React Router 7 |
| Email | Resend | — |
| Web push | py-vapid, http-ece, httpx | Service worker, Push API |
| Storage | S3-compatible, boto3, Pillow | — |
| Rate limiting | slowapi + Redis | — |
| Styling | — | CSS Modules |
| Tests | pytest | Vitest |
| Hosting | Railway | Vercel |

No TypeScript, no Tailwind, no UI component library. The backend's
`requirements.txt` is 24 packages, curated by hand from the actual import graph.

### Decisions worth naming

**Web push without `pywebpush`.** The obvious library pulls 19 transitive
packages and is synchronous. `py-vapid` for the signature, `http-ece` for the
RFC 8291 encryption and `httpx` for the POST is five packages and stays async —
no thread pool in the middle of an async job.

**Migrations run at application startup**, not as a pre-deploy step. The
platform's pre-deploy hook silently never ran, and three deployments served 500s
on a missing table. A blocking advisory lock serialises replicas, and a failure
stops the container rather than letting it answer 500 on every route that touches
the database.

**Errors are a contract.** Every failure is an `AppException` rendered as
`{"detail": {"code", "message"}}`. The frontend switches on `code`, never on the
message, so the wording can change language without breaking anything.

**Soft delete with a single door.** Deleting sets a timestamp and the daily job
purges after 30 days. Every read goes through one of two helpers that apply the
`deleted_at IS NULL` guard, so it cannot be forgotten in one query out of
fourteen and leak a deleted amount back into a total.

### Tests

1,047 backend tests and 349 frontend tests. The rule is that a guard must prove
it bites: reintroduce the defect it prevents and watch a named test fail, before
calling it a guard. A read-only end-to-end suite checks the live deployment —
security headers, CORS, that the service worker is served from the root, and that
the built bundle calls the deployed API rather than a laptop.

---

## The code

| Repository | What it holds |
|---|---|
| [OubliePas-backend](https://github.com/AlexisIvan2000/OubliePas-backend) | The API, the daily job, the migrations |
| [OubliePas-frontend](https://github.com/AlexisIvan2000/OubliePas-frontend) | The React client |

Setup, configuration and deployment live in each repository's own README. They
are not repeated here — a duplicated instruction is an instruction that will be
wrong in six months.

## Status

Live. The API runs on Railway alongside PostgreSQL, Redis and a scheduled worker;
the client is deployed on Vercel. French and English throughout, light and dark
themes, and a progressive web app that can be installed on a phone.
