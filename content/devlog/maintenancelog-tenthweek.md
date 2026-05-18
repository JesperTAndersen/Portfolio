---
title: "Maintenance Log - Tenth Week: Frontend Auth, Routing, and API Client Structure"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Frontend", "React", "Vite", "ReactRouter", "Auth"]
series: ["Maintenance Log"]
date: 2026-05-08
draft: false
---

# Devlog Week 10: Frontend Auth, Routing, and API Client Structure

This entry covers the "plumbing" for the React frontend: the route layout, guarded routes by role, auth rehydration, and an API layer split by domain.

## What Changed

- Vite + React + React Router setup with a domain-organized structure (`pages/`, `components/`, `utils/`, `context/`, `styles/`)
- Declarative nested routing with a layout route and `<Outlet />`
- Auth context for user state, token rehydration, and role checks
- `ProtectedRoute` component for nestable route protection
- API layer split by domain with a shared `apiRequest()` helper

---

## Routing: Nested Layout + Guards

The router is set up as nested routes:

- `/` redirects to `/login`
- Public route:
  - `/login`
- Authenticated routes are grouped behind a `ProtectedRoute` wrapper
- A layout route (the `Header` component) provides the app shell while children render into `<Outlet />`
- Dynamic segments are used for resource detail routes (`:id`)

The key point is that `ProtectedRoute` is *nestable*, so stricter roles can wrap a smaller route subset.

## Auth Layer: Context + Rehydration

The auth state lives in `AuthContext`:

- `authUser`: current user object (or `null`)
- `authReady`: prevents early route decisions before token rehydration finishes
- `login(user)` / `logout()`
- `hasRole(requiredRole)`: role hierarchy check (ADMIN > MANAGER > TECHNICIAN > AUTHENTICATED)

On mount, the app calls `getMe()` when a token is present, which makes refresh persistent (no forced relogin on reload).

## Protected Routes

`ProtectedRoute` enforces three layers, in order:

1. Wait for `authReady`
2. If unauthenticated → redirect to `/login`
3. If `requiredRole` is present and not satisfied → show an "Insufficient permissions" state

Because the guard is implemented as an `<Outlet />` wrapper, the same logic can be reused at multiple route depths.

## API Layer: Domain Split + Shared Request Helper

API calls are split by domain in `utils/`:

- `assetApi.js` (assets)
- `logApi.js` (logs)
- `employeeApi.js` (employees)

All of them call through a shared `apiRequest(url, { method, body })` helper that:

- builds consistent fetch calls
- attaches the JWT as a Bearer token
- centralizes error handling

Token operations are intentionally isolated in `apiClient.js` (set/read/remove), so the rest of the app doesn't manipulate storage directly.

---

## Up Next

- Build the first real feature pages (assets list + asset detail)
- Add list/detail filtering driven by URL query parameters or controlled state
- Gate create-log functionality behind TECHNICIAN+ and align UI with roles
