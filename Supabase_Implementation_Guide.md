# Supabase Implementation Guide (Beginner Friendly)

This guide explains how Supabase is integrated into the ArchionLabs project. We use Supabase for authentication (logging in/out) and for securely calling backend APIs. This document breaks down the implementation step-by-step so that it is easy to understand.

## Table of Contents
1. [Overview](#1-overview)
2. [Required Libraries](#2-required-libraries)
3. [Environment Variables](#3-environment-variables)
4. [Supabase Clients explained](#4-supabase-clients-explained)
5. [Route Protection (Middleware)](#5-route-protection-middleware)
6. [Authentication Flow (Login & Callback)](#6-authentication-flow-login--callback)
7. [Authenticated Backend API Calls](#7-authenticated-backend-api-calls)

---

## 1. Overview
In this project, Supabase is primarily used to handle user authentication. It provides secure ways for our Next.js frontend to talk to Supabase via various "Clients". We also use the Supabase session token to verify users when they make requests to our separate backend services (Build, Sim, Community, Viewer).

## 2. Required Libraries
We use the official Supabase packages for Next.js:
- `@supabase/supabase-js`: The core client library that makes API requests to Supabase.
- `@supabase/ssr`: Essential for Server-Side Rendering (SSR) in Next.js (App Router). It helps manage cookies and sessions securely across server and client boundaries.

## 3. Environment Variables
For Supabase to work, we need a few environment variables defined in our `.env` file:
- `NEXT_PUBLIC_SUPABASE_URL`: The URL of your Supabase project.
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`: The safe-to-expose anonymous key for public queries and auth.
- `SUPABASE_SERVICE_ROLE_KEY`: A secret key that bypasses all security rules (Row Level Security). **Never expose this to the browser.**

## 4. Supabase Clients explained
In Next.js App Router, code runs either on the browser (Client) or on the server (Server Components). Therefore, we need different types of Supabase clients depending on where the code is executing. These are stored in `src/lib/supabase/`.

### A. The Client-Side Supabase Client (`client.ts`)
Used inside React components that have `"use client"` at the top (like interactive UI, forms, buttons).
```typescript
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```
**Why?** It automatically looks for authentication cookies stored in the browser and handles user sessions.

### B. The Server-Side Supabase Client (`server.ts`)
Used inside React Server Components and Server Actions. It securely accesses cookies on the server using `next/headers`.
```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(url, key, {
    cookies: {
      getAll() { return cookieStore.getAll(); },
      setAll(cookiesToSet) { /* Updates cookies if the session refreshes */ }
    }
  });
}
```
**Why?** Server-side code cannot access the standard browser `document.cookie`, so we provide ways to get and set cookies using Next.js tools.

### C. The Admin Supabase Client (`admin.ts`)
Used **strictly on the server-side** when you need to perform actions that bypass Row Level Security (RLS). Returns a client using the `SUPABASE_SERVICE_ROLE_KEY`.

## 5. Route Protection (Middleware)
We want to ensure that only logged-in users can access the dashboard, and logged-in users shouldn't access the login page again.
This is handled in `src/middleware.ts`.

1. **Session Refreshing**: The middleware creates a server client and gets the user using `await supabase.auth.getUser()`. This automatically refreshes the user's secure token if it's expired.
2. **Dashboard Protection**: If the user visits `/dashboard` but has no valid session, they are redirected to `/login`.
3. **Login Redirection**: If a logged-in user visits `/login`, they are redirected to `/dashboard`.

## 6. Authentication Flow (Login & Callback)
### The Login Page (`src/app/login/page.tsx`)
This is a Client Component (`"use client"`). We import our browser client:
```typescript
const supabase = createClient();
```
We offer multiple login methods:
- **Email/Password sign up:** `supabase.auth.signUp({ email, password })`
- **Email/Password sign in:** `supabase.auth.signInWithPassword({ email, password })`
- **Google OAuth:** `supabase.auth.signInWithOAuth({ provider: "google" })`

### The Callback Route (`src/app/auth/callback/route.ts`)
When using OAuth (Google) or Email Confirmations, Supabase redirects the user to our website with a special `code` in the URL.
This server-side Route Handler intercepts that link, trades the `code` for an actual user session (cookie), and then redirects the user to the `/dashboard`:
```typescript
const supabase = await createClient(); // Server client
await supabase.auth.exchangeCodeForSession(code); // Trades code for session
return NextResponse.redirect(`${origin}/dashboard`); // Go to dashboard
```

## 7. Authenticated Backend API Calls
Our Next.js app needs to communicate with our python backend services securely. This is implemented in `src/lib/api.ts`.
We map endpoints into an `authFetch` function:
1. It fetches the current user session securely using the server side Client (`createClient()`).
2. Extracts the `access_token` JWT.
3. Automatically attaches it to every request going to our APIs using the `Authorization: Bearer <token>` header.
```typescript
if (accessToken) {
  headers["Authorization"] = `Bearer ${accessToken}`;
}
```
Because of this handy helper, developers only need to write:
`const { authFetch } = await createAuthenticatedFetch();`
`await authFetch("build", "/api/v1/generate/projects");`
And authentication is magically handled!
