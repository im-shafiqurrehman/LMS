# Project Setup and Config

Every real project starts with a week of decisions that seem administrative but are actually foundational. The folder structure you pick at the start shapes how easy every future feature is to add. The config pattern you choose determines whether a leaked `.env` file is a catastrophe or a recoverable mistake. The linting setup determines whether the codebase stays readable after six weeks of momentum.

This chapter gets EduFlow standing. By the end, the server runs, connects to the database, reads config safely from the environment, and rejects bad config at startup — before serving a single request.

---

## How most projects fail before they start

The pattern goes like this: someone initialises a Node project, writes a `server.js` that imports `dotenv` and reads `process.env.DATABASE_URL` directly in the route file, and pushes to GitHub with a real database password in the commit history. Three weeks later, the project has grown enough that nobody can find where config is read; secrets are scattered across ten files; the `.env` file was accidentally committed in week two and the GitHub secret scanner is sending alerts.

None of this is hard to prevent. You prevent it by making two decisions at the start:

1. **All config is read from the environment, in one place, at startup.** If any required variable is missing or malformed, the server refuses to start with a clear error. This is "fail fast" — better to crash at boot with a readable message than to crash at 3am during a request because `JWT_SECRET` is undefined.
2. **`.env` never enters Git.** Not once, not "just temporarily." The `.gitignore` is the first file you create.

✅ Create `.gitignore` before any other file, with `.env` and `.env.*` in it.
❌ Don't add `.env` to Git and plan to remove it "later." Later never comes, and `git log` is permanent.

---

## Folder structure

You will use a **module-based structure**, not a layer-based one. Here's the difference, and why it matters.

**Layer-based** puts all routes together, all services together, all models together:

```
src/
  routes/
    auth.routes.js
    course.routes.js
  services/
    auth.service.js
    course.service.js
  models/
    user.model.js
    course.model.js
```

This looks organised. The problem is that when you work on authentication, you're jumping between three folders constantly — routes, services, models — because they're separated by type, not by feature. At ten features, it becomes a navigation puzzle.

**Module-based** groups everything for a feature together:

```
src/
  modules/
    auth/
      auth.routes.js
      auth.service.js
      auth.validators.js
    courses/
      courses.routes.js
      courses.service.js
      courses.validators.js
  lib/
    mongodb.js     ← the MongoDB client singleton
    redis.js       ← the Redis client
    email.js       ← the Resend email client
  middleware/
    authenticate.js
    authorize.js
    errorHandler.js
  config/
    index.js       ← all env config, validated at boot
  app.js
  server.js
```

This is the structure you will build. Every feature is self-contained in its folder. When you add notifications, you add a `notifications/` module — you don't scatter files across three directories.

Create this structure now — empty folders and placeholder files — before writing logic. The structure is the commitment.

---

## Scaffold the project

Open your terminal in the directory where EduFlow will live.

**Step 1 — Initialise the project**

```bash
mkdir eduflow && cd eduflow
npm init -y
git init
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore
echo "node_modules" >> .gitignore
```

✅ Verify: `git status` shows `.gitignore` as a new file and nothing else.
❌ Don't run `npm install` before `.gitignore` exists — `node_modules` can accidentally be staged.

**Step 2 — Install core dependencies**

```bash
npm install express cors helmet morgan dotenv zod
npm install --save-dev nodemon eslint prettier
```

