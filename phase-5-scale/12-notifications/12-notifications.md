# Notifications — Email That Actually Matters

Email notifications are the thread that connects EduFlow's events to the real world. When a student enrolls, they should get a receipt. When their certificate is ready, they should know. When an instructor's course is approved, they should hear. These aren't nice-to-haves — they're the communication layer that makes the platform feel professional and trustworthy.

This chapter adds transactional email via **Resend** and introduces the pattern of sending email **after the response** — not during the request.

---

## Why Resend

Sending email via SMTP yourself is a path to spam folders and deliverability headaches. Resend is a developer-first email API with clean Node.js support, reliable deliverability, and a generous free tier. The alternative you'll most often see is SendGrid or Nodemailer with an SMTP relay — Resend is cleaner for new projects.

> **Worth reading:** Search "transactional email vs marketing email" — the difference matters for how email services treat your messages. Transactional email (receipts, confirmations, notifications) has different compliance rules than marketing campaigns.

---

## Install and configure Resend

```bash
npm install resend
```

Configure in `src/lib/email.js`:

```js
// Initialise the Resend client with config.resendApiKey
// Export a sendEmail(to, subject, html) helper function
// The helper should:
//   - Call resend.emails.send({ from: 'EduFlow <noreply@yourdomain.com>', to, subject, html })
//   - Log success or failure
//   - Never throw — a failed email must not crash the request
```

The "never throw" rule is important. If the email service is down, the enrollment still happened — the student paid. Don't let an email failure roll back a payment confirmation.

---

## Email templates

Create `src/lib/email-templates.js` with functions that return HTML strings for each email type. Keep the HTML simple — most email clients render a stripped-down subset of HTML.

Templates to build:

### Welcome email

Triggered: after email verification (`POST /auth/verify-email` success).

Content:
- Subject: `Welcome to EduFlow, [name]!`
- Body: confirmation they're verified, a link to browse the catalogue

### Enrollment confirmation

Triggered: inside `enrollStudent()` in the Stripe webhook handler.

Content:
- Subject: `You're enrolled in [Course Title]`
- Body: course name, instructor name, a link to start the first lesson, the amount charged

### Certificate ready

Triggered: inside `generateCertificate()`.

Content:
- Subject: `Your certificate for [Course Title] is ready`
- Body: congratulations message, a link to download the certificate

### Course approved (to instructor)

Triggered: inside the admin course approval service function.

Content:
- Subject: `Your course "[Title]" has been approved`
- Body: the course is now live, a link to view it in the catalogue

### Course rejected (to instructor)

Triggered: inside the admin course rejection service.

Content:
- Subject: `Action needed: "[Title]" needs changes`
- Body: the rejection reason (pass it from the admin's decision), guidance to edit and resubmit

---

## Sending email after the response

There is a subtlety in when to send email. The naive approach calls `sendEmail()` inside the request handler and `await`s it:

```js
// Slow — holds the request open while email sends
await enrollStudent(studentId, courseId);
await sendEmail(to, subject, html);   // might take 500ms
res.json({ success: true });
```

The student's browser waits for the email API call to complete before receiving their success response. Email sending can take hundreds of milliseconds — sometimes longer if the email service is slow or briefly down.

The better pattern: send the response first, then fire the email:

```js
// Fast — response goes out immediately
await enrollStudent(studentId, courseId);
res.json({ success: true });

// Email fires after the response is sent — the student doesn't wait for it
sendEmail(to, subject, html).catch(err => logger.error('Email failed:', err));
```

This works for transactional emails that are "best effort" — they should be sent, but a failure shouldn't block the core operation. For critical cases (like sending a verification OTP), you may want to await the email, but even then consider wrapping in a short timeout.

---

## Wire the emails

Go through each trigger point and add the `sendEmail()` call:

| Trigger location | Template to send |
|---|---|
| `auth.service.js` → after email verification | Welcome email |
| `enrollments.service.js` → `enrollStudent()` | Enrollment confirmation |
| `certificates.service.js` → after PDF upload | Certificate ready |
| `admin.service.js` → course approval | Course approved (to instructor) |
| `admin.service.js` → course rejection | Course rejected (to instructor) |

For the enrollment confirmation, you need the student's email address — join against the `User` table in the `enrollStudent` query.

✅ Test each email by triggering the event and checking the Resend dashboard (test mode delivers to your Resend account inbox).
❌ Don't hardcode "from" email addresses — put them in config.

---

## Definition of Done

- [ ] Welcome email is sent after email verification
- [ ] Enrollment confirmation email is sent after a successful Stripe webhook
- [ ] Certificate ready email is sent when a certificate is generated
- [ ] Course approval and rejection emails are sent to the instructor
- [ ] A Resend API failure logs the error but does not return a 500 to the client
- [ ] Email sending does not add more than ~50ms to the request response time (fire-and-forget works)

Write in `learning-log/12-notifications.md`:

1. Why is email sent after the response is sent rather than before?
2. What is the risk of `await`-ing the email send inside the request handler?
3. Why should a failed email not cause the overall operation (enrollment, certificate generation) to fail?
