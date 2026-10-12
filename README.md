# Spark’26 — Event Registration, QR Pass & Attendance Platform

The **Spark’26 CSE Symposium** website (`Spark26_RoyalBlue_Purple_Light.html`) turned into a complete,
database-connected application. The existing royal-blue/purple frontend is preserved exactly — same
layout, sections, animations, 3D effects, modals and typography — and every mock/localStorage
behaviour now runs against **PostgreSQL** through a **Node.js + Express + Prisma** backend.

| Layer | Technology |
| --- | --- |
| Backend | Node.js 18+, Express 4 |
| Database | PostgreSQL 14+ (single master database) |
| ORM & migrations | Prisma 6 |
| Auth | Server-side sessions (PostgreSQL store), bcrypt hashes, HTTP-only cookies, CSRF tokens |
| QR passes | `qrcode` (real, scannable QR) + HMAC-signed opaque payloads |
| QR scanning | `html5-qrcode` (camera) with native `BarcodeDetector` when available + manual ID fallback |
| Excel exports | ExcelJS (genuine `.xlsx`) |
| Uploads | `multer` + `file-type` content sniffing, stored under `var/uploads/` |
| Validation | Zod (server-side, field-level errors) |
| Tests | `node:test` + supertest (API/DB/Excel) and Puppeteer + `jsQR` (browser, camera, QR decoding) |

---

## 1. Quick start (exact commands)

### 1.1 Prerequisites

* Node.js **18.18+** (tested on 20.20)
* PostgreSQL **14+** (tested on 17)
* A modern browser (recent Chrome/Edge/Firefox/Safari)

### 1.2 Create the database and role

```bash
# Linux (Debian/Ubuntu) — install once
sudo apt-get update && sudo apt-get install -y postgresql
sudo pg_ctlcluster 17 main start          # or: sudo systemctl start postgresql

# Create the role + database (also a separate test database)
sudo -u postgres psql -c "CREATE ROLE spark26 WITH LOGIN PASSWORD 'spark26dev' CREATEDB;"
sudo -u postgres psql -c "CREATE DATABASE spark26      OWNER spark26;"
sudo -u postgres psql -c "CREATE DATABASE spark26_test OWNER spark26;"
```

macOS: `brew install postgresql@17 && brew services start postgresql@17`, then run the same three `psql` commands (**without** `sudo -u postgres`).
Windows: install PostgreSQL from the official installer and run the three `CREATE …` statements in **pgAdmin → Query Tool** or `psql`.

### 1.3 Configure the app

```bash
cd spark26
# A ready-to-run `.env` with LOCAL DEVELOPMENT defaults is already included.
# Before any real deployment, replace SESSION_SECRET, ADMIN_EMAIL and ADMIN_PASSWORD:
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"   # → paste as SESSION_SECRET in .env
# Every supported variable is documented in .env.example
```

> **⚠ Security:** the bundled `.env` contains throwaway development values only (local database password,
> dev session secret, demo admin). Change them before hosting this anywhere, and never commit `.env` —
> it is already listed in `.gitignore`.

### 1.4 Install, migrate, seed, run

```bash
npm install                 # installs dependencies
npm run db:generate         # generate the Prisma client (also runs automatically on install)
npm run db:deploy           # apply migrations (creates all tables)
npm run db:seed             # creates the first admin + the 8 Spark'26 events
npm run check               # prints an environment + database sanity report
npm start                   # → http://localhost:3000
```

Development with auto-reload: `npm run dev`

### 1.5 Create the first administrator (secure)

There is **no public admin signup endpoint**. Admin accounts are provisioned only from the server:

```bash
# Option A — via environment (used by the seed)
ADMIN_EMAIL="coordinator@college.edu" ADMIN_PASSWORD="A-strong-passphrase-2026" \
  npm run db:seed

# Option B — the provisioning script (prompts for the password, nothing on screen, no shell history)
npm run admin:create -- --email coordinator@college.edu --name "S. Kumar" --role SUPERADMIN

# Rotate an existing admin password
npm run admin:create -- --email coordinator@college.edu --rotate
```

Roles: `SUPERADMIN` (everything), `ADMIN` (events, registrations, attendance, exports), `SCANNER` (QR check-in only).

