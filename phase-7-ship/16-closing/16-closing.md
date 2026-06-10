# Document, Demonstrate, and Close

You've built EduFlow end to end. It runs in production, serves real HTTPS traffic, handles payments, protects video lessons behind enrollment checks, tracks progress, generates certificates, and runs behind a reverse proxy with a process manager. That's a real system.

This chapter closes the loop: you document it, demonstrate it, write it up, and prove you understand every part of it.

---

## Draw the architecture

Create an architecture diagram showing how the pieces connect. Include:

- The client (browser / Next.js frontend)
- Nginx (reverse proxy)
- Node.js / Express (the API, running in PM2 cluster)
- MongoDB (the database)
- Redis (cache + rate limiter)
- Cloudinary (video and file storage)
- Stripe (payment processing, webhooks)
- Resend (email delivery)

Use any tool: draw.io, Excalidraw, Miro, or a Mermaid diagram in your README. Save it as `docs/architecture.png` (or `.svg`) and reference it in the README.

---

## Draw the ERD

Export your full entity-relationship diagram. The easiest way:

```bash
npx prisma-erd-generator
```

Or use Prisma Studio's built-in ERD view, or draw it manually. Every table, every column, every relationship. Save as `docs/erd.png`.

---

## Write the README

Your `README.md` is the first thing anyone sees when they visit your repository. Write it like a product page for a developer audience:

**Sections to include:**

1. **Project overview** — what EduFlow is in two sentences
2. **Tech stack** — Node.js, Express, MongoDB, Prisma, Redis, Cloudinary, Stripe, Resend, Next.js, Docker, Nginx, PM2
3. **Architecture diagram** — embed the image
4. **ERD** — embed the image
5. **Local setup** — step-by-step from `git clone` to `npm run dev`, including Docker Compose and `.env` setup
6. **API reference** — a table of every endpoint: method, path, auth required, brief description
7. **Key engineering decisions** — 4–5 paragraphs on the decisions that shaped the system:
   - Why MongoDB over a relational database
   - Why JWT with refresh tokens (and the trade-offs)
   - Why Cloudinary for video (vs storing in the database)
   - Why webhooks drive enrollment (not the payment redirect)
   - One more of your choosing
8. **Known limitations and future work** — be honest about what's missing or would need work at real scale

A README that describes *why* decisions were made is what separates a portfolio project from a tutorial follow-along.

---

## Record a demo video

Record a 5–10 minute walkthrough:

1. Start at the live URL (`https://api.yourdomain.com`)
2. Show the Swagger/Postman collection (or plain curl commands) exercising the main flows:
   - Register → verify email → login
   - Browse the catalogue (show the Redis cache hit in logs)
   - Enroll in a course (trigger the Stripe webhook via CLI)
   - Watch a lesson (show the signed URL in the response)
   - Mark all lessons complete → certificate generated
   - Admin approval flow
3. Show the PM2 process list and the Nginx config
4. Show the database (Prisma Studio) with real data

Upload to YouTube (unlisted is fine) and link in the README.

---

## Portfolio write-up

Write a 400–600 word article about building EduFlow. Publish on Dev.to, Hashnode, or LinkedIn. The structure:

- What the project does and why you built it
- The most interesting engineering decision you made (pick one and go deep)
- What you'd do differently if you built it again
- A link to the repo and the live URL

This is not optional. The write-up forces you to articulate your decisions in plain language — which is exactly what a technical interview asks you to do. Doing it now, while the code is fresh, is the best time.

---

## Bar-raiser

Pick one of the following to implement — or propose your own that doesn't compromise data isolation or payment integrity:

- **Automatic subtitle generation** — integrate Cloudinary's auto-captioning for video lessons and expose the transcript via a lesson endpoint
- **Instructor payout tracking** — calculate per-instructor revenue (enrollment amount × platform commission) and expose it in the instructor dashboard
- **Course ratings and reviews** — students can rate a completed course (1–5 stars) and leave a text review; the catalogue shows average rating
- **Referral codes** — instructors get a shareable referral link; enrollments via the link record the referral and grant a discount

Choose the one that interests you most. Build it, document it in the README under "Extensions," and be ready to explain the schema and the access control choices.

---

## Evaluation

You'll walk through EduFlow in a live conversation — the viva. Expect questions like:

- "A student says they paid but aren't enrolled. Walk me through how you'd diagnose that."
- "How does your system ensure an instructor can't read another instructor's enrollments?"
- "What would break at 10,000 concurrent users? What would you fix first?"
- "Why did you use cursor pagination on the catalogue? What would change if you used offset?"
- "Explain what happens, step by step, when a student's access token expires and they make a request."

The viva is not a memory test. It's a reasoning test. If you understand every decision you made, you'll answer these without hesitation. If you copied code without understanding it, the answers will stall.

Your learning log is the evidence you understood the work. Bring it.

---

## Final checklist

- [ ] Architecture diagram exists in `docs/` and is referenced in the README
- [ ] ERD exists in `docs/` and is referenced in the README
- [ ] README includes local setup instructions (a new developer can clone and run it)
- [ ] README includes the API reference table
- [ ] README includes the key engineering decisions section
- [ ] Demo video is recorded and linked in the README
- [ ] Portfolio article is published and linked in the README
- [ ] The live URL responds to `GET /health` with a valid TLS certificate
- [ ] Every learning log file (`learning-log/NN-*.md`) is committed and answered
- [ ] The bar-raiser feature is implemented and documented

All boxes ticked. You're done. Ship it, share it, and be proud of it — you built something real.
