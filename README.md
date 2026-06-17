# EduFlow LMS

A full-stack Learning Management System built with **Next.js**, **Express.js**, **MongoDB**, **Mongoose**, and **Redux Toolkit**.

This repository contains the complete course documentation for building EduFlow from scratch — from project skeleton to production deployment.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14 (App Router), TypeScript, Tailwind CSS |
| State Management | Redux Toolkit, React-Redux |
| Backend | Express.js (Node.js) |
| Database | MongoDB (Atlas in production, Docker locally) |
| ODM | Mongoose |
| Authentication | JWT (access + refresh token pattern) |
| Payments | Stripe (PaymentIntents + webhooks) |
| Caching | Redis (ioredis) |
| Search | MongoDB Atlas Search |
| Deployment | VPS + PM2 + Nginx + Let's Encrypt |

## Course Outline

See [COURSE-OUTLINE.md](./COURSE-OUTLINE.md) for the complete chapter list.

## Phases

| Phase | Chapters | Focus |
|-------|---------|-------|
| 1 — Foundation | 01–04 | Skeleton, config, database connection |
| 2 — Identity | 05–08 | Data model, auth, authorization |
| 3 — Learning Core | 09–12 | Catalogue, instructor tools |
| 4 — Transactions | 13–16 | Payments, enrollment, lesson access |
| 5 — Scale | 17–18 | Progress, caching, search, rate limiting |
| 6 — Admin + Ship | 19–20 | Admin dashboard, VPS deployment |

## Learning Log

Write your chapter notes in `learning-log/`. Use the template at `learning-log/00-template.md`.

## Key Decisions

- **MongoDB over PostgreSQL**: flexible course schema, horizontal sharding, schema evolution without migrations
- **Mongoose over native driver**: schema validation, pre-save hooks (password hashing), .populate() for N+1 prevention
- **Redux Toolkit over React Query**: one source of truth for all frontend state, no library duplication
- **Cursor pagination**: no skip() scan cost at scale; no page-drift on real-time data
- **Two-token auth**: short-lived access tokens (15m), long-lived refresh tokens stored hashed — revocable without token expiry
- **Stripe webhook idempotency**: enrollment created only once even if Stripe delivers the webhook multiple times

## Running Locally

```bash
# 1. Start the database
docker compose up -d

# 2. Backend
cd server
cp .env.example .env   # fill in secrets
npm install
npm run seed
npm run dev            # http://localhost:3000

# 3. Frontend
cd client
cp .env.local.example .env.local
npm install
npm run dev -- --port 3001  # http://localhost:3001
```

## Documentation Structure

Each chapter folder contains numbered sub-chapters (`.md` files). Work through them in order. Each chapter ends with a Definition of Done checklist — do not advance with unticked boxes.