---

## 2. What was preserved, and what was added

**Preserved exactly:** page layout and content, royal-blue/purple/light theme, all animations
(slide transitions, 3D card tilt, prism canvas, marquee, QR scan-line, pixel stagger), event cards,
student form, My Pass page, admin portal, modals, toasts, responsive breakpoints, all existing element
IDs (`#eventGrid`, `#regForm`, `#qr`, `#adminPortal`, `#eventForm`, `#regsTable`, …) and event handlers.

**Changed only where the backend required it:**

| File | Change |
| --- | --- |
| `public/index.html` | The main `<script>` was extracted verbatim to `public/app.js` (wiring it made it maintainable); added the mobile **Contact** nav link, real date/time & capacity/status inputs in the admin event form, event-image upload field, participant list, QR scanner panel, on-spot registration panel, pass retrieval panel, and additive styles in one `<style id="sp26-backend-ui">` block. |
| `public/app.js` | localStorage demo store → REST API layer; QR grid generator → server-rendered genuine QR; hardcoded credentials → real login; CSV export → server `.xlsx`; new admin/scanner/on-spot code. |
| `src/**`, `prisma/**` | New backend, schema, migrations, seed scripts. |

The original uploaded file is untouched at `uploads/Spark26_RoyalBlue_Purple_Light.html`; the working copy is `public/index.html`.

---

## 3. Database

Single master database — **no per-event tables**. Events, students, registrations and attendance share
normalised tables linked by foreign keys (`prisma/schema.prisma`, migration `prisma/migrations/*_init`).

| Table | Purpose | Key constraints |
| --- | --- | --- |
| `students` | One row per person | `register_number` **unique**, `email` **unique**, normalised on write |
| `events` | Event catalogue | `code` + `slug` unique, `status` (DRAFT/PUBLISHED/CANCELLED/COMPLETED), `registration_mode` (WEBSITE/ON_SPOT/BOTH), capacity, deadline, image/brochure URLs, timestamptz start/end |
| `registrations` | Student × Event | **`UNIQUE(student_id, event_id)`** — a student can join many events but never the same one twice; opaque `qr_token` unique; `source`, `status`, `attendance_status`, `checked_in_at`, `checked_in_by_admin_id` |
| `admin_users` | Staff accounts | `email` unique, bcrypt `password_hash`, `role`, `status` |
| `audit_logs` | Admin action trail | admin/action/entity/timestamp (+ip), never passwords or QR tokens |
| `site_settings` | Home-page hero image URL | key/value |
| `admin_sessions` | Session store | created automatically by `connect-pg-simple` |

Event-wise participant tables and Excel worksheets are **queries over `registrations` filtered by
`event_id`** — the same organisation, without a fragile schema. All times are stored as UTC instants and
displayed in **Asia/Kolkata (IST)** everywhere (cards, passes, exports, admin tables).

---

## 4. API reference

All responses are JSON (`{ ok: true, … }` or `{ ok: false, error: { code, message, details? } }`).
Admin routes require a session cookie; every state-changing admin call additionally requires the
`x-csrf-token` header returned by login/session.

### Public
| Method | Path | Notes |
| --- | --- | --- |
| `GET` | `/api/health` | liveness |
| `GET` | `/api/events?search=&category=` | **published only**, live counts, availability, IST labels |
| `GET` | `/api/events/:idOrCode` | published event detail |
| `POST` | `/api/registrations` | website registration → creates student + registration and returns the QR pass |
| `GET` | `/api/passes/token/:token` | pass retrieval by unguessable capability token |
| `POST` | `/api/passes/lookup` | pass retrieval by **register number + email** (both must match) |
| `GET` | `/api/settings` | public site settings (hero image) |

### Admin — authentication
`POST /api/admin/auth/login` · `POST /api/admin/auth/logout` · `GET /api/admin/auth/session`

