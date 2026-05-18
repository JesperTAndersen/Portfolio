---
title: "Maintenance Log - Eleventh Week: Assets, Logs, Filtering, and Creating Maintenance Entries"
tags: ["Devlog", "Project", "3Sem", "MaintenanceLog", "Frontend", "React", "Assets", "Logs", "UX"]
series: ["Maintenance Log"]
date: 2026-05-12
draft: false
---

# Devlog Week 11: Assets, Logs, Filtering, and Creating Maintenance Entries

This entry focuses on the asset/log workflow: fetching real data from the API, filtering lists, displaying asset details, and adding log creation behind role checks.

## What Changed

- `AssetList` now fetches from the real backend API
- Active/inactive filtering implemented via a reusable `Select` component
- `AssetDetail` fetches asset + logs in parallel using `Promise.all`
- Status and task type filters wired to server-side filtering
- "Create log" is role-gated (TECHNICIAN+) and also hidden for inactive assets
- `CreateLog` includes client-side validation and prevents double-submit
- CSS Modules adopted for scoped styling

---

## Asset List: Fetch + Filter Pattern

`AssetList` implements a reusable pattern used elsewhere in the frontend:

- local state for `loading`, `error`, and the data array
- `useEffect` drives fetches
- a `Select` updates filter state
- the filter state is part of the effect dependency array, so changing a filter re-fetches cleanly

Active/inactive filtering is handled by converting a select value into `true` / `false` / `null`.

## Asset Detail: Parallel Requests + Filtered Logs

For the detail view, the page needs two independent data sources:

- asset summary (`getAssetById(id)`)
- logs for the asset (`getLogsForAsset(id, status, taskType)`)

Both are fetched at the same time with `Promise.all`, so the page doesn't wait for one request to finish before starting the other.

Filtering is intentionally server-side: changing `status` or `taskType` requests a filtered list from the backend. This keeps the frontend logic simple and ensures the filter behavior matches the API.

The empty state is also filter-aware:

- with no filters: "No logs for this asset yet"
- with filters: "No logs match those filters"

## Create Log: Guarded + Validated

Log creation is protected on two levels:

- route protection: only TECHNICIAN+ can access the route
- UI gating: the "Create log" button is only rendered if the user is TECHNICIAN+ *and* the asset is active

The form uses `datetime-local` with a max attribute to prevent future timestamps, validates before calling the API, and uses a `submitting` state to prevent double submission.

---

## Up Next

- Finish the employee flows (list + profile pages)
- Add admin actions (deactivate/reactivate) with role-gated UI
- Add password change for the logged-in user
