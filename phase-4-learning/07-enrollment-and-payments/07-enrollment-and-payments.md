# Enrollment and Payments

Enrollment is the moment EduFlow makes money. It's also the moment where correctness matters most — a student who pays and isn't enrolled, or a student who's enrolled without paying, are both catastrophic in different ways. This chapter builds the enrollment flow with Stripe handling the payment, and a webhook confirming it.

---

## Why webhooks, not a redirect

The naive approach to payment: the student clicks "Pay," gets redirected to Stripe, Stripe redirects them back to your `/payment/success` page, and your server enrolls them. This is wrong. The redirect can fail — the tab closes, the connection drops, the browser navigates away. You lose the confirmation. The student paid and isn't enrolled. Support tickets follow.

The correct approach: Stripe sends an **event** to your server via a webhook when the payment is confirmed on their side. Your server listens for the `payment_intent.succeeded` event and enrolls the student only then. The webhook is server-to-server — it doesn't depend on the student's browser staying open. The redirect is only for UX (showing "payment successful"); enrollment happens on the webhook, not the redirect.

> **Worth reading:** Search "Stripe webhooks guide" — the official Stripe docs explain the webhook lifecycle clearly. Read the "Receive events" section before implementing.

---

## The Enrollment schema

Add to `prisma/schema.prisma`:

```prisma
model Enrollment {
  id                   String    @id @default(uuid())
  studentId            String
  student              User      @relation("StudentEnrollments", fields: [studentId], references: [id])
  courseId             String
  course               Course    @relation(fields: [courseId], references: [id])
  amountPaid           Decimal   @db.Decimal(10, 2)
  stripePaymentIntentId String   @unique
  enrolledAt           DateTime  @default(now())

  progressRecords ProgressRecord[]
  certificate     Certificate?

  @@unique([studentId, courseId])
  @@index([studentId])
  @@index([courseId])
}
```

`@@unique([studentId, courseId])` prevents a student from being enrolled in the same course twice — an important guard against duplicate webhooks.

`amountPaid` stores what was actually charged at enrollment time. If the course price changes later, existing enrollment records are unaffected — they show what the student actually paid.

Run: `npx prisma migrate dev --name add-enrollment`

---

## Install Stripe

```bash
npm install stripe
```

Configure in `src/lib/stripe.js`:

```js
// Initialise and export the Stripe client using config.stripeSecretKey
```

---

## Scaffold the enrollment module

```bash
mkdir -p src/modules/enrollments
touch src/modules/enrollments/enrollments.routes.js
touch src/modules/enrollments/enrollments.service.js
```

Mount at `/api/enrollments` in `app.js`.

---

## Create a payment intent

```
POST /api/enrollments/checkout
```

Protected: `authenticate` + `authorize('STUDENT')`.

Request body:

```json
{ "courseId": "uuid" }
```

Response:

```json
{
  "clientSecret": "pi_abc_secret_xyz",
  "paymentIntentId": "pi_abc"
}
```

The service must:

1. Find the course — return 404 if not found or not `PUBLISHED`
2. Check that the student is not already enrolled — return 409 if they are
3. Create a Stripe PaymentIntent: `stripe.paymentIntents.create({ amount, currency, metadata: { studentId, courseId } })`
   - `amount` is in cents (multiply the course price in dollars by 100)
   - `metadata` is critical — it's what the webhook uses to know who to enroll in which course
4. Return the `client_secret` to the frontend

The frontend uses the `client_secret` with Stripe.js to render the payment form. The server never handles card details.

✅ Verify: calling this endpoint creates a PaymentIntent in your Stripe test dashboard.
❌ Don't store card details anywhere. Stripe handles that; your server only ever sees the PaymentIntent ID.

---

## The Stripe webhook

```
POST /api/webhooks/stripe
```

This route is **public** (no `authenticate` middleware) — it's called by Stripe's servers, not the student's browser. But it must verify the request came from Stripe using the webhook signature.

Create `src/modules/webhooks/stripe.webhook.js`.

**Critical:** This route must use `express.raw()` middleware instead of `express.json()` — Stripe's signature verification requires the raw request body. Mount the webhook route before the `express.json()` middleware in `app.js`, or apply `express.raw()` specifically to this route.

```js
// Webhook handler — shape
// 1. Get the raw body and the 'stripe-signature' header
// 2. Call stripe.webhooks.constructEvent(rawBody, signature, config.stripeWebhookSecret)
//    — throws if the signature is invalid
// 3. Switch on event.type:
//    case 'payment_intent.succeeded': enrollStudent(event.data.object)
// 4. Return HTTP 200 immediately — Stripe will retry if it doesn't get 200
```

The `enrollStudent` function:

1. Extract `studentId` and `courseId` from `paymentIntent.metadata`
2. Extract `amount` from `paymentIntent.amount_received` (convert cents back to dollars)
3. Extract `stripePaymentIntentId` from `paymentIntent.id`
4. Use `prisma.enrollment.upsert` with `where: { studentId_courseId: { studentId, courseId } }` — upsert handles the case where Stripe fires the webhook more than once (idempotency)
5. If the enrollment was newly created, trigger the enrollment confirmation email (queued for the notifications chapter)

> **Worth reading:** Search "Stripe webhook idempotency" — Stripe can deliver the same event more than once. Your webhook handler must be idempotent (safe to run multiple times). The `@@unique([studentId, courseId])` constraint and the `upsert` together make it so.

**Testing with Stripe CLI:**

```bash
# Install Stripe CLI, then:
stripe listen --forward-to localhost:3000/api/webhooks/stripe
stripe trigger payment_intent.succeeded
```

This simulates a successful payment locally without a real card.

✅ After the triggered event, an enrollment row exists in the database.
✅ Triggering the same event again does not create a duplicate enrollment (upsert works).

---

## Check enrollment (for lesson access gating)

Add to `enrollments.service.js`:

```js
async function isEnrolled(studentId, courseId) {
  const enrollment = await prisma.enrollment.findUnique({
    where: { studentId_courseId: { studentId, courseId } }
  });
  return !!enrollment;
}
```

Export it. You'll import it in the lesson access service next chapter.

---

## Definition of Done

- [ ] `POST /api/enrollments/checkout` creates a Stripe PaymentIntent and returns the `clientSecret`
- [ ] The Stripe webhook correctly enrolls a student on `payment_intent.succeeded`
- [ ] A second webhook event for the same payment does not create a duplicate enrollment
- [ ] An already-enrolled student attempting checkout receives 409
- [ ] `amountPaid` on the enrollment record matches the course price at time of enrollment
- [ ] The webhook rejects requests with an invalid Stripe signature (returns 400)

Write in `learning-log/07-enrollment-and-payments.md`:

1. Why is the enrollment triggered by the webhook and not the payment redirect?
2. What is idempotency, and how does the `upsert` + unique constraint make the webhook idempotent?
3. Why does `amountPaid` exist on the enrollment record rather than referencing the course's current price?
4. Why must the Stripe webhook use `express.raw()` instead of `express.json()`?
