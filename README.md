# City of Atlanta Ambassador / Marshal Signup (MVP)

Signup + light admin verification for **Browns Mill**, **Alfred Tup Holmes**, **Chastain (North Fulton)**, and **Candler Park**. Built for **zero‑install usage on city computers** (browser only), **GitHub Codespaces** development, and **Vercel** deployment with **Vercel Postgres/Neon**.

---

## Quick Links

* [Architecture](#architecture)
* [Data Model](#data-model)
* [Local & Codespaces Setup](#local--codespaces-setup)
* [Database (Vercel Postgres)](#database-vercel-postgres)
* [Environment Variables](#environment-variables)
* [Run & Seed](#run--seed)
* [Deployment on Vercel](#deployment-on-vercel)
* [Routes & UI](#routes--ui)
* [Security & City-Network Mode](#security--city-network-mode)
* [Roadmap (Phone Check‑In / Geofence)](#roadmap-phone-check-in--geofence)
* [Testing & QA](#testing--qa)
* [Operations: Logs, Backups, Monitoring](#operations-logs-backups-monitoring)
* [Troubleshooting](#troubleshooting)
* [Appendix](#appendix)

---

## Architecture

**Stack**

* **Next.js 14+ App Router** (TypeScript) + **Tailwind CSS**
* **Prisma** ORM with **PostgreSQL** (Vercel Postgres/Neon)
* Minimal **Edge Middleware** for admin Basic Auth
* Deployed to **Vercel** (HTTPS by default)

**Why this stack?**

* Browser‑only dev possible via **GitHub Codespaces**.
* Server actions and API routes make a tiny backend easy to maintain.
* Prisma keeps the data model explicit and portable.

**High‑Level Flow**

```
Visitor → / (Signup Form) → /api/signup (POST) → Prisma → Postgres
Admin → /admin (Basic Auth) → Toggle “Checked” (verified)
```

---

## Data Model

Prisma schema highlights:

```
Role: AMBASSADOR | MARSHAL

Ambassador {
  id, role, firstName, lastName, email (unique), phone?, notes?,
  verified ("checked"), createdAt, updatedAt
  assignments: CourseAssignment[]
}

Course {
  id, name (unique), slug (unique), lat?, lng?, createdAt, updatedAt
  assignments: CourseAssignment[]
}

CourseAssignment { // Ambassador ↔ Course (M:N)
  id, ambassadorId, courseId, availability?, preferred (bool)
  @@unique([ambassadorId, courseId])
}
```

**Seeded Courses** (with coordinates for future geofence check‑in):

* Browns Mill Golf Course — `browns-mill` — lat: `33.6867`, lng: `-84.3473`
* Alfred Tup Holmes Golf Course — `tup-holmes` — lat: `33.7567`, lng: `-84.4811`
* North Fulton / Chastain Park GC — `chastain-park` — lat: `33.8689`, lng: `-84.3865`
* Candler Park GC — `candler-park` — lat: `33.7706`, lng: `-84.3412`

> You can change/add courses in `prisma/seed.ts` and re‑run `npm run db:seed`.

---

## Local & Codespaces Setup

### Option A — **GitHub Codespaces** (Recommended; zero local installs)

1. Push the repo to GitHub.
2. In GitHub → **Code → Create codespace on main**.
3. In the Codespaces terminal, set env and run (see [Run & Seed](#run--seed)).

### Option B — Local (Node 18+)

1. Install Node 18+ and Postgres (or use Vercel Postgres connection).
2. `cp .env.example .env` → set values (see below).
3. `npm i` → `npm run db:push` → `npm run db:seed` → `npm run dev`.

> **Zero‑install on City computers:** All runtime use is via Vercel HTTPS URL in a browser. Admins don’t need Node, Git, or Vercel CLI.

---

## Database (Vercel Postgres)

* In **Vercel Dashboard**: **Add New → Storage → Postgres** → create DB.
* Copy the **`DATABASE_URL`** (includes SSL and credentials). Example:

  ```
  postgresql://USER:PASSWORD@HOST:PORT/DB?sslmode=require
  ```
* Use the same URL in development (Codespaces) and production (Vercel Environment Variables). Prisma will **create** the schema with `db:push`.

---

## Environment Variables

Create `.env` (do not commit to Git):

```
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DB?sslmode=require"
ADMIN_PASSWORD="change-me"
```

* **DATABASE\_URL** — Required. Vercel Postgres connection string.
* **ADMIN\_PASSWORD** — Basic Auth password for `/admin`. Any username is accepted; only the password is checked in middleware.

> Store **prod** secrets only in **Vercel → Project → Settings → Environment Variables**.

---

## Run & Seed

From Codespaces or local dev:

```bash
npm i
npm run db:push   # create tables from Prisma schema
npm run db:seed   # seed the four courses
npm run dev       # http://localhost:3000
```

**Package scripts**

* `postinstall`: `prisma generate`
* `db:push`: `prisma db push`
* `db:seed`: `tsx prisma/seed.ts`

---

## Deployment on Vercel

1. In **Vercel → Add New → Project → Import Git Repository**.
2. Framework should auto‑detect **Next.js**.
3. Add **Environment Variables**:

   * `DATABASE_URL`
   * `ADMIN_PASSWORD`
4. Click **Deploy**.

**City computers usage:** Visit your Vercel URL in the browser:

* Public signup: `/`
* Admin: `/admin` (browser will prompt for Basic Auth password)

---

## Routes & UI

### Public – `/`

* **Role**: Ambassador or Marshal
* **Name, Email, Phone** (email unique)
* **Course multi‑select**: Browns Mill, Tup Holmes, Chastain, Candler Park
* **Availability/Notes**
* Consent checkbox to be contacted

On submit, form posts to `POST /api/signup`.

### API – `POST /api/signup`

* Accepts `FormData` fields: `role`, `firstName`, `lastName`, `email`, `phone?`, `availability?`, `notes?`, `courses[]`
* Upserts an Ambassador by email and upserts CourseAssignments for selected courses.
* Returns `{ ok: true }` on success; non‑200 on valida­tion errors.

### Admin – `/admin` (Basic Auth)

* Lists ambassadors (latest first)
* Shows role, email, courses, created date
* **Toggle “Checked/Unchecked”** updates `verified` boolean via server action

---

## Security (City-Network Mode)

* **HTTPS by default** on Vercel.
* **No installs on City PCs**: Admin uses a browser only; never store `.env` on public machines.
* **Basic Auth**: `/admin` guarded by password only; rotate `ADMIN_PASSWORD`.
* Optional **IP allowlist** (Edge Middleware). Example:

  ```ts
  // middleware.ts (extend existing admin guard)
  const ALLOWED = ["x.x.x.x/32", "y.y.y.y/32"]; // replace with City NAT/IPs
  function ipAllowed(ip: string) {
    return ALLOWED.some(a => a.split("/")[0] === ip);
  }
  export function middleware(req: NextRequest) {
    if (req.nextUrl.pathname.startsWith("/admin")) {
      const ip = req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() || "";
      if (!ipAllowed(ip)) return new NextResponse("Forbidden", { status: 403 });
      // Basic Auth password check continues here…
    }
    return NextResponse.next();
  }
  ```
* **PII Minimization**: We store only contact fields necessary to coordinate volunteers.
* **Auditability**: Add `verifiedBy` + `verifiedAt` columns later if formal auditing is required.

---

## Roadmap (Phone Check‑In / Geofence)

**Goal:** When an ambassador is physically at their assigned course, they can tap **“I’m here”** in the mobile web app to record attendance.

**Implementation plan:**

1. **Schema**: Add `Attendance { id, ambassadorId, courseId, lat, lng, createdAt }`.
2. **Geofence**: Compare `navigator.geolocation` reading to course `lat/lng` using a Haversine distance function. Allow if within N meters (e.g., 150m).
3. **UI**: `/checkin` shows nearest course and disabled state if outside radius; one‑tap create.
4. **Abuse controls**: Rate‑limit, require GPS high‑accuracy, and store coarse IP for anomaly detection.
5. **Admin**: Attendance table, course filters, CSV export.

---

## Testing & QA

**Unit**

* Validate `POST /api/signup` input handling and Prisma upsert behavior.
* Mock Prisma client in tests.

**E2E** (Playwright)

* Submit signup with multiple course selections → verify DB rows.
* Admin login + toggle “Checked” → assert verified field flips.

**Accessibility**

* Form fields have labels; ensure focus states and tab order.
* High contrast for buttons and status messages.

---

## Operations: Logs, Backups, Monitoring

* **App logs**: Vercel project logs per deployment.
* **Database**: Use Neon/Vercel Postgres automated backups (enable snapshots/retention in provider dashboard).
* **Metrics**: Optional — add Vercel Analytics or a lightweight endpoint metrics counter.

**Backups/Restore**

* Schedule daily logical backups; test restore quarterly.
* Provide export (`COPY TO CSV`) for City reporting needs.

---

## Troubleshooting

* **401/403 on `/admin`**: Missing/incorrect `ADMIN_PASSWORD`, or blocked by IP allowlist if enabled.
* **`DATABASE_URL` errors**: Ensure `sslmode=require` and correct credentials; make sure the DB is reachable from Codespaces for seeding.
* **Prisma Client not generated**: Run `npm i` (postinstall) or `npx prisma generate`.
* **Seed idempotency**: Courses are upserted; re‑running seed is safe.
* **Vercel deployment succeeds but admin list is empty**: Did you seed? Use `npm run db:seed` against the production DB (or import minimal data).

---

## Appendix

**Project Layout (key files)**

```
src/
  app/
    page.tsx              # Public signup form
    admin/page.tsx        # Admin list + "Checked" toggle (server action)
    api/signup/route.ts   # POST handler for signup
  lib/prisma.ts           # Prisma singleton
prisma/
  schema.prisma
  seed.ts
middleware.ts             # Basic Auth (and optional IP allowlist)
README.md
```

**Common Commands**

```bash
npm run db:push   # sync schema to DB
npm run db:seed   # seed the 4 courses
npm run dev       # start dev server on 3000
```

**Example cURL**

```bash
curl -X POST https://YOUR-APP.vercel.app/api/signup \
  -F role=AMBASSADOR \
  -F firstName=Sky \
  -F lastName=Rivera \
  -F email=sky@example.org \
  -F phone=4045551234 \
  -F availability="Sat mornings" \
  -F notes="Happy to cover clinics" \
  -F courses=browns-mill -F courses=candler-park
```

**Change/Extend the Data Model**

1. Edit `prisma/schema.prisma`.
2. `npm run db:push` (dev) or create a migration if you need versioning.
3. Update UI/admin as needed.

---

**License & Ownership**
Internal pilot for City of Atlanta course volunteer coordination by Solomon Watkins. Adapt text as needed for your department’s standards.

