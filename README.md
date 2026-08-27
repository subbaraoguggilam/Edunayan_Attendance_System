# Edunayan Attendance Management System

A complete web-based attendance management system built with **Next.js 14 (App Router)**,
**MongoDB Atlas** + **Mongoose**, and **Tailwind CSS**, ready to deploy on **Vercel**.


Credentials are hardcoded in `lib/auth.js` exactly as requested for this workshop deployment.
> **Security note:** for a real production system you'd store hashed passwords in the
> database instead of hardcoding plaintext ones in source code. This is intentionally
> simplified for a workshop demo.

## Features

- **Authentication** — JWT-based session stored in an httpOnly cookie. `middleware.js`
  protects `/teacher/*` (Teacher only) and `/admin/*` (Heads only) routes at the edge.
- **Attendance marking** — Teacher (or Heads) pick a section (A/B/C) and any date, then
  mark each student Present/Absent. Saved via upsert so re-marking a date updates it.
- **Attendance viewing & Excel export** — Heads can view any section/date and download an
  `.xlsx` report for a date range (single section or all sections) using SheetJS (`xlsx`).
- **Daily syllabus** — Heads add/update the topic + notes taught in each section per date.
- **Analytics dashboard** — Heads see:
  - Section-wise attendance % (bar chart)
  - Section share of present marks (pie chart)
  - Date-wise attendance % history (line chart)
  - Student-wise attendance trend table, sorted lowest-first to flag at-risk students
  - Built with `recharts`, data aggregated server-side with MongoDB's aggregation pipeline.
- **Student roster** — Heads can add students to a section from `/admin` (needed before
  attendance can be marked for them). A `npm run seed` script is also included.

## Tech Stack

- Next.js 14 (App Router, API routes)
- MongoDB Atlas + Mongoose 8
- JWT (`jsonwebtoken` on the server, `jose` for edge-compatible middleware verification)
- `xlsx` (SheetJS) for Excel export
- `recharts` for charts
- Tailwind CSS

## Project Structure

```
app/
  api/
    auth/{login,logout,me}/route.js   # session endpoints
    students/route.js                 # GET list / POST add student
    attendance/route.js               # GET records / POST mark attendance
    attendance/export/route.js        # GET .xlsx download (Heads only)
    syllabus/route.js                 # GET / POST syllabus (Heads write)
    analytics/route.js                # GET aggregated stats (Heads only)
  login/page.js                       # login screen
  teacher/page.js                     # Teacher dashboard
  admin/page.js                       # Heads landing + roster manager
  admin/attendance/page.js            # Heads: view + export attendance
  admin/syllabus/page.js              # Heads: add/update syllabus
  admin/analytics/page.js             # Heads: charts dashboard
components/Navbar.js
lib/{dbConnect,auth,dateUtils,useSession}.js
models/{Student,Attendance,Syllabus}.js
middleware.js
scripts/seedStudents.js
vercel.json
.env.example
```

## 1. Set up MongoDB Atlas

1. Create a free cluster at https://www.mongodb.com/cloud/atlas.
2. Under **Database Access**, create a database user with a username/password.
3. Under **Network Access**, add `0.0.0.0/0` (allow access from anywhere) so Vercel's
   serverless functions can connect — or add Vercel's specific IP ranges if you prefer.
4. Click **Connect → Drivers**, copy the connection string. It looks like:
   ```
   mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/edunayan_attendance?retryWrites=true&w=majority
   ```
   Make sure to add a database name (e.g. `edunayan_attendance`) before the `?`.

## 2. Local development

```bash
npm install
cp .env.example .env.local
# edit .env.local and paste your MONGODB_URI and a random JWT_SECRET

npm run seed     # optional: adds sample students to sections A, B, C
npm run dev      # starts at http://localhost:3000
```

Generate a strong `JWT_SECRET` quickly with:
```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

## 3. Deploy to Vercel

### Option A — Vercel CLI
```bash
npm i -g vercel
vercel login
vercel            # first deploy, follow prompts (link/create project)
vercel env add MONGODB_URI production
vercel env add JWT_SECRET production
vercel --prod
```

### Option B — Vercel Dashboard
1. Push this project to a GitHub repo.
2. Go to https://vercel.com/new and import the repo.
3. Vercel auto-detects Next.js — no build settings need to change.
4. Before the first deploy (or right after), go to **Project → Settings → Environment
   Variables** and add:
   - `MONGODB_URI` = your Atlas connection string
   - `JWT_SECRET` = a long random string
5. Deploy. Once live, run `npm run seed` locally (pointed at the same `MONGODB_URI`) to
   populate sample students, or add them via the **Overview** page as a Head.

The included `vercel.json` references these as Vercel "secrets" (`@mongodb_uri`,
`@jwt_secret`) for CLI-based secret management; if you set the env vars directly in the
Dashboard instead (recommended, simplest), you can safely delete the `"env"` block from
`vercel.json` — Vercel will just use the Dashboard-configured variables.

## 4. First login

Go to `/login` and sign in with any of the three accounts above. Teacher lands on
`/teacher`; Heads land on `/admin`.

## Notes on data model

- Attendance and syllabus dates are normalized to `00:00:00 UTC` of the calendar day so a
  given date always maps to exactly one record per student (attendance) or per section
  (syllabus), regardless of what time of day it's saved.
- Marking attendance twice for the same student/date **updates** the existing record
  (upsert) rather than creating a duplicate — so Subbarao can safely correct mistakes.
