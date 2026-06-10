# Next.js Frontend Setup

The backend is running. Now you build the interface — the screens where students browse, enroll, and learn. You use **Next.js**, a React framework that gives you server-side rendering, file-based routing, and API routes on the same codebase. Combined with **Tailwind CSS** for styling, you'll ship a professional frontend without fussing over CSS frameworks.

This chapter creates the frontend repository, sets up the project structure, configures API communication to your backend, and verifies the first page loads.

---

## Why Next.js

**Next.js** is a framework built on top of React. Without it, you'd need to: wire up your own routing, manage server-side rendering manually, bundle and minify your code, and handle environment variables. Next.js does all of that for you, so you write application code, not infrastructure.

For EduFlow, Next.js gives you:

1. **File-based routing.** Create a file at `app/courses/page.tsx` and it automatically becomes a route at `/courses`. No route configuration needed.
2. **Server Components by default.** Your React components can run on the server, keeping API calls and secrets out of client code — then they return pre-rendered HTML to the browser.
3. **API routes.** You can write backend logic (`app/api/...`) without a separate server, but in EduFlow you'll use the Node API you already built — Next.js routes just proxy to it.
4. **Built-in CSS support.** Tailwind CSS integrates seamlessly. You import styles and write utility classes; Next.js bundles them and ships only the CSS actually used.
5. **Type safety with TypeScript.** Write `.tsx` files; TypeScript catches errors at build time, not runtime.

---

## Create the frontend project

**Step 1 — Initialize the Next.js project**

```bash
cd /path/to/projects
npx create-next-app@latest eduflow-web --typescript --tailwind --eslint

# When prompted:
# ✔ Would you like to use ESLint? Yes
# ✔ Would you like to use Tailwind CSS? Yes
# ✔ Would you like your code inside a `src/` directory? Yes
# ✔ Would you like to use App Router? Yes (not Pages Router)
# ✔ Would you like to use Turbopack? No (keep it stable)
# ✔ Would you like to customize the import alias? Yes
# Change from @/* to @/* (keep the default)
```

This creates a new Next.js project with TypeScript, Tailwind CSS, and ESLint already configured. The "App Router" is the modern way to structure Next.js projects (file-based routing).

**Step 2 — Create the folder structure**

```bash
cd eduflow-web
mkdir -p src/components
mkdir -p src/pages
mkdir -p src/lib
mkdir -p src/hooks
mkdir -p src/types
mkdir -p src/styles
touch src/lib/api-client.ts
touch src/lib/auth.ts
touch src/types/index.ts
```

The structure:
- `src/app/` — routes and pages (created by `create-next-app`)
- `src/components/` — reusable React components
- `src/lib/` — utility functions (API calls, auth helpers)
- `src/hooks/` — custom React hooks
- `src/types/` — TypeScript types and interfaces
- `src/styles/` — global styles (if needed beyond Tailwind)

---

## Environment and configuration

Create `.env.local` in the project root (the local environment file, never committed):

```
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:3000/api
NEXT_PUBLIC_APP_NAME=EduFlow
```

The `NEXT_PUBLIC_` prefix makes these variables available to the browser. Anything without this prefix is only available on the server.

Update `.gitignore` to add:

```
.env
.env.local
.env.*.local
.next/
```

---

## API client configuration

Create `src/lib/api-client.ts`. This file handles all communication with the backend API. Every fetch call goes through this client so you have one place to manage errors, auth tokens, and request formatting.

Shape:

```ts
// src/lib/api-client.ts

const API_URL = process.env.NEXT_PUBLIC_API_URL;

export const apiClient = {
  async get<T>(endpoint: string): Promise<T> {
    // Fetch with token from localStorage, handle 401, return JSON
  },

  async post<T>(endpoint: string, body: any): Promise<T> {
    // POST with token, body as JSON
  },

  async put<T>(endpoint: string, body: any): Promise<T> {
    // PUT with token, body as JSON
  },

  async delete<T>(endpoint: string): Promise<T> {
    // DELETE with token
  },
};
```

