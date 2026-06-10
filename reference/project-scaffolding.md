# Project Scaffolding Reference

Everything you need to stand the project up from scratch. Use this alongside Chapter 3.

---

## `docker-compose.yml`

Place this in the project root. It runs MongoDB and Redis locally without installing them on your machine.

```yaml
# docker-compose.yml
services:
  mongodb:
    image: mongo:7
    container_name: eduflow_mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: yourpassword
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: eduflow_redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mongodb_data:
  redis_data:
```

**Commands:**

```bash
# Start both services in the background
docker compose up -d

# Stop both services (data persists in volumes)
docker compose stop

# Stop and remove volumes (wipe all data — use carefully)
docker compose down -v

# See logs
docker compose logs -f mongodb
docker compose logs -f redis

# Open a MongoDB shell
docker compose exec mongodb mongosh -u root -p yourpassword
```

---

## Complete project file tree

This is the full target structure. You build it incrementally across chapters.

```
eduflow/
├── .env                        ← Never committed
├── .env.example                ← Committed — same keys, no real values
├── .gitignore
├── docker-compose.yml
├── ecosystem.config.js         ← Chapter 16 (PM2)
├── package.json
├── package-lock.json
│
├── prisma/
│   ├── schema.prisma
│   ├── seed.js                 ← Chapter 5
│   └── migrations/
│       └── ...                 ← Auto-generated — committed
│
├── src/
│   ├── app.js                  ← Express app setup
│   ├── server.js               ← Entry point — starts listening
│   │
│   ├── config/
│   │   └── index.js            ← Env validation (Zod) — Chapter 3
│   │
│   ├── lib/
│   │   ├── AppError.js         ← Custom error class
│   │   ├── asyncHandler.js     ← Async route wrapper
│   │   ├── cloudinary.js       ← Chapter 6
│   │   ├── email.js            ← Chapter 12 (Resend)
│   │   ├── email-templates.js  ← Chapter 12
│   │   ├── prisma.js           ← Prisma client singleton
│   │   └── redis.js            ← Chapter 11 (ioredis)
│   │
│   ├── middleware/
│   │   ├── authenticate.js     ← JWT verification — Chapter 4
│   │   ├── authorize.js        ← Role enforcement — Chapter 5
│   │   ├── errorHandler.js     ← Global error handler
│   │   ├── rateLimiter.js      ← Chapter 13
│   │   ├── upload.js           ← Multer + Cloudinary — Chapter 6
│   │   └── validate.js         ← Zod request body validation
│   │
│   └── modules/
│       ├── auth/
│       │   ├── auth.helpers.js
│       │   ├── auth.routes.js
│       │   ├── auth.service.js
│       │   └── auth.validators.js
│       │
│       ├── courses/
│       │   ├── courses.routes.js       ← Public catalogue routes
│       │   ├── courses.service.js
│       │   ├── courses.validators.js
│       │   ├── instructor.routes.js    ← Instructor-only routes
│       │   └── instructor.service.js
│       │
│       ├── lessons/
│       │   ├── lessons.routes.js
│       │   └── lessons.service.js
│       │
│       ├── enrollments/
│       │   ├── enrollments.routes.js
│       │   └── enrollments.service.js
│       │
│       ├── certificates/
│       │   ├── certificates.routes.js
│       │   └── certificates.service.js
│       │
│       ├── admin/
│       │   ├── admin.routes.js
│       │   └── admin.service.js
│       │
│       └── webhooks/
│           └── stripe.webhook.js
│
├── logs/                       ← PM2 log output (gitignored)
├── docs/
│   ├── architecture.png        ← Chapter 16
│   └── erd.png                 ← Chapter 16
│
└── learning-log/
    ├── 00-template.md
    ├── 02-why-postgresql-and-prisma.md
    ├── 03-project-setup-and-config.md
    ├── 04-authentication.md
    ├── 05-authorization.md
    ├── 06-course-catalogue.md
    ├── 07-instructor-course-management.md
    ├── 08-enrollment-and-payments.md
    ├── 09-lesson-access-and-video.md
    ├── 10-progress-and-certificates.md
    ├── 11-search.md
    ├── 12-caching-and-performance.md
    ├── 13-notifications.md
    ├── 14-rate-limiting.md
    ├── 15-admin-dashboard.md
    └── 16-deploy.md
```

