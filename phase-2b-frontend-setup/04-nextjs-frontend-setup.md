# Next.js Frontend Setup

Now that the backend API is standing, you need a frontend to consume it. This chapter sets up a Next.js application that will serve as the user interface for EduFlow — course browsing, enrollment, lesson viewing, and progress tracking.

---

## Why Next.js

**Next.js** is a React framework that provides server-side rendering, static site generation, and API routes out of the box. For EduFlow, this matters because:

1. **SEO-friendly course pages** — Course catalogue pages need to be crawlable by search engines. Server-side rendering ensures Google can index your content without executing JavaScript.
2. **Fast initial page loads** — The first paint happens faster when HTML is rendered on the server, not client-side.
3. **API routes for BFF pattern** — Next.js API routes act as a Backend-for-Frontend, letting you aggregate multiple backend calls into a single frontend request and handle CORS centrally.
4. **Built-in routing and optimisation** — File-based routing, automatic code splitting, and image optimisation are handled for you.

The alternative is a plain React SPA with client-side routing. That works, but you lose SEO benefits and first-paint performance. For a public-facing course platform, those matter.

> **Worth reading:** Search "Next.js vs React SPA" — the Next.js documentation explains the performance and SEO trade-offs clearly.

---

## Scaffold the Next.js app

Open your terminal in the directory where EduFlow will live (the same parent directory as your backend).

**Step 1 — Create the Next.js app**

```
npx create-next-app@latest frontend --typescript --tailwind --eslint
```

When prompted:

- Would you like to use App Router? → **Yes** (App Router is the modern Next.js architecture)
- Would you like to customise the default import alias? → **No**

This creates a `frontend/` directory with a complete Next.js setup including TypeScript, Tailwind CSS, and ESLint.

**Step 2 — Install additional dependencies**

```
cd frontend
npm install axios @tanstack/react-query zustand
npm install --save-dev @types/node
```

What each one does:

- `axios` — HTTP client for making API requests to your Express backend
- `@tanstack/react-query` — Data fetching and caching library (handles loading states, retries, cache invalidation)
- `zustand` — Lightweight state management for global app state (auth, notifications)
- `@types/node` — TypeScript definitions for Node.js (needed for environment variables)

---

## Project structure

Your Next.js app uses the **App Router** (the `app/` directory). Here is the layout you will build:

```
frontend/
├── app/
│   ├── layout.tsx              ← root layout (navbar, global styles)
│   ├── page.tsx                ← homepage (course catalogue)
│   ├── courses/
│   │   ├── page.tsx            ← course listing page
│   │   └── [id]/
│   │       └── page.tsx        ← individual course detail page
│   ├── dashboard/
│   │   ├── page.tsx            ← student dashboard
│   │   └── instructor/
│   │       └── page.tsx        ← instructor dashboard
│   ├── login/
│   │   └── page.tsx
│   └── register/
│       └── page.tsx
├── components/
│   ├── ui/                     ← reusable UI components (buttons, cards)
│   ├── CourseCard.tsx
│   ├── LessonPlayer.tsx
│   └── Navbar.tsx
├── lib/
│   ├── api.ts                  ← axios instance with base URL and interceptors
│   └── auth.ts                 ← auth utilities (token storage, refresh logic)
├── stores/
│   └── authStore.ts            ← Zustand store for auth state
├── types/
│   └── index.ts                ← TypeScript type definitions
└── .env.local                  ← environment variables (never committed)
```

Create this structure now:

```
mkdir -p app/courses/\[id\] app/dashboard/instructor app/login app/register
mkdir -p components/ui lib stores types
touch lib/api.ts lib/auth.ts stores/authStore.ts types/index.ts
```

---

## Environment configuration

Create `.env.local` in the `frontend/` directory. **Never commit this file** — add it to `.gitignore` immediately:

```
echo ".env.local" >> .gitignore
```

Then create the file:

```
# .env.local — frontend environment variables
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_FRONTEND_URL=http://localhost:3001
```

---

## Configure the API client

Create `lib/api.ts`. This axios instance handles authentication headers and token refresh automatically:

```ts
// lib/api.ts — shape only, not the implementation
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: { 'Content-Type': 'application/json' },
});

// Request interceptor: attach the access token to every request
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor: attempt token refresh on 401, redirect to login on failure
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // attempt refresh, then retry — redirect to /login if refresh fails
    }
    return Promise.reject(error);
  }
);

export default api;
```

---

## TypeScript types

Create `types/index.ts`. These types mirror the shape of your Mongoose models' documents as they come back from the API:

```ts
// types/index.ts
export interface User {
  id: string;
  email: string;
  role: 'student' | 'instructor' | 'admin';
  createdAt: string;
}

export interface Course {
  id: string;
  instructorId: string;
  title: string;
  slug: string;
  description: string;
  coverImageUrl: string;
  price: number;
  category: string;
  status: 'draft' | 'pending' | 'published' | 'rejected';
  createdAt: string;
  updatedAt: string;
}

export interface Lesson {
  id: string;
  courseId: string;
  title: string;
  videoUrl: string;
  durationSeconds: number;
  position: number;
}

export interface Enrollment {
  id: string;
  studentId: string;
  courseId: string;
  amountPaid: number;
  enrolledAt: string;
}

export interface ProgressRecord {
  id: string;
  enrollmentId: string;
  lessonId: string;
  completedAt: string;
}
```

Notice the field names use camelCase to match the Mongoose model definitions you will build in Chapter 3.05.

---

## Auth store with Zustand

Create `stores/authStore.ts`:

```ts
// stores/authStore.ts
import { create } from 'zustand';
import { User } from '@/types';

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  setUser: (user: User | null) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  setUser: (user) => set({ user, isAuthenticated: !!user }),
  logout: () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    set({ user: null, isAuthenticated: false });
  },
}));
```

---

## Run the development server

Start the Next.js dev server:

```
npm run dev -- -p 3001
```

The frontend runs on `http://localhost:3001`, keeping it separate from your backend on port 3000.

Update `.env.local` if you use a different port:

```
NEXT_PUBLIC_FRONTEND_URL=http://localhost:3001
```

---

## Definition of Done

- [ ] `npm run dev -- -p 3001` starts the Next.js server without errors
- [ ] The app loads at `http://localhost:3001`
- [ ] `.env.local` exists and is in `.gitignore`
- [ ] The folder structure matches the layout described above
- [ ] `lib/api.ts` is configured with axios interceptors for auth headers and token refresh
- [ ] TypeScript types in `types/index.ts` match the Mongoose model shapes
- [ ] The auth store is created in `stores/authStore.ts`

All boxes ticked? Then continue to Chapter 5 — Authentication. If not, that is where today's work is.
