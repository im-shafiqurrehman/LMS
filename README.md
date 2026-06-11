# EduFlow — LMS Project Guide

> A DevWeekends project guide for building a full-stack Learning Management System from scratch, deployed to production.

---

## What you are building

**EduFlow** is a production-grade Learning Management System. Instructors publish courses made of video lessons. Students enroll, watch lessons, track progress, and earn certificates. An admin approves courses and oversees the platform.

**Stack:** Node.js + Express · MongoDB · Redis · Cloudinary · Stripe · Resend · Next.js · Docker · Nginx · PM2

**Difficulty:** Advanced — assumes you can already build CRUD APIs and climbs into auth, authorization, media delivery, caching, payments, rate limiting, and production deployment.

---

## Folder structure of this guide

```
LMS/
├── 01-introduction/
│   ├── 01.01-what-youre-building.md
│   ├── 01.02-how-to-use-this-course.md
│   ├── 01.03-the-product-and-its-people.md
│   ├── 01.04-scope.md
│   ├── 01.05-prerequisites.md
│   └── 01.06-course-outline.md
│
├── phase-2-why-and-stack/
│   ├── 02-why-mongodb.md
│   └── 03-project-setup-and-config.md
│
├── phase-2b-frontend-setup/
│   └── 04-nextjs-frontend-setup.md
│
├── phase-3-identity/
│   ├── 03-authentication/
│   │   ├── 03.01-set-the-scene.md
│   │   ├── 03.02-the-approaches.md
│   │   ├── 03.03-what-a-jwt-is.md
│   │   ├── 03.04-password-hashing.md
│   │   ├── 03.05-auth-schema.md
│   │   ├── 03.06-scaffold-auth-module.md
│   │   ├── 03.07-register-endpoint.md
│   │   ├── 03.08-verify-login-refresh-logout.md
│   │   └── 03.09-middleware-and-checklist.md
│   └── 04-authorization/
│       └── 04-authorization.md
│
├── phase-4-learning/
│   ├── 05-course-catalogue/
│   │   └── 05-course-catalogue.md
│   ├── 06-instructor-course-management/
│   │   └── 06-instructor-course-management.md
│   ├── 07-enrollment-and-payments/
│   │   └── 07-enrollment-and-payments.md
│   ├── 08-lesson-access-and-video/
│   │   └── 08-lesson-access-and-video.md
│   ├── 09-progress-and-certificates/
│   │   └── 09-progress-and-certificates.md
│   └── 10-qa-system/
│       └── 10-qa-system.md
│
├── phase-5-scale/
│   ├── 10-search/
│   │   └── 10-search.md
│   ├── 11-caching-and-performance/
│   │   └── 11-caching-and-performance.md
│   ├── 12-notifications/
│   │   └── 12-notifications.md
│   └── 13-rate-limiting/
│       └── 13-rate-limiting.md
│
├── phase-6-admin/
│   └── 14-admin-dashboard/
│       └── 14-admin-dashboard.md
│
├── phase-7-ship/
│   ├── 15-deploy/
│   │   └── 15-deploy.md
│   └── 16-closing/
│       └── 16-closing.md
│
└── learning-log/
    └── 00-template.md      ← copy for each chapter
```

---

## Reading order

Work through chapters in the numbered order. Each chapter builds on the previous one.

| Phase | Chapters | What you build |
|---|---|---|
| **Phase 1 — Foundations** | 01, 02, 03 | The project, the database choice (MongoDB), the running server |
| **Phase 2 — Frontend** | 04 | Next.js frontend setup, API client, auth state |
| **Phase 3 — Identity** | 05, 06 (auth + authz) | Who users are and what they're allowed to do |
| **Phase 4 — Learning product** | 07–11 | Catalogue, courses, enrollment, lessons, progress, certificates, Q&A |
| **Phase 5 — Scale** | 12–15 | Search, caching, notifications, rate limiting |
| **Phase 6 — Admin** | 16 | Platform oversight, approval workflow, metrics |
| **Phase 7 — Ship** | 17, 18 | Production deploy, documentation, demo, close |

---

## How progress is gated

Every chapter ends in either a **Definition of Done** (build chapters) or **Key Takeaways** (concept chapters). You do not move to the next chapter until every box is ticked. The learning log answers must be written and committed before you proceed.

This is not optional — it's the mechanism that keeps a self-paced course honest.

---

## The complexity ladder this course climbs

| Rung | Chapter(s) |
|---|---|
| Project setup, config validation, env safety | 03 |
| Next.js frontend setup, API client, state management | 04 |
| Core schema design, data modelling with MongoDB | 02, 06, 07 |
| Full auth: hashing, access + refresh tokens, email verification | 05 |
| Authorization: role checks + ownership isolation (IDOR prevention) | 06 |
| Pagination (cursor), N+1 prevention | 07 |
| Payment processing, webhooks, idempotency | 09 |
| Signed media URLs, access-gated content | 10 |
| Q&A system: questions and instructor answers | 11 |
| Full-text search with MongoDB Atlas Search | 12 |
| Redis caching with invalidation | 13 |
| Transactional email (fire-and-forget pattern) | 14 |
| Rate limiting with Redis store | 15 |
| Production deploy: PM2, Nginx, Let's Encrypt | 17 |

---

## Key technologies — quick reference

| Technology | Purpose | Chapter introduced |
|---|---|---|
| Express.js | HTTP framework | 03 |
| MongoDB | Primary database | 02 |
| mongodb (native driver) | Database driver | 02 |
| Zod | Request + env validation | 03 |
| bcrypt | Password hashing | 05 |
| jsonwebtoken | JWT signing + verification | 05 |
| Next.js | Frontend framework | 04 |
| React Query | Data fetching + caching | 04 |
| Zustand | State management | 04 |
| Cloudinary | Video + file storage | 09 |
| Stripe | Payment processing | 10 |
| Redis / ioredis | Caching + rate limiter store | 13 |
| Resend | Transactional email | 14 |
| pdfkit | Certificate PDF generation | 11 |
| Docker Compose | Local services (MongoDB + Redis) | 03 |
| PM2 | Process management | 17 |
| Nginx | Reverse proxy + HTTPS termination | 17 |
| Certbot | TLS certificate management | 17 |

---

## Learning log

Create a `learning-log/` folder in your EduFlow repo. For every chapter, copy `00-template.md`, rename it to `NN-chapter-name.md`, and write your answers before moving on. Commit after each chapter. This folder is the evidence that you understood the work — and your preparation for the viva.

---

*EduFlow — built chapter by chapter, explained decision by decision.*
