# Next.js Frontend Setup

Now that the backend API is standing, you need a frontend to consume it. This chapter sets up a Next.js application and wires in **Redux Toolkit** as the single state management solution for EduFlow — no third-party data-fetching libraries, no extra abstractions. Just Next.js and Redux, kept simple.

---

## Why Next.js

**Next.js** is a React framework that provides server-side rendering, static site generation, and file-based routing out of the box. For EduFlow, this matters because:

1. **SEO-friendly course pages** — Course catalogue pages need to be crawlable by search engines. Server-side rendering ensures Google can index your content without executing JavaScript.
2. **Fast initial page loads** — The first paint happens faster when HTML is rendered on the server, not client-side.
3. **File-based routing** — Pages live in the `app/` directory; no router configuration needed.
4. **Built-in image and font optimisation** — Handled automatically.

> **Worth reading:** Search "Next.js App Router vs Pages Router" — the App Router (the `app/` directory, introduced in Next.js 13) is the current standard. This course uses App Router throughout.

---

## Why Redux for state management

Global state in EduFlow is limited but important: who is logged in, what their role is, and whether they have a valid token. This state is needed in the navbar, in protected route guards, and in API call headers. It must survive a page navigation.

**Redux Toolkit** is the official, batteries-included way to use Redux. It removes the boilerplate that made classic Redux tedious (no more action type constants, no manual reducers), while keeping Redux's core strength: **a single, predictable store** that every component can read from.

The alternatives:

- **Zustand** — lightweight, but adds a dependency for something Redux handles cleanly.
- **React Context + useReducer** — built in to React, but re-renders the entire subtree on every state change and has no dev-tools story.
- **React Query / TanStack Query** — excellent for server-state caching, but that is not the problem here. EduFlow fetches data directly with `axios` and stores what matters in Redux.

For EduFlow: **Redux Toolkit + React Redux** only. No other state library.

---

## Scaffold the Next.js app

Open your terminal in the EduFlow root directory.

**Step 1 — Create the Next.js app**

```
npx create-next-app@latest frontend --typescript --tailwind --eslint
```

When prompted:

- Would you like to use the App Router? → **Yes**
- Would you like to customise the default import alias? → **No**

This creates `frontend/` with TypeScript, Tailwind CSS, and ESLint configured.

**Step 2 — Install dependencies**

```
cd frontend
npm install axios @reduxjs/toolkit react-redux
```

What each one does:

- `axios` — HTTP client for API requests. Configured with a base URL and an auth-header interceptor.
- `@reduxjs/toolkit` — Redux Toolkit. Provides `createSlice`, `configureStore`, and `createAsyncThunk`.
- `react-redux` — The React bindings that let components read from and dispatch to the store.

That is the complete dependency list. Nothing else is needed for state management or data fetching.

---

## Project structure

```
frontend/
├── app/
│   ├── layout.tsx              ← root layout — wraps children with Redux Provider
│   ├── page.tsx                ← homepage (course catalogue)
│   ├── courses/
│   │   ├── page.tsx            ← course listing
│   │   └── [id]/
│   │       └── page.tsx        ← course detail
│   ├── dashboard/
│   │   ├── page.tsx            ← student dashboard
│   │   └── instructor/
│   │       └── page.tsx        ← instructor dashboard
│   ├── login/
│   │   └── page.tsx
│   └── register/
│       └── page.tsx
├── components/
│   ├── Providers.tsx           ← Redux Provider wrapper (client component)
│   ├── Navbar.tsx
│   ├── CourseCard.tsx
│   └── LessonPlayer.tsx
├── lib/
│   └── api.ts                  ← axios instance — reads token from Redux store
├── store/
│   ├── store.ts                ← configureStore — wires all slices together
│   ├── authSlice.ts            ← auth state: user, accessToken, isAuthenticated
│   └── courseSlice.ts          ← course state: catalogue list, loading flag
├── types/
│   └── index.ts                ← TypeScript types matching Mongoose model shapes
└── .env.local
```

Create this structure now:

```
mkdir -p app/courses/\[id\] app/dashboard/instructor app/login app/register
mkdir -p components lib store types
touch lib/api.ts
touch store/store.ts store/authSlice.ts store/courseSlice.ts
touch components/Providers.tsx
touch types/index.ts
```

---

## Environment configuration

Create `.env.local` and add it to `.gitignore` immediately:

```
echo ".env.local" >> .gitignore
```

```
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:3000
```

---

## TypeScript types

Create `types/index.ts`. These types mirror your Mongoose model document shapes as they arrive from the API:

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
  totalLessons: number;
  totalDuration: number;
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

---

## Redux store

### `store/authSlice.ts`

Manages who is currently logged in:

```ts
// store/authSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { User } from '@/types';

interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
}

const initialState: AuthState = {
  user: null,
  accessToken: null,
  isAuthenticated: false,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    setCredentials(
      state,
      action: PayloadAction<{ user: User; accessToken: string }>
    ) {
      state.user            = action.payload.user;
      state.accessToken     = action.payload.accessToken;
      state.isAuthenticated = true;
    },
    logout(state) {
      state.user            = null;
      state.accessToken     = null;
      state.isAuthenticated = false;
    },
  },
});

export const { setCredentials, logout } = authSlice.actions;
export default authSlice.reducer;
```

