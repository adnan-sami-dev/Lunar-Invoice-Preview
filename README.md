# 🌙 Lunar Invoice

<img width="1392" height="865" alt="Screenshot 2026-11-10 012342" src="https://github.com/user-attachments/assets/0b53a3f7-5082-4d53-b410-27cb9b4b72a1" />


> A multi-user invoicing platform for private tutors, built around one idea: **get paid without the awkward conversation.**

Tutors sign up, manage their students, create invoices, track payments, and send invoices directly to students' inboxes.

**[🚀 Live Demo](https://invoicing-app-lemon.vercel.app/)**

## About This Repository

This is a **source-available portfolio excerpt**. The full implementation lives in a private repository to keep the IP clean.

If you'd like to see the complete codebase, I'm happy to:

- Walk through any part of it live during an interview, or
- Grant read access to the private repo after an initial conversation.

Just reach out: [sami.adnandev@gmail.com](mailto:sami.adnandev@gmail.com) · [LinkedIn](https://www.linkedin.com/in/adnan-sami-3ab549220/)


**Stack:** Next.js 16 (App Router) · TypeScript · Tailwind CSS · Clerk · Neon (Postgres) · Brevo · Vercel

---

## Table of Contents

- [The Problem](#the-problem)
- [Features](#features)
- [Architecture Decisions & Trade-offs](#architecture-decisions--trade-offs)
- [Bugs I Fixed](#bugs-i-fixed)
- [What I'd Do Differently](#what-id-do-differently)
- [What I Learned](#what-i-learned)

---

## The Problem

Private tutors run their billing through WhatsApp, spreadsheets, and memory. They forget to invoice. They feel awkward chasing payments. They lose track of who owes what.

Existing tools are built for agencies and franchises: CRMs, marketing automation, multi-tutor dashboards. A solo tutor with 5–15 students doesn't need that. They need something **simple, automated, and non-awkward.**

---

## Features

- **Clerk authentication:** sign up, sign in, and session management handled by Clerk
- **Multi-user data isolation:** every query is scoped to the logged-in tutor
- **Student management:** add, list, and remove students with email validation
- **Invoice creation:** amount, description, due date, and status (pending / paid)
- **Mark as paid:** one-click status update with timestamp
- **Delete invoice:** scoped to the tutor, with confirmation
- **Send invoice via email:** Brevo API
- **Dashboard stats:** total earned, pending amount, invoices paid
- **Light/dark mode:** CSS variables + `localStorage` + no-flash inline script
- **Raw SQL migrations:** version-controlled, no ORM

---

## Architecture Decisions & Trade-offs

### 1. Raw SQL over an ORM

**The decision:** I use `@neondatabase/serverless` with tagged template literals instead of Prisma or Drizzle.

**Why:** I started with Prisma 8 (a release candidate) and hit breaking changes: no `url` in `schema.prisma`, a different client package (`@prisma/orm-postgres`), and a config file that didn't match the docs. Rather than fight the abstraction, I stepped back and asked: *do I need an ORM for a 3-table schema?*

The answer was no. Raw SQL gives me:

- Full control over every query
- No upgrade pain (SQL hasn't changed in 40 years)
- Zero abstraction between my code and the database
- A transferable skill (SQL works everywhere)

**The trade-off:** I lose type safety on query results and have to write migrations manually. For a solo project with 3 tables, that's acceptable. For a team of 10 with 50 tables, I'd revisit this.

**Migration strategy:** Numbered SQL files (`001_init.sql`, `002_remove_tutor_names.sql`) committed to git. Never edit a migration that has already run. Add a new one.

---

### 2. Lazy Sync over Webhooks for Tutor Creation

**The problem:** My `students` table has a foreign key to `tutors`. When a tutor signed up through Clerk, their row didn't exist in my database yet. Adding their first student threw:

```text
insert or update on table "students" violates foreign key constraint
Key (tutor_id)=(user_xyz) is not present in table "tutors"
```

**The textbook fix:** A Clerk webhook (`user.created`) that inserts the tutor the moment they sign up.

**Why I rejected it:** Webhooks require a public endpoint, Svix signature verification, retry handling, and ngrok for local development. Clerk's own data shows only 30% of their customers use webhooks for this. For a solo project, that's infrastructure I don't need.

**What I built instead:** `ensureTutor()`, an idempotent upsert that runs before any action that needs the tutor:

```sql
INSERT INTO tutors (id, email)
VALUES (${userId}, ${email})
ON CONFLICT (id) DO NOTHING
```

**Why this works:**

- **Idempotent:** safe to run every time
- **No race condition:** the check happens at the moment of action, not at sign-up
- **Works in local dev:** no public URL needed
- **Catches existing users:** anyone created before the code was deployed gets caught on their next action

**The trade-off:** If I ever need to send a welcome email on sign-up or build an admin dashboard showing "all tutors who signed up this week," I'll need the webhook. For now, I don't.

---

### 3. Multi-User Data Isolation

Every query is scoped to the logged-in tutor:

```sql
WHERE tutor_id = ${userId}
```

This applies to:

- Fetching students
- Fetching invoices
- Marking invoices as paid
- Deleting invoices
- Verifying student ownership before creating an invoice

**Why this matters:** Without the `tutor_id` check, one tutor could mark another tutor's invoice as paid or delete their students. The `WHERE` clause is the security boundary.

**Defense in depth:**

| Layer | Location | Purpose |
|-------|----------|---------|
| Middleware | `proxy.ts` | Blocks unauthenticated requests at the edge |
| Layout check | `auth()` in `(home)/layout.tsx` | Catches expired sessions |
| Resource check | `WHERE tutor_id = userId` | Ensures data isolation |

---

### 4. Email Deliverability

**The problem:** Resend's `onboarding@resend.dev` only sends to your own account email. To send to students, you need a verified domain.

**The solution:** Once a domain is verified in Resend, you can send from any address at that domain. The `from` field uses the format:

```text
${tutorName} via Lunar Invoice <invoices@yourdomain.com>
```

The `replyTo` field points to the tutor's email, so if a student replies, it goes to the tutor, not the app.

**Why you can't send "from" the tutor:** That's spoofing, and every email provider blocks it. Sending from your app with the tutor's name in the display name and their email in `replyTo` is the professional standard.

---

### 5. Theme Toggle Without Flash

**The problem:** A theme restore script ran before React hydrated. The server rendered `<html>` without the `dark` class, but the script added it. React saw a mismatch and threw a hydration error.

**The fix:** `suppressHydrationWarning` on the `<html>` tag. This tells React: *"I know the attributes differ. It's intentional."*

**The lesson:** The script runs in the window between server render and client hydration, which is exactly the window React can't reconcile. `suppressHydrationWarning` is the escape hatch for this specific case.

---

## Bugs I Fixed

| Bug | Root Cause | Fix | Lesson |
|-----|------------|-----|--------|
| Foreign key constraint on student insert | Tutor row didn't exist yet | `ensureTutor()` upsert with `ON CONFLICT DO NOTHING` | Foreign keys are a feature, not a bug |
| Infinite redirect loop | `auth().protect()` in root layout wrapped the sign-in page | Moved auth check to `(home)` layout; auth pages live outside it | Layouts wrap everything below them, including auth pages |
| Hydration mismatch | Theme script ran before React hydrated | `suppressHydrationWarning` on `<html>` | The script runs between server render and client hydration |
| Date rendering error | Neon returns `Date` objects; React can't render them | Convert to string before passing to client | Complex types must be serialized at the server/client boundary |
| `async` in Client Component | Client Components can't be `async` | Moved auth check to a Server Component | `'use client'` changes what's allowed |
| `instant` in Client Component | `instant` export only works in Server Components | Wrapped Clerk component in `<Suspense>` instead | The error message's suggested fix may not apply to your file |

---

## What I'd Do Differently

1. **Add the `ensureTutor` fallback from day one.** Hitting the foreign key error taught me why it exists, but I'd rather have written it first.
2. **Start with a domain.** Testing email with `onboarding@resend.dev` works, but a verified domain unlocks real deliverability and makes the app look production-ready.
3. **Add a webhook for `user.created`.** The lazy sync works, but a webhook would let me send a welcome email and build an admin dashboard.
4. **Write integration tests for the Server Actions.** I tested manually, but a test suite would catch regressions when I refactor.
5. **Add rate limiting to the email action.** Right now, a tutor could spam a student's inbox. A simple "1 email per invoice per hour" check would prevent abuse.

---

## What I Learned

- **Events are triggers. State is truth.** Don't derive meaning from which event fired first.
- **Foreign keys protect data integrity.** They're a feature, not an obstacle.
- **Abstractions have a cost.** Sometimes the simplest tool is the one you already understand.
- **Not every bug is worth fixing.** Some are SDK limitations. Ship the 95%.
- **The server/client boundary is where bugs live.** Dates, functions, and secrets can't cross it unchanged.