### Admin — events (auth + CSRF; write roles: SUPERADMIN/ADMIN)
`GET /api/admin/events` (list + dashboard stats + counts) · `GET /api/admin/events/:id`
`POST /api/admin/events` (create, optional `image`/`brochure` multipart) · `PATCH /api/admin/events/:id`
`POST /api/admin/events/:id/image` · `POST /api/admin/events/:id/brochure`
`POST /api/admin/events/:id/status` (`PUBLISHED`/`DRAFT`/`CANCELLED`/`COMPLETED`) · `POST /api/admin/events/:id/archive`
`DELETE /api/admin/events/:id` (refused with 409 when registrations exist) · `POST /api/admin/settings/home-hero` · `DELETE /api/admin/settings/home-hero` · `GET /api/admin/audit`

### Admin — registrations & exports
`GET /api/admin/registrations` (filter by event, search, attendance, paginated)
`GET /api/admin/events/:id/participants` · `POST /api/admin/registrations` (on-spot)
`GET /api/admin/events-summary`
`GET /api/admin/events/:id/registrations.xlsx` → `Spark26_<Event_Name>_Registrations.xlsx`
`GET /api/admin/export/master.xlsx` → `Spark26_Master_Registrations_<date>.xlsx`

### Admin — attendance
`POST /api/admin/attendance/verify` (QR scanner; returns `CHECKED_IN`, `ALREADY_CHECKED_IN`, `WRONG_EVENT`, `INVALID_CODE`, `UNKNOWN_REGISTRATION`, `CANCELLED_REGISTRATION`, `EVENT_NOT_SCANNABLE`)
`PATCH /api/admin/registrations/:id/attendance` (manual correction) · `GET /api/admin/attendance/summary?eventId=`

---

## 5. How the core flows work

**Student registration** — `POST /api/registrations` validates every field server-side (Zod), then inside a
transaction takes an advisory lock per event, re-checks publication/deadline/capacity, upserts the single
student row, and inserts the registration. A `UNIQUE(student_id, event_id)` constraint is the final guard, so
duplicate or concurrent submissions can never create two rows. The QR is generated **server-side**: a
256-bit opaque token + HMAC signature, returned as an SVG (pass card) and PNG (download). The QR contains
no personal data, no credentials and no admin access.

