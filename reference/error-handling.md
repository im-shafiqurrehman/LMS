# Error Handling — Reference and Implementation Guide

Consistent error handling is the difference between an API that helps developers debug and one that silently returns 500 and forces them to dig through logs. EduFlow uses a single, centralized error handler and a lightweight custom error class. This document explains both and shows how to use them throughout the codebase.

---

## The custom error class

Create `src/lib/AppError.js`:

```js
// src/lib/AppError.js
export class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.name = 'AppError';
    Error.captureStackTrace(this, this.constructor);
  }
}
```

Usage inside any service function:

```js
import { AppError } from '../../lib/AppError.js';

// 404 — resource not found
throw new AppError('Course not found.', 404);

// 409 — conflict
throw new AppError('An account with this email already exists.', 409);

// 403 — forbidden
throw new AppError('You are not enrolled in this course.', 403);

// 400 — bad request / business rule violation
throw new AppError('A course must have at least 3 lessons before submission.', 400);
```

By throwing `AppError` with a status code attached, the global error handler can respond with the right HTTP status without needing any logic inside route handlers.

---

## The global error handler

Create `src/middleware/errorHandler.js`:

```js
// src/middleware/errorHandler.js
// This must be added to app.js AFTER all routes — Express identifies error handlers by their 4-arg signature.

export function errorHandler(err, req, res, next) {
  // Log the error (in production, send to a logging service like Sentry)
  console.error(`[${new Date().toISOString()}] ${req.method} ${req.path}`, {
    message: err.message,
    stack: process.env.NODE_ENV === 'development' ? err.stack : undefined,
  });

  // Prisma-specific errors
  if (err.code === 'P2002') {
    // Unique constraint violation (e.g. duplicate email)
    return res.status(409).json({ error: 'A record with that value already exists.' });
  }

  if (err.code === 'P2025') {
    // Record not found (findUniqueOrThrow / updateOrThrow)
    return res.status(404).json({ error: 'Record not found.' });
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({ error: 'Invalid token.' });
  }

  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({ error: 'Token expired.' });
  }

  // Zod validation errors (if you throw them from the validator middleware)
  if (err.name === 'ZodError') {
    return res.status(400).json({
      error: 'Validation failed',
      details: err.errors.map(e => ({ field: e.path.join('.'), message: e.message }))
    });
  }

  // Our own AppError
  if (err.name === 'AppError') {
    return res.status(err.statusCode).json({ error: err.message });
  }

  // Multer errors (file upload)
  if (err.code === 'LIMIT_FILE_SIZE') {
    return res.status(400).json({ error: 'File too large. Maximum size is 500MB.' });
  }

  // Stripe errors
  if (err.type === 'StripeInvalidRequestError') {
    return res.status(400).json({ error: 'Payment request failed: ' + err.message });
  }

  // Default — unhandled exception
  const statusCode = err.statusCode || err.status || 500;
  const message = process.env.NODE_ENV === 'production'
    ? 'An unexpected error occurred.'   // Don't leak internals in production
    : err.message;

  return res.status(statusCode).json({ error: message });
}
```

Wire it in `app.js` **after all routes**:

```js
// app.js — last middleware
app.use(errorHandler);
```

---

## The validator middleware

Create `src/middleware/validate.js`. Every route that accepts a request body uses this:

```js
// src/middleware/validate.js
import { ZodError } from 'zod';

export function validate(schema) {
  return (req, res, next) => {
    try {
      // Parse and replace req.body with the validated + typed data
      req.body = schema.parse(req.body);
      next();
    } catch (err) {
      if (err instanceof ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: err.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message
          }))
        });
      }
      next(err);
    }
  };
}
```

Usage on a route:

```js
import { validate } from '../../middleware/validate.js';
import { registerSchema } from './auth.validators.js';

router.post('/register', authLimiter, validate(registerSchema), registerHandler);
```

The handler receives a fully validated `req.body` — no need to check individual fields inside the handler.

---

## Async wrapper — never forget try/catch again

Express doesn't catch async errors automatically (in Express 4). Unhandled promise rejections in route handlers crash the process. Wrap every async handler:

```js
// src/lib/asyncHandler.js
export const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};
```

Usage:

```js
import { asyncHandler } from '../../lib/asyncHandler.js';

router.post('/register', authLimiter, validate(registerSchema), asyncHandler(async (req, res) => {
  await authService.registerUser(req.body);
  res.status(201).json({ message: 'Account created. Check your email for a verification code.' });
}));
```

Any error thrown inside the async function is caught by `asyncHandler` and forwarded to the global error handler via `next(err)`.

> **Note:** Express 5 (in stable release) handles async errors natively — the `asyncHandler` wrapper is not needed. If you upgrade to Express 5, remove all the wrappers.

---

## Error handling rules — apply these everywhere

1. **Service functions throw `AppError`** with the correct status code and a human-readable message. They never call `res.json()` directly.
2. **Route handlers never contain business logic** — they call a service function, handle the happy path, and let errors bubble to the global handler.
3. **The global error handler is the only place that calls `res.status().json()`** for errors.
4. **Prisma errors are caught in the global handler** — you don't need try/catch around every Prisma call for common errors (unique violation, not found). Only wrap in try/catch when you need different behaviour than the default.
5. **Email failures are swallowed silently** — the `.catch()` on fire-and-forget email calls logs the error and discards it. A failed email must not propagate to the response.
6. **In production, 500 errors never leak internal detail** — the message is `"An unexpected error occurred."` The actual error is logged for the engineer, not returned to the client.

---

## Error patterns quick reference

| Scenario | Throw | Result |
|---|---|---|
| Email already registered | `new AppError('An account with this email already exists.', 409)` | HTTP 409 |
| Resource not found (no ownership issue) | `new AppError('Course not found.', 404)` | HTTP 404 |
| Resource exists but not owned by requester | `new AppError('Course not found.', 404)` — **use 404, not 403** | HTTP 404 |
| User is authenticated but wrong role | Return 403 from `authorize()` middleware | HTTP 403 |
| User not enrolled, tries to access lesson | `new AppError('Enroll in this course to access this lesson.', 403)` | HTTP 403 |
| Business rule violated (e.g. too few lessons) | `new AppError('A course must have at least 3 lessons before submission.', 400)` | HTTP 400 |
| Stripe payment fails | Handled by Stripe error type in global handler | HTTP 400 |
| Webhook signature invalid | `new AppError('Invalid webhook signature.', 400)` | HTTP 400 |
| Deactivated user tries to log in | `new AppError('Your account has been deactivated. Contact support.', 403)` | HTTP 403 |