---

## `.gitignore`

Create this as the **first** file in the project:

```gitignore
# Environment
.env
.env.*
!.env.example

# Dependencies
node_modules/

# Logs
logs/
*.log

# Build output
dist/

# OS junk
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
*.swo

# Prisma (keep migrations, not the generated client in some setups)
# prisma/migrations/ — COMMIT these, they are part of the project
```

---

## `package.json` — the scripts you need

```json
{
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js",
    "lint": "eslint src/",
    "format": "prettier --write src/",
    "db:migrate": "prisma migrate dev",
    "db:migrate:prod": "prisma migrate deploy",
    "db:seed": "node prisma/seed.js",
    "db:studio": "prisma studio",
    "db:reset": "prisma migrate reset"
  },
  "prisma": {
    "seed": "node prisma/seed.js"
  }
}
```

---

## `src/app.js` — the complete setup

```js
// src/app.js — wire order matters
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';

import { config } from './config/index.js';
import { errorHandler } from './middleware/errorHandler.js';

// Routers (import as you build each module)
import { authRouter } from './modules/auth/auth.routes.js';
import { coursesRouter } from './modules/courses/courses.routes.js';
import { instructorRouter } from './modules/courses/instructor.routes.js';
import { lessonsRouter } from './modules/lessons/lessons.routes.js';
import { enrollmentsRouter } from './modules/enrollments/enrollments.routes.js';
import { certificatesRouter } from './modules/certificates/certificates.routes.js';
import { adminRouter } from './modules/admin/admin.routes.js';
import { stripeWebhookRouter } from './modules/webhooks/stripe.webhook.js';

export const app = express();

// ── Security headers ───────────────────────────────────────────────────────
app.use(helmet());

// ── CORS ───────────────────────────────────────────────────────────────────
app.use(cors({
  origin: config.frontendUrl,
  credentials: true,
}));

// ── Stripe webhook MUST use raw body — mount before express.json() ─────────
app.use('/api/webhooks', stripeWebhookRouter);

// ── Body parsing ───────────────────────────────────────────────────────────
app.use(express.json());

// ── HTTP request logging ───────────────────────────────────────────────────
app.use(morgan(config.nodeEnv === 'production' ? 'combined' : 'dev'));

// ── Health check ───────────────────────────────────────────────────────────
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// ── Route mounting ─────────────────────────────────────────────────────────
app.use('/api/auth', authRouter);
app.use('/api/courses', coursesRouter);
app.use('/api/instructor', instructorRouter);
app.use('/api/lessons', lessonsRouter);
app.use('/api/enrollments', enrollmentsRouter);
app.use('/api/certificates', certificatesRouter);
app.use('/api/admin', adminRouter);

// ── 404 handler (must be after all routes) ─────────────────────────────────
app.use((req, res) => {
  res.status(404).json({ error: `Route ${req.method} ${req.path} not found.` });
});

// ── Global error handler (must be last, 4-arg signature required) ──────────
app.use(errorHandler);
```

---

## `src/server.js` — the entry point

```js
// src/server.js
import { config } from './config/index.js';  // ← validates env at boot
import { app } from './app.js';
import { prisma } from './lib/prisma.js';

async function main() {
  // Verify database connection before accepting traffic
  await prisma.$connect();
  console.log('✅  Database connected');

  app.listen(config.port, () => {
    console.log(`🚀  EduFlow API running on port ${config.port}`);
    console.log(`    Environment : ${config.nodeEnv}`);
    console.log(`    Started at  : ${new Date().toISOString()}`);
  });
}

main().catch((err) => {
  console.error('❌  Failed to start server:', err);
  process.exit(1);
});
```