`setCredentials` is dispatched after a successful login or token refresh. `logout` is dispatched when the user logs out or when a refresh attempt fails.

### `store/courseSlice.ts`

Manages the course catalogue list loaded from the API:

```ts
// store/courseSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { Course } from '@/types';

interface CourseState {
  catalogue: Course[];
  isLoading: boolean;
  error: string | null;
}

const initialState: CourseState = {
  catalogue: [],
  isLoading: false,
  error: null,
};

const courseSlice = createSlice({
  name: 'courses',
  initialState,
  reducers: {
    setLoading(state, action: PayloadAction<boolean>) {
      state.isLoading = action.payload;
    },
    setCatalogue(state, action: PayloadAction<Course[]>) {
      state.catalogue = action.payload;
      state.isLoading = false;
      state.error     = null;
    },
    setError(state, action: PayloadAction<string>) {
      state.error     = action.payload;
      state.isLoading = false;
    },
  },
});

export const { setLoading, setCatalogue, setError } = courseSlice.actions;
export default courseSlice.reducer;
```

### `store/store.ts`

Wires the slices into one store and exports the typed hooks:

```ts
// store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import authReducer   from './authSlice';
import courseReducer from './courseSlice';

export const store = configureStore({
  reducer: {
    auth:    authReducer,
    courses: courseReducer,
  },
});

export type RootState   = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Typed hooks — use these everywhere instead of the plain useDispatch / useSelector
export const useAppDispatch: () => AppDispatch        = useDispatch;
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

The typed `useAppDispatch` and `useAppSelector` hooks give you full TypeScript autocompletion when reading state or dispatching actions.

---

## Redux Provider

Next.js App Router renders `layout.tsx` on the server by default. The Redux `<Provider>` must be a **client component** — wrap it:

```tsx
// components/Providers.tsx
'use client';
import { Provider } from 'react-redux';
import { store } from '@/store/store';

export function Providers({ children }: { children: React.ReactNode }) {
  return <Provider store={store}>{children}</Provider>;
}
```

Then use it in the root layout:

```tsx
// app/layout.tsx
import { Providers } from '@/components/Providers';
import './globals.css';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

✅ Every page and client component in the app now has access to the Redux store.
❌ Do not import `store` directly in components to read state — always use `useAppSelector`. Direct imports bypass React's render cycle.

---

## API client

Create `lib/api.ts`. The axios instance reads the access token directly from the Redux store — no localStorage juggling:

```ts
// lib/api.ts
import axios from 'axios';
import { store } from '@/store/store';
import { logout } from '@/store/authSlice';

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: { 'Content-Type': 'application/json' },
});

// Attach the access token to every request
api.interceptors.request.use((config) => {
  const token = store.getState().auth.accessToken;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// On 401, dispatch logout so the UI reflects the session expiry
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      store.dispatch(logout());
    }
    return Promise.reject(error);
  }
);

export default api;
```

---

## How data fetching works (no React Query)

With Redux Toolkit and `axios`, data fetching is a two-step pattern:

1. **Dispatch `setLoading(true)`** before the request.
2. **Dispatch `setCatalogue(data)`** on success, or **`setError(message)`** on failure.

Example in a page component:

```tsx
// app/courses/page.tsx — shape only
'use client';
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '@/store/store';
import { setLoading, setCatalogue, setError } from '@/store/courseSlice';
import api from '@/lib/api';

export default function CoursesPage() {
  const dispatch  = useAppDispatch();
  const { catalogue, isLoading, error } = useAppSelector((s) => s.courses);

  useEffect(() => {
    dispatch(setLoading(true));
    api.get('/api/courses')
      .then((res) => dispatch(setCatalogue(res.data.data)))
      .catch((err) => dispatch(setError(err.message)));
  }, [dispatch]);

  if (isLoading) return <p>Loading...</p>;
  if (error)     return <p>Error: {error}</p>;

  return (
    <ul>
      {catalogue.map((course) => (
        <li key={course.id}>{course.title}</li>
      ))}
    </ul>
  );
}
```

This is intentionally simple. The dispatch pattern is the same everywhere: loading → fetch → success or error. No library wrappers, no hooks to memorise beyond `useAppSelector` and `useAppDispatch`.

---

## Run the development server

```
npm run dev -- -p 3001
```

The frontend runs on `http://localhost:3001`, separate from the backend on port 3000.

---

## Definition of Done

- [ ] `npm run dev -- -p 3001` starts the Next.js server without TypeScript errors
- [ ] The app loads at `http://localhost:3001`
- [ ] `.env.local` is created and listed in `.gitignore`
- [ ] The Redux store is configured in `store/store.ts` with `auth` and `courses` slices
- [ ] `<Providers>` wraps the root layout so the store is available to all pages
- [ ] `lib/api.ts` reads the access token from the Redux store (not `localStorage`)
- [ ] `types/index.ts` defines `User`, `Course`, `Lesson`, `Enrollment`, and `ProgressRecord`
- [ ] No `zustand`, `@tanstack/react-query`, or other state-management libraries are installed

All boxes ticked? Then continue to Chapter 5 — Authentication. If not, that is where today's work is.
