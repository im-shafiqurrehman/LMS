# Notifications — Transactional Email

Three events in EduFlow trigger email notifications: welcome on registration, enrollment confirmation, and certificate ready. This chapter sends them via **Resend** using a fire-and-forget pattern.

---

## Why Resend

Sending email directly from your Express server (via SMTP) works in development but fails in production at scale. IP reputation, deliverability, and bounce handling are complex problems. Resend is a developer-focused transactional email API that handles all of that — you call an API, the email gets delivered.

Install the Resend SDK:

```
npm install resend
```

---

## The email helper

Create `src/lib/email.js`:

```js
// src/lib/email.js — shape only
import { Resend } from 'resend';
import { config } from '../config/index.js';

const resend = new Resend(config.resendApiKey);

export async function sendWelcomeEmail({ to, name }) {
  await resend.emails.send({
    from: 'EduFlow <noreply@yourdomain.com>',
    to,
    subject: 'Welcome to EduFlow',
    html: `<p>Hi ${name}, welcome to EduFlow...</p>`,
  });
}

export async function sendEnrollmentEmail({ to, name, courseTitle }) {
  await resend.emails.send({
    from: 'EduFlow <noreply@yourdomain.com>',
    to,
    subject: `You're enrolled: ${courseTitle}`,
    html: `<p>Hi ${name}, you are now enrolled in ${courseTitle}...</p>`,
  });
}

export async function sendCertificateEmail({ to, name, courseTitle, pdfUrl }) {
  await resend.emails.send({
    from: 'EduFlow <noreply@yourdomain.com>',
    to,
    subject: `Your certificate for ${courseTitle} is ready`,
    html: `<p>Hi ${name}, congratulations on completing ${courseTitle}. <a href="${pdfUrl}">Download your certificate</a>.</p>`,
  });
}
```

---

## The fire-and-forget pattern

Email sending is not critical to the response the user receives. If the welcome email fails to send, the registration still succeeded — the user should not receive a 500 error because of an email problem.

The pattern: call the email function without `await` and handle errors with `.catch()`:

```js
// In auth.service.js — after saving the user
sendWelcomeEmail({ to: user.email, name: user.email }).catch(err =>
  console.error('Failed to send welcome email:', err)
);

// Return the success response immediately — don't wait for email
return { message: 'Registration successful.' };
```

The email is sent asynchronously in the background. The user gets their response instantly; the email catches up a moment later.

✅ Wrap email calls in `.catch()` so a failed send does not throw an unhandled rejection.
❌ Do not `await` email sends inside the main response path — a slow or failed email service would delay every response.

---

## Where each email is sent

| Event | Where it is triggered | Email function |
|-------|-----------------------|---------------|
| Registration | `auth.service.js`, after `user.save()` | `sendWelcomeEmail` |
| Enrollment | `enrollments.service.js`, after creating the `Enrollment` document | `sendEnrollmentEmail` |
| Certificate issued | `progress.service.js`, after `generateCertificate` completes | `sendCertificateEmail` |

---

## Definition of Done

- [ ] A new registration triggers a welcome email (verify in the Resend dashboard)
- [ ] A completed Stripe payment triggers an enrollment confirmation email
- [ ] Course completion triggers a certificate-ready email with the PDF link
- [ ] An email send failure does not return an error to the user
