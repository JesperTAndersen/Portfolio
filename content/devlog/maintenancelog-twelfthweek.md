---
title: "Maintenance Log - Twelfth Week: User Profiles, Employee Admin Actions, and Form Patterns"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Frontend", "React", "Employees", "Auth", "Forms"]
series: ["Maintenance Log"]
date: 2026-05-16
draft: false
---

# Devlog Week 12: User Profiles, Employee Admin Actions, and Form Patterns

This entry covers the employee-facing parts of the UI: profile views, edit flows, admin actions, and the form patterns used to keep components maintainable.

## What Changed

- `UserProfile` supports both "me" and viewing other employees via dynamic route params
- Profile page extracted into focused subcomponents:
  - `EditProfileForm`, `AdminActions`, `PasswordChangeForm`
- Form handling uses a mix of controlled inputs and a `FormData` approach (when appropriate)
- `useEffect` race-condition prevention via an `ignore` flag to avoid setting state after unmount
- Admin-only employee actions (deactivate/reactivate) gated by role and disallowed on own profile

---

## User Profile: “Me” vs “Other Employee”

The profile route supports multiple entry points:

- own profile: loaded via `getMe()`
- other employee profile: loaded via `getEmployeeById(id)`

The page derives `isOwnProfile` from the route param and the authenticated user, and then chooses the correct API call.

## Race Condition Prevention in Effects

Because profile navigation can happen quickly, the page guards state updates with an `ignore` flag inside `useEffect`:

- set `ignore = true` in the cleanup function
- only call `setUser` / `setError` / `setLoading` if not ignored

This prevents React warnings and subtle UI bugs when an async request resolves after the component has already unmounted.

## Component Extraction + SRP

As the profile page grew, it was split into subcomponents with single responsibilities:

- `EditProfileForm`: edit UI and update call; uses callbacks to update the parent view
- `AdminActions`: admin-only deactivate/reactivate actions (not shown for own profile)
- `PasswordChangeForm`: only shown for own profile

The parent stays focused on data loading and permissions, while the subcomponents focus on UI + one action.

## Form Patterns: Controlled vs FormData

Not all inputs need to be controlled. For the edit form, using `FormData` is a simple option that reduces state boilerplate when the form is not reused for complex live validation.

This is a practical tradeoff: keep controlled inputs for places that benefit from instant validation/formatting, and use `FormData` when the form can be treated as a submission payload.

---

## Up Next

- Implement `EmployeeList` using the same fetch + filter + navigate pattern as assets
- Reuse the shared `Select` and card pattern for consistency
