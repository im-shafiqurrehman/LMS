# Next.js Frontend Setup

Now that the backend API is standing, you need a frontend to consume it. This chapter sets up a Next.js application that will serve as the user interface for EduFlow — course browsing, enrollment, lesson viewing, and progress tracking.

---

## Why Next.js

**Next.js** is a React framework that provides server-side rendering, static site generation, and API routes out of the box. For EduFlow, this matters because:

1. **SEO-friendly course pages** — Course catalogue pages need to be crawlable by search engines. Server-side rendering ensures Google can index your content without executing JavaScript.
2. **Fast initial page loads** — The first paint happens faster when HTML is rendered on the server, not client-side.
3. **API routes for BFF pattern** — Next.js API routes act as a Backend-for-Frontend, letting you aggregate multiple backend calls into a single frontend request and handle CORS centrally.
4. **Built-in routing and optimization** — File-based routing, automatic code splitting, and image optimization are handled for you.

The alternative is a plain React SPA with client-side routing. That works, but you lose SEO benefits and first-paint performance. For a public-facing course platform, those matter.

> **Worth reading:** Search "Next.js vs React SPA" — the Next.js documentation explains the performance and SEO trade-offs clearly.

---

## Scaffold the Next.js app

Open your terminal in the directory where EduFlow will live (the same parent directory as your backend).

**Step 1 — Create the Next.js app**

```bash
npx create-next-app@latest frontend --typescript --tailwind --eslint
```

When prompted:
- Would you like to use App Router? → **Yes** (App Router is the modern Next.js architecture)
- Would you like to customize the default import alias? → **No**

This creates a `frontend/` directory with a complete Next.js setup including TypeScript, Tailwind CSS, and ESLint.

**Step 2 — Install additional dependencies**

```bash
cd frontend
npm install axios react-query zustand
npm install --save-dev @types/node
```

What each one does:
- `axios` — HTTP client for making API requests to your backend
- `react-query` — Data fetching and caching library (handles loading states, retries, cache invalidation)
- `zustand` — Lightweight state management for global app state (auth, cart, etc.)
- `@types/node` — TypeScript definitions for Node.js (needed for environment variables)

---

## Project structure

Your Next.js app uses the **App Router** (the `app/` directory structure). Here's the layout you'll build:

```
frontend/
├── app/
│   ├── layout.tsx           ← root layout (navbar, global styles)
│   ├── page.tsx             ← homepage (course catalogue)
│   ├── courses/
│   │   ├── page.tsx         ← course listing page
│   │   └── [id]/
│   │       └── page.tsx     ← individual course detail page
│   ├── dashboard/
│   │   ├── page.tsx         ← student dashboard (enrolled courses, progress)
│   │   └── instructor/
│   │       └── page.tsx     ← instructor dashboard (course management)
│   ├── login/
│   │   └── page.tsx         ← login page
│   └── register/
│       └── page.tsx         ← registration page
├── components/
│   ├── ui/                  ← reusable UI components (buttons, cards, modals)
│   ├── CourseCard.tsx       ← course display card
│   ├── LessonPlayer.tsx     ← video lesson player
│   └── Navbar.tsx           ← navigation bar
├── lib/
│   ├── api.ts               ← axios instance with base URL and interceptors
│   └── auth.ts              ← auth utilities (token storage, refresh logic)
├── stores/
│   └── authStore.ts         ← Zustand store for auth state
├── types/
│   └── index.ts             ← TypeScript type definitions
└── .env.local               ← environment variables (never committed)
```

Create this structure now:

```bash
mkdir -p app/courses/[id] app/dashboard/instructor app/login app/register
mkdir -p components/ui lib stores types
touch lib/api.ts lib/auth.ts stores/authStore.ts types/index.ts
```

---

## Environment configuration

Create `.env.local` in the `frontend/` directory:

```env
# .env.local — frontend environment variables
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_FRONTEND_URL=http://localhost:3001
```

**Never commit `.env.local`**. Add it to `.gitignore` immediately:

```bash
echo ".env.local" >> .gitignore
```

---

## Configure the API client

Create `lib/api.ts`:

```typescript
// lib/api.ts
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor: add auth token to every request
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor: handle 401 errors (token expired)
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token expired - trigger refresh logic
      const refreshToken = localStorage.getItem('refresh_token');
      if (refreshToken) {
        try {
          const response = await axios.post(`${process.env.NEXT_PUBLIC_API_URL}/api/auth/refresh`, {
            refresh_token: refreshToken,
          });
          const { access_token } = response.data;
          localStorage.setItem('access_token', access_token);
          // Retry the original request
          error.config.headers.Authorization = `Bearer ${access_token}`;
          return api.request(error.config);
        } catch (refreshError) {
          // Refresh failed - redirect to login
          localStorage.removeItem('access_token');
          localStorage.removeItem('refresh_token');
          window.location.href = '/login';
        }
      } else {
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  }
);

export default api;
```

This axios instance:
- Sets the base URL from environment variables
- Automatically adds the access token to every request
- Handles token refresh on 401 errors
- Redirects to login when refresh fails

---

## TypeScript types

Create `types/index.ts`:

```typescript
// types/index.ts
export interface User {
  id: string;
  email: string;
  role: 'student' | 'instructor' | 'admin';
  created_at: string;
}

export interface Course {
  id: string;
  instructor_id: string;
  title: string;
  slug: string;
  description: string;
  cover_image_url: string;
  price: number;
  category: string;
  status: 'draft' | 'pending' | 'published' | 'rejected';
  created_at: string;
  updated_at: string;
}

export interface Lesson {
  id: string;
  course_id: string;
  title: string;
  video_url: string;
  duration_seconds: number;
  position: number;
}

export interface Enrollment {
  id: string;
  student_id: string;
  course_id: string;
  amount_paid: number;
  enrolled_at: string;
}

export interface ProgressRecord {
  id: string;
  enrollment_id: string;
  lesson_id: string;
  completed_at: string;
}
```

These types mirror your backend Prisma schema, ensuring type safety across the full stack.

---

## Auth store with Zustand

Create `stores/authStore.ts`:

```typescript
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

This lightweight store manages auth state globally and can be accessed from any component.

---

## Update the root layout

Edit `app/layout.tsx`:

```typescript
// app/layout.tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'EduFlow - Learn Anything, Anywhere',
  description: 'A modern learning management system',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

---

## Run the development server

Start the Next.js dev server:

```bash
npm run dev
```

The frontend will run on `http://localhost:3000` by default. If your backend is also on port 3000, change the frontend port:

```bash
npm run dev -- -p 3001
```

Then update `.env.local`:
```env
NEXT_PUBLIC_FRONTEND_URL=http://localhost:3001
```

---

## Definition of Done

- [ ] `npm run dev` starts the Next.js server without errors
- [ ] The app loads at `http://localhost:3001` (or your chosen port)
- [ ] `.env.local` exists and is in `.gitignore`
- [ ] The folder structure matches the layout described above
- [ ] `lib/api.ts` is configured with axios interceptors
- [ ] TypeScript types in `types/index.ts` match your backend schema
- [ ] The auth store is created and functional

All boxes ticked? Then continue to the next chapter. If not, that's where today's work is.
