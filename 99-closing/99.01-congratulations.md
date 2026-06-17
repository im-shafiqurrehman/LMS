# Close Out — Document, Demonstrate, and Reflect

You built it. Now you own it: a real URL, a working application, a codebase you understand. This chapter closes the course.

---

## Documentation

A codebase without documentation is a system only the original author understands — and even they forget after six months.

Create a `docs/` folder in your repo and write:

- **`docs/architecture.md`** — A diagram (draw.io, Excalidraw, or plain ASCII) showing the request flow: browser → Nginx → Next.js frontend / Express backend → MongoDB, Redis, Cloudinary, Stripe, Resend.
- **`docs/erd.md`** — The entity-relationship diagram: all Mongoose models, their fields, and the references between them.
- **`docs/api.md`** — A list of every API endpoint: method, path, auth required, request body shape, response shape.

This documentation has two audiences: an interviewer who wants to understand the system quickly, and your future self in six months.

---

## The demo video

Record a 3–5 minute screen recording that shows the full product loop:

1. Register as an instructor, create a course, add a lesson, submit for review.
2. Approve the course as an admin.
3. Register as a student, find the course in the catalogue, enroll (use a Stripe test card).
4. Watch the first lesson, mark it complete.
5. Complete all lessons, download the certificate.

Host it on YouTube (unlisted is fine) and add the link to your `README.md`.

---

## The portfolio write-up

The demo video shows what you built. The write-up explains why. Write 400–600 words on:

- The technical decisions you made and why (MongoDB + Mongoose, cursor pagination, signed Cloudinary URLs)
- One thing that turned out to be harder than expected and how you solved it
- One thing you would do differently now

Publish it on LinkedIn, your blog, or a dev platform (dev.to, Hashnode). This is the artefact that persists after the demo video is gone.

---

## Bar-raiser (one line)

Pick one creative feature or technical improvement that does not break any core rules (data isolation, no hardcoded secrets). Ideas: real-time progress updates via Server-Sent Events, course completion leaderboard, instructor revenue dashboard, automated email sequences after enrollment.

Implement it and document the decision.

---

## The viva

You will be asked to walk through how EduFlow works and answer "what would break if…" questions. Typical questions:

- "Walk me through what happens when a student buys a course."
- "What would break if you removed the compound index on `{ status, createdAt }` from courses?"
- "How does the refresh token pattern prevent an attacker from using a stolen access token indefinitely?"
- "Why does the ownership check return 404 instead of 403?"

Your learning log entries are your preparation. If you wrote genuine answers — not summaries of the chapter, but your own reasoning — you are ready.

---

## Final Definition of Done

- [ ] `docs/architecture.md`, `docs/erd.md`, and `docs/api.md` exist and are accurate
- [ ] A demo video covers the full product loop and is linked from `README.md`
- [ ] A portfolio write-up is published and linked
- [ ] The bar-raiser feature is implemented and documented
- [ ] Every chapter's learning log file is committed

_EduFlow is shipped. You built a real system, understood every decision you made, and can explain it out loud. That is the whole point._
