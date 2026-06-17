# EduFlow LMS — Course Outline

**Stack:** Next.js · Express.js · MongoDB · Mongoose · Redux Toolkit  
**Difficulty:** Advanced  
**Pace:** Full-time, ~50–60 hrs/week  

---

## How to read this outline

Every chapter below links to its folder. Each folder contains 10–22 numbered sub-chapters — small, focused files you work through in order. Do not skip. The Definition of Done at the end of each chapter is a **gate**: every box ticked before you move on.

---

## Phase 1 — Foundation (Week 1)

| # | Chapter | Sub-chapters | Gate |
|---|---------|-------------|------|
| 01 | [Introduction](./01-introduction/) | 01.01 – 01.07 | Course outline read, working rhythm set |
| 02 | [Project Skeleton](./02-project-skeleton/) | 02.01 – 02.20 | Both apps boot, health check returns `{ status: "ok" }` |
| 03 | [Why MongoDB?](./03-why-mongodb/) | 03.01 – 03.08 | Can explain document vs relational, embed vs reference |
| 04 | [Connect and Configure](./04-connect-and-configure/) | 04.01 – 04.12 | Server fails fast on bad config; Mongoose connects |

---

## Phase 2 — Identity (Week 1–2)

| # | Chapter | Sub-chapters | Gate |
|---|---------|-------------|------|
| 05 | [Data Model](./05-data-model/) | 05.01 – 05.14 | All Mongoose models defined, seeded, verified |
| 06 | [Authentication API](./06-authentication-api/) | 06.01 – 06.22 | Register, verify, login, refresh, logout all working |
| 07 | [Login and Registration — Frontend](./07-login-and-registration/) | 07.01 – 07.14 | Register, verify, login, logout working in the browser |
| 08 | [Authorization](./08-authorization/) | 08.01 – 08.11 | Role middleware in place; IDOR prevented via ownership checks |

---

## Phase 3 — Learning Core (Week 2–3)

| # | Chapter | Sub-chapters | Gate |
|---|---------|-------------|------|
| 09 | [Course Catalogue API](./09-course-catalogue-api/) | 09.01 – 09.20 | Paginated, filtered catalogue endpoint working |
| 10 | [Browse Courses — Frontend](./10-browse-courses/) | 10.01 – 10.13 | Catalogue browsable in the browser with filters |
| 11 | [Instructor Courses API](./11-instructor-courses-api/) | 11.01 – 11.20 | Instructor can create, edit, add lessons, submit for review |
| 12 | [Manage My Courses — Frontend](./12-manage-my-courses/) | 12.01 – 12.13 | Instructor dashboard fully functional in the browser |

---

## Phase 4 — Transactions and Progress (Week 3–4)

| # | Chapter | Sub-chapters | Gate |
|---|---------|-------------|------|
| 13 | [Enrollment and Payments](./13-enrollment-and-payments/) | 13.01 – 13.20 | Stripe checkout works; enrollment confirmed via webhook |
| 14 | [Enroll in a Course — Frontend](./14-enroll-in-a-course/) | 14.01 – 14.10 | Student can pay and enroll in the browser |
| 15 | [Lesson Access API](./15-lesson-access-api/) | 15.01 – 15.18 | Lessons, progress, certificates, Q&A all working |
| 16 | [Watch a Lesson — Frontend](./16-watch-a-lesson/) | 16.01 – 16.12 | Student can watch lessons, track progress, download certificate |

---

## Phase 5 — Scale (Week 4)

| # | Chapter | Sub-chapters | Gate |
|---|---------|-------------|------|
| 17 | [Progress and Certificates](./17-progress-and-certificates/) | 17.01 – 17.15 | Progress aggregation accurate; certificates generated as PDF |
| 18 | [Performance and Search](./18-performance-and-search/) | 18.01 – 18.20 | Redis cache live; Atlas Search endpoint working; rate limits in place |
| 19 | [Admin Dashboard](./19-admin-dashboard/) | 19.01 – 19.12 | Admin can approve/reject courses; platform metrics visible |

---

## Phase 6 — Ship (Week 4–5)

| # | Chapter | Sub-chapters | Gate |
|---|---------|-------------|------|
| 20 | [Deploy](./20-deploy/) | 20.01 – 20.15 | App live at a public HTTPS URL; secrets in env, not in Git |
| 99 | [Closing](./99-closing/) | 99.01 – 99.05 | ERD, architecture doc, demo video, bar-raiser chosen |

---

## Learning Log

Every concept chapter ends with a prompt to write in `learning-log/`. Use the template at [learning-log/00-template.md](./learning-log/00-template.md).

---

## Quick Reference

| Topic | Chapter |
|-------|---------|
| Mongoose connection | 04 |
| User / RefreshToken models | 05, 06 |
| JWT access + refresh tokens | 06 |
| Role-based access control | 08 |
| N+1 query fix | 09 |
| Cursor pagination | 09 |
| Stripe webhook + idempotency | 13 |
| Signed video URLs | 15 |
| Redis cache + invalidation | 18 |
| Atlas Search | 18 |
| Rate limiting | 18 |
| VPS + PM2 + Nginx + SSL | 20 |