Every method should:
- Add `Authorization: Bearer <token>` header if the token exists in localStorage
- Handle `401 Unauthorized` by clearing localStorage and redirecting to `/login`
- Throw descriptive errors instead of silent failures

---

## Auth context and hooks

Create `src/lib/auth.ts`. This manages authentication state — checking if the user is logged in, storing/clearing tokens, and triggering login redirects.

Shape:

```ts
// src/lib/auth.ts

export function getAuthToken(): string | null {
  // Read token from localStorage; return null if missing
}

export function setAuthToken(token: string): void {
  // Save token to localStorage
}

export function clearAuthToken(): void {
  // Remove token from localStorage
}

export function isAuthenticated(): boolean {
  // Return true if token exists and hasn't expired
}
```

Later chapters will add a React Context to share auth state across components.

---

## Tailwind CSS configuration

Tailwind is already configured by `create-next-app`. The config lives in `tailwind.config.ts`. You don't need to touch it yet — Tailwind's defaults are production-ready. Later, you'll add custom colors or typography if the design demands it.

Verify Tailwind works by checking `src/app/globals.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

These three directives inject Tailwind's base styles, component classes (like `.btn`), and utilities (like `.flex`, `.p-4`) into your CSS.

---

## The first page

Replace the default homepage with a real one. Update `src/app/page.tsx`:

```tsx
export default function Home() {
  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
      <div className="mx-auto max-w-4xl px-6 py-20 text-center">
        <h1 className="text-5xl font-bold text-gray-900 mb-4">
          Welcome to EduFlow
        </h1>
        <p className="text-xl text-gray-600 mb-8">
          Learn from industry experts. Move at your own pace.
        </p>
        <a
          href="/courses"
          className="inline-block bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-3 px-8 rounded-lg transition"
        >
          Browse Courses
        </a>
      </div>
    </main>
  );
}
```

This uses Tailwind utility classes: `min-h-screen` for full height, `bg-gradient-to-br` for a gradient background, `px-6 py-20` for padding, etc. Every style is written as a class name, not in a separate CSS file.

---

## Run the frontend

```bash
npm run dev
```

This starts the Next.js development server on `http://localhost:3000` (or the next available port).

**Note:** Your backend API also runs on `3000`. In development, run them on different ports:
- Backend: `PORT=3000` (default)
- Frontend: `PORT=3001`

Update `.env.local`:

```
NEXT_PUBLIC_API_URL=http://localhost:3000/api
```

And start the frontend:

```bash
npm run dev -- -p 3001
```

Now both are running: backend on `3000`, frontend on `3001`. They can talk to each other via the `NEXT_PUBLIC_API_URL`.

---

## Folder structure checklist

Verify your folder structure looks like this:

```
eduflow-web/
  src/
    app/
      page.tsx          ← the homepage
      layout.tsx        ← root layout
      globals.css       ← Tailwind directives
    components/         ← empty for now
    lib/
      api-client.ts     ← API communication
      auth.ts           ← auth helpers
    hooks/              ← empty for now
    types/
      index.ts          ← shared TypeScript types
  .env.local            ← never commit, never push
  package.json
  tsconfig.json
  tailwind.config.ts
  next.config.js
```

---

## Definition of Done

- [ ] `npm run dev` starts the frontend without errors on `http://localhost:3001` (or available port)
- [ ] The homepage loads with the title "Welcome to EduFlow"
- [ ] `.env.local` is in `.gitignore` and contains `NEXT_PUBLIC_API_URL`
- [ ] `src/lib/api-client.ts` exists and exports the `apiClient` object
- [ ] `src/lib/auth.ts` exists with token management functions
- [ ] TypeScript finds no errors: `npm run build` succeeds

Write your learnings in `learning-log/04-frontend-setup.md` and commit before proceeding.
