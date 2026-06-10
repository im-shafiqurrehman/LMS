# Viva Preparation Guide

The viva is a live conversation, not a quiz. You will be asked to explain your system, defend your decisions, and reason through hypothetical failures. The questions don't test memory — they test whether you understand what you built and why.

This guide gives you the full question bank. Use it two ways: as a study guide before the viva, and as a self-assessment checkpoint at the end of each chapter.

---

## How the viva works

Expect 45–60 minutes. The conversation covers three areas:

1. **System walkthrough** — "Walk me through what happens when a student clicks 'Enroll'." You trace the request end-to-end.
2. **Decision defence** — "Why did you choose cursor pagination over offset?" You explain the decision and what the alternative would have cost.
3. **Failure scenarios** — "What happens if Redis goes down while a student is browsing the catalogue?" You reason through the system's behaviour under failure.

The interviewer is not trying to catch you out. They are checking whether you actually understand your own system, or whether you assembled it from copied code without knowing what it does.

---

## The question bank

### Architecture and design

1. Walk me through EduFlow's architecture from a student clicking "Buy Course" to them watching the first lesson. Name every component that handles the request at each step.

2. Why did you choose PostgreSQL over MongoDB for this project? Give me a concrete example from the EduFlow schema where MongoDB would have caused a problem.

3. Your application makes heavy use of foreign keys and cascade deletes. What does `onDelete: Cascade` actually do in PostgreSQL, and what would happen if it weren't there?

4. Why does the project separate `app.js` from `server.js`? What does that make possible that having both in one file would make harder?

5. What is the Prisma client singleton pattern, and what specific problem does it solve in a hot-reloading development environment?

---

### Authentication and security

6. What is the difference between hashing and encryption? Why is bcrypt used for passwords instead of AES?

7. What is a salt, and what specific attack does it prevent? Walk me through a rainbow table attack on an unsalted hash.

8. Walk me through the two-token pattern. Why is the access token short-lived? Why is the refresh token stored hashed in the database?

9. What would go wrong if you stored the refresh token as plaintext rather than hashed?

10. A user's access token is compromised and leaks to an attacker. What damage can the attacker do, and when does it stop?

11. What does `jwt.verify()` check, and what does it explicitly *not* check?

12. Why does the algorithm need to be specified explicitly in `jwt.verify({ algorithms: ["HS256"] })`? What attack does omitting it enable?

13. Why does the login endpoint return the same error message for a wrong email and a wrong password?

---

### Authorization and data isolation

14. What is the difference between authentication and authorization? Give me a concrete example of each in EduFlow.

15. What is an IDOR vulnerability? Walk me through exactly how an attacker could exploit one in EduFlow if the ownership check in `updateCourse` were missing.

16. Why does the ownership check return 404 instead of 403 when a resource belongs to another user?

17. Why is `authorize('ADMIN')` applied at the router level in `admin.routes.js` rather than on each individual route?

18. Why can't you put the enrollment check (whether a student is enrolled) in the `authorize` middleware instead of in the service function?

---

### Data modelling

19. Why does `amountPaid` live on the `Enrollment` record rather than being derived from the current `Course.price`?

20. What does `@@unique([studentId, courseId])` on `Enrollment` prevent, and why is that important for the Stripe webhook?

21. The `Lesson` model has `@@unique([courseId, position])`. What would go wrong during lesson reordering if this constraint didn't exist?

22. Why does `emailVerifiedAt` store a DateTime rather than a Boolean `isVerified`?

23. What is the `deactivatedAt` pattern on `User`, and how does it differ from deleting the user? When would you prefer one over the other?

---

### Payments

24. Why is enrollment triggered by a Stripe webhook rather than by the payment redirect URL?

25. What is idempotency, and how does the `upsert` on `Enrollment` make the webhook idempotent?

26. Why must the Stripe webhook endpoint use `express.raw()` instead of `express.json()`?

27. What is a PaymentIntent? What is the difference between creating a PaymentIntent and a Charge?

28. A webhook event fires twice due to a Stripe retry. Walk me through exactly what happens in your system on the second call.

---

### Performance and caching

29. What is the N+1 query problem? Show me a specific example of where it would occur in EduFlow without `include`.

30. What is cursor pagination? Walk me through the SQL (or Prisma query) for fetching the second page of catalogue results.

31. What is the cache-aside pattern? Walk me through a cache miss and a cache hit on the catalogue endpoint.

32. What is cache invalidation, and when does EduFlow invalidate the catalogue cache? What would happen if you forgot to invalidate after approving a course?

33. What does TTL on a cache entry do? Why does the catalogue cache have a TTL even when explicit invalidation is implemented?

34. What is a GIN index, and why does full-text search use it instead of a B-tree index?

---

### Failure scenarios (the hard ones)

35. Redis goes down while a student is browsing the catalogue. What happens? What does the student see? What do your logs show?

36. The Resend email service is down when a student enrolls. What happens to the enrollment? What happens to the confirmation email?

37. A student's browser tab closes right after Stripe processes the payment, before the redirect completes. Are they enrolled? Why?

38. Two concurrent requests both mark the final lesson complete for the same enrollment at the same millisecond. How many certificates are generated? Walk me through exactly why.

39. Your VPS runs out of disk space during a video upload. What happens? Is the lesson in a corrupted state? How would you recover it?

40. The `JWT_SECRET` is accidentally committed to the public GitHub repo. What is the impact? What do you do in the next 10 minutes?

---

### Deployment and operations

41. Why does Node.js run behind Nginx rather than listening on port 80 directly?

42. What does PM2's cluster mode do? How does it relate to Node.js being single-threaded?

43. What is the difference between `prisma migrate dev` and `prisma migrate deploy`? Why must you use `deploy` in production?

44. Walk me through what happens when the VPS reboots. Does EduFlow come back up automatically? How?

45. A student reports that lessons are loading slowly. Walk me through how you would diagnose the problem — starting from a support ticket, no other information.

---

## Preparing effectively

**For each question:** write a two-paragraph answer in your own words. The first paragraph defines the concept. The second gives a concrete EduFlow example.

**Don't memorise answers** — understand the concept well enough to produce an answer you've never said out loud before. The viva conversation will not follow this exact script.

**Use your learning log** — if a question relates to a chapter, re-read what you wrote in the learning log. If your log answer is thin, expand it now.

**Practice out loud** — say the answer to an empty room, a phone camera, or a study partner. The gap between "I understand this" and "I can explain it clearly" closes only through speaking.

---

## The bar for passing

You pass if:

- You can trace any request end-to-end without pausing to look things up
- You can explain every decision with a concrete reason (not "because the chapter said so")
- You can reason through failure scenarios even if you haven't seen that exact scenario before
- You notice when a question has a trade-off and you name both sides before picking one

You don't pass if:

- You can't explain code in your own repository
- You answer "I don't know" more than twice without recovering
- You describe what the code does but can't explain why it was written that way
