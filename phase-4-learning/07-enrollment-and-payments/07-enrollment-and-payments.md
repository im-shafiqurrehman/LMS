# Enrollment and Payments

A student browsing the catalogue sees a course they want. They click "Enroll." This chapter builds that flow: collecting payment via Stripe and granting access to the course only after the payment succeeds.

---

## Why not confirm enrollment on the frontend

The most obvious approach: the student pays on the frontend, the frontend tells the backend "payment succeeded," and the backend enrolls them. The problem: the frontend is client code. Anyone can send that "payment succeeded" message without paying anything. You cannot trust the client to report payment success.

The correct approach uses **Stripe webhooks**. The flow:

1. The backend creates a Stripe Payment Intent and returns its `client_secret` to the frontend.
2. The frontend uses the Stripe.js library to collect card details and confirm the payment directly with Stripe's servers.
3. Stripe sends an `payment_intent.succeeded` webhook event to your backend.
4. The backend verifies the webhook signature, then enrolls the student.

The enrollment happens only when Stripe's servers tell your backend it succeeded — never based on the client's self-report.

---

## The Enrollment model

Create `src/models/enrollment.model.js`:

```js
// src/models/enrollment.model.js — shape only
import mongoose from 'mongoose';

const enrollmentSchema = new mongoose.Schema(
  {
    studentId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    courseId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Course',
      required: true,
    },
    amountPaid:             { type: Number, required: true }, // price snapshot
    stripePaymentIntentId:  { type: String, required: true, unique: true },
    enrolledAt:             { type: Date, default: Date.now },
  }
);

// A student cannot enroll in the same course twice
enrollmentSchema.index({ studentId: 1, courseId: 1 }, { unique: true });
enrollmentSchema.index({ courseId: 1 });

export const Enrollment = mongoose.model('Enrollment', enrollmentSchema);
```

Notice `amountPaid`. This stores what the student actually paid at enrollment time — not the course's current price. If the instructor raises the price next month, no existing enrollment is affected.

---

## The payment flow

Scaffold:

```
mkdir -p src/modules/enrollments
touch src/modules/enrollments/enrollments.routes.js
touch src/modules/enrollments/enrollments.service.js
```

**Step 1 — Create a Payment Intent**

`POST /api/enrollments/create-payment-intent`  
Auth: student  
Body: `{ courseId }`

The service must:

1. Fetch the course and confirm it is published.
2. Check that the student is not already enrolled: `Enrollment.findOne({ studentId, courseId })` — if found, return 409.
3. Call `stripe.paymentIntents.create({ amount: course.price * 100, currency: 'usd', metadata: { studentId, courseId } })`.
4. Return `{ clientSecret: paymentIntent.client_secret }`.

The frontend uses the `clientSecret` to complete payment with Stripe.js. Nothing is stored in your database yet — enrollment happens in step 2.

**Step 2 — Handle the Stripe webhook**

`POST /api/enrollments/webhook`  
No auth — verified by signature instead.

This endpoint must be registered **before** `express.json()` in `app.js`, with the raw request body. Stripe requires the raw body to verify the signature:

```js
// In app.js — raw body for the webhook, before express.json()
app.use('/api/enrollments/webhook', express.raw({ type: 'application/json' }));
```

The webhook handler must:

1. Verify the signature: `stripe.webhooks.constructEvent(req.body, req.headers['stripe-signature'], config.stripeWebhookSecret)`.
2. If the signature fails, return 400.
3. Handle only `payment_intent.succeeded` events.
4. Extract `studentId` and `courseId` from `event.data.object.metadata`.
5. Create the enrollment: `new Enrollment({ studentId, courseId, amountPaid: amount / 100, stripePaymentIntentId: id })`.
6. Return 200.

Idempotency: Stripe may deliver the same webhook multiple times. The `unique: true` index on `stripePaymentIntentId` ensures a duplicate delivery does not create a duplicate enrollment — the second insert will fail with a duplicate key error, which you catch and ignore.

Install Stripe:

```
npm install stripe
```

> **Worth reading:** Search "Stripe webhooks Node.js" — the official Stripe documentation on webhook verification is clear and includes Node.js examples.

---

## Definition of Done

- [ ] `POST /api/enrollments/create-payment-intent` returns a Stripe `clientSecret` for a valid course
- [ ] Attempting to create a payment intent for an already-enrolled course returns 409
- [ ] The webhook endpoint verifies the Stripe signature and rejects unverified requests with 400
- [ ] A `payment_intent.succeeded` event creates an enrollment with the correct `amountPaid`
- [ ] A duplicate webhook (same `stripePaymentIntentId`) does not create a duplicate enrollment

Write in `learning-log/07-enrollment-and-payments.md`:

1. Why can you not trust the frontend to report payment success?
2. What does Stripe's webhook signature verification protect against?
3. Why does the `Enrollment` model store `amountPaid` rather than referencing the course price at query time?
4. What does idempotency mean in the context of the Stripe webhook handler?