What each one does, briefly:
- `express` — the HTTP framework
- `cors` — allows the frontend (running on a different port) to make requests to the API
- `helmet` — sets sensible security headers on every response with one line
- `morgan` — logs every incoming request so you can see what the server is doing
- `dotenv` — loads `.env` into `process.env` at startup
- `zod` — validates and parses the environment config at boot (you'll use it for request validation too)
- `nodemon` — restarts the server when files change, so you don't restart manually

**Step 3 — Install MongoDB driver**

```bash
npm install mongodb
```

This installs the official MongoDB Node.js driver. You'll use it directly to connect to MongoDB and perform queries. No schema file needed — MongoDB is schemaless at the database level.

**Step 4 — Create the folder structure**

```bash
mkdir -p src/modules/auth src/modules/courses src/modules/enrollments src/modules/qa
mkdir -p src/lib src/middleware src/config
touch src/app.js src/server.js
touch src/config/index.js
touch src/lib/mongodb.js
touch src/middleware/authenticate.js src/middleware/authorize.js src/middleware/errorHandler.js
```

Verify the structure looks right: `find src -type f` should list every file you just created.

---

## Config validation at startup

Create `src/config/index.js`. This file does one thing: it reads every environment variable the application needs, validates each one, and exports a typed config object. If anything is missing or wrong, it throws before the server starts.

The shape it must produce:

```js
// src/config/index.js — shape only, not the implementation
{
  port: 3000,                // number, default 3000
  nodeEnv: "development",   // "development" | "production" | "test"
  databaseUrl: "...",        // required string
  jwtSecret: "...",          // required string, min 32 chars
  jwtRefreshSecret: "...",   // required string, min 32 chars
  jwtAccessExpiry: "15m",    // default "15m"
  jwtRefreshExpiry: "7d",    // default "7d"
  cloudinaryCloudName: "...", // required string
  cloudinaryApiKey: "...",    // required string
  cloudinaryApiSecret: "...", // required string
  redisUrl: "...",            // required string
  resendApiKey: "...",        // required string
  stripeSecretKey: "...",     // required string
  stripeWebhookSecret: "...", // required string
  frontendUrl: "...",         // required string (for CORS)
}
```

Use **Zod** to define and parse this schema. When validation fails, Zod gives you a clear error message listing every missing or malformed variable — far better than `Cannot read property of undefined` deep in a route handler.

> **Worth reading:** Search "Zod environment validation Node.js" — there are several short guides showing the pattern. The T3 stack's `env.js` pattern is a good reference.

✅ The server should print each missing variable's name when startup fails, not just a generic error.
❌ Don't use `process.env.JWT_SECRET` directly in a route file. All env access goes through `src/config/index.js`.

**Hint:** Zod's `.safeParse()` returns a result object with a `success` boolean and an `error` with a `.format()` method — that formatted error is what you print to the console before calling `process.exit(1)`.

---

## The MongoDB client singleton

Create `src/lib/mongodb.js`. This file exports a single MongoDB client instance. In development, Node's hot-reload can create multiple instances of the client if you're not careful — each one holds its own connection pool, and you'll hit connection limits fast.

The pattern to implement:

```js
// src/lib/mongodb.js — shape only
// In development: attach the client to `global` so hot-reload reuses it
// In production: just create it once (the module system handles singleton behaviour)
// Export it as `client` and a helper function `getDb()` that returns the database
```

Every module that needs the database imports from this file: `import { getDb } from '../lib/mongodb.js'`. Never create a new `MongoClient` anywhere else.

---

## Wire the app

Create `src/app.js` and `src/server.js`. The split is deliberate: `app.js` builds and configures the Express application (middleware, routes), and `server.js` starts it listening. This separation makes the app easier to test — tests can import `app.js` without starting a server.

`src/app.js` must:
- Import and apply `helmet()`, `cors()`, `morgan()`, `express.json()`
- Apply your error-handling middleware last (after routes)
- Export the app

`src/server.js` must:
- Import the config (triggering validation at boot)
- Import the app
- Connect to the database (a quick MongoDB `connect()` to verify the URL is reachable)
- Start listening on `config.port`
- Log a clear startup message: the port, the environment, and the timestamp

---

## Your `.env` file

Create `.env` in the project root (not in `src/`). It is never committed.

```
# .env — local development values only
NODE_ENV=development
PORT=3000

DATABASE_URL=mongodb://admin:yourpassword@localhost:27017/eduflow?authSource=admin

JWT_SECRET=replace-me-with-a-32-char-minimum-secret-string
JWT_REFRESH_SECRET=replace-me-with-a-different-32-char-secret
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

REDIS_URL=redis://localhost:6379

RESEND_API_KEY=

STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=

FRONTEND_URL=http://localhost:3000
```

Leave Cloudinary, Resend, and Stripe blank for now — the server startup validation will fail for those. Temporarily remove them from the required list in `config/index.js` and add a `// TODO: required before chapter N` comment. Add them back when that chapter arrives.

✅ The only `.env` values you fill in now: `DATABASE_URL` (MongoDB URI), `JWT_SECRET`, `JWT_REFRESH_SECRET`, `REDIS_URL`.

---

## Run the database locally with Docker

You need a local MongoDB instance and a local Redis instance. Docker keeps them isolated from your machine.

Create `docker-compose.yml` in the project root:

```yaml
services:
  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: yourpassword
      MONGO_INITDB_DATABASE: eduflow
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  mongodb_data:
```

Start them:

```bash
docker compose up -d
```

Verify MongoDB is reachable:

```bash
docker compose exec mongodb mongosh -u admin -p yourpassword --eval "db.adminCommand('ping')"
```

If you see `{ ok: 1 }`, the database is up.

> **Worth reading:** Search "Docker Compose for local development Node.js" — search results will give you several guides on why volumes matter (your data survives container restarts) and why pinning image versions matters (reproducibility).

---

## Verify the database connection

Since MongoDB is schemaless, there's no migration step. You'll verify the connection works in `src/server.js` when you call `client.connect()`. If the connection string is correct, the server starts. If not, it fails with a clear error.

You'll create collections as you need them throughout the course — MongoDB creates collections automatically when you first insert a document.

---

## Definition of Done

- [ ] `npm run dev` starts the server without errors; the console prints port, environment, and timestamp
- [ ] The server refuses to start if `DATABASE_URL` is missing or malformed — the error names the missing variable
- [ ] No `.env` file or real secret appears in `git log` or `git status`
- [ ] `docker compose up -d` starts MongoDB and Redis; the MongoDB connection succeeds
- [ ] `GET /health` returns `{ "status": "ok" }` (add this as a sanity-check route in `app.js`)
- [ ] The folder structure matches the layout described above

All boxes ticked? Then continue to Chapter 4 — Authentication. If not, that's where today's work is.