**Pass retrieval** — the pass is not authorised by knowing a registration ID. Retrieval happens either
through the unguessable 256-bit token (stored on the student's own device) or by proving
**register number + email** match. Guessed IDs return `404`.

**Attendance** — the admin picks the event, then scans. The server verifies the HMAC, looks the token up,
verifies the registration belongs to the selected event, and performs an atomic
`updateMany({ where: { id, attendanceStatus: 'PENDING' }, data: { PRESENT, checkedInAt, checkedInByAdminId } })`.
Only one concurrent scan can flip the row, so repeats report *already checked in* with the original time and
never increment counts. Every successful check-in is written to `audit_logs`.

**Registration modes** — `WEBSITE` (online only), `ON_SPOT` (public registration disabled; admins register at
the venue), `BOTH`. On-spot records are identical in shape (real registration ID, QR token, participant list,
Excel, scanner-eligible) and carry `source = ON_SPOT` plus the registering admin.

**Excel** — workbooks are generated on the server from the live database on every download: styled header row,
frozen header, auto-filter, sized columns, split registration/attendance columns, and an event summary block.
The master workbook adds a `Summary` sheet plus one worksheet per event and an `About` sheet that states the
database is authoritative. Attendance is **never** changed by editing a file.

---

## 6. Testing

```bash
npm test                 # 48 automated API / database / Excel / security tests (isolated test DB)
npm run test:e2e         # 35 browser acceptance checks (headless Chrome, real frontend + API)
npm run test:camera      # camera QR check-in using a synthetic video stream containing a real pass
npm run check            # environment + database sanity report
```

`npm test` runs against `TEST_DATABASE_URL` (migrations are applied automatically; the live database is
never touched). Long/repeated runs of the browser suites can trip the login rate limiter by design —
start the server with `DISABLE_RATE_LIMITS=1 npm start` while testing.

Results from the verification run are in `docs/verification.md`. Screenshots land in `var/e2e-shots/`.

---

## 7. Security checklist

* bcrypt (cost 12) password hashes; passwords only ever read from env or a hidden prompt.
* Server-side sessions in PostgreSQL, HTTP-only + SameSite=Lax cookies, `secure` in production, rolling expiry.
* CSRF synchroniser token required on every cookie-authenticated write (`403` otherwise).
* Zod validation on all inputs; safe error messages (no stack traces, ORM or SQL details).
* Parameterised queries everywhere via Prisma; unique/foreign-key constraints for integrity.
* Rate limits: login (10 / 15 min / ip+email), registration (40 / 15 min), QR verify (120 / min), pass lookup (20 / 15 min).
* Uploads: magic-byte sniffing (JPG/PNG/WEBP, PDF for brochures), 5 MB cap, randomised safe filenames,
  path-traversal guard, served from `/uploads` with `dotfiles: deny`.
* Helmet with a strict CSP that still allows the original inline styles/scripts and Google Fonts.
* Audit logging of publishing, edits, registrations, check-ins, exports and logins — no secrets or PII.
* `.env` is git-ignored and `.env.example` contains no real secrets.

### Production notes

* `NODE_ENV=production`, `COOKIE_SECURE=true`, `APP_ORIGIN=https://your-domain`, terminate TLS at a proxy and set `trust proxy`.
* Put `var/uploads` on persistent storage (or swap `persistUpload()` for S3-compatible object storage).
* Back up with `pg_dump`: `pg_dump "$DATABASE_URL" -Fc -f spark26-$(date +%F).dump` on a schedule, and verify restores.
* Run migrations on deploy with `npm run db:deploy` (never `migrate dev`).

---

## 8. Project structure

```
spark26/
├─ public/                 # the Spark'26 frontend
│  ├─ index.html           # original design (script extracted, backend panels added)
│  ├─ app.js               # API-integrated application code
│  └─ vendor/html5-qrcode.min.js
├─ src/
│  ├─ app.js  server.js    # Express app + bootstrap
│  ├─ config/env.js        # typed environment access
│  ├─ lib/                 # prisma, time (IST), ids, qr, audit, http, validation schemas
│  ├─ middleware/          # auth (session/CSRF/roles), validation, rate limits, uploads
│  ├─ routes/              # public, adminAuth, adminEvents, adminRegistrations, adminAttendance
│  └─ services/            # events, registrations, passes, attendance, excel
├─ prisma/                 # schema, migrations, seed
├─ scripts/                # create-admin, check, test-db-prepare, e2e-browser, e2e-camera-scan
├─ tests/                  # api, attendance, excel, security (+ harness)
├─ docs/verification.md    # test evidence
└─ var/uploads/            # uploaded event images, brochures, home hero photo
```

---

## 9. Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Missing required environment variable: SESSION_SECRET` | copy `.env.example` → `.env` and set a long random value |
| `Can't reach database server` | start PostgreSQL (`sudo pg_ctlcluster 17 main start`) and check `DATABASE_URL` |
| Public events list is empty | `npm run db:seed`, and make sure events are **Published**, not drafts |
| `Too many sign-in attempts` | rate limiting is working — wait 15 minutes or restart the server locally |
| Camera button says “Camera unavailable” | browsers only expose the camera on `https://` or `localhost`; use the manual registration-ID verification or a tunnel |
| Excel download 401 in a new tab | the admin session cookie is required — download from inside the admin panel |
| Uploaded image 404 | confirm `UPLOAD_DIR` exists and is writable; files are written to `var/uploads/<category>/` |

---

## 10. Known limitations / your action items

1. **Set your own admin credentials and `SESSION_SECRET`** in `.env` before real use; change
   `ADMIN_PASSWORD` and rotate it (`npm run admin:create -- --email … --rotate`).
2. **Camera scanning needs HTTPS** on the real deployment (or `localhost`). The manual
   registration-ID fallback works everywhere.
3. **Brochures** accept images and PDFs; the student "Details" modal links to the file rather than
   previewing PDFs inline.
4. **Pass tokens live on the student's device** (localStorage) like a saved ticket. A student who
   clears storage uses *Find my pass* (register number + email) — by design, a registration ID alone is
   never enough.
5. SMS/email notifications are **not** included; the schema and audit trail are ready for them.
6. Deleting an event with registrations is intentionally refused (409) — archive it instead so
   attendance history survives.
